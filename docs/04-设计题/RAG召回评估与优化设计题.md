# RAG 召回评估与优化设计题（未实现）

> **证据边界**：本文全部是**设计提案**，NL2DSL 项目里**没有实现**。
> 面试口径必须是「我们没做，但如果要做我会这样设计」，不要说成"我做过"。
> 已实现部分见 [RAG 检索链专项](../03-专项/RAG检索链专项.md)。

---

## 0. 定位

一句话：**"我的检索链能跑通，但我没有量化过它好不好。这份是补量化的方案。"**

现有能力：四集合召回 + 精排 + 阈值 + 回退（已实现）。
缺口：检索层没有 `Recall@K` / `MRR` / `nDCG`，没有分通道归因，阈值是拍的。

---

## 一、缺口清单

| 缺口 | 后果 |
|---|---|
| 无检索层指标 | 只知道端到端 DSL 对不对，不知道是"没召回"还是"召回了没用对" |
| 无分通道对比 | 不知道关键词通道和向量通道各贡献多少 |
| 阈值未校准 | 0.5 是拍的，可能长期走回退分支 |
| 无标注集 | 想改进也没有基线可对比 |
| 无精排前后对比 | 无法证明 rerank 真的有增益 |

---

## 二、未采用的通用技术

### 2.1 检索评估

| 技术 | 说明 |
|---|---|
| **Recall@K** | 前 K 条里覆盖了多少应该召回的内容 |
| **MRR** | 第一个正确结果排名的倒数均值 |
| **nDCG@K** | 考虑排序位置的加权命中率 |
| 分通道归因 | 关键词 / 向量 / 融合 / 精排后各算一次 |
| 标注集 | query → 应该召回哪些 name |

### 2.2 融合

| 技术 | 说明 |
|---|---|
| **RRF** | 按排名倒数加权融合，无需调权重，天然处理分数不可比 |
| 加权分数融合 | 两路各给权重（需要标注调参） |
| MMR | 提升结果多样性，避免召回 5 条几乎一样 |

### 2.3 查询侧

| 技术 | 说明 |
|---|---|
| Query 改写 | 术语表驱动，把口语改写成标准指标名 |
| Query 扩展 | 一个 query 生成多个变体并行检索 |
| HyDE | 先让 LLM 写假设答案，再用答案去检索 |
| 指代消解 | 多轮场景，把"那华南呢"补全 |

### 2.4 索引侧

| 技术 | 说明 |
|---|---|
| 父子块切分 | 文档型知识库才需要（当前是结构化配置，不需要） |
| 元数据过滤 | 检索前按 domain/租户/时间过滤 |
| 软删除 / 全量 diff | 解决"只增不删" |
| 批量 embedding | 关键词通道从 N 次调用改成 1 次 batch |

### 2.5 向量库

| 技术 | 说明 |
|---|---|
| Milvus Server / PGVector / ES | 生产级：副本、并发、倒排混合检索 |

---

## 三、怎么补

### 3.1 先诊断，别急着加技术

```
① 建 50 条标注（query → 应该召回的 name，按集合分组）
② 算当前各集合 Recall@5 / MRR
③ 看最弱的是哪个集合、哪类 query
④ 针对性改一处 → 重跑 → 对比
```

> 面试口径：**"检索链有四个集合、两条通道、一层精排，盲改等于赌博。我先建标注集算基线，再决定改哪。"**

### 3.2 方案 A：最小可用（1~2 天）

新增 `nl2dsl/evaluation/retrieval/`：

```
retrieval/
├── dataset.py      # 读标注 YAML
├── metrics.py      # recall_at_k / mrr / ndcg_at_k
└── runner.py       # 调 RAGRetriever，按集合算指标出报告
```

标注格式：

```yaml
cases:
  - id: r001
    query: "各品牌的流水"
    expected:
      metrics: [gmv]
      terms: [GMV]
      schema: [brand]
```

指标实现（约 20 行）：

```python
def recall_at_k(ranked: list[str], relevant: set[str], k: int) -> float:
    if not relevant:
        return 1.0
    return sum(1 for n in ranked[:k] if n in relevant) / len(relevant)

def mrr(ranked: list[str], relevant: set[str]) -> float:
    for i, n in enumerate(ranked, 1):
        if n in relevant:
            return 1.0 / i
    return 0.0

def ndcg_at_k(ranked, relevant, k):
    dcg = sum(1.0 / math.log2(i + 1) for i, n in enumerate(ranked[:k], 1) if n in relevant)
    idcg = sum(1.0 / math.log2(i + 1) for i in range(1, min(len(relevant), k) + 1))
    return dcg / idcg if idcg else 0.0
```

**分通道对比**（最有价值）：

```python
report = {
    "keyword_only":  {...},   # 只走关键词通道
    "vector_only":   {...},   # 只走向量通道
    "hybrid_rerank": {...},   # 最终链路
}
```

### 3.3 方案 B：两个最值钱的改进（1 周）

**1）RRF 替代硬拼接**

```python
def rrf_fuse(ranked_lists: list[list[str]], k: int = 60) -> list[str]:
    scores: dict[str, float] = {}
    for ranked in ranked_lists:
        for rank, name in enumerate(ranked, 1):
            scores[name] = scores.get(name, 0.0) + 1.0 / (k + rank)
    return [n for n, _ in sorted(scores.items(), key=lambda x: -x[1])]
```

现状是"关键词优先 + 语义补充"的硬拼接，**排序信息在合并时丢失**（第 1 名和第 9 名没区别）。RRF 正好解决。

**2）阈值分布校准**

跑全部标注集，把 `rerank_score` 画直方图，取 **P70 分位数**当阈值；同时把"走回退分支的比例"作为监控——回退率过高说明阈值不合理。

### 3.4 方案 C：完整（1~2 月）

1. 检索评估接进 CI：`--fail-on-regression` 同时检查 Recall@5（下降 > 0.03 即阻断）。
2. 分通道 + 分集合归因报告。
3. 标注集扩到 200 条（术语别名、英文缩写、多维度、时间表达、否定、歧义）。
4. 同步做全量 diff，清理已删除实体的向量。
5. 关键词通道批量 embedding。

---

## 四、设计类追问

**Q1：怎么评估检索效果？**
> "四层：检索层 Recall@K / MRR / nDCG；融合层对比单通道与融合后的增益；精排层对比精排前后 NDCG；端到端看 DSL 正确率。我现在只做了第四层。"

**Q2：混合检索两路分数怎么融合？**
> "我现在是硬拼接，简单但丢排序信息。更好的做法是 RRF：按排名倒数加权，不用调权重，对分数不可比的情况天然友好。"

**Q3：阈值怎么定？**
> "不该拍。拿标注集跑分数分布，取分位数（比如 P70），再把'空召回回退率'当监控指标——回退率高说明阈值定错了。"

**Q4：检索慢怎么优化？**
> "先看瓶颈：关键词通道现在每个词单独 embed，批量化是一步到位的优化；然后加语义缓存；最后才考虑换更小的 embedding 模型。"

**Q5：怎么证明 RAG 有用？**
> "做 A/B：关掉检索上下文 vs 打开，跑同一份评测集看 DSL 准确率差值。我的评测框架支持矩阵对比，RAG 可以做成一个开关参与比较。"

**Q6：知识库更新了怎么办？**
> "现在是 mtime 增量同步，只重跑变化文件。但有坑：删除实体不清理旧向量，会召回废弃字段。修法是对集合做全量 diff，或给记录加版本字段软删除。"

---

## 五、速查

| 问题 | 答案（未实现部分） |
|---|---|
| 检索层指标 | Recall@K / MRR / nDCG（无） |
| 融合方案 | RRF 替代硬拼接（无） |
| 阈值 | 应取分位数而非 0.5（未校准） |
| 标注集 | 50 条起步，目标 200 条（无） |
| 分通道归因 | 关键词 / 向量 / 融合 / 精排后各算一次（无） |
| 清理策略 | 全量 diff 或版本字段软删除（无） |
| 落地顺序 | 标注集 → 指标 → 分通道归因 → RRF → 阈值校准 |
| 一句话 | **"检索链我能讲清每层为什么这么设计；检索质量我还没量化——这是最大待补项。"** |
