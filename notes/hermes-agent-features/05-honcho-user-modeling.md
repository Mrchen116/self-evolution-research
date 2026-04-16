# Hermes Agent 深度研究 05：用户模型（Honcho Dialectic User Modeling）

**研究日期**: 2026-04-16  
**代码仓库**: `hermes-agent`

---

## 核心结论

Hermes 的 Honcho 集成是一个非常**工程化且注重延迟优化**的用户建模层。它不是简单的"把对话存起来下次塞 prompt"，而是构建了一个**异步、分层、辩证式**的用户理解系统：

1. **数据收集完全隐式**：没有显式反馈机制，只靠对话历史 + agent 主动写入的 memory 镜像
2. **更新频率可配置**：通过 `writeFrequency`、`contextCadence`、`dialecticCadence` 三个正交旋钮控制
3. **两层 prefetch 架构**：基础上下文（session summary + peer representation/card）和辩证推理层（dialectic supplement）分别异步刷新
4. **辩证式用户建模**：通过多轮 `peer.chat()` 自我审计、reconcile，生成对用户当前状态的理解
5. **Prompt 注入位置巧妙**：用户模型内容被注入到 **user message** 中而非 system prompt，以保留 system prompt 前缀缓存

---

## 1. 收集哪些数据来做用户建模？

Honcho 层的数据来源**完全隐式**，没有点赞/点踩、没有用户直接纠正：

### 1.1 对话历史
每轮对话的用户消息和 assistant 消息都会通过 `sync_turn()` 同步到 Honcho（`plugins/memory/honcho/__init__.py:863-892`）。超过 25K 字符的消息会被分块，并加上 `[continued]` 前缀。

### 1.2 Built-in Memory 写入镜像
当 agent 使用内置 `memory` 工具添加/替换用户画像条目时，`on_memory_write()` 会把这些事实同步为 Honcho 的 `conclusions`（`__init__.py:894-910`）。

### 1.3 Session 上下文
Session 初始化时，通过 `session.context()` 加载已有消息（`session.py:225`）。

### 1.4 一次性文件迁移
首次启用时，`MEMORY.md`、`USER.md`、`SOUL.md` 会被作为文件上传到 Honcho，用于初始化 user/AI peer 表示（`session.py:751-848`）。

**总结**：Honcho 看到的信号是观察性的——用户说了什么，以及 agent 推断并记录下来的事实。

---

## 2. 何时、如何更新用户模型？

有三个独立的配置旋钮控制更新频率：

| 配置项 | 默认值 | 含义 |
|--------|--------|------|
| `writeFrequency` | `async` | 消息刷写到 Honcho 的时机：`async`（后台线程）、`turn`（每轮同步）、`session`（会话结束批量）、或整数 N（每 N 轮） |
| `contextCadence` | `1` | 两次 `context()` API 调用之间的最小 turn 数（基础层刷新频率） |
| `dialecticCadence` | `3` | 两次 `peer.chat()` 调用之间的最小 turn 数（辩证推理层刷新频率） |

### 2.1 Turn 生命周期

**Turn 开始**（`run_agent.py:8445-8448`）：
```python
provider.on_turn_start(turn_number, message)
```
更新内部 turn 计数器，为 cadence 计算做准备。

**预取**（`run_agent.py:8458-8463`）：
```python
provider.prefetch_all()
```
在**主 LLM 调用前**执行，返回的是**上一轮**后台工作已经准备好的缓存上下文。

**同步 + 排队下一轮预取**（`run_agent.py:11209-11212`）：
```python
provider.sync_all(user_msg, assistant_msg)
provider.queue_prefetch_all()
```
Assistant 响应 finalized 后，先把当前 turn 同步到 Honcho，然后启动后台线程为**下一轮**准备新的 context 和 dialectic。

### 2.2 异步 Dialectic 更新

`queue_prefetch()`（`__init__.py:631-677`）会启动 daemon thread，调用 `_run_dialectic_depth()`，根据 `dialecticDepth` 配置可能执行 1-3 次 `peer.chat()`。结果缓存在 `_prefetch_result` 中，供下一轮 `prefetch_all()` 消费。

### 2.3 首回合特殊处理

由于没有上一轮预取，第一回合的 dialectic 会**同步执行**，并带一个 bounded timeout（默认 8 秒），防止首回合空白（`__init__.py:568-596`）。

---

## 3. 用户模型是什么格式？

用户模型**不是单个 JSON blob**，而是一个**分层文本表示**，由以下几部分组装而成：

### 3.1 Peer Card
通过 `peer.get_card()` / `peer.card()` 获取的精选事实列表，例如：
```python
["Name: Eri", "Prefers dark mode", "Works in Python"]
```

### 3.2 Peer Representation
由 Honcho 后端生成的自由文本摘要，通过 `peer.representation()` 或 `peer.context()` 获取。

### 3.3 Session Summary
当前会话范围的文本摘要，来自 `session.context(summary=True)`。

### 3.4 Conclusions
Agent 或内置 memory 工具写入的持久事实，通过 `conclusions_scope.create([...])` 存储。

### 3.5 组装成 Prompt Block

`_format_first_turn_context()`（`__init__.py:432-459`）把这些文本组装成如下结构：

```text
## Session Summary
...

## User Representation
...

## User Peer Card
...

## AI Self-Representation
...

## AI Identity Card
...
```

最终，`build_memory_context_block()`（`agent/memory_manager.py:65-80`）会把它包在 `<memory-context>` 围栏中：

```text
<memory-context>
...
</memory-context>
```

---

## 4. 用户模型如何检索并注入 Agent Prompt？

### 4.1 两层 Prefetch 架构

**基础上下文层** —— `get_prefetch_context()`（`session.py:625-677`）：
- 获取 session summary + user representation/card + AI representation/card
- 缓存在 `_base_context_cache` 中
- 按 `contextCadence` 刷新

**辩证补充层** —— `_run_dialectic_depth()`（`__init__.py:772-812`）：
- 获取 LLM 合成的关于用户当前状态的推理
- 缓存在 `_prefetch_result` 中
- 按 `dialecticCadence` 刷新

### 4.2 注入流程

**System Prompt 静态头**：
`_build_system_prompt()`（`run_agent.py:3287-3406`）调用 `self._memory_manager.build_system_prompt()`，返回一个静态头（如 `# Honcho Memory\nActive (hybrid mode)...`）。这个头**每 session 只计算一次**，利于前缀缓存。

**每轮动态上下文注入**：
`prefetch_all()` 返回的实时上下文在** API 调用时**被注入到 **user message** 中（`run_agent.py:8539-8550`），而不是塞进 system prompt。这是为了**保留 system prompt 前缀缓存**，减少重复 token 成本。

### 4.3 Recall Mode 控制注入行为

Honcho 提供三种 recall mode（`__init__.py:480-507`）：

| 模式 | 自动注入 | 工具可用性 |
|------|----------|-----------|
| `hybrid` | 是 | 工具可用 |
| `context` | 是 | 工具隐藏 |
| `tools` | 否 | 工具可用，需显式调用 |

默认是 `hybrid`，即自动注入 + 同时保留 Honcho 相关工具给 agent 主动调用。

---

## 5. 辩证 / 质疑机制（Dialectic）

这是 Honcho 集成中最有特色的部分，实现于 `_run_dialectic_depth()`（`__init__.py:772-812`）。

### 5.1 Prompt 选择：冷启动 vs 热会话

**冷启动**（`_base_context_cache` 为空，`__init__.py:721-726`）：

**英文原版**：
```text
Who is this person? What are their preferences, goals, and working style?
Focus on facts that would help an AI assistant be immediately useful.
```

**中文翻译**：
```text
这个人是谁？他们的偏好、目标和工作风格是什么？
专注于能帮助 AI 助手立即变得有用的事实。
```

**热会话**（已有 base context，`__init__.py:727-731`）：

**英文原版**：
```text
Given what's been discussed in this session so far, what context about this user
is most relevant to the current conversation? Prioritize active context over
biographical facts.
```

**中文翻译**：
```text
鉴于本次会话到目前为止讨论的内容，关于这个用户的哪些上下文与当前对话最相关？
优先关注活跃上下文，而非传记性事实。
```

### 5.2 多轮深度（1–3 passes）

`_build_dialectic_prompt()`（`__init__.py:713-750`）定义了三类 pass：

**Pass 0**：冷/暖启动 prompt（如上）

**Pass 1：Self-audit**（`__init__.py:732-740`）

**英文原版**：
```text
What gaps remain in your understanding...
Synthesize what you actually know about the user's current state and immediate
needs, grounded in evidence from recent sessions.
```

**中文翻译**：
```text
你的理解中还存在哪些空白……
综合你实际了解到的关于用户当前状态和即时需求的信息，
并以最近会话中的证据为基础。
```

**Pass 2：Reconciliation**（`__init__.py:741-750`）
检查前几轮之间的矛盾，输出最终综合结论。

### 5.3 提前退出 heuristic

`_signal_sufficient()`（`__init__.py:752-770`）判断某一轮输出是否"足够好"：
- 若 >100 字符且包含结构化特征（标题、列表等），或
- 若 >300 字符（无结构）

满足则跳过更深的 pass。

### 5.4 推理级别控制

每个 pass 使用相对于基础配置递减的 reasoning level（`__init__.py:684-712`）：
- 例如 `dialecticDepth=2` 时，pass 0 用 `minimal`，pass 1 用 `base`
- 可通过 `dialecticDepthLevels` 自定义

### 5.5 执行方式

Dialectic query 通过 `peer.chat()` 发到 **Honcho 后端**执行（`session.py:493-556`），而不是通过主 LLM provider。这意味着 Honcho 服务端有自己的 LLM 来运行这些辩证推理。

---

## 6. 关键文件索引

| 内容 | 文件 | 行号 |
|------|------|------|
| `HonchoMemoryProvider` 完整生命周期 | `plugins/memory/honcho/__init__.py` | 1-1054 |
| Provider `__init__`（recall_mode, cadence, depth） | `plugins/memory/honcho/__init__.py` | 189-226 |
| `initialize()` — session init, memory migration, pre-warm | `plugins/memory/honcho/__init__.py` | 267-405 |
| `_format_first_turn_context()` — 组装上下文文本 | `plugins/memory/honcho/__init__.py` | 432-459 |
| `system_prompt_block()` — 静态头注入 system prompt | `plugins/memory/honcho/__init__.py` | 461-507 |
| `prefetch()` — 两层上下文注入 + 首回合同步超时 | `plugins/memory/honcho/__init__.py` | 509-615 |
| `queue_prefetch()` — 后台线程刷新 | `plugins/memory/honcho/__init__.py` | 631-677 |
| `_build_dialectic_prompt()` — 冷/暖/self-audit/reconcile | `plugins/memory/honcho/__init__.py` | 713-750 |
| `_signal_sufficient()` — 提前退出 heuristic | `plugins/memory/honcho/__init__.py` | 752-770 |
| `_run_dialectic_depth()` — 多轮辩证执行 | `plugins/memory/honcho/__init__.py` | 772-812 |
| `sync_turn()` — 持久化对话 turn | `plugins/memory/honcho/__init__.py` | 863-892 |
| `on_memory_write()` — 镜像内置 memory 到 conclusions | `plugins/memory/honcho/__init__.py` | 894-910 |
| `HonchoSessionManager` | `plugins/memory/honcho/session.py` | 1-1256 |
| `dialectic_query()` — 调用 `peer.chat()` | `plugins/memory/honcho/session.py` | 493-556 |
| `get_prefetch_context()` — 获取 summary + representation + card | `plugins/memory/honcho/session.py` | 625-677 |
| `create_conclusion()` — 写入 peer 持久事实 | `plugins/memory/honcho/session.py` | 1071-1118 |
| `HonchoClientConfig` | `plugins/memory/honcho/client.py` | 1-677 |
| `MemoryManager` — orchestration | `agent/memory_manager.py` | 1-374 |
| `build_memory_context_block()` — 包 `<memory-context>` | `agent/memory_manager.py` | 65-80 |
| `_build_system_prompt()` — 组装 system prompt 层 | `run_agent.py` | 3287-3406 |
| `on_turn_start()` | `run_agent.py` | 8445-8448 |
| `prefetch_all()` | `run_agent.py` | 8458-8463 |
| `sync_all() + queue_prefetch_all()` | `run_agent.py` | 11209-11212 |
| Dialectic depth 测试 | `tests/honcho_plugin/test_session.py` | 1-1146 |
| Async writer / prefetch cache 测试 | `tests/honcho_plugin/test_async_memory.py` | 1-470 |

---

## 设计洞察

1. **隐式信号 + 异步架构 = 零摩擦**：Honcho 不打扰用户索要反馈，所有学习都在后台异步完成。`writeFrequency=async` 时，网络 I/O 完全不会阻塞主 agent 的响应。

2. **Dialectic 是多轮自我审计，不是单次 summary**：Pass 0 收集、Pass 1 质疑自己还有什么没理解、Pass 2 reconcile 矛盾。这种结构化的"质询"比单次 LLM summary 更能产出有深度的用户理解。

3. **注入 user message 而非 system prompt 是性能优化**：为了保留 system prompt 的前缀缓存（对 Claude 等模型尤为重要），Hermes 把动态 Honcho 上下文放到 user message 中。这是一个非常"生产级"的延迟/成本优化决策。

4. **三层缓存避免重复 API 调用**：`_base_context_cache`、`_prefetch_result`、以及 `MemoryManager` 的 own cache 构成了多层缓存体系。Cadence 配置让用户可以在"实时性"和"API 成本"之间做权衡。

5. **Honcho 是服务端 heavy 的设计**：不同于内置 memory 的纯本地文件方案，Honcho 把 dialectic reasoning 放在自己的后端执行。这意味着用户建模的质量部分依赖于 Honcho 服务本身的 LLM 能力，而不仅是主 agent 的模型。
