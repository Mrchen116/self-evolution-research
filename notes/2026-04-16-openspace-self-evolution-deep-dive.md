# OpenSpace Self-Evolution 深度拆解

> 分析对象：`OpenSpace/`（HKUDS/OpenSpace）
> 分析日期：2026-04-16

---

## 一、项目定位

OpenSpace 优化的不是底层 agent 代码，而是 **skill 目录里的工件**：

- `SKILL.md` 主说明文件
- skill 目录中的辅助脚本 / 配置 / 示例文件
- 这些工件在真实任务中被选择、注入、执行、分析、再进化

它本质上是一个 **在线 self-evolution skill runtime**：

- 执行真实任务时自动录制轨迹
- 任务结束后用分析 agent 判断 skill 是否真的被用了、是否完成任务、是否该进化
- 立即触发 `FIX / DERIVED / CAPTURED`
- 另有两条后台触发线：
  - 工具退化触发修复
  - 指标监控触发修复或派生

所以 OpenSpace 的“进化对象”是 skill DAG，而不是单个 prompt，也不是 agent 本体源码。

---

## 二、完整流程图

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                         OpenSpace Self-Evolution Loop                        │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 0: 初始化                                                                │
│ ──────────────────────────────────────────────────────────────────────────── │
│ OpenSpace.initialize()                                                       │
│   1. 建 LLMClient / GroundingClient / RecordingManager                       │
│   2. SkillRegistry 从 3 类目录 discover：                                     │
│      - OPENSPACE_HOST_SKILL_DIRS                                             │
│      - config_grounding.json 里的 skill_dirs                                │
│      - openspace/skills/ 内建 skills                                         │
│   3. SkillStore.sync_from_registry()                                         │
│      - 给每个已发现 skill 建初始 SkillRecord                                 │
│      - skill_id 来自 skill 目录下的 .skill_id sidecar                        │
│   4. 初始化 ExecutionAnalyzer + SkillEvolver                                 │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: 任务开始前选 skill                                                     │
│ ──────────────────────────────────────────────────────────────────────────── │
│ _select_and_inject_skills(task)                                              │
│   1. 先按历史质量过滤明显差的 skill                                           │
│   2. 若 skill 数 > 10：BM25 → embedding 预筛                                 │
│   3. LLM 只看 skill_id + description + 质量摘要，选最多 N 个                 │
│   4. 选中后才把完整 SKILL.md 注入 GroundingAgent                             │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: 执行真实任务 + 录制                                                    │
│ ──────────────────────────────────────────────────────────────────────────── │
│ OpenSpace.execute(task)                                                      │
│   1. GroundingAgent 迭代调用 shell/gui/mcp/web/system 等工具                 │
│   2. RecordingManager 持久化：                                                │
│      - metadata.json                                                         │
│      - conversations.jsonl                                                   │
│      - traj.jsonl                                                            │
│   3. 任务完成 / 失败后，在 finally 里进入分析阶段                             │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 3: 执行后分析                                                            │
│ ──────────────────────────────────────────────────────────────────────────── │
│ ExecutionAnalyzer.analyze_execution()                                        │
│   1. 读取 metadata / conversations / traj                                    │
│   2. 构造分析 prompt：                                                        │
│      - 用户任务                                                               │
│      - 已选 skill 内容                                                        │
│      - 工具定义 + 实际用到的工具                                              │
│      - 会话日志（优先级截断，不是简单 tail truncate）                         │
│      - traj 工具时间线                                                        │
│   3. 运行 analysis agent loop（最多 5 轮，可调工具）                          │
│   4. 输出 ExecutionAnalysis：                                                 │
│      - task_completed                                                        │
│      - tool_issues                                                           │
│      - skill_judgments                                                       │
│      - evolution_suggestions[]                                               │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 4: 写回指标 / 版本账本                                                   │
│ ──────────────────────────────────────────────────────────────────────────── │
│ SkillStore.record_analysis()                                                 │
│   1. execution_analyses 插 1 行                                              │
│   2. skill_judgments 插多行                                                   │
│   3. 对每个被选中的 skill 原子更新计数：                                      │
│      - total_selections                                                      │
│      - total_applied                                                         │
│      - total_completions                                                     │
│      - total_fallbacks                                                       │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                        ┌──────────────┴──────────────┐
                        ▼                             ▼
┌──────────────────────────────┐    ┌─────────────────────────────────────────┐
│ STEP 5A: Trigger 1           │    │ STEP 5B: 后台 Trigger 2 / 3            │
│ post-analysis 立即进化        │    │ 周期性后台进化                          │
│ process_analysis()           │    │ _maybe_evolve_quality()                │
│                              │    │   - tool degradation                   │
│ 分析中若产生 suggestion：     │    │   - every 5 executions metric check   │
│   FIX / DERIVED / CAPTURED   │    │ 两者都先经 LLM confirmation 再执行     │
└──────────────────────────────┘    └─────────────────────────────────────────┘
                        │                             │
                        └──────────────┬──────────────┘
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 6: SkillEvolver 执行进化                                                 │
│ ──────────────────────────────────────────────────────────────────────────── │
│ 统一流程：                                                                    │
│   1. build EvolutionContext                                                  │
│   2. 跑 evolution agent loop（最多 5 轮，可继续调工具）                       │
│   3. 输出 skill edit 内容 + <EVOLUTION_COMPLETE>                             │
│   4. apply_with_retry()：最多 3 次                                            │
│      - 支持 FULL / DIFF / PATCH 三种编辑格式                                  │
│      - 结构校验失败则把错误回喂给 LLM 重试                                    │
│   5. 生成 SkillRecord + SkillLineage                                          │
│   6. 写入 SQLite，必要时失活父节点，更新 .skill_id，热注册回 registry         │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 7: 新 skill 进入下一轮检索与执行                                          │
│ ──────────────────────────────────────────────────────────────────────────── │
│ FIX   : 同目录、同 path、新 skill_id，父版本 is_active=False                  │
│ DERIVED: 新目录、新 skill_id，可单亲增强或多亲合并                            │
│ CAPTURED: 新根节点，新目录，无父节点                                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、核心数据模型

### 3.1 `SkillMeta` vs `SkillRecord`

`SkillMeta` 是 discovery / selection 阶段的轻量对象，只关心：

- `skill_id`
- `name`
- `description`
- `path`

`SkillRecord` 才是进化系统的一等公民，额外持有：

- lineage
- tool dependencies / critical tools
- 质量计数器
- recent analyses
- active / inactive 状态

也就是说，**registry 管“可发现”，store 管“可进化”**。

### 3.2 `skill_id` 的真实身份机制

OpenSpace 没有用目录名当主键，而是给每个 skill 目录写一个 `.skill_id` sidecar：

- 初次 discover：`{name}__imp_{uuid8}`
- FIX / DERIVED / CAPTURED 后由 evolver 重写 sidecar

这个设计很关键，因为：

- skill 可以移动目录但 ID 保持稳定
- FIX 虽然复用同一个目录，但会产生新版本 ID
- registry / DB / LLM prompt 全都用这个 ID 作为统一引用

### 3.3 `SkillLineage` 的版本 DAG 语义

OpenSpace 明确实现了 DAG，而不是平铺版本列表。

| 来源 | 父节点 | 代数规则 | 文件系统语义 |
|------|--------|----------|--------------|
| `IMPORTED` | 0 | `0` | 初次导入 |
| `CAPTURED` | 0 | `0` | 新建目录 |
| `FIXED` | 1 | `parent.generation + 1` | 同名、同目录、同 path；旧版本失活 |
| `DERIVED` | 1+ | `max(parent.gen)+1` | 新目录；支持多亲合并 |

额外值得注意：

- `content_snapshot` 保存整个 skill 目录的全量快照
- `content_diff` 保存 unified diff
- **多亲 DERIVED 时 `content_diff=""`**，因为作者认为“创作式合并”不适合硬算单亲 patch

---

## 四、任务执行前：skill 是怎么被选出来的

### 4.1 不是把所有 SKILL.md 全塞进上下文

`SkillRegistry.select_skills_with_llm()` 的设计相当克制：

1. 先拿 `SkillStore.get_summary()` 的历史质量数据
2. 过滤掉明显差的 skill
3. 如果本地 skill 数量超过 `PREFILTER_THRESHOLD = 10`
   - 先跑 BM25
   - 再跑 embedding rerank
4. LLM 看到的只是：
   - `skill_id`
   - `description`
   - 质量摘要（如 `success 3/5 = 60%`）
5. 只有被选中之后，才把完整 skill body 注入 agent

这意味着 OpenSpace 做的是 **两阶段 skill 检索**：

- 阶段 1：轻量 catalog 级选择
- 阶段 2：选中后再加载全文

### 4.2 质量过滤的实际规则

在把 skill catalog 给 LLM 之前，会先剔除：

- `selections >= 2` 且 `completions == 0`
- `applied >= 2` 且 `fallbacks / applied > 0.5`

这一步的作用不是最终裁决，而是 **不给明显退化 skill 继续污染上下文**。

### 4.3 注入时的颗粒度

注入的不是 skill 摘要，而是完整 `SKILL.md` 内容；但这个阶段只对被选中的少量 skill 生效。

所以 OpenSpace 的关键假设是：

> skill selection 是稀疏的，skill evolution 必须建立在“被真正选择过且可观测”的历史上。

---

## 五、任务执行后：Analyzer 到底怎么分析

### 5.1 分析输入不是只看最终回答

Analyzer 会加载三类工件：

- `metadata.json`
- `conversations.jsonl`
- `traj.jsonl`

其中：

- `conversations.jsonl` 记录 reasoning / tool call / tool result 的 delta
- `traj.jsonl` 是结构化的工具调用时间线
- `metadata.json` 提供选中的 `skill_id`、可用工具、执行状态等

### 5.2 会话日志不是简单截断，而是“优先级预算”

`conversation_formatter.py` 做了很细的优先级编排：

| 优先级 | 内容 |
|--------|------|
| P0 | 用户原始指令 |
| P1 | 最后一轮 assistant 响应 |
| P2 | tool call + tool error |
| P3 | 非最终 assistant reasoning、带 embedded summary 的 tool result |
| P4 | 普通 tool success result |
| P5 | iteration 之间的 system guidance |

总预算默认 `80,000 chars`。若超长：

- 先保 P0-P3
- 再尽量塞 P4 / P5
- 若连高优内容都超长，就继续压缩旧轮次、优先保 final iteration

这和 DGM 那种单纯 `[:250000]` 硬截断完全不是一个思路。

### 5.3 Analyzer 不是单次 LLM call，而是一个小 agent

`ExecutionAnalyzer._run_analysis_loop()` 最多跑 5 轮：

- 普通轮：允许调工具
- 最后一轮：强制关工具，必须输出 JSON

可用工具包括：

- 执行时那批真实工具
- 额外的 `read_file` / `list_dir` / `run_shell`

prompt 里明确写了：

> 大多数情况下 trace 已经足够，只有证据模糊时才用工具二次调查

也就是说，它不是“日志摘要器”，而是一个 **带复查能力的 postmortem agent**。

### 5.4 输出对象：`ExecutionAnalysis`

`ExecutionAnalysis` 不是一段自由文本总结，而是后续 store / quality / evolver 都会消费的**结构化判定对象**。它在代码里的字段语义大致是：

| 字段 | 含义 | 下游用途 |
|------|------|----------|
| `task_id` | 这次 execution 的主键 | 对应 recording 和 `execution_analyses` 表 |
| `timestamp` | 这次任务的分析时间 | 任务级时间戳 |
| `task_completed` | Analyzer 对“任务是否真的完成”的最终裁决，不看 agent 自报 `success` | 决定 skill completion 统计 |
| `execution_note` | 任务为什么成 / 败的短摘要 | 给人读，也作为后续 evolution 上下文 |
| `tool_issues` | 哪些工具在这次任务里“实际有问题”，包括语义失败，不只是报错 | 回灌 ToolQualityManager，拉低 `recent_success_rate` |
| `skill_judgments[]` | 对每个被选中 skill 判“是否真的进入执行链”，不是只看它有没有被注入 | 更新 `selections / applied / completions / fallbacks` |
| `evolution_suggestions[]` | 已经成型的进化动作，不是泛建议；包含 `fix / derived / captured`、目标 skill 和方向 | 直接驱动 Trigger 1 执行进化 |
| `analyzed_by` | 产出分析的模型 | 记录来源 |
| `analyzed_at` | 分析完成时间 | 审计 / 调试 |

其中有两个派生语义也很重要：

- `candidate_for_evolution = len(evolution_suggestions) > 0`
- `skill_ids = [j.skill_id for j in skill_judgments]`

也就是说，是否进入 Trigger 1，不靠额外布尔开关，而是看 suggestion 列表是否为空。

### 5.5 两个很细的工程处理

#### A. skill_id 自动纠错

LLM 很容易把 `weather__imp_61f694cb` 写成 `...61f694bc`。

Analyzer 会对未知 ID 做：

- 同前缀匹配
- 编辑距离搜索
- 唯一近邻替换

这是一个很实用的“LLM 结果纠偏器”。

#### B. tool issue 去重

规则层已经知道某次工具调用是不是报错；LLM 只补 **语义失败**。

如果某工具在 `traj` 里所有调用都已经是 error，Analyzer 输出的同类 issue 会被过滤，不重复污染 tool quality 系统。

---

## 六、三条进化触发线

### 6.1 Trigger 1：post-analysis

这是最直接的链路：

1. 任务执行完
2. Analyzer 产出 suggestion
3. `SkillEvolver.process_analysis()` 立即执行

适合：

- 某个 skill 明确误导 agent
- 某个 skill 成功但值得增强
- 本次没 skill，但 agent 发现了可复用流程

### 6.2 Trigger 2：tool degradation

当 tool quality manager 发现某工具最近 success rate 下降时：

1. 找出依赖该工具的 skill
2. 先做规则筛选
3. 再调用 `_llm_confirm_evolution()` 让 LLM 判断是否真要 FIX
4. 通过后进入 evolve

并且它做了 anti-loop：

- `_addressed_degradations[tool_key]` 记录“这个退化问题已修过哪些 skill”
- 工具恢复后会清空
- 再次退化时才重新考虑

### 6.3 Trigger 3：metric monitor

每 5 次 execution 后触发一次后台检查：

- `min_selections=5`
- 看 `applied_rate / completion_rate / effective_rate / fallback_rate`
- 先规则判定
- 再让 LLM 二次确认

这条线的目标不是“解决某次具体失败”，而是 **基于统计行为修 skill**。

---

## 七、SkillEvolver 的真实执行机制

### 7.1 统一上下文：`EvolutionContext`

无论来自哪条触发线，都会整理成：

- trigger 类型
- suggestion（FIX / DERIVED / CAPTURED）
- parent skill records / contents / dirs
- recent analyses
- tool degradation summary 或 metric summary
- 可用工具

所以 evolver 的后半段是统一的，只是上下文不同。

### 7.2 Evolution loop 是“看 token 终止”，不是“看有没有调工具”

`_run_evolution_loop()` 最多 5 轮，终止条件不是 tool calls，而是 assistant 输出：

- `<EVOLUTION_COMPLETE>`
- `<EVOLUTION_FAILED>`

这意味着：

- LLM 可以多轮查资料
- 也可以不查资料直接给 edit
- 最后一轮会被强制要求收敛

这个设计比常见的“无工具调用即结束”更稳，因为 skill 编辑很可能需要一轮思考但不需要工具。

### 7.3 Patch 应用层支持三种格式

`patch.py` 支持：

- `FULL`：完整文件内容
- `DIFF`：SEARCH/REPLACE
- `PATCH`：`*** Begin Patch` 多文件格式

并且适配三类操作：

- `fix_skill()`：原地修
- `derive_skill()`：复制父目录后修改；多亲时直接新建目录
- `create_skill()`：从零新建 skill

### 7.4 失败后不是放弃，而是“apply-retry”

`_apply_with_retry()` 会在以下情况重试：

- patch parse error
- patch apply 失败
- 结构校验失败

重试时会把：

- 上一次错误
- 当前磁盘上的真实文件内容

喂回 LLM，让它按 ground truth 修第二版。最多 3 次。

### 7.5 三种进化的文件系统与 DB 语义

#### FIX

- 同目录
- 同 path
- 新 `skill_id`
- 父版本 `is_active=False`
- registry 用新 ID 替换旧 ID

这意味着 FIX 是“版本替换”，不是原记录就地改写。

#### DERIVED

- 新目录
- 新 `skill_id`
- 单亲：增强版
- 多亲：merge / compose
- 继承所有 parent 的 tool deps / critical tools / tags 的并集

#### CAPTURED

- 新目录
- 无父节点
- skill 根目录优先级：
  - 显式 `capture_dir`
  - 从本次分析中所用 skill 所在根目录推断
  - registry 第一个 skill root 兜底

这个“捕获落在哪个 host skill dir”是多 host-agent 场景里非常关键的工程细节。

---

## 八、SQLite 账本：为什么它真的是版本系统

### 8.1 存储结构

`SkillStore` 的核心表：

- `skill_records`
- `skill_lineage_parents`
- `execution_analyses`
- `skill_judgments`
- `skill_tool_deps`
- `skill_tags`

数据库默认在：

- `<project_root>/.openspace/openspace.db`

并使用：

- WAL
- busy timeout
- 读写分离连接
- checkpoint on close

这说明作者确实把它当长期运行系统，而不是 demo。

### 8.2 指标更新语义

`record_analysis()` 对每个被选中的 skill 原子更新：

- `total_selections += 1`
- `total_applied += skill_applied`
- `total_completions += (skill_applied and task_completed)`
- `total_fallbacks += (not skill_applied and not task_completed)`

于是：

- `applied_rate = applied / selections`
- `completion_rate = completions / applied`
- `effective_rate = completions / selections`
- `fallback_rate = fallbacks / selections`

这个定义非常实用，因为它把“被选中但没用上”当成一种质量信号，而不是简单记成失败。

### 8.3 版本可追溯性

OpenSpace 不只存当前 skill 内容，还存：

- 全目录 `content_snapshot`
- 全目录 unified diff
- parent list
- generation
- source task id
- created_by

这让它具备三个能力：

- 可视化 lineage tree
- 精准回看某版本到底改了什么
- 在 cloud / dashboard 里展示技能演化历史

---

## 九、我认为 OpenSpace 最重要的设计判断

### 9.1 它优化的是“执行知识”，不是“模型参数”

skill 是 markdown + 脚本 + 目录快照，因此：

- 修改成本低
- 可解释
- 可审计
- 可分享

这和 Hermes 的“优化 skill body 文本”类似，但 OpenSpace 更进一步，把 **多文件目录** 当成版本单元。

### 9.2 它把“真实任务后的复盘”做成了一等公民

很多 self-improvement 系统只在 benchmark 上离线跑。OpenSpace 的核心价值是：

- 真实任务就是训练信号
- 失败是 FIX 信号
- 成功也是 DERIVED / CAPTURED 信号

这使它天然适合长尾环境，而不是只适配固定评测集。

### 9.3 它真正实现了“版本 DAG + 在线指标 + 后台修复”三件套

这三件事同时成立，系统才像“在线自进化平台”：

- DAG 解决版本历史
- 指标解决何时触发
- 后台修复解决系统长期稳定性

OpenSpace 在这三项上都不是概念，而是有明确代码落地。

---

## 十、局限与代价

### 10.1 没有固定 benchmark 作为强约束

OpenSpace 的优化目标主要来自：

- 真实任务
- skill 统计指标
- LLM postmortem 判断

优点是贴近生产，缺点是：

- 难做严格横向比较
- LLM judgment 本身会有漂移
- 不同任务分布变化会影响进化方向

### 10.2 质量指标只覆盖“被选中的 skill”

如果 selector 没把一个本该有用的 skill 选出来，这个问题不会直接体现在该 skill 的 counters 上，而会体现为 selection 侧问题。

所以 OpenSpace 里其实有两层优化问题：

- skill content 本身
- skill retrieval / selection 策略

当前代码对前者做得更深。

### 10.3 进化仍然依赖 LLM 的结构化输出稳定性

虽然加了：

- ID 纠错
- apply retry
- final round 强收敛

但 `FIX / DERIVED / CAPTURED` 仍然高度依赖 LLM 输出 patch 的质量。

---

## 十一、一句话结论

OpenSpace 是这三个项目里 **最接近“在线技能生态系统”** 的一个：它不是拿 benchmark 去调一个 prompt，而是把 skill 当成带 lineage、指标、依赖、全量快照的长期资产，在真实任务闭环里持续修、分叉、捕获新技能。

如果把 self-evolution 理解成“系统在运行中自己积累可复用执行知识”，OpenSpace 是目前这三个仓库里工程闭环最完整的一种实现。
