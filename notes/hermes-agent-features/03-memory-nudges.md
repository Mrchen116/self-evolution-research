# Hermes Agent 深度研究 03：Agent-Curated Memory + Periodic Nudges

**研究日期**: 2026-04-16  
**代码仓库**: `hermes-agent`

---

## 核心结论

Hermes 的 memory 写入不是被动的"把所有对话存下来"，而是**由 agent 自主策展（curate）**的。系统通过两种 nudge 机制提醒 agent：
1. **System prompt 中的内联 guidance**（始终存在）
2. **基于 turn 计数器的后台 review nudge**（默认每 10 个 turn 一次）

决定是否写入 memory、写什么、以什么粒度写，**完全由 LLM（主 agent 或后台 review agent）自主决策**。没有硬性的 heuristic 自动写 memory。

Memory 的物理存储是简单的 **Markdown 文本文件**（`MEMORY.md` 和 `USER.md`），用 `§` 符号分隔条目，写入时带文件锁。

---

## 1. Nudge 到底是什么？（精确 Prompt 文本）

Hermes 有两类 nudge：Memory review nudge 和 Skill review nudge。它们共用同一套后台 review 框架。这里聚焦 memory。

### 1.1 后台 Review Prompt

当 memory nudge 触发时，fork 的 review agent 会收到以下三种 prompt 之一：

#### Memory-only review prompt
**代码位置**: `run_agent.py:2278-2287`

**英文原版**：
```text
Review the conversation above and consider saving to memory if appropriate.

Focus on:
1. Has the user revealed things about themselves — their persona, desires,
   preferences, or personal details worth remembering?
2. Has the user expressed expectations about how you should behave, their work
   style, or ways they want you to operate?

If something stands out, save it using the memory tool.
If nothing is worth saving, just say 'Nothing to save.' and stop.
```

**中文翻译**：
```text
回顾上面的对话，如果合适的话，考虑保存到 memory 中。

重点关注：
1. 用户是否透露了关于他们自己的信息——他们的个性、愿望、偏好，
   或值得记住的个人细节？
2. 用户是否表达了对你应该如何表现、他们的工作风格、或他们希望你
   如何运作的期望？

如果有突出的东西，用 memory 工具保存下来。
如果没什么值得保存的，只需说 'Nothing to save.' 然后停止。
```

#### Combined review prompt（与 skill nudge 同时触发）
**代码位置**: `run_agent.py:2299-2311`

**英文原版**：
```text
Review the conversation above and consider two things:

**Memory**: Has the user revealed things about themselves — their persona,
desires, preferences, or personal details? Has the user expressed expectations
about how you should behave, their work style, or ways they want you to operate?
If so, save using the memory tool.

**Skills**: Was a non-trivial approach used to complete a task that required trial
and error, or changing course due to experiential findings along the way, or did
the user expect or desire a different method or outcome? If a relevant skill
already exists, update it. Otherwise, create a new one if the approach is reusable.

Only act if there's something genuinely worth saving.
If nothing stands out, just say 'Nothing to save.' and stop.
```

**中文翻译**：
```text
回顾上面的对话，考虑两件事：

**Memory**：用户是否透露了关于他们自己的信息——他们的个性、愿望、
偏好或个人细节？用户是否表达了对你应该如何表现、他们的工作风格、
或他们希望你如何运作的期望？如果是，用 memory 工具保存下来。

**Skills**：是否用了非平凡的方法来完成任务？是否经历了试错，或在过程中
根据经验发现改变了方向？或者用户是否期望或希望用不同的方法或得到不同的结果？
如果相关 skill 已经存在，更新它。否则，如果这个做法可以复用，就创建一个新的。

只有确实有值得保存的东西时才行动。
如果没什么突出的，只需说 'Nothing to save.' 然后停止。
```

### 1.2 System Prompt 中的常驻 Guidance

**代码位置**: `agent/prompt_builder.py:144-156`

**英文原版**：
```python
MEMORY_GUIDANCE = (
    "Use the memory tool to remember important facts about the user and your own "
    "evolving best practices. Be concise. If a memory becomes outdated or wrong, "
    "replace or remove it."
)
```

**中文翻译**：
```text
使用 memory 工具来记住关于用户的重要事实，以及你自己不断演化的最佳实践。
要简洁。如果某个 memory 已经过时或错误，替换或删除它。
```

只要 `memory` 工具可用，这段 guidance 就会一直出现在 system prompt 中，提醒主 agent **主动**使用 memory 工具。

---

## 2. Nudge 何时、如何注入 Agent 上下文？

### 2.1 Memory Nudge 的触发逻辑

**先明确两个概念**：
- **User turn**：用户发一条消息，到 agent 给出最终回复，算**一轮完整对话**。
- **Tool iteration**：agent 内部一次"思考 → 调用工具 → 拿到结果"的循环；**一个 user turn 里可以有多个 tool iteration**。

用人话解释触发时机：

> **当 agent 和用户已经对话了大约 10 个 turn，但 agent 一直没有主动调用 `memory` 工具去记录点什么时，系统就会认为"agent 可能已经聊了不少但忘了做笔记"，于是在当前 turn 结束后，自动在后台启动一个 review agent，让它审视这轮对话里有没有值得记住的用户信息或经验。**

这跟 skill review 类似，都是**定期复盘机制**。区别在于 skill review 数的是 tool call 次数，而 memory review 数的是**用户 turn 数**（即来回对话的轮数）。

**重置条件**：只要 agent 在某一轮主动调用了 `memory` 工具（add/replace/remove），memory 的计数器就会归零。

| 配置项 | 默认值 | 代码位置 |
|--------|--------|----------|
| `memory.nudge_interval` | `10` | `run_agent.py:1207-1220` |

**计数器逻辑**：
- `self._turns_since_memory` 每完成一次 `run_conversation`（即每轮用户对话）后 +1（`run_agent.py:8245`）
- 触发条件（`run_agent.py:8238-8248`）：
  ```python
  if (self._memory_nudge_interval > 0
          and "memory" in self.valid_tool_names
          and self._memory_store):
      self._turns_since_memory += 1
      if self._turns_since_memory >= self._memory_nudge_interval:
          _should_review_memory = True
          self._turns_since_memory = 0
  ```

### 2.2 注入机制：后台 Thread，不污染主上下文

**关键点**：nudge **不是**直接插入到主 agent 的当前对话上下文中的。

流程如下：
1. 主 agent 正常完成对用户消息的响应
2. 在 turn 结束时，如果 `_should_review_memory == True`，主 agent 启动 `_spawn_background_review`（`run_agent.py:11216-11226`）
3. 该函数在一个 **daemon thread** 中 fork 一个新的 `AIAgent`
4. Review agent 获得当前对话的 `messages_snapshot` 和 review prompt
5. Review agent 自主决定是否调用 `memory` 工具

这意味着：
- 用户不会感觉到"被 nudge 打断"
- 主 agent 的上下文不会被 review 过程污染
- Memory 写入是**异步后台**发生的

### 2.3 Skill Nudge 的触发（作为对比）

Skill nudge 基于 **tool iteration** 而非 user turn：
- `self._iters_since_skill` 在每个 tool-calling iteration 后 +1（`run_agent.py:8519-8523`）
- 阈值默认也是 `10`（`run_agent.py:1335-1341`）
- 触发后同样走后台 review

### 2.4 计数器重置规则

当 agent **实际执行了**对应工具时，计数器归零：
- 调用 `memory` 工具后 → `_turns_since_memory = 0`（`run_agent.py:7319-7323`）
- 调用 `skill_manage` 工具后 → `_iters_since_skill = 0`（`run_agent.py:7577-7581`）

**注意**：如果工具调用被 plugin block 了，计数器**不会**重置（`tests/run_agent/test_run_agent.py:1544-1557`）。这防止了 agent 因为被 block 而反复尝试绕过 nudge。

---

## 3. 什么决定 memory 是否被写入？

**答案：Agent 输出决定。**

没有任何自动 heuristic（比如"用户说了 3 次就喜欢"就自动写）。后台 review agent 是一个完整的 `AIAgent` fork，拥有相同的模型和工具。它被给予 review prompt 后，可以：
- 调用 `memory` 工具（`add`/`replace`/`remove`）
- 或者输出 "Nothing to save." 并停止

主 agent 也可以在正常 turn 中**主动**调用 `memory` 工具（system prompt 鼓励这样做）。

---

## 4. Memory 格式

Built-in memory 使用**自定义纯文本格式**，存储在两个 Markdown 文件中：

### 4.1 文件位置
- `~/.hermes/memories/MEMORY.md` —— agent 自己的笔记
- `~/.hermes/memories/USER.md` —— 用户画像

### 4.2 条目分隔符
**`§`（section sign）**，即 `\n§\n`。

**代码位置**: `tools/memory_tool.py:57`

每个文件是由多个 entry 组成的列表，entry 之间用 `\n§\n` 分隔。

### 4.3 System Prompt 中的渲染示例

在 system prompt 里，memory 会被渲染成（`tools/memory_tool.py:391-407`）：

```text
══════════════════════════════════════════════
MEMORY (your personal notes) [45% — 990/2200 chars]
══════════════════════════════════════════════
User prefers concise answers.

§

Always use git status before making changes.
```

可以看到，system prompt 会**实时显示使用率百分比**（`pct% — current/limit chars`），让 agent 感知 memory 容量。

### 4.4 容量限制

| 文件 | 默认字符限制 | 配置项 |
|------|-------------|--------|
| MEMORY.md | 2200 | `memory_char_limit` |
| USER.md | 1375 | `user_char_limit` |

**代码位置**: `tools/memory_tool.py` 相关配置在 `cli-config.yaml.example:384-386` 等处。

---

## 5. Memory 存储在哪里？

### 5.1 Built-in Memory：纯文本文件
- 物理位置：`~/.hermes/memories/MEMORY.md` 和 `~/.hermes/memories/USER.md`
- 写入机制：原子写入（temp file + `os.replace`）+ 文件锁（`.lock` 文件）

**代码位置**: `tools/memory_tool.py:432-460`

文件锁实现：
```python
lock_path = memory_file_path.with_suffix(memory_file_path.suffix + ".lock")
```

### 5.2 Session History：SQLite
所有对话消息存储在 SQLite（`hermes_state.py`），并建有 FTS5 索引用于 cross-session search。但这是**对话历史**，不是 curated memory。

### 5.3 External Memory Providers
Hermes 支持通过插件接入外部 memory provider（Honcho、Hindsight、Mem0 等），由 `agent/memory_manager.py` 和 `agent/memory_provider.py` 抽象管理。但：
- 外部 provider 是**可选且附加的**
- 同一时间只能运行一个外部 provider + built-in 文件 memory
- 外部 provider 走的是自己的 API/SDK，不属于 built-in nudge 体系

---

## 6. 关键文件索引

| 内容 | 文件 | 行号 |
|------|------|------|
| Nudge interval 默认配置 | `run_agent.py` | 1207-1220, 1335-1341 |
| Memory review prompt 原文 | `run_agent.py` | 2278-2287 |
| Combined review prompt 原文 | `run_agent.py` | 2299-2311 |
| 后台 review 线程实现 | `run_agent.py` | 2313-2412 |
| Turn-based memory nudge 触发 | `run_agent.py` | 8238-8248 |
| Iteration-based skill nudge 触发 | `run_agent.py` | 8519-8523, 11198-11204 |
| Memory tool执行后计数器归零 | `run_agent.py` | 7319-7323 |
| Skill工具执行后计数器归零 | `run_agent.py` | 7577-7581 |
| 被 plugin block 时不重置计数器 | `tests/run_agent/test_run_agent.py` | 1544-1557 |
| Built-in memory 工具实现 | `tools/memory_tool.py` | 1-581 |
| Memory 文件路径解析 | `tools/memory_tool.py` | 53-55 |
| System prompt 注入 memory block | `run_agent.py` | 3287-3380 |
| Memory guidance system prompt | `agent/prompt_builder.py` | 144-156 |
| Memory provider 抽象 | `agent/memory_provider.py` | 1-232 |
| Memory manager  orchestration | `agent/memory_manager.py` | 1-374 |
| SQLite session store | `hermes_state.py` | 1-1239 |

---

## 设计洞察

1. **Memory Formation Policy 是 LLM-driven，不是 rule-driven**：Hermes 把"什么时候写 memory、写什么"这个问题完全交给了模型自己判断。好处是灵活，坏处是可能漏掉重要信息或写入无关信息，完全取决于模型质量。

2. **后台 review 是低摩擦的 memory 触发机制**：不需要打断主对话，每 10 个 turn 自动 review 一次，既给了 agent 写 memory 的机会，又不会干扰用户体验。这是一个非常聪明的工程权衡。

3. **简单文件格式降低了工程复杂度**：不用向量数据库、不用复杂的 schema，就是两个带分隔符的 markdown 文件。这让读写极其简单，但也限制了检索能力（跨 session recall 靠的是独立的 FTS5 session search，不是 memory 文件本身的检索）。

4. **实时显示 memory 使用率百分比**：在 system prompt 中显示 `[45% — 990/2200 chars]`，给 agent 提供了明确的容量信号，促使它主动压缩或清理过时 memory。这是很多人忽略但非常有效的设计细节。
