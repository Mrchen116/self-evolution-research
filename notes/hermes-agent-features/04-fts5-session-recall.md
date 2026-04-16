# Hermes Agent 深度研究 04：跨 Session 检索回忆（FTS5 + LLM Summarization）

**研究日期**: 2026-04-16  
**代码仓库**: `hermes-agent`

---

## 核心结论

Hermes 的跨 session 回忆不是自动注入 prompt 的，而是实现为一个**显式工具 `session_search`**。当 agent 怀疑有历史上下文相关时，它会主动调用该工具。

底层架构非常工程化：
1. **存储层**：所有消息存入本地 SQLite，并通过 FTS5 虚拟表建立全文索引
2. **检索层**：模型给出搜索 query，系统进行 FTS5 `MATCH` 查询并按 `rank` 排序
3. **压缩层**：命中会话的完整对话被加载后，按 query 匹配位置截断到 ~100K 字符
4. **摘要层**：截断后的对话 transcript 并行送给一个便宜的 auxiliary LLM（默认 Gemini Flash）做摘要
5. **使用层**：摘要结果以 JSON tool response 形式返回给主模型，由主模型自行消化

---

## 1. 数据库 Schema：Sessions + Messages + FTS5

### 1.1 核心表结构

**文件**: `/Users/czj/Repos/opensource-hub/self-evolution/hermes-agent/hermes_state.py`（lines 36-91）

```sql
CREATE TABLE IF NOT EXISTS schema_version (
    version INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS sessions (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,
    user_id TEXT,
    model TEXT,
    model_config TEXT,
    system_prompt TEXT,
    parent_session_id TEXT,
    started_at REAL NOT NULL,
    ended_at REAL,
    end_reason TEXT,
    message_count INTEGER DEFAULT 0,
    tool_call_count INTEGER DEFAULT 0,
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    cache_read_tokens INTEGER DEFAULT 0,
    cache_write_tokens INTEGER DEFAULT 0,
    reasoning_tokens INTEGER DEFAULT 0,
    billing_provider TEXT,
    billing_base_url TEXT,
    billing_mode TEXT,
    estimated_cost_usd REAL,
    actual_cost_usd REAL,
    cost_status TEXT,
    cost_source TEXT,
    pricing_version TEXT,
    title TEXT,
    FOREIGN KEY (parent_session_id) REFERENCES sessions(id)
);

CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,
    content TEXT,
    tool_call_id TEXT,
    tool_calls TEXT,
    tool_name TEXT,
    timestamp REAL NOT NULL,
    token_count INTEGER,
    finish_reason TEXT,
    reasoning TEXT,
    reasoning_details TEXT,
    codex_reasoning_items TEXT
);
```

**设计要点**：
- `parent_session_id` 支持 session 血缘关系（用于压缩触发 split 和 delegation 子会话）
- 当前 schema version 是 `6`（line 34）
- 启用 WAL mode（`PRAGMA journal_mode=WAL`，line 157），支持并发读 + 单写

### 1.2 FTS5 虚拟表与触发器

**代码位置**: `hermes_state.py:93-112`

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=id
);

CREATE TRIGGER IF NOT EXISTS messages_fts_insert AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, content) VALUES (new.id, new.content);
END;

CREATE TRIGGER IF NOT EXISTS messages_fts_delete AFTER DELETE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content) VALUES('delete', old.id, old.content);
END;

CREATE TRIGGER IF NOT EXISTS messages_fts_update AFTER UPDATE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content) VALUES('delete', old.id, old.content);
    INSERT INTO messages_fts(rowid, content) VALUES (new.id, new.content);
END;
```

FTS5 表同步通过 SQLite trigger 自动维护，无需应用层手动更新索引。

---

## 2. 搜索查询是如何构造的？

### 2.1 入口：`search_messages()`

**代码位置**: `hermes_state.py:990-1091`

### 2.2 Query 清洗：`_sanitize_fts5_query()`

**代码位置**: `hermes_state.py:938-988`

这是非常重要的一步，防止模型生成的 query 包含 FTS5 特殊字符导致语法错误：

1. **保留合法引号短语**：提取成对的双引号内容
2. **剥离特殊字符**：去掉未配对的 `+`、`{}`、`()`、`"`、`^`
3. **清理 `*`**：合并连续的 `*`，去掉开头的 `*`
4. **去掉 dangling boolean**：删除开头/结尾的 `AND`/`OR`/`NOT`
5. **包裹特殊 term**：对未引号的连字符/点号 term（如 `chat-send`）加双引号包裹

### 2.3 动态 SQL 构建

WHERE 子句（`hermes_state.py:1018-1037`）：
- `messages_fts MATCH ?` —— 必须
- `source_filter` → `s.source IN (...)` —— 可选
- `exclude_sources` → `s.source NOT IN (...)` —— 可选
- `role_filter` → `m.role IN (...)` —— 可选

### 2.4 实际执行的 SQL

```sql
SELECT
    m.id,
    m.session_id,
    m.role,
    snippet(messages_fts, 0, '>>>', '<<<', '...', 40) AS snippet,
    m.content,
    m.timestamp,
    m.tool_name,
    s.source,
    s.model,
    s.started_at AS session_started
FROM messages_fts
JOIN messages m ON m.id = messages_fts.rowid
JOIN sessions s ON s.id = m.session_id
WHERE {where_sql}
ORDER BY rank
LIMIT ? OFFSET ?
```

**代码位置**: `hermes_state.py:1040-1058`

**注意**：搜索 query 就是模型调用 `session_search` 时传入的字符串，没有自动用当前任务或对话上下文做扩展。

---

## 3. 摘要化流程：从 FTS5 命中到 LLM Summary

### 3.1 Step A：FTS5 检索

`session_search()` 调用 `db.search_messages(query=..., limit=50, ...)`（`tools/session_search_tool.py:337-343`）。

### 3.2 Step B：Session 准备与去重

**代码位置**: `tools/session_search_tool.py:354-403`

处理逻辑：
1. **按根 session 分组**：子 session 通过 `parent_session_id` 链追溯到根 session
2. **去重**：同一个根 session 只保留一个结果
3. **排除当前 session 血统**：当前 session 及其所有 parent/child 都被排除，避免重复回忆已经在上下文里的内容
4. **Source 过滤**：默认排除 `source="tool"` 的隐藏 session（`_HIDDEN_SESSION_SOURCES`，line 242）

### 3.3 Step C：加载与格式化对话

对每个唯一 session：
- 加载完整对话：`db.get_messages_as_conversation(session_id)`（line 409）
- 格式化为可读 transcript：`_format_conversation()`（lines 56-87）
- 截断到 ~100K 字符：`_truncate_around_matches()`（lines 90-172）

#### `_truncate_around_matches()` 的截断策略

这是整个 recall 系统中最精巧的算法之一：

1. **尝试完整短语匹配**（case-insensitive）
2. 若短语未命中，寻找所有 query term 在 200 字符窗口内的共现位置
3. 退而求其次，找单个 term 的位置
4. 选择能覆盖最多匹配位置的窗口起点，窗口前后比例为 **25% / 75%**（偏向匹配点之后的内容）

这个策略确保摘要 LLM 看到的是**与 query 最相关的上下文片段**，而不是整个冗长对话。

### 3.4 Step D：LLM 摘要

**代码位置**: `tools/session_search_tool.py:175-236`

#### System Prompt

**英文原版**：
```python
system_prompt = (
    "You are reviewing a past conversation transcript to help recall what happened. "
    "Summarize the conversation with a focus on the search topic. Include:\n"
    "1. What the user asked about or wanted to accomplish\n"
    "2. What actions were taken and what the outcomes were\n"
    "3. Key decisions, solutions found, or conclusions reached\n"
    "4. Any specific commands, files, URLs, or technical details that were important\n"
    "5. Anything left unresolved or notable\n\n"
    "Be thorough but concise. Preserve specific details (commands, paths, error messages) "
    "that would be useful to recall. Write in past tense as a factual recap."
)
```

**中文翻译**：
```text
你正在回顾一段过去的对话记录以帮助回忆发生了什么。
请围绕搜索主题总结这段对话，需包含：
1. 用户问了什么或想完成什么
2. 采取了哪些行动以及结果如何
3. 关键决策、找到的解决方案或达成的结论
4. 任何重要的具体命令、文件、URL 或技术细节
5. 任何未解决或值得注意的事项

要详尽但简洁。保留对回忆有用的具体细节（命令、路径、错误信息）。
用过去时态写成事实性回顾。
```

#### User Prompt

**英文原版**：
```python
user_prompt = (
    f"Search topic: {query}\n"
    f"Session source: {source}\n"
    f"Session date: {started}\n\n"
    f"CONVERSATION TRANSCRIPT:\n{conversation_text}\n\n"
    f"Summarize this conversation with focus on: {query}"
)
```

**中文翻译**：
```text
搜索主题: {query}
会话来源: {source}
会话日期: {started}

对话记录:
{conversation_text}

请围绕以下主题总结这段对话: {query}
```

#### 模型调用

```python
response = await async_call_llm(
    task="session_search",
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_prompt}
    ],
    temperature=0.1,
    max_tokens=MAX_SUMMARY_TOKENS,  # 10,000
)
```

**关键点**：`task="session_search"` 会路由到一个便宜的 auxiliary model，通常是 **Gemini Flash**，通过 `agent/auxiliary_client.py` 配置（lines 2575+）。这保证了大量历史 session 的摘要不会烧掉主模型（如 Claude/GPT-4）的 token。

### 3.5 Step E：并行执行

所有 session 的摘要请求通过 `asyncio.gather()` 并行发送（`tools/session_search_tool.py:425-431`）。`_run_async()`（lines 440-441）负责在 CLI、gateway、worker thread 等不同上下文中安全地管理事件循环。超时默认为 60 秒。

---

## 4. 摘要结果如何注入当前 Agent Prompt？

**答案是：不直接注入 prompt，而是以 tool response 返回。**

`session_search` 是**显式工具**，不是自动 memory injection。模型必须主动调用它。

工具返回的 JSON（`tools/session_search_tool.py:478-484`）：

```json
{
    "success": true,
    "query": "...",
    "results": [
        {"session_id": "...", "source": "...", "summary": "..."},
        ...
    ],
    "count": 3,
    "sessions_searched": 5
}
```

主模型在 tool response 中看到这些摘要，然后在后续回复中自行引用。

### System Prompt 如何鼓励使用？

**代码位置**: `agent/prompt_builder.py:158-162`

**英文原版**：
```python
SESSION_SEARCH_GUIDANCE = (
    "When the user references something from a past conversation or you suspect "
    "relevant cross-session context exists, use session_search to recall it before "
    "asking them to repeat themselves."
)
```

**中文翻译**：
```text
当用户提到过去对话中的某件事，或者你怀疑存在相关的跨 session 上下文时，
使用 session_search 去回忆它，而不是让用户重复一遍。
```

当 `session_search` 在可用工具集中时，这段 guidance 会被追加到 system prompt（`run_agent.py:3320-3321`）。

---

## 5. 搜索结果的排序与过滤

### 5.1 排序
FTS5 结果按内置 `rank` 函数排序（`hermes_state.py:1056`：`ORDER BY rank`）。

### 5.2 过滤层

| 过滤类型 | 实现 | 代码位置 |
|----------|------|----------|
| Source 排除 | 默认排除 `source="tool"` | `session_search_tool.py:242` |
| Role 过滤 | 可选 `role_filter`（如只搜 `user,assistant`） | `hermes_state.py:1028-1037` |
| 当前 session 排除 | 排除当前 session 及其整个血统链 | `session_search_tool.py:381-397` |
| 根 session 去重 | 子 session 追溯到根，每根只留一个 | `session_search_tool.py:388-403` |
| 数量限制 | `limit` 被钳制在 `[1, 5]`，默认 3 | `session_search_tool.py:316-321` |

---

## 6. 关键文件索引

| 内容 | 文件 | 行号 |
|------|------|------|
| SQLite schema (sessions, messages) | `hermes_state.py` | 36-91 |
| FTS5 虚拟表 + 触发器 | `hermes_state.py` | 93-112 |
| `_sanitize_fts5_query()` | `hermes_state.py` | 938-988 |
| `search_messages()` | `hermes_state.py` | 990-1091 |
| `session_search()` 工具入口 | `tools/session_search_tool.py` | 297-488 |
| `_format_conversation()` | `tools/session_search_tool.py` | 56-87 |
| `_truncate_around_matches()` | `tools/session_search_tool.py` | 90-172 |
| `_summarize_session()` | `tools/session_search_tool.py` | 175-236 |
| `SESSION_SEARCH_GUIDANCE` | `agent/prompt_builder.py` | 158-162 |
| `SESSION_SEARCH_SCHEMA` | `tools/session_search_tool.py` | 500-544 |
| Agent loop 拦截 `session_search` | `run_agent.py` | 7209-7219, 7659-7673 |
| `async_call_llm(task="session_search")` | `agent/auxiliary_client.py` | 2575+ |
| FTS5 search 测试 | `tests/test_hermes_state.py` | 270-480 |
| `session_search` 工具测试 | `tests/tools/test_session_search.py` | 1-378 |
| Session storage 文档 | `website/docs/developer-guide/session-storage.md` | 1-300 |
| Session search 用户文档 | `website/docs/user-guide/sessions.md` | 292-318 |

---

## 设计洞察

1. **On-demand tool 而非自动注入**：Hermes 选择让 agent 显式调用 `session_search`，而不是像 Honcho 那样自动把 summarized context 注入 prompt。这避免了每次请求都检索历史带来的延迟和成本，但也要求 agent 足够"自觉"地去调用工具。

2. **FTS5 是非常务实的选择**：不用向量数据库，直接用 SQLite 内置 FTS5。对于本地 CLI agent 来说，这消除了额外依赖，查询速度足够快，且 `rank` 排序已经能提供可用的相关性。

3. **Truncation 算法是隐藏亮点**：`_truncate_around_matches()` 的多阶段匹配策略（短语 → 共现窗口 → 单 term + 25/75 偏置）确保 auxiliary LLM 拿到的是高密度相关信息，而不是被淹没在海量对话中。

4. **Auxiliary model 分工明确**：摘要任务路由到便宜的 Gemini Flash（`task="session_search"`），主模型只负责看摘要后的结果。这是一个非常成本意识强的设计，对于需要频繁检索历史的场景至关重要。

5. **当前 session 血统排除防止重复**：通过 `parent_session_id` 链排除当前 session 及其相关子 session，避免 agent 花 token 去回忆"它本来就已经知道"的内容。这个细节体现了对 token 效率的重视。
