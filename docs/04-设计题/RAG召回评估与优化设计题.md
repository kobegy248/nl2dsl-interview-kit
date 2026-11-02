# RAG 召回评估与优化设计题（未实现 · 教学版）

> **证据边界**：本文全部是**设计方案与教学**，NL2DSL 项目里**没有实现**这些评估能力。
> 面试口径必须是「我们没做，但如果要做我会这样设计」，不要说成"我做过"。
> 已实现的检索链见 [RAG 检索链专项](../03-专项/RAG检索链专项.md)。
>
> **这份文档怎么读**：每一节都按「是什么 → 为什么 → 公式 → 手算例子 → 代码 → 什么时候用 → 坑」的顺序讲，目标是看完能自己写一个检索评估脚本。

---

## 0. 先回答一个问题：为什么检索要单独评？

现在项目里能评的是**端到端**：问一句"各品牌的流水"，看最终 DSL 的 metric 是不是 `gmv`。

但端到端正确，**掩盖了两种完全不同的失败**：

```
失败 A（检索没召回）：RAG 根本没把 gmv 的指标定义放进上下文
                      → 模型不知道有 gmv 这个东西 → 只能猜，猜错了

失败 B（召回了没用对）：上下文里有 gmv，但模型没选它
                      → 这是生成/排序的问题，不是检索的问题
```

**这两者的修法完全不同**：A 要改召回（阈值、关键词词典、embedding），B 要改 prompt 或生成链路。如果你只有端到端分数，你永远不知道该修哪边。

> 一句话：**检索评估回答的是"该看到的东西，模型看到了吗"，端到端评估回答的是"模型最后答对了吗"。中间还隔着模型自己这一层。**

再举一个更尖锐的场景：假设你把阈值从 0.5 调到 0.8，端到端准确率没变。这有两种可能——

1. 调高了，召回更精，但该召回的被误杀 → 准确率没降是因为模型靠 prompt 里的强制规则兜住了（"流水→gmv"写死在 system prompt 里）；
2. 调高了，召回没变化，因为你这条查询的 rerank 分数本来就都在 0.9 以上。

**没有检索层指标，你连"这个改动到底有没有生效"都不知道。**

---

## 1. 地基：标注集（Ground Truth）

一切指标的前提是"知道什么该被召回"。这需要一份标注集。

### 1.1 标注集长什么样

```
每个用例 = (query, 每个集合里应该召回哪些 name)
```

```yaml
# retrieval_cases.yaml
cases:
  - id: r001
    query: "各品牌的流水"
    expected:
      metrics: [gmv]            # metrics 集合里，应该召回 gmv 这一条
      terms:   [GMV]            # terms 集合里，应该召回 GMV 这一条
      schema:  [brand]          # schema 集合里，应该召回 brand 这一条
      history: []               # 不要求命中 history

  - id: r002
    query: "让利最多的牌子"
    expected:
      metrics: [total_discount]
      terms:   [total_discount]
      schema:  [brand]
```

### 1.2 关键设计决策（很多人在这栽跟头）

| 决策 | 怎么做 | 为什么 |
|---|---|---|
| **按集合分组** | expected 里每个集合单独列 | 四个集合的语义不同，混在一起算 Recall 没有意义 |
| **标到 name 粒度** | 标 `gmv` 而不是标 `metrics` 集合 | name 是最终能落到实体的最小单位 |
| **允许空 expected** | `history: []` 是合法的 | 不是每道题都要命中所有集合 |
| **标"相关"不是"正确"** | 一个 query 可以有多条相关 | Recall 的分母是"应该召回的相关条数"，不是 1 |

### 1.3 标注集要多大、怎么来

- **起步 50 条**，覆盖六类：术语别名（流水→gmv）、英文缩写（AOV→avg_order_value）、多维度、时间表达、否定、歧义。
- **来源**：① 把现有 `scripts/test_rag_coverage.py` 的 8 条扩写；② 从线上审计表 `question` 字段抽样真实问法；③ LLM 生成 + 人工校验（生成 100 条，人筛出 50 条靠谱的）。
- **质量红线**：标注必须人工过一遍。LLM 自己标自己评 = 循环论证。

---

## 2. 五个指标，逐个教

### 2.1 Hit Rate@K（最简单，先讲）

**是什么**：前 K 条里，只要命中任意一条相关结果，这道题就算过。

**公式**：

```
Hit Rate@K = 命中任意相关结果的 query 数 / 总 query 数
```

**手算**：假设 3 道题，各自 top3 里有没有相关：

| query | top3 | 有相关吗 |
|---|---|---|
| q1 | gmv, avg_order_value, sales_amount | ✅（gmv） |
| q2 | customer_count, total_balance, sales_amount | ❌ |
| q3 | region, brand, channel | ✅（brand） |

Hit Rate@3 = 2/3 ≈ 0.67。

**什么时候用**：想看"整体上用户能不能命中"，做高层健康度看板。
**坑**：它只看"有没有"，不看"排第几"、不看"命中了多少条"。一个 query 召回 5 条但只有第 5 条相关，Hit Rate 还是 1。

### 2.2 Recall@K（最常用）

**是什么**：该召回的相关结果里，前 K 条覆盖了多少比例。

**公式**：

```
Recall@K = |相关结果 ∩ 前K条| / |相关结果|
```

**手算**：query"各品牌的流水"，metrics 集合里应该召回 `{gmv}`（1 条），实际 top3 = `[gmv, avg_order_value, sales_amount]`：

```
Recall@3 = |{gmv} ∩ {gmv, avg_order_value, sales_amount}| / |{gmv}| = 1/1 = 1.0
```

换个查询"让利最多的牌子"，相关 `{total_discount}`，实际 top3 = `[sales_amount, gmv, customer_count]`：

```
Recall@3 = 0/1 = 0.0
```

**什么时候用**：这是**最该先算的指标**。它回答的核心问题是"该找的东西找到了吗"——检索的第一性目标。
**坑**：分母是"相关条数"。如果你的相关集标了 3 条但只召回 1 条，Recall = 1/3 = 0.33，而不是 1。所以标注集里相关条目要标全。

**代码**：

```python
def recall_at_k(ranked: list[str], relevant: set[str], k: int) -> float:
    if not relevant:
        return 1.0                      # 没要求召回 → 不扣分
    hits = sum(1 for name in ranked[:k] if name in relevant)
    return hits / len(relevant)
```

### 2.3 Precision@K

**是什么**：前 K 条里，有几条是相关的。

**公式**：

```
Precision@K = |相关结果 ∩ 前K条| / K
```

**手算**：top3 = `[gmv, avg_order_value, sales_amount]`，相关 `{gmv}`：

```
Precision@3 = 1/3 ≈ 0.33
```

**什么时候用**：想看"召回得干不干净"。Recall 高但 Precision 低 = 捞了一堆垃圾进来，会污染 prompt、吃掉 token 预算。
**坑**：和 Recall 是跷跷板——K 越大 Recall 越高、Precision 越低。所以通常两个一起报，或者用 F1。

### 2.4 MRR（Mean Reciprocal Rank）

**是什么**：第一个相关结果排第几，取倒数后求平均。

**公式**：

```
RR = 1 / (第一个相关结果的排名)
MRR = 所有 query 的 RR 的平均值
```

**手算**：

| query | 第一个相关排第几 | RR |
|---|---|---|
| q1（gmv 排第 1） | 1 | 1/1 = 1.0 |
| q2（total_discount 排第 2） | 2 | 1/2 = 0.5 |
| q3（brand 排第 4） | 4 | 1/4 = 0.25 |

MRR = (1.0 + 0.5 + 0.25) / 3 ≈ 0.58。

**什么时候用**：对**"第一个结果对不对"敏感**的场景。你的场景里尤其重要——因为上下文预算有限，往往只有排最前面的结果会进 prompt。
**坑**：只关心第一个，后面的顺序无所谓。如果场景是"要给模型看 top5"，MRR 会低估"排第 2 也够好"的收益，这时用 nDCG 更合适。

**代码**：

```python
def mrr(ranked: list[str], relevant: set[str]) -> float:
    for i, name in enumerate(ranked, 1):
        if name in relevant:
            return 1.0 / i
    return 0.0
```

### 2.5 nDCG@K（最"排序敏感"）

**是什么**：考虑**排序位置**的命中率——排前面的命中比排后面的值钱。

**公式**（二元相关，即命中=1 不中=0）：

```
DCG@K  = Σ(i=1..K) 相关_i / log2(i+1)
IDCG@K = 理想情况（所有相关排最前）的 DCG
nDCG@K = DCG / IDCG
```

**手算**：相关 `{total_discount}`，top4 = `[sales_amount(0), total_discount(1), gmv(0), customer_count(0)]`：

```
DCG@4  = 0/log2(2) + 1/log2(3) + 0/log2(4) + 0/log2(5)
       = 0 + 1/1.585 + 0 + 0 = 0.631

IDCG@4 = 1/log2(2) + 0 + 0 + 0 = 1.0     # 理想：total_discount 排第 1

nDCG@4 = 0.631 / 1.0 = 0.631
```

**什么时候用**：**精排前后对比必须用它**——因为精排的价值就是"把对的东西排到前面"，只有位置敏感的指标能测出来。
**坑**：多级相关（非常相关/一般相关/不相关）时 rel 不是 0/1，要用 `2^rel - 1` 这类增益函数。你现在标注是二元相关，用 0/1 就行，但要知道这个扩展。

**代码**：

```python
import math

def ndcg_at_k(ranked: list[str], relevant: set[str], k: int) -> float:
    dcg = sum(1.0 / math.log2(i + 1)
              for i, name in enumerate(ranked[:k], 1) if name in relevant)
    idcg = sum(1.0 / math.log2(i + 1)
               for i in range(1, min(len(relevant), k) + 1))
    return dcg / idcg if idcg > 0 else 0.0
```

### 2.6 五个指标怎么选（对比表）

| 指标 | 测什么 | 排序敏感 | 你的场景优先级 |
|---|---|---|---|
| Recall@K | 该找的找到了吗 | 否 | ⭐⭐⭐ 先算这个 |
| MRR | 第一个相关的排第几 | 只关心第一个 | ⭐⭐⭐ 预算紧张时必看 |
| nDCG@K | 整体排序质量 | 是 | ⭐⭐⭐ 精排对比必用 |
| Precision@K | 召回干不干净 | 否 | ⭐⭐ 配合 Recall 看 |
| Hit Rate@K | 有没有命中 | 否 | ⭐ 高层健康度 |

**建议报告格式**：每集合报 `Recall@5` + `MRR` + `nDCG@5`，再加一个全局 `Hit Rate@5`。不要只报一个数。

---

## 3. 分通道归因：定位"哪一层坏了"

这是把评估从"打分"升级成"诊断"的关键一步。

你的检索链是：

```
查询 → [关键词通道] ──┐
     → [向量通道]   ──┼→ 合并去重 → 精排 → 阈值 → top_k
```

**分通道归因 = 分别只开一条通道，各算一遍 Recall**：

```python
report = {
    "keyword_only":   recall_per_collection(retriever_keyword_only, cases),
    "vector_only":    recall_per_collection(retriever_vector_only, cases),
    "hybrid_rerank":  recall_per_collection(retriever_full, cases),   # 最终链路
}
```

**读结果的四条规则**：

| 现象 | 结论 | 该改什么 |
|---|---|---|
| 关键词高、向量低 | 短实体精确匹配是主力，向量通道没帮上 | 查 embedding 是否被短文本拖累，或降 vector 权重 |
| 向量高、关键词低 | 语义相似有用，关键词词典没覆盖到 | 补 jieba 词典 / 术语别名 |
| 两路都低、融合高 | 融合在起作用（两条通道互补） | 保持，可试 RRF 进一步提 |
| 两路都高、融合没更高 | 融合没增益，甚至有损 | 检查合并去重逻辑是否丢序 |

> 没有这个归因，你调检索就是蒙着眼睛改参数。

---

## 4. 精排增益怎么测

精排（CrossEncoder）值不值得上、阈值怎么调，都需要量化。

**做法**：拿同一批 query，跑两次——

```
基线：粗召回（不精排），直接按原始相似度取 top5
精排后：粗召回 top10 → CrossEncoder → 取 top5
```

各算 nDCG@5，对比：

```
ndcg_gain = nDCG(精排后) - nDCG(基线)
```

- `gain > 0` 且显著 → 精排有用，值；
- `gain ≈ 0` → 精排白跑，要么粗召回已经够好，要么 reranker 没吃上正确的输入；
- `gain < 0` → 精排在添乱（可能 reranker 对短中文实体不敏感），要换模型或调阈值。

> 这一步能直接回答面试官问的"你的 rerank 真的有用吗"——不是靠感觉，是靠 nDCG 差值。

---

## 5. 阈值校准（0.5 为什么是错的，怎么改对）

现在的 `threshold=0.5` 是拍的。正确做法是**用数据定阈值**：

### 5.1 看分数分布

把所有标注集的 query 跑一遍精排，收集所有候选的 `rerank_score`，画直方图：

```
相关候选的分数分布： 集中在 0.7 ~ 0.95
不相关候选的分布：   集中在 0.1 ~ 0.4
```

如果两者分得很开 → 阈值取中间 0.5~0.6 就很好。
如果两者重叠严重 → 说明 reranker 分不开，阈值怎么调都白搭，得先换模型或加特征。

### 5.2 用验证集扫阈值

在验证集上从 0.2 到 0.9 扫一遍，找让 `Recall@5` 最高（或 F1 最高）的阈值：

```python
best_threshold, best_score = None, -1
for t in [0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9]:
    retriever.set_threshold(t)
    score = eval_recall(cases, retriever)
    if score > best_score:
        best_threshold, best_score = t, score
```

### 5.3 加一个监控：回退率

你的链路里"某集合全被过滤 → 回退 top3"（`retriever.py:234-236`）。这个回退发生的比例就是**回退率**。

```
回退率 = 触发回退的集合-查询数 / 总集合-查询数
```

- 回退率 > 30% → 阈值定太高，精排形同虚设；
- 回退率 ≈ 0 → 阈值定太低，过滤没起任何作用。

> 一个健康的阈值，应该是"该过滤的过滤掉，不该过滤的极少误杀"，回退率大致落在 5%~15%。

---

## 6. RRF 融合：把"硬拼接"升级成"看排名"

### 6.1 现在的合并方式有什么问题

`_coarse_retrieve` 现在的做法是：**先放关键词通道的结果，再补语义通道的**（`retriever.py:176-189`）。

问题：合并后，**关键词通道的第 1 名和第 9 名没有区别**——都是"先放进去"，语义通道里一个高相关的结果反而排到了关键词通道的垃圾后面。

### 6.2 RRF 的直觉

RRF（Reciprocal Rank Fusion）的核心想法：**不融合分数，融合排名**。

```
为什么不能直接融合分数？
- 关键词通道：语义相似度（cosine）在 0~1
- 向量通道：也是 cosine 0~1，但两个 embedding 空间不同
- 精排：CrossEncoder logit，量纲又不一样
三个分数不可比，直接加权是在比较苹果和橘子。

为什么排名可以融合？
排名是"相对位置"，天然跨通道可比：第 1 就是第 1，不管分数是多少。
```

### 6.3 公式

```
RRF_score(d) = Σ_{每条通道 L}  1 / (k + rank_L(d))

d = 某个候选
rank_L(d) = 候选 d 在通道 L 里的排名（1 起）
k = 平滑常数，通常取 60
```

**为什么分母有个 +60**：如果没有这个 60，第 1 名和第 2 名的权重是 1/1 vs 1/2，差一倍；第 50 名和第 51 名是 1/50 vs 1/51，几乎没差。加 60 是为了**让高排名之间的差距变小、更平滑**，避免某条通道的第 1 名完全碾压其他通道。60 是经验值，不是魔法数字。

### 6.4 手算

假设两个通道对同一个候选 `gmv` 的排名：关键词通道排第 1，向量通道排第 3，k=60：

```
RRF_score(gmv) = 1/(60+1) + 1/(60+3) = 1/61 + 1/63 ≈ 0.0164 + 0.0159 = 0.0323
```

另一个候选 `sales_amount`：关键词第 5，向量第 2：

```
RRF_score = 1/65 + 1/62 ≈ 0.0154 + 0.0161 = 0.0315
```

`gmv` 胜出（0.0323 > 0.0315）。注意看：虽然两个候选在各自通道里有输有赢，RRF 通过排名把"综合位置"算出来了，而且**不需要任何权重参数**。

### 6.5 代码

```python
def rrf_fuse(ranked_lists: list[list[str]], k: int = 60) -> list[str]:
    scores: dict[str, float] = {}
    for ranked in ranked_lists:
        for rank, name in enumerate(ranked, 1):
            scores[name] = scores.get(name, 0.0) + 1.0 / (k + rank)
    return sorted(scores, key=lambda n: scores[n], reverse=True)
```

### 6.6 什么时候用 RRF，什么时候用加权

| 场景 | 用 |
|---|---|
| 通道分数量纲不同（cosine / BM25 / 精排分） | RRF |
| 已有标注可以调权重 | 加权融合（可学习，但会过拟合） |
| 通道少（2 条）、分数同源 | 加权就够，RRF 收益不大 |
| 通道多、想快速上线、不想调参 | RRF |

---

## 7. Query 改写 / 扩展（补"同义词"的另一条路）

你的项目已经做了一半：**术语别名进索引**（`sync.py:124-131`）。但查询侧还能再补一层。

### 7.1 术语表驱动的改写（零成本，最该先做）

```python
def expand_query(query: str, alias_map: dict[str, str]) -> str:
    """把口语词替换/补上标准名。alias_map: 别名 -> 标准指标名"""
    hits = [std for alias, std in alias_map.items() if alias in query]
    return query if not hits else f"{query} {' '.join(hits)}"

# "各品牌的流水" → "各品牌的流水 gmv"
```

为什么有用：把"别名命中"显式带进查询向量，提高向量通道对该标准名的相似度。

### 7.2 伪相关反馈（PRF）

第一轮检索拿 top-N 结果，把里面的高频词（去掉停用词）追加到原 query，再检索一次。**经典但成本翻倍**，一般作为"第一轮效果差"的补救，不做默认路径。

### 7.3 LLM 改写（慎用）

让 LLM 把口语改写成"检索友好的标准问法"。**问题**：多一次 LLM 调用，延迟和成本都涨，而且可能引入幻觉。**建议**：先用 7.1 的规则版本，只有规则覆盖不到、且业务量证明值得时再上 LLM。

---

## 8. 一个完整走查：把上面所有指标串一遍

假设标注集里 2 道题，只看 metrics 集合，K=3：

```
case r001: query="各品牌的流水",  相关={gmv}
case r002: query="让利最多的牌子", 相关={total_discount}
```

实际检索结果：

```
r001 → ranked = [gmv, avg_order_value, sales_amount]
r002 → ranked = [sales_amount, total_discount, gmv]
```

**算一遍**：

```
Recall@3:  r001 = 1/1 = 1.0,  r002 = 1/1 = 1.0   → 平均 1.0
MRR:       r001 = 1/1 = 1.0,  r002 = 1/2 = 0.5   → 平均 0.75
nDCG@3:    r001 = 1/log2(2) = 1.0
           r002 = 0/log2(2) + 1/log2(3) = 0.631
           → 平均 0.816
Precision@3: r001 = 1/3 = 0.33, r002 = 1/3 = 0.33 → 平均 0.33
```

**读结论**：

- Recall=1.0 说明该找的都找到了（没漏）；
- MRR=0.75 说明第一个结果基本对（r002 排第二，略差）；
- nDCG=0.816 说明排序整体不错，但 r002 还能再前进一步；
- Precision=0.33 说明 top3 里混了 2 条不相关的——**如果预算只装得下 2 条，这些垃圾会挤掉有用的**。

> 这个走查能看明白：**四个指标各说一件事，缺哪个都会漏掉一个维度的问题。**

---

## 9. 落地代码（可直接接进现有 RAGRetriever）

新建 `nl2dsl/evaluation/retrieval/`，三个文件：

### 9.1 metrics.py

```python
"""检索评估指标：recall_at_k / precision_at_k / mrr / ndcg_at_k。"""
import math

def _top_names(ranked: list[dict], k: int) -> list[str]:
    """从 RAGRetriever 返回的 hit dict 列表里取前 k 个 name。"""
    return [h.get("name", "") for h in ranked[:k]]

def recall_at_k(ranked, relevant, k=5):
    rel = set(relevant)
    if not rel:
        return 1.0
    return sum(1 for n in _top_names(ranked, k) if n in rel) / len(rel)

def precision_at_k(ranked, relevant, k=5):
    rel = set(relevant)
    top = _top_names(ranked, k)
    if not top:
        return 0.0
    return sum(1 for n in top if n in rel) / len(top)

def mrr(ranked, relevant):
    rel = set(relevant)
    for i, n in enumerate(_top_names(ranked, len(ranked)), 1):
        if n in rel:
            return 1.0 / i
    return 0.0

def ndcg_at_k(ranked, relevant, k=5):
    rel = set(relevant)
    top = _top_names(ranked, k)
    dcg = sum(1.0 / math.log2(i + 1)
              for i, n in enumerate(top, 1) if n in rel)
    idcg = sum(1.0 / math.log2(i + 1)
               for i in range(1, min(len(rel), k) + 1))
    return dcg / idcg if idcg > 0 else 0.0
```

### 9.2 runner.py

```python
"""对标注集跑检索评估，按集合 + 按通道输出指标。"""
from nl2dsl.evaluation.retrieval.metrics import recall_at_k, mrr, ndcg_at_k

COLLECTIONS = ["schema", "metrics", "terms", "history"]

def evaluate(retriever, cases, k=5):
    report = {c: {"recall": [], "mrr": [], "ndcg": []} for c in COLLECTIONS}
    for case in cases:
        raw = retriever.retrieve_hybrid(case["query"], top_k=k)
        for col in COLLECTIONS:
            expected = case["expected"].get(col, [])
            if not expected:
                continue                     # 该题不考这个集合
            ranked = raw.get(col, [])
            report[col]["recall"].append(recall_at_k(ranked, expected, k))
            report[col]["mrr"].append(mrr(ranked, expected))
            report[col]["ndcg"].append(ndcg_at_k(ranked, expected, k))
    return {c: {m: (sum(v)/len(v) if v else None) for m, v in ms.items()}
            for c, ms in report.items()}
```

### 9.3 分通道归因

```python
# 关键词通道：只调 retrieve_by_keywords
# 向量通道：只调 retrieve
# 最终链路：retrieve_hybrid（含精排 + 阈值 + 回退）
def channel_report(retriever, cases, k=5):
    def run(fn):
        for case in cases:
            raw = fn(case["query"])
            ...
    return {
        "keyword_only":  run(lambda q: retriever.retrieve_by_keywords(q)),
        "vector_only":   run(lambda q: retriever.retrieve(q)),
        "hybrid_rerank": run(lambda q: retriever.retrieve_hybrid(q)),
    }
```

> 注意：`retrieve()` 和 `retrieve_by_keywords()` 返回的结构和 `retrieve_hybrid()` 一致（都是 `{集合: [hit dict]}`），所以可以复用同一个评估函数。

---

## 10. 面试怎么答（这套讲法的"输出"）

**被问"你们怎么评估 RAG 检索效果"**：

> "分四层。第一层是检索层，用 Recall@5 / MRR / nDCG 测'该召回的有没有召回、排得对不对'，这里我需要一份标注集，按集合标到 name 粒度。第二层是融合层，对比单通道和融合后的增益。第三层是精排层，对比精排前后的 nDCG 差值。第四层才是端到端 DSL 正确率。
> 我目前只做了第四层，这是我要补的第一件事。补完之后我还会做分通道归因——分别只开关键词通道和向量通道各算一遍 Recall，这样能定位是'没召回'还是'召回了没用对'。"

**被问"Rerank 有什么用、怎么证明"**：

> "粗召回是双塔，query 和 doc 分开编码，精度有限。CrossEncoder 把两者拼起来编码，能捕捉交互信息，排序更准。怎么证明它有用：拿同一批 query，分别跑'不精排直接取 top5'和'精排后取 top5'，比 nDCG 差值，正增益才说明值。"

**被问"混合检索两路分数怎么融合"**：

> "我现在的实现是硬拼接，简单但丢排序信息。更好的做法是 RRF：不融合分数而融合排名，因为两路分数的量纲不可比，排名才是跨通道可比的。RRF 用 1/(k+rank) 加权，k 取 60 做平滑，不需要调权重。"

---

## 11. 速查

| 问题 | 答案 |
|---|---|
| 为什么单独评检索 | 端到端对 ≠ 检索对；失败分"没召回"和"没用好"两类，修法不同 |
| 地基 | 标注集：query → 各集合应召回的 name |
| Recall@K | 该找的覆盖了多少，最该先算 |
| MRR | 第一个相关排第几的倒数均值，预算紧张时必看 |
| nDCG@K | 排序质量，精排对比必用 |
| Precision@K | 召回干不干净，配 Recall 用 |
| 分通道归因 | 分别只开关键词/向量通道各算 Recall，定位哪层坏了 |
| 精排增益 | 精排前后 nDCG 差值 |
| 阈值校准 | 看分数分布 + 验证集扫阈值 + 监控回退率（5%~15%） |
| RRF | 融合排名而非分数，1/(k+rank)，k=60，免调参 |
| Query 改写 | 术语表规则改写（零成本）→ PRF → LLM 改写（慎用） |
| 一句话 | **先建标注集算 Recall，再做分通道归因，才能从"打分"升级到"诊断"。** |
