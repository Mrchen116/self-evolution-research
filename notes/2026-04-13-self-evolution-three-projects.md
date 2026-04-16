# Self-Evolution 三项目深度对比：OpenSpace / DGM / Hermes

> 研究日期：2026-04-13

---

## 横向对比总表

> 覆盖：优化对象 · 优化指标 · 评测集运行方式 · 评测结果→优化过程 · 版本管理

| 维度 | OpenSpace | DGM | Hermes |
|------|-----------|-----|--------|
| **优化对象** | Skill 文件（SKILL.md）的内容与工作流描述；三种演化类型：FIX（修复退化）/ DERIVED（派生增强）/ CAPTURED（从成功执行提炼新 skill） | Agent 代码库本体（`coding_agent.py`、`tools/`、`utils/`）；输出为 Python 源码 diff | Skill 文件的 body 文本（SKILL.md 正文，不含 YAML frontmatter） |
| **优化指标** | 在线运行指标，无固定 benchmark。Skill 粒度：applied_rate / completion_rate / effective_rate / fallback_rate；工具粒度：success_rate / latency | SWE-bench Verified / Polyglot 上的 solved-task rate（accuracy_score）；分阶段 small→medium→full 评测 | 合成 eval 集上的 composite fitness：correctness×0.5 + procedure_following×0.3 + conciseness×0.2 − length_penalty；benchmark（TBLite / YC-Bench）仅作 regression gate，不参与 fitness |
| **评测集运行方式** | **在线**，无独立评测集。每次真实任务执行后自动录制并触发分析；周期性指标监控也可触发批量演化 | **离线分阶段**。small 子集（~10 题）→ 通过阈值后跑 medium（~50 题）→ 可选 full；每次在隔离 Docker 容器内运行，有 30min 超时 | **离线合成**。强模型读 skill 文本生成约 20 个 `(task_input, expected_behavior_rubric)` 样例，按 train / val / holdout 切分；也支持接入 sessiondb 挖真实记录或手工 golden set |
| **评测结果→优化过程** | 多轮 analysis agent（最多 5 轮，配备 read_file / run_shell / MCP 工具）读取优先级压缩轨迹 → 产出 ExecutionAnalysis（含 FIX/DERIVED/CAPTURED 建议）→ skill evolver 执行演化 | 单次 o1 LLM 调用（system message 含完整代码库源码；user message 含截断日志 + benchmark 测试结果）→ 输出 problem statement（GitHub issue 格式）→ coding agent 多轮（Docker 内）修改代码仓 → 生成 `model_patch.diff` | DSPy GEPA 多轮迭代：每轮用当前 skill 文本跑 trainset → 关键词重叠快速打分 → 反思 LLM 提出改进版 skill 文本 → valset 验证 → 保留最优；最终在 holdout 上 LLM-as-judge 精确评分（注：当前实现存在 evolved_body 提取 bug，见下文详细节） |
| **版本管理** | SQLite 持久化的 SkillStore；每个 skill 有 SkillLineage（origin 枚举 + parent_skill_ids + diff）；形成**版本 DAG**；SkillRecord 保留滚动窗口 50 条历史分析；回退可按 lineage 追溯父版本 | **平铺 archive**（run_id 列表）；每个节点以 `metadata.json` + 累积 git patch 存储；`parent_commit` 字段记录亲代关系；parent 采样：score × 1/(1+children_count)（默认 score_child_prop）；所有可编译节点默认全保留（keep_all）；`dgm_metadata.jsonl` 按代追加记录 archive 快照 | 每次运行产出独立时间戳目录（`output/{skill}/{timestamp}/`）含 baseline + evolved + metrics.json；PLAN 中规划 git branch + PR；**无 DAG**，各次运行相互独立；回退靠 git revert |

---

## 一句话定性

| 项目 | 定性 |
|------|------|
| OpenSpace | 在线进化，skill 是一等公民，有完整的版本谱系和在线监控闭环 |
| DGM | 离线进化，agent 改自己的代码，o1 做诊断，coding agent 做实施，benchmark 做裁判 |
| Hermes | 离线进化，GEPA 优化 skill 文本，合成数据驱动，基础设施完整但核心存在实现 bug |

---

## 优化信号机制专题：完整轨迹还是打点？

---

## OpenSpace — 混合策略 + 智能优先级截断

### 录制的原始信号

任务执行时录制两个文件：
- `conversations.jsonl`：完整对话轮次（含 agent 推理过程）
- `traj.jsonl`：每步工具调用结构化记录（step, timestamp, tool, command, parameters, result）

### 静态预处理：优先级预算截断

把原始录制压缩到 **80,000 字符**以内，规则如下：

| 优先级 | 内容 | 策略 |
|--------|------|------|
| P0 CRITICAL | 用户指令 | 永不截断 |
| P1 CRITICAL | 最终迭代 assistant response | 永不截断 |
| P2 HIGH | 工具调用 + 工具报错（成对） | 高优先保留 |
| P3 HIGH | 旧迭代 assistant 推理；工具结果中有摘要则取摘要 | 预算够才保留 |
| P4 MEDIUM | 工具成功结果 | 预算足时保留 |
| P5 LOW | 系统 guidance 消息 | 优先丢 |

各 section 单独上限：工具报错 1000 chars，工具成功 800 chars，工具参数 500 chars，skill 内容 8000 chars/条。

skill 内容截断时**不是静默截断**，而是留展开入口（`analyzer.py:497`）：
```
... [truncated at 8000 chars — use read_file("/path/to/SKILL.md") to see full content]
```

### 分析过程：多轮 agent loop，不是单次调用

压缩后的轨迹交给一个 **analysis agent** 处理，最多 **5 轮**（`_ANALYSIS_MAX_ITERATIONS = 5`）。

**agent 拿到的工具**：
- `read_file`、`list_dir`、`run_shell`（文件系统 + shell）
- 任务执行时用过的所有工具（MCP 工具等）原封不动传入——即分析 agent 可以重现执行

**设计意图**（prompt 里原话）：
> "In most cases the trace above is sufficient — only use tools when evidence is ambiguous or you need to verify specific details."

**轮次机制**：
- 每轮：LLM 决定是否调工具；有工具调用则执行并继续
- 无工具调用：直接解析 JSON，结束
- 第 5 轮（最后一轮）：强制关闭工具，注入系统消息 "This is your FINAL round — you MUST output the JSON analysis object now"

实际上大多数分析 **1 轮**完成，工具调用是应对轨迹证据模糊时的兜底。

### 分析结果结构

```python
ExecutionAnalysis {
    task_completed: bool,
    execution_note: str,                   # 任务级别的自由文本分析
    tool_issues: List[str],                # 有问题的工具 key（语义失败，非规则层已捕获的）
    skill_judgments: List[SkillJudgment],  # 每个 skill 是否被实际应用
    evolution_suggestions: [               # FIX / DERIVED / CAPTURED
        { type, target_skill_ids, category, direction }
    ]
}
```

**tool_issues 的去重机制**：规则层已经记录了每次工具调用的 success/error；LLM 只补充规则层漏掉的**语义失败**（调用成功但结果不对）。若某工具的所有调用规则层已全标为 error，LLM 的重复报告会被过滤掉。

### 关键文件

- `openspace/skill_engine/conversation_formatter.py` — 优先级截断逻辑
- `openspace/skill_engine/analyzer.py` — 分析 agent loop（`_run_analysis_loop`）、prompt 构建、结果解析

---

## DGM — 全量 Markdown 日志 + 硬截断

### 录制的原始信号

三类文件，全部来自 coding agent 跑 benchmark 的输出：
- `{entry}.md`：agent 完整运行日志（Markdown 格式，逐步记录推理和工具调用）
- `{entry}.json`：predicted patch（agent 生成的代码 diff）+ eval_result
- `report.json`（或 `{entry}_eval.md`）：benchmark 测试结果（FAIL_TO_PASS / PASS_TO_PASS 两类测试的通过情况）

### 长轨迹处理：硬截断，无智能压缩

```python
if len(md_log) > 250000:
    md_log = md_log[:250000] + "\n<log clipped>"
```

没有优先级，没有摘要，可能截断在推理中途。250K 字符是硬边界。

### 诊断过程：单次 LLM 调用，但 system message 塞入完整代码库

这是与 OpenSpace 最关键的区别：**DGM 的 `diagnose_problem()` 是单次 LLM 调用，没有 agent loop，没有工具**。

但 prompt 的构造非常特殊：

**system message** = coding agent 架构说明 + **当前代码库完整源码**
- 包含：`coding_agent.py`、`tools/` 下所有 `.py`、`utils/` 下所有 `.py`（排除几个运维类文件）
- 如果存在历史 patch，也一并附入
- 也就是说：LLM 拿到的不是"如何理解代码"的说明，而是**真实可读的完整代码文本**

**user message** = 运行日志（截断后）+ GitHub issue + predicted patch + test patch + 测试结果

使用的模型是 `o1-2024-12-17`（reasoning model，非 Claude）。

**输出结构**（JSON）：
```json
{
  "log_summarization": "agent 如何尝试解决问题的分析",
  "potential_improvements": "可改进的方向",
  "improvement_proposal": "选一个最高价值的改进详细描述",
  "implementation_suggestion": "针对当前代码库具体怎么改",
  "problem_description": "把上面两个包装成 GitHub issue 格式"
}
```

`problem_description` 就是最终交给 coding agent 去实现的输入。

### 特殊路径：预置 prompt 模板

entry_id 不是真实 benchmark 任务时，走预置模板而非日志分析：
- `'solve_empty_patches'`：应对 agent 没生成 patch 的情况
- `'solve_stochasticity'`：应对 agent 随机性导致结果不稳定
- `'solve_contextlength'`：应对 context 超长报错

polyglot 模式还有 25% 概率随机触发 stochasticity 模板（不管实际失败原因）。

### 完整流程

```
parent agent 代码库（当前版本）
    +
child 运行日志（截断 250K）+ benchmark 测试结果
    ↓
diagnose_problem()：单次 o1 调用 → problem statement（GitHub issue 格式）
    ↓
coding_agent.py 在 Docker 中运行（多轮 coding agent）：按 problem statement 修改 /dgm/ 代码库
    ↓
生成 model_patch.diff
    ↓
run_harness_swe/polyglot()：跑 benchmark 评测（10 → 50 → 200 任务分阶段）
    ↓
diagnose_improvement()：单次 LLM 调用，评估改动效果（score -2 到 +2）
    ↓
结果写入 archive（DGM_outer.py 管理）
```

### 关键文件

- `dgm/prompts/self_improvement_prompt.py` — 日志截断、prompt 构建、`get_current_code()`（读入代码库）
- `dgm/self_improve_step.py` — 完整流程编排
- `dgm/coding_agent.py` — 执行代码修改的多轮 coding agent
- `dgm/prompts/diagnose_improvement_prompt.py` — 改后效果评估 prompt

---

## Hermes — 合成 eval 集 + GEPA，无长轨迹问题

### 正常 agent 逻辑下的完整流程

**第一步：造测试题（一次 LLM 调用）**

把 skill 文件喂给强模型，生成 20 道题：
```
题目: "帮我 review 这段 Python 代码的安全问题"
期望行为: "应该识别出 SQL injection，说明危害，给出修复建议"
```

**第二步：用当前 skill 答题（每题一次 LLM 调用）**

对每道题调用：
```
system/instructions: "[skill 文件全文]"
user: "帮我 review 这段代码..."
```
LLM 产出 chain-of-thought + output。**这就是 Hermes 里的"trace"**——不是多步 agent session，就是单次 LLM 调用的推理链。所以根本不存在长轨迹问题。

**第三步：打分**

- 快速路径（优化迭代中）：关键词重叠启发式，0.3 + 0.7 × overlap
- 精确路径（验证阶段）：LLM-as-judge，多维打分
  ```python
  FitnessScore {
      correctness: float,          # 权重 0.5
      procedure_following: float,  # 权重 0.3
      conciseness: float,          # 权重 0.2
      length_penalty: float,       # 减分
      feedback: str                # 文字反馈，供 GEPA 读取
  }
  ```

**第四步：GEPA 改 skill 文本**

GEPA 做的事：把所有失败案例打包，再调用一次 LLM：
```
"以下是 agent 按当前 skill 做题失败的案例：
  [task, expected, actual_output, score] × N
当前 skill 文本：[skill 全文]
请分析 skill 哪里写得不好，给出改进版本。"
```
LLM 输出改进后的 skill 文本。GEPA 在 val set 验证，多轮迭代保留最优。

**第五步：写回 skill 文件，提 PR。**

### 当前实现的一个 Bug

`evolve_skill.py:184`：
```python
evolved_body = optimized_module.skill_text  # ← 有问题
```

`self.skill_text` 是普通 Python 属性，GEPA 优化完之后改的是 DSPy 内部 predictor 的 Signature instruction，**不会回写到 `self.skill_text`**。导致 `evolved_skill.md` 和 `baseline_skill.md` 内容相同。

**影响范围**：只影响"写回 skill 文件"这一步，其余基础设施（数据集生成、打分、约束校验）都是完整可用的。整个仓库更像"框架搭好、核心接线断了"的状态。

**关键文件**：
- `evolution/core/dataset_builder.py` — 合成数据集生成
- `evolution/core/fitness.py` — 打分函数
- `evolution/skills/skill_module.py` — DSPy 封装
- `evolution/skills/evolve_skill.py` — 主流程（bug 在此）
