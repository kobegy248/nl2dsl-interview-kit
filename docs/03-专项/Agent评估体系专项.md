# Agent 评估体系专项（已实现）

> **证据边界**：本文只讲**已实现**的评测能力（V1/V2 打分、真实链路执行、回归门禁、运行时 LLM 裁判），有 `file:line` 证据。
> 「LLM-as-Judge 校准」「统计显著性」「检索层指标」等**未实现**内容，见 [评估方法论升级设计题](../04-设计题/评估方法论升级设计题.md)。

---

## 0. 三句话（开场口径）

1. **我的评测框架里没有 LLM-as-judge，这是刻意的**——能确定性判定的，绝不交给模型判。
2. **但项目里有"LLM 当裁判"**，只是位置不同：它跑在**运行时**（`confidence` 节点、`verify_dsl` 节点），不参与离线打分。
3. **核心方法是 canonical 规范化 + 二元判定**：先消除表面差异，再判等价/不等价。

---

## 一、评测体系分两代

| | V1（`evaluation/scoring.py`） | V2（`evaluation/v2_*.py`） |
|---|---|---|
| 定位 | 端到端黑盒评测（调真实 API） | 语义理解基准（纯内存比对） |
| 维度 | 4 类 12 维 | 5 维（intent / metric / filter / planner / governance） |
| 评分 | 加权综合分，**允许部分分** | **二元 0/1，无部分分** |
| 速度 | 慢（要建库、跑 SQL） | 快（不连库） |
| 稳定性 | 脆（数据一变就挂） | 稳（只比 DSL 语义） |
| 用途 | 发布前全链路验收 | 日常迭代、CI 回归门禁 |

V1 的 12 维权重（`scoring.py:14-31`，合计 1.0）：

| 类别 | 维度 | 权重 |
|---|---|---|
| Semantic 56% | intent / metric / dimension / filter | 8% / 20% / 12% / 16% |
| Planning 14% | join / limit / order_by | 7% / 4% / 3% |
| Execution 20% | sql_success / result_accuracy | 10% / 10% |
| Governance 10% | permission / masking / audit | 4% / 3% / 3% |

V2 权重（`v2_runner.py:27-33`）：`intent 0.20 / metric 0.30 / filter 0.20 / planner 0.20 / governance 0.10`，且**动态归一化**——只对本次用例实际参与的维度重新分配，未涉及的维度记 N/A 而非 0 分。

---

## 二、V2 核心：Canonical Resolver（先规范化再比对）

六个解析器（`evaluation/canonical/`）：

| 解析器 | 做什么 | 例子 |
|---|---|---|
| metric | alias / field+func → metric_id | `sales_amount` ≡ `SUM(pay_amount)` |
| dimension | 维度别名 → 物理列 | `region` ≡ `region_code` |
| value | 业务值 → 编码值 | `华东` ≡ `HD` |
| time | 自然语言时间 → 范围+粒度 | `2024年` ≡ `["2024-01-01","2024-12-31"]` |
| join | 表名 → 业务实体 | `customer_dim` ≡ `c` |
| order | 排序归一化 | 单 dict / 列表 / 字符串 → `(field, direction)` |

**等价边界是显式定义的**：

| 属于语义等价 | 不属于语义等价 |
|---|---|
| metric alias 不同 | 不同指标（GMV vs SalesAmount） |
| 维度别名不同 | 不同业务概念（Revenue vs Profit） |
| 值编码不同（华东/HD） | 不同过滤值（华东 vs 华南） |
| 时间写法不同 | 不同时间范围（2024年 vs Q1） |
| `between [a,b]` ≡ `>=a and <=b` | 不同聚合函数（SUM vs AVG） |

---

## 三、评分器：全部 0/1

`scorers/base.py` 的契约：

```python
class Scorer(ABC):
    """二元评估：1.0（通过）或 0.0（不通过）。不存在部分得分。"""
```

**为什么二元**（设计文档 §10.2 原话）：

> 部分分（0.8、0.9）主观性太强："alias 对了但 field 错了" → 0.8？这取决于 alias 和 field 哪个更重要。Canonical 化后，要么等价（100），要么不等价（0），判定标准客观。

唯一的"内部平均"是 `planner_scorer`：维度 / limit / order / joins / time_range / post_process 各判一次 0/1，再取平均。

---

## 四、真实链路评测：严禁拿 expected 造 actual

`evaluation/execution.py` 写死：

> **严禁从 `expected` 构造 actual DSL。**
> `llm` 模式下若没有可用 LLM Client，返回 `unavailable`，**不静默退化**到规则生成。

统一观测对象 `EvaluationObservation`：

```python
status                # success | warning | clarification | error | unavailable
dsl_before_optimizer  # 优化前
dsl_after_optimizer   # 优化后（评分用这个）
sql / data / trace
injected_filters      # 本次注入的租户/行级过滤
```

三个保护：

1. **不可评分状态不崩**：`error/clarification/unavailable` → 全 0 判不通过，报告照常产出。
2. **治理注入的过滤条件评分前剥离**（`_strip_injected_filters`）：否则 `tenant_id` 会被误判成"语义过滤错"。
3. **矩阵评测**：`generator(rule|llm) × optimizer(on|off)` 四组合，每种独立环境。

---

## 五、回归门禁：baseline + 三重身份校验

`evaluation/baseline.py` `evaluate_regression()`：

| 机制 | 做法 |
|---|---|
| 身份校验（fail-closed） | `schema_version` / `dataset_hash` / `matrix_combos` 任一缺失或不等 → 直接判失败，不比分数 |
| 数据集指纹 | 对用例集合算 hash，改一条 expected 就换 hash |
| 维度阈值 | 任一维度下降 > `0.02` → 回归 |
| 用例阈值 | 单用例下降 > `0.10` → 回归 |
| CI 集成 | `--fail-on-regression` → 有回退时非零退出 |

> 身份校验的意义：**换了数据集还比分数 = 假回归/假提升**。很多团队的基线就是这么烂掉的。

---

## 六、运行时的 LLM 裁判（关键：不在评测里）

### 6.1 `confidence` 节点（`agent/confidence.py`）

```
syntax   = validator.validate(dsl) → 1.0/0.0        （规则）
semantic = LLM 判断 DSL 是否回答问题 → 0~1          （LLM 裁判）
history  = 1.0（未实现）

confidence = min(syntax, semantic) * history
```

| 分数 | 路由 |
|---|---|
| ≥ 0.8 | continue |
| 0.6~0.79 | warning |
| < 0.6 | clarify |

三个工程细节：

- **LLM 不可用/解析失败 → 中性 0.5 且不阻断**（`neutral_*` 直接 continue），不能因 LLM 抖动误拦用户；
- **子查询模式降级**：Agent 派发的子查询即使低置信度，也从 clarify 降为 warning；
- **结果可解释**：输出 syntax / semantic / history / 路由四段。

### 6.2 `verify_dsl` 节点（`graph/nodes.py:1360`）

执行后让 LLM 自检"DSL 和结果是否回答了原始问题"：输入原始问题 + 最终 DSL + 结果前 3 行；输出 `PASS / WARN / FAIL` + 一句原因；**warning-only，不阻断**。

### 6.3 为什么这不叫"评测"

| 维度 | 评测框架（V2） | 运行时裁判 |
|---|---|---|
| 何时跑 | 离线、CI | 每个线上请求 |
| 判定对象 | expected vs actual DSL | 问题 vs DSL/结果 |
| 输出 | 0/1 分数 | 置信度 / PASS-WARN-FAIL |
| 可复现 | 完全可复现 | 不可复现（模型/温度/版本） |

> 口径：**"评测要可复现、可比较、可门禁，所以绝不让模型参与；运行时需要覆盖结构化比对覆盖不到的语义合理性，才用 LLM 兜底，而且只告警不判死。"**

---

## 七、这个实现的已知边界

| 边界 | 说明 |
|---|---|
| 无置信区间 | 只报点估计，小样本波动大（10 例三次跑出 72.8/86.9/89.8） |
| 无显著性检验 | 版本对比只看均值差 |
| 无检索层指标 | 只评端到端 |
| 无 judge 校准 | 运行时的 LLM 打分没验过一致性 |
| masking 评测逻辑有 bug | 用"原始值≠脱敏值"判已脱敏，而生产链路 `mask()` 未被调用 |

> 前四条是"没做"，最后一条是"做错了"——性质不同，都要主动说。

---

## 八、相关追问（只回答"你怎么做的"）

**Q1：你们用 LLM-as-judge 吗？**
> "评测框架里不用，刻意的——核心维度都能 canonical 化，交给模型反而不可复现。但运行时有两个 LLM 裁判：confidence 判 DSL 质量决定是否追问，verify_dsl 执行后自检。它们只告警、不阻断。"

**Q2：为什么二元评分？**
> "部分分权重是主观的。'alias 对了 field 错了'给 0.8 是因为你觉得 alias 更重要，换个人就是 0.5。canonical 化后要么等价要么不等价，而且便于做门禁——回退就是回退，没有'轻微回退'的扯皮。"

**Q3：怎么防"假评测"？**
> "三道：actual 必须来自真实链路，不许从 expected 造；LLM 不可用返回 unavailable 不静默降级；治理注入的过滤条件评分前剥离，避免误判。"

**Q4：回归门禁怎么做？**
> "baseline + 三重身份校验（schema/dataset_hash/matrix）+ 维度 0.02、用例 0.10 阈值 + 非零退出。身份不一致直接判失败，不比较分数。"

**Q5：为什么 V2 不测 SQL？**
> "SQL 执行测试慢、脆、依赖数据、不能换库；语义测试纯内存、快、稳定、可换库。SQL 执行保留为 V1 的低频全链路验收。"

---

## 九、速查

| 项 | 值 |
|---|---|
| 评测框架用 LLM-judge 吗 | 不用，刻意不用 |
| 哪里用了 LLM 裁判 | confidence（质量分）、verify_dsl（PASS/WARN/FAIL） |
| 核心方法 | Canonical 规范化 + 二元判定 |
| V1 / V2 | 4 类 12 维加权 / 5 维二元 + 动态权重 |
| 真实链路 | 禁止从 expected 造 actual；unavailable 不降级 |
| 门禁 | dataset_hash + matrix 三重校验 + 阈值 + 非零退出 |
| 已知边界 | 无置信区间 / 无显著性 / 无检索指标 / masking 评测有 bug |
| 下一步 | 见 [评估方法论升级设计题](../04-设计题/评估方法论升级设计题.md) |
