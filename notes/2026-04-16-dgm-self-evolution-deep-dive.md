# DGM Self-Evolution 深度拆解

> 分析对象：`dgm/`（jennyzzt/dgm, Darwin Gödel Machine）
> 分析日期：2026-04-16

---

## 一、项目定位

DGM 的自进化对象不是 skill，也不是 prompt 片段，而是 **agent 自己的代码仓**：

- `coding_agent.py`
- `tools/*.py`
- `utils/*.py`

它的核心闭环是：

1. 用当前 agent 跑 benchmark
2. 读取失败日志、预测 patch、官方私测结果
3. 让强模型提出一个“如何改 agent 自己”的 GitHub issue
4. 再让 coding agent 去实现这个 issue
5. 把新补丁重新放回 benchmark 上验
6. 把通过最低门槛的子代放进 archive，继续繁殖

因此 DGM 是一个 **benchmark-driven self-code-modification loop**。

---

## 二、完整流程图

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                           DGM Self-Improvement Loop                          │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 0: 初始化 archive                                                       │
│ ──────────────────────────────────────────────────────────────────────────── │
│ DGM_outer.initialize_run()                                                  │
│   - 新跑：archive = ['initial']                                             │
│   - 续跑：读取上一轮 dgm_metadata.jsonl 的最后一个 archive                   │
│   - 把缓存好的 initial/ 或 initial_polyglot/ 拷进 output_dgm/<run_id>/     │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: 从 archive 里挑 parent + 待分析 entry                                │
│ ──────────────────────────────────────────────────────────────────────────── │
│ choose_selfimproves()                                                       │
│   1. 读取 archive 每个节点 metadata.json                                    │
│   2. 收集 accuracy / unresolved_ids / empty_patch_ids / children_count      │
│   3. 按策略采 parent：                                                       │
│      - random                                                                │
│      - score_prop                                                            │
│      - score_child_prop (默认)                                               │
│      - no_darwin baseline                                                    │
│   4. 对每个 parent 决定本轮要“修什么”：                                      │
│      - 某个 unresolved benchmark entry                                      │
│      - solve_empty_patches 模板                                              │
│      - solve_stochasticity 模板                                              │
│      - solve_contextlength 模板                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: 在新 Docker 容器里重建 parent agent                                   │
│ ──────────────────────────────────────────────────────────────────────────── │
│ self_improve_step.self_improve()                                            │
│   1. 启 dgm 容器                                                             │
│   2. 若 polyglot：替换 coding_agent.py / 删除 SWE 专属工具                   │
│   3. 沿 parent_commit 链取出全部 ancestor patch                             │
│   4. 依次 patch 到 /dgm                                                     │
│   5. 做一个临时 git commit，记为 base_commit                                │
│   6. pip install -r requirements.txt                                        │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 3: 诊断“当前 agent 哪里该进化”                                           │
│ ──────────────────────────────────────────────────────────────────────────── │
│ diagnose_problem()                                                          │
│   system message:                                                            │
│     - coding agent 概述                                                     │
│     - 当前 dgm 代码全文（coding_agent.py + tools/ + utils/ + 历史 patch）    │
│   user message:                                                              │
│     - agent 运行日志 md                                                     │
│     - benchmark 任务描述 / GitHub issue                                     │
│     - agent 预测 patch                                                      │
│     - benchmark 私有 test patch                                             │
│     - benchmark 测试结果                                                    │
│   输出 JSON → 组装成 problem_statement（像一个 GitHub issue）                │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 4: 让 coding agent 改自己的代码                                          │
│ ──────────────────────────────────────────────────────────────────────────── │
│ 在容器中执行：                                                               │
│   timeout 1800 python /dgm/coding_agent.py                                   │
│      --problem_statement "<自进化 issue>"                                    │
│      --git_dir /dgm/                                                         │
│      --base_commit <临时 commit>                                             │
│      --self_improve                                                          │
│                                                                              │
│ coding_agent.py 最终输出：                                                   │
│   /dgm/model_patch.diff                                                      │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 5: 把新 patch 拿回宿主机并评测                                           │
│ ──────────────────────────────────────────────────────────────────────────── │
│ 复制回：                                                                     │
│   - self_evo.md                                                              │
│   - model_patch.diff                                                         │
│                                                                              │
│ patch 不为空才继续：                                                         │
│   - 跑 SWE-bench 或 Polyglot harness                                         │
│   - small subset                                                            │
│   - 若达到阈值，再跑 medium subset                                           │
│   - 聚合 overall_performance                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 6: 过滤“可进入 archive 的子代”                                           │
│ ──────────────────────────────────────────────────────────────────────────── │
│ filter_compiled() + is_compiled_self_improve()                              │
│   不是编译器意义上的 compiled，而是：                                         │
│   - overall_performance 有关键字段                                            │
│   - 不是空 patch                                                             │
│   - 评测提交数足够                                                            │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 7: 更新 archive                                                         │
│ ──────────────────────────────────────────────────────────────────────────── │
│ update_archive()                                                             │
│   - 默认 keep_all：所有通过最低门槛的子代都入 archive                        │
│   - 记录到 dgm_metadata.jsonl：                                              │
│       generation                                                             │
│       selfimprove_entries                                                    │
│       children                                                               │
│       children_compiled                                                      │
│       archive                                                                │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 8: 下一代继续从 archive 里抽 parent                                      │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、archive 机制：它如何表示“进化树”

### 3.1 节点不是分支，也不是完整 checkout，而是 patch stack

每个 run 目录里保存：

- `metadata.json`
- `model_patch.diff`
- 评测产物

真正重建某个 parent 时，不是 checkout 某个 git branch，而是：

1. 从 `parent_commit` 一路往上追溯
2. 用 `get_model_patch_paths()` 收集所有祖先 patch
3. 在新容器里从原始 `/dgm` 依次 apply 这些 patch

所以 archive 的语义是：

> 节点 = 一串按 ancestry 顺序叠加起来的 patch

这比存整个代码树省空间，但也意味着：

- 祖先 patch 一旦损坏，后代都受影响
- 没有真正的 per-node git commit checkout

### 3.2 默认 parent 采样策略：`score_child_prop`

每个候选 parent 会计算：

- `score = sigmoid(10 * (accuracy - 0.5))`
- `child_penalty = 1 / (1 + children_count)`
- 采样概率 ∝ `score * child_penalty`

这相当于同时偏好：

- 分数更高的 parent
- 子代较少、还没被过度开发的 parent

作者想实现的是一种很轻量的“探索 / 利用”平衡。

### 3.3 选 entry 的方式不是“找最难题”，而是从失败桶里抽样

对 SWE 模式，优先从：

- `empty_patch_ids`
- `unresolved_ids`

里选。

另外还有三种“元问题模板”：

- `solve_empty_patches`
- `solve_stochasticity`
- `solve_contextlength`

也就是说，DGM 不是只盯 benchmark 题本身，还会把 **agent 基础设施问题** 当成可进化目标。

---

## 四、`diagnose_problem()`：真正关键的不是日志，而是 system message

### 4.1 DGM 最核心的 prompt 结构

`diagnose_problem()` 调用 `get_diagnose_prompt_swe()` / `..._polyglot()`，产出两部分：

#### system message

包含：

- coding agent 总结
- 当前 agent 代码全文
  - `coding_agent.py`
  - `tools/`
  - `utils/`
  - 以及已有历史 patch

排除若干文件：

- `utils/evo_utils.py`
- `utils/docker_utils.py`
- `utils/swe_log_parsers.py`
- `utils/eval_utils.py`（polyglot 分支）
- `prompts/self_improvement_prompt.py`

#### user message

包含：

- agent 运行日志 `.md`
- benchmark 任务描述 / GitHub issue
- agent 预测 patch
- benchmark 私测 patch
- benchmark 测试日志

### 4.2 关键点：诊断模型看到的是“完整当前 agent 实现”

这和很多 self-reflection 系统很不一样。

DGM 不是简单让模型说：

> “看起来这次失败了，给几个建议。”

而是让模型同时看到：

- agent 真实代码
- agent 真实行为
- benchmark 的裁判结果

因此 `problem_description` 不是泛泛而谈，而是一个面向当前代码库的改造 issue。

### 4.3 日志处理方式非常粗暴：250K 硬截断

`process_selfimprove_eval_logs()` 里：

```python
if len(md_log) > 250000:
    md_log = md_log[:250000] + "\n<log clipped>"
```

没有：

- priority truncation
- 按 tool error 保序
- 专门保最终轮次

所以 DGM 的反思质量很依赖日志前部信息是否足够。

### 4.4 输出 JSON 的真正用途

诊断模型输出字段包括：

- `log_summarization`
- `potential_improvements`
- `improvement_proposal`
- `implementation_suggestion`
- `problem_description`

但真正下游被 coding agent 消化的是：

```text
coding_agent_summary + 
"# To Implement" +
implementation_suggestion +
problem_description
```

换句话说，前面几个字段只是中间解释，**最后喂给 coding agent 的仍然是“实现任务说明书”**。

---

## 五、coding agent 的真实行为，比 README 想象得更简单

### 5.1 `coding_agent.py` 的 `forward()` 非常薄

主流程其实只有：

1. 拼一段 instruction
2. 调 `chat_with_agent(...)`
3. 结束后直接 `diff_versus_commit(...)` 写成 `model_patch.diff`

并没有在 `forward()` 里实现：

- 显式 test-debug loop
- 多候选 patch 比较
- regression test 驱动的 rerank
- 失败后自动 retry

### 5.2 真正的 agent loop 在 `llm_withtools.py`

`chat_with_agent()` 做的是：

- 给模型注册 tools
- 用模型原生 tool calling
- 只要模型继续发 tool call，就继续执行
- 直到模型停止调工具，保存最终消息历史

支持：

- Claude tool use
- OpenAI Responses API function call
- 手写 `<tool_use>` fallback

这意味着 DGM 的“能不能改好自己”很大程度依赖：

- 模型自己的工具使用能力
- `tools/` 目录里工具描述写得够不够好

### 5.3 SWE 版本与 Polyglot 版本并不对称

SWE 版 `coding_agent.py` 里：

- `self_improve=True` 并不会切到 `OPENAI_MODEL`
- `self.code_model` 直接固定成 `CLAUDE_MODEL`

Polyglot 版 `coding_agent_polyglot.py` 才写了：

- 非 self-improve 用 Claude
- self-improve 用 `o3-mini-2025-01-31`

所以仓库里实际存在 **两个不同的自进化 agent 配置**。

### 5.4 仓库里有一些“看起来要用但没接上”的能力

SWE 版 `coding_agent.py` 里定义了：

- `get_regression_tests()`
- `run_regression_tests()`

并 import 了：

- `get_report_score`
- `score_tie_breaker`

但 `forward()` 没有调用它们。

这说明当前代码更像“最小可跑版本”，而不是把文中很多高级策略都真正接上了。

---

## 六、评测编排：实际跑的是 small → medium，full/big 没接通

### 6.1 当前真正生效的层级

在 outer loop 中：

- small subset 一定会跑
- 达到阈值后会跑 medium subset

阈值由：

- `test_more_threshold = 0.4`

控制，也就是 small 上 resolved 数达到 `len(small) * 0.4` 才扩到 medium。

### 6.2 `full_eval_threshold` 看起来存在，实际上没用上

`DGM_outer.py` 确实计算了：

- `get_full_eval_threshold()`

并把 `full_eval_threshold` 传进 `self_improve()`。

但在 `self_improve_step.py` 里：

- `full_eval_threshold` 没有被使用
- `test_task_list_big = load_json_file("./swe_bench/subsets/big.json")` 也加载了但没用

所以按当前仓库代码，**full/big eval 并没有真正接进 self-improve 主链路**。

### 6.3 “compiled” 不是编译成功，而是“评测记录完整且不是空 patch”

`is_compiled_self_improve()` 检查的是：

- `overall_performance` 有关键字段
- 至少有一个非空 patch 结果
- 提交的 benchmark 数量达到预期

只要满足这些，节点就会被认为“compiled”并可进 archive。

所以这里的 compiled 更接近：

> 这个子代的评测元数据足够完整，可以纳入进化池

---

## 七、archive 更新策略其实非常宽松

### 7.1 默认 `keep_all`

只要通过最低门槛，全部进 archive。

这意味着 archive 很容易膨胀成：

- 高分节点
- 中分节点
- 只是“能跑完”的节点

都混在一起。

### 7.2 `keep_better` 也不是只保留更优于 parent

`update_archive(..., method='keep_better')` 的比较基准是：

- `initial score - noise_leeway`

而不是：

- 父节点分数
- 当前 archive 前沿
- Pareto frontier

所以即便开启 `keep_better`，标准也依然相当松。

这说明 DGM 更像 **开放式实验 archive**，而不是精确维护一个 elite set。

---

## 八、代码级重要发现：当前实现和“理想算法”有几处偏差

这一节最值得保留，因为它们不是 README 能看出来的，只有真读代码才能发现。

### 8.1 `best` 采样策略实际上会挑到最低分 parent

`choose_selfimproves()` 里：

```python
sorted_commits = sorted(candidates, key=lambda x: candidates[x]['accuracy_score'])
parent_commits = sorted_commits[:min(selfimprove_size, len(sorted_commits))]
```

这里没有 `reverse=True`，所以 `best` 实际拿到的是 **最低分** 节点。

### 8.2 argparse 的 `choices` 有拼接 bug

```python
choices=['random', 'score_prop', 'score_child_prop' 'best']
```

Python 会把相邻字符串字面量拼成：

```python
'score_child_propbest'
```

也就是说，这里的 choices 并不是作者想写的四个选项。

### 8.3 `if unresolved_ids == 0` 是无效判断

`unresolved_ids` 是列表，不是整数。

所以这句：

```python
if unresolved_ids == 0:
    continue
```

基本不会按预期工作。后面如果 `unresolved_ids` 恰好为空，`random.choice(entry_ids)` 会直接炸。

### 8.4 `diagnose_improvement_prompt.py` 当前基本是坏的

这里有一个很实在的实现错误：

- `find_selfimprove_eval_logs()` 返回 4 个值
- `get_diagnose_improvement_prompt()` 却按 3 个值解包
- 后面又用错了 `process_selfimprove_eval_logs()` 的参数个数

结果是 post-improvement diagnosis 这条链路大概率会在构造 prompt 时直接异常，然后不断 retry，最后返回 `None`。

所以当前仓库里的 “post improve diagnose” 更像半成品。

### 8.5 full eval 相关变量是死参数

前面提过，但值得再强调一次：

- `full_eval_threshold` 传了，没用
- `test_task_list_big` 读了，没跑

这会让“分阶段跑到 full benchmark”在当前代码里变成名义功能，而不是实际功能。

---

## 九、DGM 的真正强点在哪里

### 9.1 它把“改自己”转成了标准软件工程任务

最妙的设计是这一步：

> 先让强模型看代码 + 看失败日志 + 看 benchmark 裁判，再把结论翻译成一个 GitHub issue。

这样下游 coding agent 不需要懂“自我反思理论”，它只是在做一个普通的工程实现任务。

### 9.2 它不是只看 patch 好不好，而是拿 benchmark 当唯一裁判

新 patch 是否进 archive，不看：

- patch 看起来是否优雅
- 反思文本是否自洽
- LLM 是否说“这个改进很好”

而看：

- benchmark resolved / unresolved / empty patch 结果

这是 DGM 最硬核的部分。

### 9.3 patch stack 让 ancestry 很清晰

每个子代只保存自己的新增 diff，重建时再叠祖先 patch。

优点：

- 节点存储小
- ancestry 明确
- 便于分析“哪代引入了什么改动”

缺点则是我前面提到的：祖先 patch 的可应用性成了系统隐含假设。

---

## 十、DGM 的主要局限

### 10.1 反思质量严重依赖单条日志前 250K 字符

一旦关键失败信号在后面，诊断模型就看不到。

相比 OpenSpace 的优先级截断，DGM 这里明显更粗糙。

### 10.2 当前仓库里的 coding agent 太薄

理论上 DGM 应该是：

- 诊断 agent 很强
- 执行 agent 也有多轮测试、比较、反思

但当前 checked-in `coding_agent.py` 只做了一层 tool-calling agent 封装，很多更强的执行策略并没真正接上。

### 10.3 archive 维护偏“实验记录”，不是严格进化前沿

默认 `keep_all`，再加上 compiled 判定宽松，意味着 archive 会越来越像：

- 一堆可以复现的实验节点

而不是：

- 严格受控的最优族群

### 10.4 当前实现里存在多处细节 bug

这会直接影响：

- parent 采样是否符合预期
- benchmark 深评测是否真的发生
- post-hoc diagnosis 是否可用

所以研究 DGM 时，不能只看 paper / README，必须以代码为准。

---

## 十一、一句话结论

DGM 是这三个项目里 **最像“让 agent 直接改自己代码”** 的实现：它把 benchmark 失败转成下一轮 agent 改造任务，再用 benchmark 反向筛选子代。

但从当前仓库代码来看，它更像一个非常有启发性的研究原型，而不是已经打磨成产品级闭环的系统。它最值得学的，是：

- 用 benchmark 充当唯一外部裁判
- 用强模型把失败日志翻译成工程 issue
- 用 patch stack 维护 ancestry

而最需要警惕的，是当前实现里几处会悄悄改变实验语义的代码偏差。
