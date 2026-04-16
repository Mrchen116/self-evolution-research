# Hermes Agent 深度研究 05：用户模型（Honcho Dialectic User Modeling）

**研究日期**: 2026-04-16  
**代码仓库**: `hermes-agent`

---

## 核心结论（用人话讲）

Honcho 是 Hermes 接的一个**第三方用户画像服务**。它干的事可以类比为：

> **在对话进行的同时，后台有一个"秘书"一直在观察用户，写用户画像笔记，并且每隔几轮还会用 LLM 做一轮"深度反思"：这个人现在到底需要什么？我的理解有没有漏洞？**这个更新后的画像，会在下一轮对话时悄悄塞给主 agent。

"Dialectic"（辩证）就是这套反思机制的名字——它不是简单地把对话历史压缩成摘要，而是让 LLM **多轮质询自己**，逐步提炼出对用户的深层理解。

整个系统的工程目标很明确：**让 agent 越来越懂这个用户，但不要阻塞主对话的速度，也不要烧太多 token。**

---

## 先明确两个概念

### Turn（轮）

在 Hermes 的语境里，**一个 turn = 用户发送一条消息 → agent 处理并给出最终回复的完整过程**。即使 agent 在这一轮里调了 5 个工具、发了 3 次 API 请求，仍然只算 **1 个 turn**。Honcho 的所有频率控制（`contextCadence`、`dialecticCadence`）都是按这个粒度计算的（`run_agent.py:8233`）。

### Dialectic 是什么？

Dialectic 是 **Honcho 后端提供的一项 LLM 推理服务**（通过 `peer.chat()` 调用）。它不是本地 Hermes 模型在跑，而是把 prompt 发到 Honcho 的服务器，由 Honcho 用自己的 LLM 基于该用户的全部历史数据做推理，返回一段对用户当前状态/需求的洞察文本。这段文本会被注入到 user message 中，帮主 agent "更懂用户"。

---

## 1. 整体流程：Honcho 在每一轮对话中的生命周期

下面这张图讲的是 **Honcho 这个插件在 Hermes 主循环里是怎么被调用的**。代码位置：`run_agent.py:8233 - 11212`。

```
用户发送新消息
    ↓
[Turn 开始]
    ├─ _user_turn_count += 1
    ├─ on_turn_start(turn_number, user_message)  →  Honcho 记录当前是第几轮
    └─ prefetch_all(user_message)                →  Honcho 返回【本轮要注入的上下文文本】
                                                    （基础层 + 辩证层，详见第 3 节）
    ↓
[构造 API 请求]
    └─ 把 prefetch 返回的文本拼接到 user_message 后面（`run_agent.py:8539-8550`）
    ↓
[主 LLM 生成回复]
    └─ agent 看到的 prompt 里已经包含了 Honcho 提供的用户画像
    ↓
[Turn 结束]
    ├─ sync_all(user_msg, assistant_msg)         →  把这轮对话追加到 Honcho 后端
    └─ queue_prefetch_all(user_msg)              →  启动后台线程，为【下一轮】准备新的上下文
                                                    （详见第 5 节）
    ↓
用户看到回复
```

**关键理解**：
- `prefetch_all()` 不是"更新 Honcho 后端的数据"，而是**"从 Honcho 后端读取/生成数据，注入当前 turn"**。
- `sync_all()` 是 `MemoryManager` 的 orchestrator 方法（由 `run_agent.py` 调用），它会分发给每个 provider 的 `sync_turn()`。对 Honcho 来说，真正的"写"发生在 `HonchoMemoryProvider.sync_turn()` 里——把刚聊完的这轮对话同步给 Honcho 后端。
- `queue_prefetch_all()` 是"预计算"——在后台悄悄为下一轮生成洞察，这样下一轮 `prefetch_all()` 就能立刻返回缓存结果，不卡主对话。

---

## 2. 数据收集：完全隐式观察

Honcho **不会**打断用户问"你打几分？"或者"请确认你的偏好"。它收集的数据只有两类：

### 2.1 对话原文

每轮结束后，`sync_turn()` 会把这轮的 **用户原始消息 + assistant 最终回复**（不含中间 tool call）同步到 Honcho 后端（`plugins/memory/honcho/__init__.py:863-892`）。超过 25K 字符的消息会被分块，并加上 `[continued]` 前缀。

### 2.2 Agent 主动记下来的 facts

当 agent 使用内置 `memory` 工具添加/替换用户画像条目时（比如往 `USER.md` 写"用户偏好深色模式"），`on_memory_write()` 会把这些事实同步为 Honcho 的 `conclusions`（`__init__.py:894-910`）。

**注意**：本地 `MEMORY.md` / `USER.md` 是主存储，Honcho 只是额外接收一份镜像副本。

### 2.3 数据同步之后发生了什么？——Honcho 服务端是黑盒

`hermes-agent` 仓库里只有 Honcho 的**客户端适配器**代码，没有 Honcho 服务端本身的源码。因此，数据被同步到后端之后**具体怎么处理**（比如 representation 是实时更新还是异步批处理、summary 是每次写入后自动重算还是惰性生成），从 Hermes 代码里**看不到**。

但我们可以从客户端调用的 API 和返回结果，推断出 Honcho 后端至少维护着以下数据对象：

| 数据对象 | 推断来源 | 说明 |
|---------|---------|------|
| **Session messages** | `add_messages()` 写入，`context().messages` 可读 | 原始对话历史的持久化存储 |
| **Session summary** | `context(summary=True).summary` | 当前会话内容的摘要 |
| **Peer representation** | `context().peer_representation` / `peer.representation()` | 关于用户的自由文本画像 |
| **Peer card** | `context().peer_card` / `peer.get_card()` | 结构化的事实列表 |
| **Conclusions** | `conclusions_scope.create()` 写入，后续会影响 card | 持久化的推断事实 |

进一步可以推断出三种 API 的语义分工：

- **`add_messages()`** —— 只负责"存"原始消息。调用延迟低，不会立刻触发 LLM 重算。
- **`context()`** —— 负责"读已加工好的档案"。返回的 summary / representation / card 是后端已经维护好的状态，不是每次调用时才现算的。
- **`peer.chat()`** —— 这是唯一一个客户端明确知道"会调用 LLM"的 API。它让 Honcho 后端基于用户全部历史数据做一次深度推理，返回一段洞察文本。

**总结**：Hermes 把"原料"（对话原文、结论）喂给 Honcho 后端，后端自己负责消化、更新画像、生成摘要。Hermes 只需要在合适的时机通过 `context()` 和 `chat()` 把加工好的结果取回来，注入 prompt。

---

## 3. 用户画像的两层"召回"机制（为当前 turn 准备注入文本）

Honcho 不把用户画像当一张静态卡片，而是分成两层，分别按不同频率**为当前 turn 生成/读取要注入 prompt 的文本**。

### 3.1 基础事实层（Context Layer）

**这一层在做什么？**

从 Honcho 后端**读取**已经维护好的用户档案和会话摘要，格式化成一段文本，注入当前 turn 的 prompt。

**读取的内容包括**：
- **Session Summary**：当前这次会话在聊什么的摘要
- **User Representation**：Honcho 后端维护的一段自由文本摘要，比如"Eri 是一位后端工程师，最近在学 Rust..."
- **User Peer Card**：结构化的事实列表，比如 `["Name: Eri", "Prefers dark mode", "Dislikes verbose explanations"]`
- **AI Self-Representation / AI Identity Card**：agent 自身的身份和角色定位（Honcho 也管理"AI peer"）

**调用方式**：`get_prefetch_context()`（`session.py:625-677`）内部调用 `peer.context()` 或 `session.context()`，这是 Honcho 的标准 API，成本低、速度快。

**更新频率（注入频率）**：由 `contextCadence` 控制，默认每 **1 个 turn** 就读取/刷新一次。由于结果被缓存在 `_base_context_cache` 中，实际只有在 `queue_prefetch_all()` 启动后台刷新或首次调用时才会发请求。

### 3.2 辩证推理层（Dialectic Layer）

**这一层在做什么？**

调用 Honcho 后端的 **Dialectic LLM 服务**（`peer.chat()`），让它基于用户的全部历史数据**现推理一段洞察文本**，回答"基于最近聊的内容，这个用户当下最可能需要什么？"

**打个比方**：
- 基础层像是"用户的简历和档案"（从数据库直接读）
- 辩证层像是"秘书在会议进行到一半时，突然意识到的：'哦，他反复提到 deadline，说明他现在最焦虑的是时间，不是技术细节'"（用 LLM 推理生成）

**调用方式**：`_run_dialectic_depth()`（`__init__.py:772-812`）→ `dialectic_query()`（`session.py:493-556`）→ `peer.chat()`。推理在 Honcho 服务器上完成，产出的文本缓存在 `_prefetch_result` 中。

**实际调用 LLM 生成新洞察的频率**：由 `dialecticCadence` 控制，默认每 **3 个 turn** 才触发一次真正的 `_run_dialectic_depth()` 调用（发生在 turn 结束时的 `queue_prefetch()` 里）。因为这一步要调远程 LLM，成本高。

**注入到 prompt 的频率**：不是每轮都有。`_prefetch_result` 被读取一次后会立即清空（`__init__.py:600-602`）。这意味着：
- 生成一次新洞察 → **只在紧接着的下一轮注入一次**
- 之后几轮 prompt 里**没有辩证层**，只有基础事实层
- 直到下一个 `dialecticCadence` 到期，才会再次生成新的洞察

举个例子（`dialecticCadence=3`）：
- Turn 1：有辩证层 A（首回合同步计算）
- Turn 2：有辩证层 A（Turn 1 结束时预计算的）
- Turn 3：没有辩证层（`_prefetch_result` 已被清空，且 cadence 未到期）
- Turn 4：没有辩证层
- Turn 5：有辩证层 B（Turn 4 结束时满足 cadence=3，新计算的）

### 3.3 三个配置旋钮

| 配置项 | 默认值 | 含义 |
|--------|--------|------|
| `writeFrequency` | `async` | 对话内容刷写到 Honcho 后端的时机：`async`（后台线程不阻塞）、`turn`（每轮同步）、`session`（会话结束批量）、或整数 N（每 N 轮） |
| `contextCadence` | `1` | 两次基础层读取之间的最小 turn 数 |
| `dialecticCadence` | `3` | 两次辩证层 LLM 推理之间的最小 turn 数 |

这三个旋钮是正交的，让用户可以在"实时性"和"API 成本"之间做权衡。

---

## 4. 辩证推理层的内部实现展开（3.2 节的续篇）

上一节 3.2 讲了**辩证推理层是什么、产出的文本用来干什么、多久跑一次**。这一节专门展开它的**内部实现细节**：Dialectic 到底在代码的哪些地方被触发，以及它内部是怎么一步步执行的。

### 4.1 调用位置：只会在"准备往 prompt 里塞画像"的时候被触发

Dialectic **从不会在用户和 agent 聊天的中途被调用**。它只会在两个与"准备/预计算用户画像"相关的时刻出现，对应第 1 节时序图中的两个步骤：

1. **Turn N 开始时——准备注入当前 turn 的画像（`prefetch()`）**：
   - 非首回合：直接读取"上一轮结束后后台已经算好的"辩证结果。
   - 首回合特殊：因为这是第一次对话，没有"上一轮预计算"的结果，所以会临时启动一个带 8 秒 timeout 的后台线程跑一遍 Dialectic（`__init__.py:568-596`），尽量不让第一回合的 prompt 是空白。

2. **Turn N 结束时——在后台为下一轮预计算画像（`queue_prefetch()`）**：
   - 当前 turn 聊完后，系统检查 `dialecticCadence`（默认每 3 turns）。
   - 如果允许，启动一个 daemon thread 调用 `_run_dialectic_depth()`，悄悄为 **Turn N+1** 生成新的洞察文本（`__init__.py:631-677`）。

所以你可以把 Dialectic 理解为：**一个在 turn 边界处运行的"后台批处理任务"，专门负责产出高价值的洞察文本，供下一回合注入 prompt 使用。**

注意区分两个动作：
- `prefetch()` 在**每个 turn 开始时都会执行**，但通常只是**读缓存**（把上一轮预计算好的洞察注入 prompt）。
- `queue_prefetch()` 里的 `_run_dialectic_depth()` 才是**真正调用 Honcho 远端 LLM** 的地方，受 `dialecticCadence` 控制（默认每 3 turns 跑一次）。

### 4.2 Dialectic 的执行链

**`query` 从哪来？**

这个 `query` 就是**当前 turn 的用户原始消息**（`original_user_message`）。具体来说：`run_agent.py:11212` 在 turn 结束时调用 `queue_prefetch_all(original_user_message)`，把用户刚说的话作为 query 传进来。Honcho 后端会基于这句话 + 该用户的全部历史档案来做推理。

```
queue_prefetch() / prefetch()
    ↓
_run_dialectic_depth(query)   ← query = 当前 turn 的用户原始消息
    ├─ 循环 1~3 次（由 dialecticDepth 控制，默认可能是 1 或 2）
    │   ├─ _build_dialectic_prompt(pass_idx)  →  构造自我质询 prompt
    │   ├─ _resolve_pass_level(pass_idx)      →  决定本轮推理深度（minimal/low/medium...）
    │   └─ dialectic_query(session_key, prompt, reasoning_level)
    │           ↓
    │           peer.chat(query, reasoning_level=level)  →  发到 Honcho 后端 LLM
    │           ↓
    │           返回一段洞察文本
    │
    └─ _signal_sufficient(result)  →  如果结果已经够好，提前跳出循环
    ↓
缓存到 _prefetch_result，供下一轮 prefetch() 使用
```

### 4.3 多轮自我质询的 3 个 Pass

这些辩证推理不是在本地的 Hermes 主模型上跑的，而是通过 `peer.chat()` 发到 **Honcho 自己的后端 LLM** 执行。

#### Pass 0：收集印象

根据是否有已有档案，选择不同 prompt：

**冷启动**（`_base_context_cache` 为空，`__init__.py:721-726`）：
> "这个人是谁？他们的偏好、目标和工作风格是什么？专注于能帮助 AI 助手立即变得有用的事实。"

**热会话**（已有基础档案，`__init__.py:727-731`）：
> "鉴于本次会话到目前为止讨论的内容，关于这个用户的哪些上下文与当前对话最相关？优先关注活跃上下文，而非传记性事实。"

#### Pass 1：自我审计（Self-audit）

**Prompt**（`__init__.py:732-740`）：
> "你的理解中还存在哪些空白……综合你实际了解到的关于用户当前状态和即时需求的信息，并以最近会话中的证据为基础。"

这一步是让 LLM **质疑自己**："我是不是漏了什么？""我有没有在瞎猜？"

#### Pass 2：矛盾调和（Reconciliation）

检查 Pass 0 和 Pass 1 的输出有没有冲突，输出一个最终统一的结论（`__init__.py:741-750`）。

#### 提前退出机制

`_signal_sufficient()`（`__init__.py:752-770`）判断某一轮输出是否已经"足够好"：
- 若 >100 字符且包含结构化特征（标题、列表等），或
- 若 >300 字符（无结构）

满足则跳过更深的 pass，省 token。

### 4.4 推理级别控制

每个 pass 可以使用不同的 reasoning level（`__init__.py:684-712`）。例如 `dialecticDepth=2` 时，Pass 0 用 `minimal`，Pass 1 用 `base`。可通过 `dialecticDepthLevels` 自定义。

---

## 5. 怎么保证不阻塞主对话？异步 Prefetch 架构

用户不会希望 agent 每回一条消息就卡住几秒"正在分析你的人格"。所以 Hermes 设计了一个**预取 + 缓存**机制。

### 每轮对话的详细时序

**Turn N 开始时**（`run_agent.py:8445-8463`）：
1. `on_turn_start(turn_number, message)` — 更新内部计数器
2. `prefetch_all()` — 返回 **Turn N-1 的后台线程已经准备好** 的用户画像（基础层 `_base_context_cache` + 辩证层 `_prefetch_result`）
3. 主 LLM **立刻开始生成回复**

**Turn N 进行中**：主 agent 正常和用户交互，完全不受 Honcho 后台影响。

**Turn N 结束时**（`run_agent.py:11209-11212`）：
1. `sync_all(user_msg, assistant_msg)` — `MemoryManager` 的 orchestrator 方法，它会调用 Honcho provider 的 `sync_turn()`，把这轮对话追加到 Honcho 后端
2. `queue_prefetch_all()` — 启动后台 daemon thread，为 **Turn N+1** 准备新的基础层和辩证层

**首回合特殊处理**：
- 第一次对话没有"上一轮预取"，所以第一回合的辩证层会**同步执行**，但加了一个 8 秒 bounded timeout（`__init__.py:568-596`），防止首回合空白。

### 缓存分层

| 缓存 | 存什么 | 刷新频率 |
|------|--------|----------|
| `_base_context_cache` | 基础事实层（summary + representation + card） | `contextCadence`（默认 1 turn） |
| `_prefetch_result` | 辩证推理层的 LLM 输出 | `dialecticCadence`（默认 3 turns） |

核心思想就是：**"本轮用上一轮已经算好的结果"**，把延迟降到了几乎为零。

---

## 6. 用户模型最终长什么样？怎么注入 prompt？

### 6.1 格式：拼成一段结构化文本

最终给到 agent 的 user message 长这样——**用户的原话在最前面**，后面紧跟着 Honcho 注入的用户画像背景：

```text
【用户原话】
帮我看看这个 Docker 构建怎么这么慢，明天要演示了。

【Honcho 注入的画像背景】
<memory-context>
[System note: The following is recalled memory context, NOT new user input. Treat as informational background data.]

## Session Summary
用户在排查一个 Docker 部署问题...

## User Representation
Eri 是一位后端工程师，最近在学 Rust...

## User Peer Card
- Name: Eri
- Prefers dark mode

## AI Self-Representation
你是一个帮助用户写代码和解决技术问题的 AI 助手...

## AI Identity Card
- Primary goal: 提高用户开发效率

## Dialectic Insight
用户近期多次表现出 deadline 焦虑，当前最可能需要快速定位构建瓶颈的简短排查步骤，而不是深入底层原理讲解。
</memory-context>
```

**拆解一下各部分来源**：
- `## Session Summary` 到 `## AI Identity Card` → 来自 **3.1 基础事实层**（直接从 Honcho 后端读取的现成档案）
- `## Dialectic Insight` 那段 → 来自 **3.2 辩证推理层**（Honcho 后端 LLM 基于用户原话 + 历史档案现推理出的洞察）
- 用户的原话（"帮我看看这个 Docker 构建怎么这么慢..."）→ **不会出现在 `<memory-context>` 里面**，它本来就单独放在 user message 的最前面。

所以 agent 既能看到用户当下的具体请求，也能看到"这个用户是谁、现在最可能需要什么"的背景信息。

### 6.2 注入位置：user message 而非 system prompt

**这是非常重要的工程优化**：

- System prompt 是几乎不变的。如果每次都把动态用户画像塞进去，**前缀缓存（prefix caching）就会失效**，白白浪费 token
- Hermes 的解法是：system prompt 只放一个静态头（`# Honcho Memory\nActive (hybrid mode)...`），这个头**每 session 只计算一次**
- **动态的用户画像内容被注入到 user message 中**（`run_agent.py:8539-8550`）

这样 system prompt 的前缀缓存保住了，token 成本大幅下降。

### 6.3 三种工作模式（Recall Mode）

Honcho 支持三种模式，控制用户画像的可见性（`__init__.py:480-507`）：

| 模式 | 自动注入画像 | agent 能否主动查 Honcho |
|------|------------|------------------------|
| `hybrid`（默认） | 是 | 可以，相关工具可用 |
| `context` | 是 | 不能，工具被隐藏 |
| `tools` | 否 | 可以，必须主动调用工具 |

默认 `hybrid` 就是"自动注入 + 也允许主动调用"。

---

## 7. 关键文件索引

| 内容 | 文件 | 行号 |
|------|------|------|
| Honcho 集成的完整生命周期 | `plugins/memory/honcho/__init__.py` | 1-1054 |
| Provider 初始化（recall_mode / cadence / depth 配置） | `plugins/memory/honcho/__init__.py` | 189-226 |
| 初始化、session 创建、预加热、文件迁移 | `plugins/memory/honcho/__init__.py` | 267-405 |
| 组装上下文文本 `_format_first_turn_context` | `plugins/memory/honcho/__init__.py` | 432-459 |
| 预取逻辑（两层缓存 + 首回合同步超时） | `plugins/memory/honcho/__init__.py` | 509-615 |
| 启动后台辩证更新线程 `queue_prefetch` | `plugins/memory/honcho/__init__.py` | 631-677 |
| 构建辩证 prompt（3-pass）`_build_dialectic_prompt` | `plugins/memory/honcho/__init__.py` | 713-750 |
| 提前退出 heuristic `_signal_sufficient` | `plugins/memory/honcho/__init__.py` | 752-770 |
| 执行多轮辩证推理 `_run_dialectic_depth` | `plugins/memory/honcho/__init__.py` | 772-812 |
| 同步对话 turn `sync_turn` | `plugins/memory/honcho/__init__.py` | 863-892 |
| 镜像内置 memory 到 conclusions `on_memory_write` | `plugins/memory/honcho/__init__.py` | 894-910 |
| Honcho Session 管理器 | `plugins/memory/honcho/session.py` | 1-1256 |
| 调用后端 LLM 跑辩证 `dialectic_query` | `plugins/memory/honcho/session.py` | 493-556 |
| 获取基础事实层 `get_prefetch_context` | `plugins/memory/honcho/session.py` | 625-677 |
| 写入 peer 持久事实 `create_conclusion` | `plugins/memory/honcho/session.py` | 1071-1118 |
| MemoryManager orchestration | `agent/memory_manager.py` | 1-374 |
| 包 `<memory-context>` 注入 prompt `build_memory_context_block` | `agent/memory_manager.py` | 65-80 |
| 主循环中 prefetch / sync 调用 | `run_agent.py` | 8445-8463, 11209-11212 |
| 动态上下文注入 user message | `run_agent.py` | 8539-8550 |

---

## 设计洞察

1. **用户画像不是静态档案，是活的理解**：Honcho 把"用户模型"拆成了"事实层 + 辩证层"，前者便宜常更新，后者贵但更深。这比单次 summary 更能捕捉用户状态的动态变化。

2. **辩证（Dialectic）本质是 LLM 自我质询**：不是简单压缩对话，而是设计了一个"收集 → 审计 → 调和"的结构化反思流程。这产出的是"洞察"而不只是"摘要"。

3. **Prefetch 架构解决了实时性的核心矛盾**：用户不可能等 agent 每次回复前都跑一次 LLM 分析。Honcho 的方案是"本轮用上一轮已经算好的结果"，把延迟降到了几乎为零。

4. **注入 user message 是精打细算的缓存优化**：对 Claude 这类支持 system prompt 前缀缓存的模型来说，这是实打实的 token 成本优化。

5. **Honcho 是服务端 heavy 的设计**：辩证推理跑在 Honcho 自己的后端 LLM 上，不是本地模型。这意味着用户建模的质量同时取决于 Hermes 的架构设计和 Honcho 服务端的能力。
