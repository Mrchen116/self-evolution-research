# Hermes Agent 深度研究 04：跨 Session 检索回忆（FTS5 + LLM Summarization）

**研究日期**: 2026-04-16  
**代码仓库**: `hermes-agent`

---

## 核心结论

Hermes 的跨 session 回忆不是自动塞到 prompt 里的，而是实现为一个**显式工具 `session_search`**。当 agent 怀疑用户提到的东西和过去对话有关时，它要主动调用这个工具。

整个流程可以概括为四步：

```
agent 主动调用 session_search(query="部署问题")
           ↓
    [检索层] SQLite FTS5 全文搜索历史消息
           ↓
    [加工层] 加载命中会话 → 截断无关内容 → 送 auxiliary LLM 摘要
           ↓
    [返回层] 把摘要结果以 JSON 形式还给 agent
           ↓
agent 在回复中引用这些历史信息
```

关键设计：**原始对话不能直塞进 prompt（太长太杂），必须先经过「截断 + LLM 摘要」两层压缩。**

---

## 1. 入口：agent 必须显式调用工具

### 1.1 什么时候会触发？

不是自动触发，是**模型主动决策**。

System prompt 里有一段常驻 guidance（`agent/prompt_builder.py:158-162`）：

> "当用户提到过去对话中的某件事，或者你怀疑存在相关的跨 session 上下文时，使用 `session_search` 去回忆它，而不是让用户重复一遍。"

当 agent 看到用户说"还是之前那个问题"、"上次部署报错的方案"这类话时，它**应该**调用 `session_search("部署报错")`。

### 1.2 工具返回什么？

返回一个 JSON（`tools/session_search_tool.py:478-484`）：

```json
{
    "success": true,
    "query": "部署报错",
    "results": [
        {"session_id": "abc", "source": "cli", "summary": "用户上次在 Fly.io 部署时遇到..."},
        {"session_id": "def", "source": "cli", "summary": "三周前用户问过 Docker 部署..."}
    ],
    "count": 2,
    "sessions_searched": 3
}
```

agent 像读取任何 tool result 一样读取这些 summary，然后在给用户的回复中引用。

---

## 2. 检索层：FTS5 怎么找历史消息？

### 2.1 业务逻辑

所有用户和 agent 的对话都被悄悄写入本地 SQLite 数据库（`~/.hermes/state.db`）。同时，系统用 SQLite 内置的 **FTS5 全文搜索引擎**给消息内容建了索引。

当 agent 调用 `session_search` 时，它传入一个自然语言 query（比如 `"部署报错"`）。系统会：
1. 把这个 query 清洗成安全的 FTS5 搜索语法
2. 在 `messages_fts` 表里执行 `MATCH`
3. 按 SQLite 内置的 `rank` 排序，取最相关的前 N 条消息
4. 根据这些消息定位到它们属于哪些历史 session

### 2.2 存储层实现（简要）

**代码位置**: `hermes_state.py:36-112`

核心就两张表 + 一个 FTS5 虚拟表：

```sql
-- 会话元数据
CREATE TABLE sessions (id TEXT PRIMARY KEY, ...);

-- 每条消息
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT REFERENCES sessions(id),
    role TEXT,
    content TEXT,
    ...
);

-- FTS5 全文索引（虚拟表，通过 trigger 自动同步）
CREATE VIRTUAL TABLE messages_fts USING fts5(
    content, content=messages, content_rowid=id
);
```

FTS5 表的同步是通过 SQLite trigger 自动完成的（insert/update/delete 时自动维护），应用层不用操心。

### 2.3 Query 清洗

模型生成的 query 可能带特殊字符（如 `+`、`"`、`()`），直接丢进 FTS5 `MATCH` 会语法报错。所以系统先跑一遍 `_sanitize_fts5_query()`（`hermes_state.py:938-988`）：

- 保留合法的引号短语
- 去掉未配对的特殊字符
- 清理 dangling boolean（如开头结尾的 `AND`/`OR`）
- 对连字符/点号 term 加双引号包裹

### 2.4 执行的实际 SQL

```sql
SELECT
    m.id, m.session_id, m.role,
    snippet(messages_fts, 0, '>>>', '<<<', '...', 40) AS snippet,
    m.content, m.timestamp, s.source, s.started_at
FROM messages_fts
JOIN messages m ON m.id = messages_fts.rowid
JOIN sessions s ON s.id = m.session_id
WHERE messages_fts MATCH ?
ORDER BY rank
LIMIT ? OFFSET ?
```

**代码位置**: `hermes_state.py:1040-1058`

### 2.5 检索后的过滤与去重

拿到消息命中结果后，系统还要做几层过滤（`tools/session_search_tool.py:354-403`）：

| 过滤规则 | 目的 |
|---------|------|
| **排除当前 session 血统** | 不回忆 agent 已经知道的当前对话 |
| **根 session 去重** | 子 session 追溯到根，同一个根只留一个 |
| **排除 tool session** | 默认过滤掉 `source="tool"` 的隐藏会话 |
| **数量钳制** | `limit` 默认 3，最大 5 |

---

## 3. 加工层：原始对话太长，怎么压缩成可用摘要？

### 3.1 业务逻辑

检索到的历史 session 可能很长（几百条消息）。直接把完整对话塞进主模型 prompt 不现实，会烧光 token。

所以系统走了一个**两步压缩**策略：
1. **机械截断**：按 query 关键词位置，把对话截断到 ~100K 字符，保留最相关片段
2. **LLM 摘要**：把截断后的片段送给一个便宜的 auxiliary model（通常是 Gemini Flash），生成人话 summary

### 3.2 Step 1：加载并截断对话

对每个命中的 session：
- 加载完整对话历史（`db.get_messages_as_conversation`，`tools/session_search_tool.py:409`）
- 格式化成可读 transcript（`_format_conversation`，lines 56-87）
- 截断到关键词附近（`_truncate_around_matches`，lines 90-172）

#### 截断算法的精妙之处

`_truncate_around_matches()` 不是简单从开头截，而是：
1. 先尝试找到 query 短语的完整匹配位置
2. 找不到的话，找所有 query 关键词在 200 字符窗口内的共现位置
3. 再不行就找单个关键词位置
4. 选一个能覆盖最多匹配点的窗口，**窗口前后比例为 25% / 75%**（稍微偏向匹配点之后的内容）

这意味着 auxiliary LLM 看到的是**高密度相关片段**，而不是被淹没在几百条闲聊里。

### 3.3 Step 2：LLM 摘要

截断后的 transcript 被送进 auxiliary LLM 做摘要。

**System Prompt**（`tools/session_search_tool.py:207-219`）：

> "你正在回顾一段过去的对话记录以帮助回忆发生了什么。请围绕搜索主题总结这段对话，需包含：1. 用户问了什么或想完成什么；2. 采取了哪些行动以及结果如何；3. 关键决策、找到的解决方案或达成的结论；4. 任何重要的具体命令、文件、URL 或技术细节；5. 任何未解决或值得注意的事项。要详尽但简洁。保留对回忆有用的具体细节。用过去时态写成事实性回顾。"

**模型调用**：

```python
response = await async_call_llm(
    task="session_search",
    messages=[...],
    temperature=0.1,
    max_tokens=10000,
)
```

**关键细节**：`task="session_search"` 会路由到便宜的 auxiliary model（通常是 **Gemini Flash**）。这样烧的是廉价模型的 token，而不是主模型（Claude/GPT-4）的 token。

**代码位置**: `agent/auxiliary_client.py:2575+`

### 3.4 并行加速

如果命中了 3 个历史 session，系统会同时发起 3 个摘要请求（`asyncio.gather`，`tools/session_search_tool.py:425-431`），而不是串行等待。

---

## 4. 消费层：摘要结果如何被 agent 使用？

**不直接注入 prompt，而是作为 tool response 返回。**

agent 调用 `session_search` → 系统返回 JSON（见 1.2 节）→ agent 在自己的后续 reasoning 中阅读这些 summary → 决定如何在回复里引用。

这跟 Honcho 的自动注入模式完全不同：Honcho 是"系统帮你把用户模型塞进 prompt"，`session_search` 是"系统给你一个工具，你自己决定查不查、怎么用"。

---

## 5. 关键文件索引

| 内容 | 文件 | 行号 |
|------|------|------|
| SQLite schema + FTS5 触发器 | `hermes_state.py` | 36-112 |
| Query 清洗 `_sanitize_fts5_query` | `hermes_state.py` | 938-988 |
| FTS5 搜索 `search_messages` | `hermes_state.py` | 990-1091 |
| `session_search` 工具入口 | `tools/session_search_tool.py` | 297-488 |
| 对话格式化 `_format_conversation` | `tools/session_search_tool.py` | 56-87 |
| 智能截断 `_truncate_around_matches` | `tools/session_search_tool.py` | 90-172 |
| LLM 摘要 `_summarize_session` | `tools/session_search_tool.py` | 175-236 |
| 鼓励使用的 system prompt guidance | `agent/prompt_builder.py` | 158-162 |
| auxiliary model 路由 | `agent/auxiliary_client.py` | 2575+ |

---

## 设计洞察

1. **显式工具 vs 自动注入的权衡**：`session_search` 是 on-demand 的。好处是不会每次请求都检索历史（省 token、省延迟）；坏处是 agent 可能"忘记"查历史，需要 system prompt 不断提醒。

2. **两步压缩是必要设计**：原始对话动辄几万字，不经过「截断 + LLM 摘要」直接塞 prompt，token 成本爆炸。截断算法确保了 auxiliary LLM 看到的是信息密度最高的片段。

3. **FTS5 是本地 agent 的务实之选**：不需要额外部署向量数据库，SQLite 内置 FTS5 就能提供可用的全文检索和相关性排序（`ORDER BY rank`）。

4. **Auxiliary model 分工明确**：摘要这种"体力活"交给 Gemini Flash，主模型只负责看摘要后的结果做决策。这是非常成本意识强的工程选择。

5. **排除当前 session 血统是 token 效率细节**：通过 `parent_session_id` 链排除当前对话及其子会话，避免 agent 花 token 去回忆"它本来就已经知道"的内容。
