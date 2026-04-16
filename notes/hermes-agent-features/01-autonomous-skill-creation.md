# Hermes Agent 深度研究 01：自动生成 Skill（Autonomous Skill Creation）

**研究日期**: 2026-04-16  
**代码仓库**: `hermes-agent` (`/Users/czj/Repos/opensource-hub/self-evolution/hermes-agent`)

---

## 核心结论

Hermes 的 skill 自动生成不是由确定性事件触发的，而是**双层 nudge 驱动 + prompt 引导**的模型自主决策流程。系统不会说"完成了复杂任务 → 必须生成 skill"，而是持续提醒 agent，并在每约 10 个 tool iteration 后启动一个**后台 review agent**，由这个 fork 出来的 agent 自主判断当前对话是否值得沉淀为 skill。

---

## 1. 触发条件：双层机制

### 1.1 内联 System Prompt 引导（始终存在）

只要 agent 拥有 `skill_manage` 工具，system prompt 就会注入一段 `SKILLS_GUIDANCE`：

> "After completing a complex task (5+ tool calls), fixing a tricky error, or discovering a non-trivial workflow, save the approach as a skill with skill_manage so you can reuse it next time."

**代码位置**: `agent/prompt_builder.py:164-171`

这意味着主 agent 在**每一轮**都被提醒：做完复杂任务后应该考虑创建 skill。

### 1.2 后台 Review Nudge（基于计数器）

**先明确两个概念**：
- **User turn**：用户发一条消息，到 agent 给出最终回复，算**一轮完整对话**。
- **Tool iteration**：agent 内部一次"思考 → 调用工具 → 拿到结果"的循环；**一个 user turn 里可以有多个 tool iteration**。

这是真正"自动"触发的机制。用人话解释触发时机：

> **当 agent 连续调用了大约 10 次其他工具（比如文件读写、命令执行、搜索等），却一直没有调用 `skill_manage` 来管理 skill 时，系统就会认为"agent 可能已经积累了足够多经验但没有主动总结"，于是在当前用户 turn 结束后，自动在后台启动一个 review agent，让它审视这轮对话是否值得生成或更新一个 skill。**

换个角度理解：这个机制就像一个**后台秘书**，每隔一段时间（默认 10 个 tool call）检查一下 agent 最近的工作，提醒它"你是不是该写个 skill 了？"但秘书不会打断 agent 和用户的对话，而是等 agent 回完用户之后，自己在后台默默审视。

**重置条件**：只要 agent 在某一轮主动调用了 `skill_manage`（无论是创建、编辑还是 patch skill），计数器就会归零，重新开始计数。这意味着如果 agent 已经养成了主动总结的习惯，后台 nudge 就不会频繁触发。

| 配置项 | 默认值 | 代码位置 |
|--------|--------|----------|
| `skills.creation_nudge_interval` | `10` | `run_agent.py:1335-1341` |

**代码层面的计数器逻辑**：
- `self._iters_since_skill` 在每个 tool-calling iteration 后 +1，前提是本轮**没有**调用 `skill_manage`（`run_agent.py:8519-8523`）
- 当 `_iters_since_skill >= _skill_nudge_interval`（默认 10）时，设置 `_should_review_skills = True`，计数器归零（`run_agent.py:11198-11204`）
- 当用户 turn 结束且响应已完成后，如果 flag 为 true，则后台启动 `_spawn_background_review`（`run_agent.py:11218-11226`）

**关键设计**：不是主 agent 被强制打断去创建 skill，而是**响应用户之后**，在后台开一个 review agent 来做判断。

---

## 2. 完整数据流：后台 Review Agent 路径

这是 skill 自动生成的核心引擎。

### Step 1: 快照捕获
在 turn 结束时，主 agent 把当前完整的对话历史 `messages` 复制一份传入 review：

```python
messages_snapshot=list(messages)
```

**代码位置**: `run_agent.py:11220-11223`

### Step 2: 选择 Review Prompt
根据同时触发的 nudge 类型，选择三种 prompt 之一（`run_agent.py:2328-2334`）：
- `_SKILL_REVIEW_PROMPT` —— 只 review skill
- `_MEMORY_REVIEW_PROMPT` —— 只 review memory
- `_COMBINED_REVIEW_PROMPT` —— 两者一起 review

### Step 3: Fork Review Agent
创建一个新的 `AIAgent` 实例（`run_agent.py:2343-2359`），配置如下：
- 使用相同的 model / provider / platform
- `max_iterations=8`, `quiet_mode=True`
- **关键**：把 memory 和 skill 的 nudge interval 设为 0，防止 review agent 递归触发 review
- 把 `messages_snapshot` 作为 conversation history，`review prompt` 作为新的 user message

### Step 4: Review Agent 执行工具
这个 fork 出来的 agent 可以像正常 agent 一样调用 `skill_manage(action='create', ...)`。

### Step 5: 结果展示
主线程扫描 review agent 的 `_session_messages`，如果检测到 "created"、"updated"、"added" 等关键词，向用户打印简洁总结（如 "Skill 'x' created"）。

**代码位置**: `run_agent.py:2363-2397`

---

## 3. 生成 Skill 的 Prompt / 模板

Hermes **没有**一个专门的"生成 skill"的 LLM prompt 来输出 YAML frontmatter。相反，它依赖三层引导：

### 3.1 Review Prompt（触发决策）

`_SKILL_REVIEW_PROMPT` 的完整文本（`run_agent.py:2289-2297`）：

**英文原版**：
```text
Review the conversation above and consider saving or updating a skill if appropriate.
Focus on: was a non-trivial approach used to complete a task that required trial and error,
or changing course due to experiential findings along the way, or did the user expect or
desire a different method or outcome?
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

### 3.2 Tool Schema 描述（约束格式）

`skill_manage` 工具的 schema（`tools/skill_manager_tool.py:681-768`）中，`content` 字段的描述是：

> "Full SKILL.md content (YAML frontmatter + markdown body). Required for 'create' and 'edit'."

模型必须自主生成完整的 `SKILL.md` 文本，作为 `skill_manage` 调用的 `content` 参数。

### 3.3 In-Context 示例

System prompt 中的 `<available_skills>` 索引（`agent/prompt_builder.py:776-799`）会让模型看到现有 skill 的格式，从而模仿生成。

---

## 4. 输出格式与存储位置

### 4.1 文件格式

一个标准的生成 skill 是单个 `SKILL.md` 文件，结构为：
- YAML frontmatter（`---` 分隔）
- Markdown body

**必填 frontmatter 字段**（受校验）：
- `name`（最多 64 字符）
- `description`（最多 1024 字符）

**可选字段**：`version`, `author`, `license`, `platforms`, `metadata.hermes.tags`, `metadata.hermes.related_skills`, `required_environment_variables` 等。

### 4.2 目录结构

```
~/.hermes/skills/
├── <skill-name>/
│   ├── SKILL.md
│   ├── references/     (可选)
│   ├── templates/      (可选)
│   ├── scripts/        (可选)
│   └── assets/         (可选)
```

若指定了 `category`：
```
~/.hermes/skills/
├── <category>/
│   └── <skill-name>/
│       └── SKILL.md
```

**代码位置**: `tools/skill_manager_tool.py:79-81`（`SKILLS_DIR = HERMES_HOME / "skills"`）

---

## 5. 校验与安全机制

在 skill 被真正保存前，有多层校验：

### 5.1 命名校验
- `name`: regex `^[a-z0-9][a-z0-9._-]*$`, max 64 chars（`tools/skill_manager_tool.py:111-122`）
- `category`: 防止路径遍历，同样 regex，max 64 chars（`tools/skill_manager_tool.py:125-147`）

### 5.2 Frontmatter 校验
`_validate_frontmatter`（`tools/skill_manager_tool.py:150-186`）：
- 必须以 `---` 开头
- 必须有闭合 `---`
- 必须是合法 YAML
- 必须包含 `name` 和 `description`
- frontmatter 后必须有非空 body

### 5.3 内容大小限制
- `MAX_SKILL_CONTENT_CHARS = 100_000`（`tools/skill_manager_tool.py:97`）
- 若 agent 写入超此限制会被拒绝（手动放置的 skill 除外）（`tools/skill_manager_tool.py:189-201`）

### 5.4 重复检查
- `_find_skill` 扫描所有 skill 目录，若同名已存在则 creation 失败（`tools/skill_manager_tool.py:325-330`）

### 5.5 安全扫描与自动回滚
- `_security_scan_skill`（`tools/skill_manager_tool.py:56-74`）会在保存后运行 `tools.skills_guard.scan_skill`
- 若扫描返回 `allowed=False` 或 `allowed=None`（发现危险内容），会立即 `shutil.rmtree` 删除整个目录
- `edit` 和 `patch` 也有类似回滚机制

### 5.6 辅助文件路径限制
- 辅助文件只能放在 `references/`, `templates/`, `scripts/`, `assets/` 下（`tools/skill_manager_tool.py:229-254`）
- 使用 `tools.path_security.validate_within_dir` 防止目录遍历（`tools/skill_manager_tool.py:257-265`）

---

## 6. 关键文件索引

| 内容 | 文件 | 行号 |
|------|------|------|
| Skill nudge 配置与计数器 | `run_agent.py` | 1335-1341 |
| `_SKILL_REVIEW_PROMPT` | `run_agent.py` | 2289-2297 |
| `_COMBINED_REVIEW_PROMPT` | `run_agent.py` | 2299-2311 |
| 后台 review 线程实现 | `run_agent.py` | 2313-2412 |
| Skill nudge 计数增加 | `run_agent.py` | 8519-8523 |
| Skill nudge turn-end 触发 | `run_agent.py` | 11198-11204 |
| `SKILLS_GUIDANCE` system prompt | `agent/prompt_builder.py` | 164-171 |
| Skill index注入 system prompt | `agent/prompt_builder.py` | 776-799 |
| `skill_manage` tool schema | `tools/skill_manager_tool.py` | 681-768 |
| `_create_skill` 实现 | `tools/skill_manager_tool.py` | 304-358 |
| `_validate_frontmatter` | `tools/skill_manager_tool.py` | 150-186 |
| `_security_scan_skill` | `tools/skill_manager_tool.py` | 56-74 |
| 原子写入 `_atomic_write_text` | `tools/skill_manager_tool.py` | 268-298 |
| Skill CRUD 测试 | `tests/tools/test_skill_manager_tool.py` | 1-487 |
| Skill size限制测试 | `tests/tools/test_skill_size_limits.py` | 1-216 |

---

## 设计洞察

1. **不是事件触发，是 nudge + 自主决策**：Hermes 没有写死"5 个 tool call 后自动生成"，而是通过 system prompt 持续提醒 + 后台 review agent 让模型自己判断。这避免了大量低质量 skill 的生成，但也意味着生成质量完全取决于模型的判断能力。

2. **后台 fork 设计非常巧妙**：主 agent 的对话上下文不会被 review 过程污染，review 在 daemon thread 中异步进行，不影响用户交互延迟。

3. **没有专门的 skill generation LLM call**：模型直接把 `SKILL.md` 文本作为 `skill_manage` 工具的 `content` 参数生成，schema 描述就是格式约束。这简化了工程，但对模型的 in-context 格式遵循能力要求较高。
