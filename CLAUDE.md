# Self-Evolution Research Space

## 研究背景与目的

用户在研究 **agent self-improvement / self-evolution** 领域的主流开源实现。背景是：现在社区里有大量关于 agent 自进化的说法和项目，用户希望深度理解这些项目的技术细节，并判断哪些设计值得在自己的 coding agent（关联生态：Claude Code / Codex / OpenClaw）中借鉴或实现。

本目录是研究空间，所有相关仓库都 clone 到这里供直接阅读源码。

---

## 已研究的仓库

### 1. OpenSpace (`./OpenSpace/`)
- **GitHub**: https://github.com/HKUDS/OpenSpace
- **核心方向**: 进化 skill 资产库（在线、持续演化）
- **MCP 接入**: 支持 `SKILL.md`，可作为 MCP 接到 Claude Code、Codex、OpenClaw 等

### 2. DGM — Darwin Gödel Machine (`./dgm/`)
- **GitHub**: https://github.com/jennyzzt/dgm
- **核心方向**: 进化 agent 自己的代码仓库（agent 改自己）
- **评测**: SWE-bench Verified + Polyglot，solved-task rate

### 3. Hermes Agent Self-Evolution (`./hermes-agent-self-evolution/`)
- **GitHub**: https://github.com/NousResearch/hermes-agent-self-evolution
- **核心方向**: 用 DSPy + GEPA 进化 skill / prompt / tool description 文本工件
- **当前实现进度**: Phase 1（skill files）已实现；tool descriptions / system prompts / code evolution 仍在规划中

### 4. Hermes Agent (`./hermes-agent/`)
- **GitHub**: https://github.com/NousResearch/hermes-agent
- **核心方向**: 面向终端用户的自进化 Agent（skill 生成与修订、memory nudges、跨 session 回忆、Honcho 用户建模）
- **深度研究笔记**: [`notes/hermes-agent-features/`](./notes/hermes-agent-features/)
  - `01-autonomous-skill-creation.md` — 自动生成 skill
  - `02-skill-self-improvement.md` — skill 持续修订
  - `03-memory-nudges.md` — agent-curated memory + periodic nudges
  - `04-fts5-session-recall.md` — 跨 session 检索回忆（FTS5 + LLM summarization）
  - `05-honcho-user-modeling.md` — Honcho dialectic user modeling

---

## 核心研究问题与已有结论

### 优化对象三条路线

| 项目 | 优化对象 | fitness 信号 | 特点 |
|------|---------|-------------|------|
| OpenSpace | skill 工作流/说明 | 在线执行监控指标 | 离落地最近 |
| Hermes | skill/prompt/tool description 文本 | task-specific eval dataset | 工程友好 |
| DGM | agent 代码仓本体 | coding benchmark solved-rate | 自进化味道最重 |

### 问题：agent 靠哪些信号来调优，完整轨迹还是打点？（2026-04-13）

详细分析见 → [`notes/2026-04-13-self-evolution-three-projects.md`](./notes/2026-04-13-self-evolution-three-projects.md)

---

## 研究方法论与经验教训

### 做笔记/分析时：代码逻辑 + 业务逻辑必须同时讲

**教训来源**：`hermes-agent` 的 skill creation / self-improvement 笔记初稿。

**问题**：只讲"`_iters_since_skill` 计数器到 10 就触发 `_spawn_background_review`"，读者（包括未来的自己）看不懂**这到底意味着什么**。

**修正后的标准**：
- 先用**业务语言**解释触发时机："当 agent 连续调用了大约 10 次普通工具却一直没调用 `skill_manage` 时，系统会在当前 turn 结束后自动派一个后台 review agent 去审视最近对话是否值得生成/更新 skill。"
- 再用**代码逻辑**给出支撑：文件路径、变量名、阈值默认值。

**规则**：代码是"怎么实现"，业务逻辑是"这在解决什么问题、在什么场景下发生"。两者缺一不可。

---

## 待研究问题

- [ ] 从 coding agent 视角：最应该抄哪个项目的哪一层设计？
- [ ] 如何接入 Claude Code / Codex / OpenClaw 生态？
- [ ] OpenSpace 的 GDPVal 两阶段评测：冷启动 vs 热启动的具体实现
- [ ] DGM 的 archive 采样策略（"高分且子节点少"倾向）的具体实现
- [ ] Hermes GEPA 和 DSPy 标准 optimizer 的区别是什么
