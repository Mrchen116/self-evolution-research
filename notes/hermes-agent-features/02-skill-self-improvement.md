# Hermes Agent 深度研究 02：Skill 在使用中自我改进（Skills Self-Improve During Use）

**研究日期**: 2026-04-16  
**代码仓库**: `hermes-agent`

---

## 核心结论

Hermes 在代码层面**没有**一个显式的"skill revision engine"去自动比对执行轨迹和 skill 内容并改写它。Skill 的"自我改进"实际上与 skill 生成共用同一套机制：**基于计数器的后台 review loop**。每约 10 个未涉及 `skill_manage` 的 tool iteration 后，一个 fork 出来的 review agent 会审视当前对话历史，由它自主决定是否调用 `skill_manage(action="patch")` 或 `action="edit"` 来更新某个 skill。

也就是说，改进不是"执行完一个 skill 后自动对比修正"，而是**周期性由后台 agent 做全局审视**，判断对话中是否有新经验值得更新到已有 skill 中。

---

## 1. 触发条件

**先明确两个概念**：
- **User turn**：用户发一条消息，到 agent 给出最终回复，算**一轮完整对话**。
- **Tool iteration**：agent 内部一次"思考 → 调用工具 → 拿到结果"的循环；**一个 user turn 里可以有多个 tool iteration**。

用业务语言说：

> **当 agent 连续执行了大约 10 次"普通工作"（比如读写文件、运行命令、搜索网络），却一直没有调用 `skill_manage` 去创建或修改 skill 时，系统就会在当前用户 turn 结束后，自动派一个"后台 review agent"去检查：最近这段对话里有没有学到什么新东西，值得更新到已有的 skill 里？**

这跟 skill 的初次生成用的是同一套机制。它不是"某个 skill 执行失败了 → 自动修正"，而是更像一个**定期复盘机制**。

### 计数器逻辑

| 变量 | 含义 | 代码位置 |
|------|------|----------|
| `self._iters_since_skill` | 距离上次 skill 操作经过了多少个 tool iteration | `run_agent.py:8519-8523` |
| `self._skill_nudge_interval` | 触发阈值，默认 `10` | `run_agent.py:1336-1341` |

**流程**：
1. 每个 agent loop iteration 结束后，如果本轮**没有**调用 `skill_manage`，则 `_iters_since_skill += 1`
2. 如果本轮调用了 `skill_manage`（无论 create/edit/patch），计数器立即归零（`run_agent.py:7320-7323`）
3. 当 `_iters_since_skill >= _skill_nudge_interval` 且 `skill_manage` 在可用工具中时，设置 `_should_review_skills = True`，计数器归零（`run_agent.py:11198-11204`）
4. Turn 结束后，如果产生了 `final_response` 且未被中断，则启动后台 review（`run_agent.py:11218`）

**关键特点**：
- 不需要某个 skill 被加载过
- 不需要任务失败
- 完全由 iteration 间隔驱动

---

## 2. 完整数据流

当触发条件满足后，主 agent 执行以下步骤：

### Step 1: 快照对话历史
```python
messages_snapshot = list(messages)
```
**代码位置**: `run_agent.py:11221`

### Step 2: 启动后台 Review Thread
调用 `_spawn_background_review`（`run_agent.py:2313-2412`），在 daemon thread 中运行。

### Step 3: Fork Review Agent
在后台线程中：
- 实例化一个全新的 `AIAgent`，使用相同的 model/provider（`run_agent.py:2343-2349`）
- 将其 nudge interval 设为 0，防止递归触发 review（`run_agent.py:2353-2354`）
- 把 `messages_snapshot` 作为 conversation history，review prompt 作为新的 user message（`run_agent.py:2356-2358`）
- review agent 自主运行，最多 8 个 iteration

### Step 4: Review Agent 决定操作
Review agent 可以调用 `skill_manage` 进行：
- `create`：创建新 skill
- `edit`：完全重写某个已有 skill 的 `SKILL.md`
- `patch`：对已有 skill 做局部 find-and-replace

也可以输出 "Nothing to save." 并停止。

### Step 5: 结果回显
主线程扫描 review agent 的 `_session_messages`，如果检测到成功操作的关键词（"created"、"updated"、"added"、"removed"、"replaced"），向用户打印一行简洁总结。

**代码位置**: `run_agent.py:2363-2391`

### 有没有结构化比对？

**没有。** Review agent 只看到完整的对话历史和一段自然语言 review prompt。系统不会自动把"skill 原始内容"与"执行轨迹"做 diff 或结构化对比。更新什么、怎么更新，完全由 review agent 的 LLM 自主决定。

---

## 3. 修订 Skill 的 Prompt

与 skill 生成共用同一套 review prompt。定义在 `run_agent.py` 的类常量中：

### `_SKILL_REVIEW_PROMPT`（`run_agent.py:2289-2297`）

**英文原版**：
```text
Review the conversation above and consider saving or updating a skill if appropriate.

Focus on: was a non-trivial approach used to complete a task that required trial
and error, or changing course due to experiential findings along the way, or did
the user expect or desire a different method or outcome?

If a relevant skill already exists, update it with what you learned.
Otherwise, create a new skill if the approach is reusable.
If nothing is worth saving, just say 'Nothing to save.' and stop.
```

**中文翻译**：
```text
回顾上面的对话，如果合适的话，考虑保存或更新一个 skill。

重点关注：是否用了非平凡的方法来完成任务？是否经历了试错，或在过程中
根据经验发现改变了方向？或者用户是否期望或希望用不同的方法或得到不同的结果？

如果相关 skill 已经存在，用你学到的东西更新它。
否则，如果这个做法可以复用，就创建一个新 skill。
如果没什么值得保存的，只需说 'Nothing to save.' 然后停止。
```

### `_COMBINED_REVIEW_PROMPT`（`run_agent.py:2299-2311`）

当 memory nudge 和 skill nudge 同时触发时使用，要求 review agent 同时审视 memory 和 skill。

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

**注意**：这里没有 JSON schema、没有 old-vs-new 对比模板、没有 DSPy signature。修订决策完全是自然语言驱动的。

---

## 4. Old Skill vs New Skill：如何合并

系统层面**没有自动 merge 或三路 diff**。更新完全由 agent 通过 `skill_manage` 工具完成，支持两种模式：

### 4.1 `patch` — 局部模糊替换
**代码位置**: `tools/skill_manager_tool.py:397-485`

- 接收 `old_string` 和 `new_string`
- 使用 `tools.fuzzy_match.fuzzy_find_and_replace` 处理空白/缩进差异
- 如果 patch 会导致 `SKILL.md` 的 frontmatter 被破坏，则拒绝
- 成功替换后运行安全扫描，若被 block 则回滚到原始内容

### 4.2 `edit` — 完全重写
**代码位置**: `tools/skill_manager_tool.py:361-394`

- 接收完整的新的 `SKILL.md` 内容
- agent 通常需要先调用 `skill_view()` 读取现有内容，再输出完整新版本
- 写之前保存 `original_content`，写后若安全扫描失败则原子恢复

### 4.3 安全扫描与回滚

```python
# edit 的回滚
if not scan_result.get("allowed", True):
    _atomic_write_text(skill_path, original_content)
    return {"success": False, ...}

# patch 的回滚  
if not scan_result.get("allowed", True):
    restored = original_content.replace(new_string, old_string)
    _atomic_write_text(skill_path, restored)
```

**代码位置**: `tools/skill_manager_tool.py:383-388`（edit 回滚）、`477-480`（patch 回滚）

---

## 5. 持久化机制

### 5.1 存储位置
- 所有 agent 创建/更新的 skill 都在 `~/.hermes/skills/` 下
- `SKILLS_DIR = HERMES_HOME / "skills"`（`tools/skill_manager_tool.py:80-81`）

### 5.2 原子写入
`_atomic_write_text`（`tools/skill_manager_tool.py:268-298`）使用 temp file + `os.replace()`，保证不会出现半写文件。

### 5.3 版本控制
**没有内置版本控制**（没有 Git commit、没有编号版本）。唯一的回滚机制就是上述的安全扫描失败后的即时恢复。

### 5.4 缓存失效
每次 `skill_manage` 调用成功后，skill system prompt cache 会被清空（`tools/skill_manager_tool.py:667-672`），确保后续 turn 立即看到更新后的 skill。

---

## 6. 关键文件索引

| 内容 | 文件 | 行号 |
|------|------|------|
| Skill nudge 配置与默认阈值 | `run_agent.py` | 1336-1341 |
| `_iters_since_skill` 计数增加 | `run_agent.py` | 8519-8523 |
| `_iters_since_skill` 调用 skill_manage 后归零 | `run_agent.py` | 7320-7323 |
| Skill review 触发检查 | `run_agent.py` | 11198-11204 |
| `_spawn_background_review` 调用 | `run_agent.py` | 11218-11226 |
| `_SKILL_REVIEW_PROMPT` 定义 | `run_agent.py` | 2289-2297 |
| `_COMBINED_REVIEW_PROMPT` 定义 | `run_agent.py` | 2299-2311 |
| 后台 review 线程完整实现 | `run_agent.py` | 2313-2412 |
| `skill_manage` 工具 schema | `tools/skill_manager_tool.py` | 681-768 |
| `skill_manage` 动作分发器 | `tools/skill_manager_tool.py` | 616-674 |
| `_edit_skill`（完全重写） | `tools/skill_manager_tool.py` | 361-394 |
| `_patch_skill`（模糊替换） | `tools/skill_manager_tool.py` | 397-485 |
| `_atomic_write_text` | `tools/skill_manager_tool.py` | 268-298 |
| 安全扫描 + edit 回滚 | `tools/skill_manager_tool.py` | 383-388 |
| 安全扫描 + patch 回滚 | `tools/skill_manager_tool.py` | 477-480 |
| Fuzzy match 引擎 | `tools/fuzzy_match.py` | (imported at 444) |
| Skill patch 测试 | `tests/tools/test_skill_improvements.py` | 1-175 |

---

## 设计洞察

1. **"Self-improvement" 在这里是 prompt-driven background review**：代码层面没有 diff engine、没有 execution trace analyzer，只有一个周期性启动的 review agent，由它自主决定要不要改 skill。这是一种**极轻量**的设计，工程成本低，但高度依赖模型能力。

2. **Patch 比 Edit 更适合增量改进**：`patch` 模式只做局部替换，对长 skill 更友好，也减少了重写时破坏已有结构的风险。Fuzzy match 的加入让模型不需要精确匹配缩进。

3. **没有版本控制是明显短板**：一旦 review agent 的 patch 把 skill 改坏了（且安全扫描没拦住），就没有历史版本可回退。对于"越用越强"的系统来说，缺少版本回溯是一个工程风险点。
