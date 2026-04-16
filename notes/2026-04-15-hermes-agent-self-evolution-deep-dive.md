# Hermes Agent Self-Evolution 深度拆解

> 分析对象：`hermes-agent-self-evolution/`（NousResearch/hermes-agent-self-evolution）
> 分析日期：2026-04-15

---

## 一、项目定位

Hermes Agent Self-Evolution 是一个**不训练模型权重、纯靠 API 调用**的文本工件优化管道。它把 agent 的 skill、tool description、system prompt 当成"可进化参数"，用 DSPy + GEPA 自动搜索更优版本。

优化目标分三层：

| 层级 | 内容 | 当前状态 |
|------|------|----------|
| 1. Skill 文件 (SKILL.md) | 任务级别的技能说明 | **Phase 1 已实现并验证** |
| 2. Tool descriptions | 工具描述文本 | 规划中 (Phase 2) |
| 3. System prompts | 系统提示词分段 | 规划中 (Phase 3) |
| 4. Tool implementation code | Python 工具代码 | 规划中 (Phase 4，换 Darwinian Evolver) |

---

## 二、完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Hermes Self-Evolution Pipeline                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 1: 加载 Baseline Skill                                                │
│  ─────────────────────────                                                  │
│  读取 hermes-agent repo 中的 SKILL.md                                       │
│  解析出：frontmatter (YAML) + body (markdown 说明)                           │
│  例如：skills/web-search/arxiv/SKILL.md                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 2: 生成/加载 Eval Dataset                                             │
│  ────────────────────────────────                                           │
│  三种来源（三选一）：                                                         │
│    A) synthetic  ──► LLM 读取 skill 全文，生成 (task_input, expected_behavior)│
│    B) sessiondb  ──► 从真实会话历史里挖掘并筛选                             │
│    C) golden     ──► 人工标注的 JSONL 数据集                                │
│                                                                              │
│  自动拆分为：train / val / holdout（默认 50% / 25% / 25%）                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 3: 把 Skill 包装成 DSPy Module                                        │
│  ─────────────────────────────────                                          │
│                                                                              │
│  SkillModule (dspy.Module)                                                   │
│  ├── Signature: TaskWithSkill                                                │
│  │     Inputs:  skill_instructions, task_input                               │
│  │     Output:  output                                                       │
│  └── Predictor: dspy.ChainOfThought                                          │
│                                                                              │
│  这样 GEPA 就能把 skill body 当成"可优化参数"来变异                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 4: 运行优化器 (GEPA / MIPROv2 / BootstrapFewShot)                     │
│  ────────────────────────────────────────                                   │
│                                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐                 │
│  │ Bootstrap   │◄──►│  MIPROv2    │◄──►│  GEPA (Primary) │                 │
│  │ FewShot     │    │  (fallback) │    │  (Primary)      │                 │
│  └─────────────┘    └─────────────┘    └─────────────────┘                 │
│        ▲                                          │                          │
│        │                                          │                          │
│        └────────  Phase 1 实际用的 ──────────────┘                          │
│                                                                              │
│  优化器内部循环：                                                             │
│  1. 用当前 skill 版本回答 trainset 中的问题（LLM 调用）                      │
│  2. fitness function 给分                                                     │
│  3. GEPA 分析 trace，提出定向 mutation                                        │
│  4. 保留高分 variant，进入下一轮                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 5: 提取 Evolved Skill                                                 │
│  ────────────────────────                                                   │
│  从 optimized_module 中取出变异后的 skill_text                               │
│  重新组装：frontmatter + evolved body → 新的 SKILL.md                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 6: 约束校验 (Constraint Validation)                                   │
│  ─────────────────────────────────────                                      │
│  硬门槛，全通过才能继续，否则直接丢弃：                                       │
│    □ size_limit       ──► skill ≤ 15KB, tool desc ≤ 500 chars              │
│    □ growth_limit     ──► 长度增长 ≤ baseline 的 +20%                      │
│    □ non_empty        ──► 不能为空                                          │
│    □ skill_structure  ──► 必须有 YAML frontmatter + name + description     │
│    □ test_suite       ──► pytest 100% 通过（可选）                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 7: Holdout 评估 (Baseline vs Evolved)                                 │
│  ──────────────────────────────────────────                                 │
│  用同一套 holdout 数据集分别跑：                                              │
│    - baseline_module(task_input=...)                                         │
│    - optimized_module(task_input=...)                                        │
│  计算平均 score，得出 improvement %                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 8: 输出与报告                                                           │
│  ────────────────────────                                                   │
│  保存到 ./output/<skill_name>/<timestamp>/：                                 │
│    - evolved_skill.md                                                        │
│    - baseline_skill.md                                                       │
│    - metrics.json                                                            │
│  最终通过 PR 提交到 hermes-agent 仓库（人工 review，不 auto-merge）          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、核心组件详解

### 3.1 Skill → DSPy Module 的封装

文件：`evolution/skills/skill_module.py`

```python
class SkillModule(dspy.Module):
    class TaskWithSkill(dspy.Signature):
        skill_instructions: str = dspy.InputField(...)
        task_input: str = dspy.InputField(...)
        output: str = dspy.OutputField(...)

    def forward(self, task_input: str) -> dspy.Prediction:
        result = self.predictor(
            skill_instructions=self.skill_text,
            task_input=task_input,
        )
        return dspy.Prediction(output=result.output)
```

关键点：
- `skill_text` 就是 SKILL.md 的 body，它是**可优化参数**
- GEPA 会不断变异这个字符串，然后重新调用 `forward()` 评估效果
- `ChainOfThought` 内部会让 LLM 先输出 reasoning，但模块只把最终的 `output` 暴露给 fitness 函数

---

### 3.2 数据集生成机制

文件：`evolution/core/dataset_builder.py`

**Synthetic 生成流程：**

1. 把完整的 skill 文本喂给 `judge_model`（默认 `gpt-4.1`）
2. 调用 `SyntheticDatasetBuilder.GenerateTestCases` 这个 DSPy Signature
3. LLM 输出 JSON 数组，每个元素包含：
   - `task_input`: 用户可能会怎么问
   - `expected_behavior`: rubric —— 好回答应该包含什么/做什么（不是 exact match）
   - `difficulty`: easy / medium / hard
   - `category`: 测试 skill 的哪个方面
4. 自动 shuffle 并拆分为 train / val / holdout

**三种数据源对比：**

| 来源 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| synthetic | 零成本、可批量生成 | 可能不够真实 | 快速迭代、Phase 1 |
| sessiondb | 来自真实对话 | 需要外部 session 数据 | 有用户历史积累时 |
| golden | 人工精准标注 | 人力成本高 | 最终验收、关键 skill |

---

### 3.3 Fitness 的两层实现

文件：`evolution/core/fitness.py`

#### 版本 A：完整版 LLM-as-judge（代码已实现，但未在 Phase 1 使用）

```python
@dataclass
class FitnessScore:
    correctness: float         # 0-1，回答是否正确
    procedure_following: float # 0-1，是否遵循 skill 的流程
    conciseness: float         # 0-1，是否简洁
    length_penalty: float      # 0-1，过长惩罚
    feedback: str              # 文本反馈，供 GEPA 做 reflective mutation

    @property
    def composite(self) -> float:
        raw = 0.5 * correctness + 0.3 * procedure_following + 0.2 * conciseness
        return max(0.0, raw - length_penalty)
```

`length_penalty` 计算方式：
- `artifact_size / max_size > 0.9` 时开始惩罚
- 在 90% 处为 0，100% 处为 0.1，110% 处为 0.3（封顶）
- 公式：`min(0.3, (ratio - 0.9) * 3.0)`

LLM-as-judge 的输入：
- `task_input`, `expected_behavior` (rubric), `agent_output`, `skill_text`
- 由 `dspy.ChainOfThought(JudgeSignature)` 打分并输出改进建议

#### 版本 B：Phase 1 实际使用的 Heuristic Scorer

```python
def skill_fitness_metric(example, prediction, trace=None) -> float:
    agent_output = getattr(prediction, "output", "")
    expected = getattr(example, "expected_behavior", "")

    expected_words = set(expected.lower().split())
    output_words = set(agent_output.lower().split())
    overlap = len(expected_words & output_words) / len(expected_words)

    score = 0.3 + (0.7 * overlap)
    return min(1.0, max(0.0, score))
```

本质：**把 expected_behavior rubric 和 agent 最终输出做关键词重叠（token-level set intersection）**。基础分 0.3，overlap 越高分数越高。

> ⚠️ **重要发现**：报告中写的 "Baseline 0.408 → Optimized 0.569 (+39.5%)" 就是这个 heuristic scorer 的分数，**不是** LLM-as-judge 的 composite fitness。Phase 1 为了节省 API 成本没有跑完整的多维评判。

---

### 3.4 优化器选择与 fallback

文件：`evolution/skills/evolve_skill.py:156-177`

```python
try:
    optimizer = dspy.GEPA(
        metric=skill_fitness_metric,
        max_steps=iterations,
    )
    optimized_module = optimizer.compile(...)
except Exception as e:
    # GEPA 如果当前 DSPy 版本不支持，就 fallback 到 MIPROv2
    optimizer = dspy.MIPROv2(
        metric=skill_fitness_metric,
        auto="light",
    )
    optimized_module = optimizer.compile(...)
```

但 `generate_report.py` 明确说 Phase 1 用的是 **BootstrapFewShot**（比 MIPROv2 更简单的 optimizer）：

> "Optimizer: DSPy BootstrapFewShot"

BootstrapFewShot 的工作原理：
1. 用 baseline skill 跑训练集
2. 收集得分高的 execution traces
3. 选出最好的 N 个作为 few-shot demonstrations
4. 把这些 demonstrations 注入到优化后模块的 prompt 里

所以 Phase 1 的改进主要来自：**没改写 skill 本身，而是给 skill 配上了成功的执行示例**。

---

### 3.5 约束校验器 (ConstraintValidator)

文件：`evolution/core/constraints.py`

| 约束 | 规则 | 失败后果 |
|------|------|----------|
| `size_limit` | skill ≤ 15KB，tool desc ≤ 500 chars | 拒绝部署，但保存到 `evolved_FAILED.md` |
| `growth_limit` | 变异后长度 ≤ baseline +20% | 同上 |
| `non_empty` | 不能为空 | 同上 |
| `skill_structure` | 必须有 `---` frontmatter，且含 `name:` 和 `description:` | 同上 |
| `test_suite` | hermes-agent 的 pytest 100% 通过 | 可选 |

---

## 四、Phase 1 实验配置与结果

来自 `generate_report.py`：

| 参数 | 值 |
|------|-----|
| Target Skill | arxiv (arXiv 论文搜索) |
| Skill Size | 10,175 chars |
| Model | MiniMax M2.5 via OpenRouter |
| Optimizer | BootstrapFewShot |
| Training Examples | 3 (从 7 条 synthetic 中选) |
| Validation Examples | 2 (held-out) |
| Max Bootstrapped Demos | 2 |
| Optimization Rounds | 1 |
| Total Time | < 60 seconds |
| Estimated Cost | < $0.50 |

### 结果

| Metric | Baseline | Optimized | Change |
|--------|----------|-----------|--------|
| Validation Example 1 | 0.408 | 0.569 | +39.5% |
| Validation Example 2 | 0.374 | 0.374 | 0.0% |
| **Average** | **0.391** | **0.472** | **+20.7%** |

---

## 五、关键设计与取舍

### 5.1 为什么不直接优化模型权重？

Hermes 把 agent 能力拆成三层：
1. **Model Weights** —— 用 RL 训练（Tinker-Atropos），成本高、需要 GPU
2. **Instructions** —— skill / prompt / tool description，纯文本，LLM 可直接变异
3. **Tool Code** —— Python 实现，需要手动或代码级进化

Self-evolution 瞄准的是第 2 层（Instructions），因为：
- 纯文本，变异容易
- 改变立即可部署
- 效果可直接用 eval dataset 测量
- 不需要 GPU，全靠 API 调用

### 5.2 Fitness 信号的设计意图

设计了两层 fitness：
- **Fast heuristic**（keyword overlap）：优化循环中快速筛选，省 API 钱
- **LLM-as-judge**（多维 rubric）：最终精准评估，同时也是 GEPA reflective mutation 的反馈来源

这种分层是典型的"粗筛 + 精筛"策略。

### 5.3 三个优化引擎的分工

| 引擎 | 优化对象 | 许可证 | 角色 |
|------|----------|--------|------|
| DSPy + GEPA | Skill, prompt, tool desc | MIT | **主优化器** |
| DSPy MIPROv2 | Few-shot, instruction text | MIT | Fallback |
| Darwinian Evolver | Code files, algorithms | AGPL v3 | Phase 4 代码进化 |

---

## 六、待验证与下一步（来自代码与报告）

1. **真正跑 GEPA**：Phase 1 因为 GEPA 可能还没集成到当前 DSPy 版本，实际用了 BootstrapFewShot。GEPA 声称比 MIPROv2 高 +10%，比 GRPO 高 +6% 且 rollout 少 35 倍。
2. **替换 heuristic scorer 为 LLM-as-judge**：报告中把这点列为 Immediate Next Step 第 3 项。
3. **Benchmark gating**：TBLite / YC-Bench 目前只是"Planned"，不参与 fitness，只作为 regression gate。
4. **PR 自动化**：目前输出到本地，还没有自动提 PR 到 hermes-agent。

---

## 七、总结：从 coding agent 视角能借鉴什么？

| 可借鉴点 | 说明 |
|----------|------|
| **把 skill 当可优化参数** | SKILL.md 不是静态文档，而是 DSPy module 的参数，这个抽象很干净 |
| **合成 eval dataset** | 用 LLM 自动生成 (task, rubric) 对，解决了"没数据怎么优化"的冷启动问题 |
| **分层 fitness** | 快速 heuristic 用于优化循环，昂贵 LLM-as-judge 用于精评和 mutation 反馈 |
| **硬约束守门** | size / growth / structure / pytest 多道关卡，防止进化出怪物 |
| **保留 frontmatter 只进化 body** | 元数据不变，只改内容，避免破坏文件格式 |

| 需谨慎点 | 说明 |
|----------|------|
| **Phase 1 结果可能不能代表 GEPA 效果** | 实际 optimizer 是 BootstrapFewShot，改进来自加 few-shot demo，不是 skill 文本被重写 |
| **Heuristic scorer 过于粗糙** | Keyword overlap 不能捕捉语义正确性，可能选出"堆砌关键词"的变体 |
| **Synthetic eval 的真实性问题** | 自己生成的题目自己考，存在 overfitting 风险，需要 golden set 或真实 benchmark 把关 |
