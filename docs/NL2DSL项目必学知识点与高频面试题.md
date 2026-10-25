# NL2DSL 源码必学知识点与高频面试题

依据 `D:\demo\db-gpt\NL2DSL` 工程的实际代码整理，阅读日期：2026-10-23。本文的“已实现”只表示在该目录看到对应代码，不证明公司环境已经上线，也不证明个人独立开发。先掌握 P0，再学 P1；每题要能指到函数、解释输入输出，并说出一个失败分支。

## 术语速查（读正文前先看这里）

> 正文大量出现源码行话，先扫一遍这 5 个词，再看 P0 就顺了。代码路径均在 `D:\demo\db-gpt\NL2DSL`。

### 1. domain —— 业务领域

**人话：** 一套系统服务多个业务团队，电商一套、银行一套。每套有**自己的指标、维度、权限、数据库、向量库**，互相隔离。简历里"多域自治、各域配置完全隔离"就是它。

**代码一条链：**

- ① 靠文件名发现领域 — `nl2dsl/engine.py:26`：`metrics.yaml`（无前缀）→ `ecommerce`；`bank_metrics.yaml` → `bank`。
- ② 每个域一个"组件盒子" — `nl2dsl/domain_context.py:8`：`DomainContext` 装着 `registry_dict / validator / sql_builder / sandbox / executor / row_security / rag_retriever / graph`。
- ③ 数据库/向量库按域隔离 — `nl2dsl/engine.py:46`：bank 用 `data/bank.db`、`data/bank_milvus_lite.db`。
- ④ 全部塞进字典 — `nl2dsl/engine.py:68`：`self._domains: dict[str, DomainContext]`。
- ⑤ 请求带 domain 按名取图 — `nl2dsl/api_factory.py:542`：`_get_domain_graph(domain)` 找不到直接 `raise NotFoundError`。

⚠️ **高频考点**：`Engine.get_domain()`（`engine.py:83`）写了"找不到回退 ecommerce"，但 **API 实际走 `_get_domain_graph`，抛 NotFoundError、不回退**。问"未知 domain 会不会回退"要答：API 路径抛 404。

### 2. ainvoke —— 异步执行整条图

**人话：** 图在启动时编译好，像流水线拼装完毕。`ainvoke` 就是**按启动键**——塞输入进去，跑完吐输出。`a` = async，`invoke` = 执行。

**代码对应：**

- ① 编译图 — `nl2dsl/engine.py:268`：`graph = build_graph(...)`，存进 `DomainContext.graph`。
- ② 造初始状态 — `nl2dsl/api_factory.py:826`：`state = QueryState(question=..., user_id=..., status="pending", query_id=...)`。`QueryState` 定义在 `graph/state.py:57`，是图里所有节点读写的共享状态。
- ③ 造身份卡并启动 — `nl2dsl/api_factory.py:847`：`config = {"configurable": {"thread_id": query_id}}`，然后 `result = await _get_domain_graph(req.domain).ainvoke(state, config)`。
- ④ 取结果 — `nl2dsl/api_factory.py:851`：`dsl = result.get("dsl")`。`result` 还是 `QueryState`，但 `dsl/sql/data/status` 已被各节点填好。

| 东西 | 是什么 |
|------|--------|
| `state` | 初始输入 + 各节点共享的中间状态（`QueryState`） |
| `config` | 这次执行的身份卡，`thread_id` 用于隔离/恢复 |
| `invoke` / `ainvoke` / `astream` | 同步 / 异步 / 流式（SSE 用 astream） |

**面试说法：** "`ainvoke` 是 LangGraph 编译图的异步执行入口，传入初始 `QueryState` 和带 `thread_id` 的 config，跑完整条管道返回填好结果的 state。`thread_id` 用 `query_id`，检查点能按请求隔离、也支持 `/query/resume` 恢复。"

### 3. DSL —— 结构化语义契约

**人话：** LLM 不直接吐 SQL，而是吐一张**结构化"查询申请单"**，写清要查什么指标、按什么维度、什么过滤条件。系统再拿这张单子去校验、注入权限、拼 SQL。所以"DSL 是语义契约，SQL 是实现细节"。

**代码定义** — `nl2dsl/dsl/models.py:72`：`DSL(BaseModel)` 有 `metrics / dimensions / filters / having / order_by / limit / data_source / time_field / time_range / joins / post_process`。

**两个关键点：** `filters` 同时支持平铺列表和条件树（`models.py:98` 自动识别 dict → `FilterTreeNode`）；`post_process` 只支持 `group_top_n` 和 `proportion`（`models.py:61`）。

### 4. DSLValidator —— DSL 校验器

**人话：** 不查字段"形状"（那是 Pydantic 的事），查的是**语义合不合法**——指标/维度注册了没、属不属于当前数据源、有没有漏指标。

**代码** — `nl2dsl/dsl/validator.py:7`：构造时把 `registry` 里的 metrics/dimensions/data_sources 转成 `set` 存下来。`validate()`（`validator.py:43`）查：data_source 存在、指标 alias 注册、维度注册、指标/维度属于当前 data_source、必须指定 metrics 或 dimensions、post_process 引用合法。

**边界（高频追问）：** Pydantic 只保证"形状对"，`DSLValidator` 保证"语义对"。但**校验通过 ≠ 结果正确**——语义、权限、Join、真实数据、执行结果都可能出错。

### 5. SemanticRegistry —— 语义注册表

**人话：** 把 `configs/metrics.yaml` 里的指标、维度、数据源读进内存变成字典，供校验器、SQLBuilder、RAG 检索查。就是"哪些指标/维度合法"那本台账。

**代码** — `nl2dsl/semantic/registry.py:5`：`load()` 读 YAML 填 `self.metrics / dimensions / data_sources`。加载入口 `engine.py:177`，之后 `registry_dict` 传给 `DSLValidator`（`engine.py:210`）和 `SQLBuilder`（`engine.py:234`）等下游。

### 其他常见行话（一句话）

| 词 | 一句话 |
|----|--------|
| **Pydantic** | 数据校验库，定义 DSL/Filter 字段形状和枚举范围 |
| **LangGraph** | 编排框架，用「节点 + 边」定义 Agent 工作流 |
| **StateGraph** | LangGraph 定义状态机的类，`build_graph` 用它拼节点 |
| **CompiledGraph** | 编译后的图对象，`ainvoke`/`astream` 在它上面调 |
| **QueryState** | 图里所有节点共享的状态字典（`graph/state.py:57`） |
| **thread_id** | 检查点隔离键，本项目用 `query_id` |
| **reducer** | 状态字段合并函数（如 `state.py:13 add_to_list`），处理子图 trace 去重 |
| **checkpointer / InMemorySaver** | 检查点存储，用于中断恢复；内存版进程重启即丢 |
| **RAG** | 检索增强生成，从向量库召回 Schema/指标/术语/样例喂 LLM |
| **Milvus Lite** | 本地向量库，存 embedding 供检索 |
| **BGE** | 中文 embedding / reranker 模型 |
| **sqlglot** | SQL 解析/方言转换库 |
| **SQLAlchemy Core** | 用表达式树动态拼 SQL，不用 ORM |

---

## P0-1. 从 API 入口讲完整请求

**代码定位：** `nl2dsl/api.py`、`nl2dsl/api_factory.py:458`、`:870`、`:1023`；`nl2dsl/engine.py`、`nl2dsl/domain_context.py`。

**必须掌握：** `api.py` 调 `create_app()`。`create_app` 无覆盖参数时复用 Engine 的 DomainContext；带注入参数时重建组件用于测试。`POST /api/v1/query` 先校验 domain、生成 `query_id`，可选做歧义预检，再由 `AgentController.route` 分流：复杂/探索计划进 `AgentOrchestrator.run`，简单计划构造 `QueryState` 并调用对应领域图的 `ainvoke`。两条路径都在 API 层写审计。`/query/dsl` 只生成 DSL，`/query/execute` 接收 DSL 执行，不能混为一个接口。

**高频追问：**

- “一个请求究竟什么时候走 Agent，什么时候走 Graph？”答时指出执行计划类型判断，不要说所有请求都先过完整 Agent。
- “未知 domain 会不会回退到电商？”`_get_domain_graph` 直接抛 `NotFoundError`。
- “为什么 `create_app` 有无覆盖参数会走两条构造路径？测试注入会不会漏掉真实 RAG/权限配置？”

**动手验收：** 用纸画出 `/query` 的两条路径，写出 `question/user_id/tenant_id/domain/query_id` 在哪里进入 State 或 Agent，最后在哪里写入审计。对照 `tests/e2e/test_api_query.py`、`tests/integration/test_multi_domain.py`。

## P0-2. DSL 是什么，哪些校验不是 Pydantic 能做的

**代码定位：** `nl2dsl/dsl/models.py:5`、`:72`，`nl2dsl/dsl/validator.py`、`nl2dsl/dsl/semantic_validator.py`、`nl2dsl/semantic/registry.py`，`configs/metrics.yaml`。

**必须掌握：** DSL 有 `data_source`、`metrics`、`dimensions`、`filters`、`having`、`order_by`、`limit`、`time_range`、`joins`、`post_process`。`filters` 同时支持平铺列表与 `and/or/not` 条件树；`post_process` 当前模型支持 `group_top_n` 和 `proportion`。Pydantic 限定字段形状和部分范围；指标/维度是否注册、是否属于数据源、操作符和业务口径要继续交给校验器和语义注册表。

**高频追问：**

- “模型生成 `sales_amount`，是否允许它自己写 `SUM(discount_amount)`？”要从注册表与 SQLBuilder 的指标表达式优先级回答。
- “平铺 filters 和条件树有什么区别？行级权限追加过滤时如何保持原有 OR 语义？”这是需要读实现并实测的边界，不能仅凭模型定义断言安全。
- “为什么 DSL 校验通过仍不代表结果正确？”语义、权限、Join、数据和执行结果仍可能出错。

**动手验收：** 针对“华东近 7 天各品牌销售额前 5”写一份 DSL，分别让 `data_source`、指标、维度、操作符各错一次，记录是哪一层拦住。看 `tests/unit/test_semantic_validator.py` 和 `test_sql_builder.py`。

## P0-3. SQLBuilder 怎样把 DSL 变成 SQL

**代码定位：** `nl2dsl/sql_engine/builder.py:14`、`:42`、`:181`、`:293`、`:403`、`:581`；`nl2dsl/semantic/resolver.py`；`configs/metrics.yaml`。

**必须掌握：** `build(dsl)` 从 `data_source` 找主表，收集查询引用列，按配置推导必要 JOIN；指标优先用注册表达式，维度映射到物理列，`value_map` 将业务枚举转成库中取值，再组合 SELECT、WHERE、GROUP BY、HAVING、ORDER BY。存在 `post_process` 时，代码不直接对 SQL 应用全局 `LIMIT`，避免破坏分组 TopN/占比后处理。构建器用 SQLAlchemy 表达式，但 `_resolve_column` 与指标聚合仍有 `text(...)` 路径，安全不能只凭“用了 SQLAlchemy”来保证。

**高频追问：**

- “`sales_amount` 的表达式从哪来？模型写错物理字段会怎样？”看 `_metric_expr` 和指标构建分支。
- “华东这个值在哪里转为数据库编码？什么时候需要 JOIN？”看 `_translate_filter_value`、`_determine_required_joins`。
- “TopN 为什么不总是在 SQL 中加 LIMIT？WHERE 与 HAVING 的差别？”
- “post_process 在 Python 内存里分组 TopN/占比，高基数维度分组会爆内存吗？”先分清风险边界是”分组后的组数”而非原始行数（SQL 已 GROUP BY）。看 `query/post_processor.py`（`extract_top_n_per_group` 全量 `sorted`、`calculate_proportion` 全量 `sum`）与 `builder.py:581`（有 post_process 时不加 LIMIT，导致 `limit le=10000` 上限对后处理路径失效）。现状缺口：`selectivity` 标记未在执行前硬拦截、沙箱 EXPLAIN 是启发式预估、后处理纯内存无流式。改进三闸：低基数维度白名单、组数显式上限、分组 TopN 下沉窗口函数。面试把风险说成”已识别+待完善”，勿称已有保障。
- “`text(expr)` 的信任边界是什么？注册配置被污染怎么办？”

**动手验收：** 用 `configs/metrics.yaml` 中真实注册项手算一个 DSL 对应的 SQL；对照 `tests/unit/test_sql_builder.py`，确认 Join、过滤、排序与 LIMIT。此前的[SQLBuilder 专项复盘](2026-10-13-NL2DSL面试训练复盘与SQLBuilder专项.md)可作题目，但以当前源码为准。

## P0-4. LangGraph 真正的节点与循环

**代码定位：** `nl2dsl/graph/builder.py:38`、`:147`，`nl2dsl/graph/subgraphs.py`、`nl2dsl/graph/edges.py:53`、`:194`、`nl2dsl/graph/state.py`、`nl2dsl/graph/nodes.py`。

**必须掌握：** 主图从 `clarification → plan → decompose → validation 子图 → permission_check 子图 → resolve_semantic` 开始；有语义配置时才注册 `optimize_dsl`，随后经 `confidence → build_sql → scan_sql → sandbox_check`，按结果去 `human_review` 或 `execute_sql`，最后 `verify_dsl → explain → END`。校验子图内部有生成、校验、纠正循环；不要把它说成主图的一串独立顺序节点。`plan` 对非单查询可结束主图，交由 API 的 Agent 路径处理。

**高频追问：**

- “DSL 校验失败到底重试几次？”`route_after_validate` 同时数生成尝试和校验失败，阈值为 3；规则生成模式下也要避免无限循环。
- “SQL 执行失败会不会无限重试？”`route_after_execute` 按 `source="simplified"` 计数，最多再简化重试 1 次。
- “复杂度检测后会走不同节点吗？”当前 `build_sql` 后 simple/complex 都指向 `scan_sql`；不要讲成两条不同执行管道。
- “`human_review` 是真实人工审批还是图中节点？”检查节点实现和 resume 接口后再描述自动/人工边界。

**动手验收：** 不看图画出正常、澄清、校验失败、沙箱风险、执行失败五条路径；用 `tests/unit/test_graph_edges.py`、`test_graph_subgraphs.py` 验证。

## P0-5. 治理与权限：代码做到哪一步

**代码定位：** `nl2dsl/permission/row_level.py`、`column_level.py`、`nl2dsl/graph/nodes.py:911`、`:926`、`configs/permissions.yaml`。

**必须掌握：** 行权限按 `user_id` 查配置，把行过滤和配置中的 `tenant_id` 追加进 DSL；列权限目前检查 `dimensions` 中的敏感字段，另有结果行的 `mask` 方法。重要边界：`RowLevelSecurity.inject` 遇到未配置用户会原样返回 DSL；不要声称这个类本身对未知用户默认拒绝。也要确认 API/其他层是否另有身份校验，才能评价整条链路。列检查只看 dimensions 也需要结合 SQL 生成和测试分析是否覆盖指标、过滤、排序引用。

**高频追问：**

- “未配置用户的查询怎么办？租户 ID 取请求还是权限配置？”
- “用户原有 OR 条件加行权限，是否保证整体仍是 AND？”
- “模型把敏感字段放进 filter 或 metric 而非 dimensions，现有检查覆盖吗？”

**动手验收：** 设计已授权、未知用户、跨租户、敏感维度、敏感过滤五个测试输入；分别写预期与当前代码行为。参考 `tests/unit/test_permission_row.py`、`test_permission_col.py`。面试中把风险说成已识别和待完善项，勿称已有绝对保障。

## P0-6. SQL 扫描、沙箱、执行的不同职责

**代码定位：** `nl2dsl/sql_engine/scanner.py`、`nl2dsl/query/sandbox.py:24`、`nl2dsl/sql_engine/executor.py`、`nl2dsl/graph/builder.py:251`。

**必须掌握：** `SQLScanner` 用正则拦危险操作、注释、UNION 和多语句；它不是 SQL AST 白名单，不能证明任意 SQL 安全。`QuerySandbox.check` 使用 SQLite 的 `EXPLAIN QUERY PLAN` 做启发式行数估算，执行 `LIMIT` 预览，并把缺少 WHERE 判为风险；预估值不是生产库真实扫描行数，预览也可能有实际执行代价。沙箱不通过时图路由到 `human_review`。最后才到执行器。

**高频追问：** “SQLBuilder 已用 SQLAlchemy，为什么还扫描？”“沙箱 `EXPLAIN` 失败返回 -1 时如何处理？”“没有 WHERE 的小维表查询会怎样？”“MySQL 能否直接沿用 SQLite 预估逻辑？”

**动手验收：** 用 `tests/unit/test_sandbox.py` 和扫描器相关测试各举一个允许、拒绝、误报或漏报样例，说明生产设计还需要只读账号、超时、资源限制和数据库权限。

## P1-7. RAG 的真实检索与同步流程

**代码定位：** `nl2dsl/rag/retriever.py:21`、`store.py:23`、`reranker.py:13`、`sync.py:58`、`:205`，`nl2dsl/engine.py`。

**必须掌握：** 四个集合是 `schema/metrics/history/terms`。检索器从库中装载名称和中文短语，加入 jieba 字典；短实体集合按关键词分别嵌入召回，history 用整句语义检索，然后去重、可选 rerank。`auto_sync` 在启动时按 YAML 修改时间和同步状态决定是否 upsert。它不是实时配置订阅，也未看到删除旧向量的自动步骤；注册配置与索引不是原子同版本发布。

**高频追问：** “问销售额却召回订单数，从关键词、候选集、精排还是注册表查起？”“改 YAML 不重启会立刻生效吗？”“删除一个指标后旧索引如何清理？”“为什么 BGE-Reranker 只处理候选集？”

**动手验收：** 从 `retrieve_hybrid` 跟一个问题，记下每类候选和排序结果；参考 `tests/unit/test_rag_retriever.py`、`tests/integration/test_rag_rerank_integration.py`。

## P1-8. 评测、数字与可观测性

**代码定位：** `nl2dsl/evaluation/dataset.py`、`runner.py`、`v2_runner.py`、`scoring.py`、`baseline.py`、`v2_reporter.py`；`nl2dsl/audit/`；`tests/evaluation/v2/`。

**必须掌握：** 区分 DSL 结构合法、指标/维度/过滤正确、SQL 执行成功、结果正确和治理生效。评测 v2 有数据集、各类 scorer、baseline、回归比较与报告；讲成绩时必须带数据集版本、样本数、成功定义、模型/配置版本、环境和分位耗时。API 简单/复杂路径都写审计，但审计字段要按路径与错误分支核对。

**高频追问：** “90% 准确率具体是什么分母？”“800ms→80ms 是否端到端 P95？”“准确率提高、用户体验却变差怎么解释？”“评测集会不会把模拟样例当线上数据？”

**动手验收：** 找一份实际评测输出和原始样本，复算一项分数；找一条失败样本沿 trace 定位。找不到原始证据时，面试不要报精确百分比。

## P1-9. 多领域、Agent 与生产入口

**代码定位：** `nl2dsl/engine.py:26`、`nl2dsl/domain_context.py`、`nl2dsl/agent/controller.py`、`orchestrator.py`、`nl2dsl/api_factory.py:904`。

**必须掌握：** Engine 发现领域并构建各领域上下文；API 先验证 domain，再按 AgentController 的计划类型选择单查询 Graph 或 Agent 编排。跨领域/复杂查询要关注子查询是否共享身份、租户、领域和审计上下文。不要仅凭 `AGENTS.md` 架构图描述调用顺序，以上述函数实际分支为准。

**高频追问：** “银行和电商的指标配置如何隔离？”“复杂问题拆成两条查询，失败一条时返回什么？”“同一个 query_id 能否追踪所有子查询？”

**动手验收：** 读 `tests/integration/test_multi_domain.py`、`test_agent_orchestrator.py`，找一个简单问题和一个复杂问题的不同入口。

## 面试前最终验收

1. **白板 10 分钟：** 画 `/api/v1/query` 的简单与复杂两条路径，明确 Graph 子图和 Agent 分流。
2. **现场 15 分钟：** 用 `configs/metrics.yaml` 中一个真实指标手写 DSL 和预期 SQL，指出注册口径、值映射、JOIN、LIMIT 与后处理。
3. **故障 10 分钟：** 解释未知指标、未配置用户、检索过期、沙箱风险、SQL 执行失败各在哪层处理及当前边界。
4. **证据 5 分钟：** 准备自己确实开发过的函数/提交/测试、一个失败修复案例、一个能复算的评测数字。公司版、开源版、设计方案分别标清。

延伸练习见[深度模拟面试训练](高宇-NL2DSL深度模拟面试训练.md)。该工程目录不是当前资料仓库的一部分，本文引用的源码路径请在 `D:\demo\db-gpt\NL2DSL` 中打开。

## 高级 Agent 开发工程师：压力追问题库

下面每题都要求回答四件事：**当前代码怎么做、触发条件、失败后的状态、你会怎么验证或改进**。标有“设计题”的改进方案不能冒充项目已实现。先自己口述，再查源码和测试。

### A. Agent 路由与规划（10 题）

源码：`nl2dsl/agent/controller.py`、`planner.py`、`models.py`、`orchestrator.py:237`；`nl2dsl/api_factory.py:904`。

1. `/query` 已经调用一次 `AgentController.route`，进入 `AgentOrchestrator.run` 后又调用一次。两次结果若不一致，会发生什么？怎样设计才能保持单次请求的计划一致性？
2. `ComplexExecutionPlan` 与 `ExplorationPlan` 都进入 Orchestrator，但 exploration 在 `_run_exploration_path` 中是否真的执行探索循环？指出当前能力边界。
3. 规划器把本该拆成两条的比较问题误判为 `single_query`，你如何用 trace 定位？会在哪一步产生错误结果？
4. 实体抽取从用户问题取出指标、维度时，怎样防止“销售额”和“GMV”同名但不同口径？实体提示与注册表谁有最终解释权？
5. 子查询计划中 `depends_on` 的合法性在哪层验证？缺失依赖、环、重复 ID 分别如何处理？
6. 用户问“今年比去年增长多少”，计划要拆成什么子查询？时间基准和分母为零如何表示？
7. 比较、趋势、排行、占比四种意图需要哪些不同的结果结构？怎样避免 Aggregator 猜错数值列？
8. 如果一条复合查询跨银行和电商两个 domain，现有 `DomainContext` 与 dispatcher 能否直接支持？需要怎样的授权和路由契约？
9. 规划器使用 LLM 时，怎样约束输出的子查询数量、依赖深度和预计执行成本？超预算如何降级？【设计题】
10. 什么时候应该让 Agent 自主规划，什么时候固定工作流更合适？结合本项目 API 分流讲成本、可解释性和安全边界。

### B. 子查询并发、依赖与聚合（10 题）

源码：`nl2dsl/agent/dispatcher.py`、`aggregator.py`、`orchestrator.py:463`。

11. `MAX_PARALLEL_SUB_QUERIES=3` 只限制独立子查询还是所有子查询？如果单个 Graph 内部再发并发请求，全局并发上限还成立吗？
12. `asyncio.gather` 等待全部独立子查询，一个子查询卡住会怎样？怎样设单任务超时与总预算？【设计题】
13. dispatcher 用 `thread_id=f"sub_{sub_query.id}"`。两个请求都生成 `sq-1` 且共用 `InMemorySaver` 时，检查点可能如何相互影响？怎样构造隔离测试？
14. 子查询 state 的 `query_id` 从 base_state 传递，复杂路径目前传的是空字符串；父请求审计 ID 与子查询 trace 如何关联？
15. 有依赖的子查询在依赖完成后串行执行。若依赖返回 `warning` 或 `clarification`，下游该继续、跳过还是要求用户澄清？当前代码怎么判？
16. 依赖链有环，dispatcher 如何结束？报错能否准确指出成环节点，而不只是“未满足依赖”？
17. 三条子查询中一条失败，Orchestrator 何时返回 `warning`，何时返回 `error`？聚合器是否会把失败项的空数据当作 0？
18. 比较查询两期数据只有一期成功，能否输出增长率？怎样防止部分结果被解释成完整结论？
19. Aggregator 对数值列使用启发式查找时，遇到订单数、金额、折扣额三列，如何保证聚合的是计划指定的指标？
20. 取消 HTTP 请求后，后台子查询是否继续执行？如何向图传播取消信号，同时保留必要审计？【设计题】

### C. State、Graph 与恢复（10 题）

源码：`nl2dsl/graph/state.py`、`builder.py:147`、`edges.py`、`subgraphs.py`、`nl2dsl/engine.py:73`。

21. `trace` 和 `dsl_attempts` 使用 reducer 追加并按字典相等去重。如果两次真实事件内容完全相同，会不会丢一条？怎样设计事件 ID？
22. 子图把父图的 trace 带回，为什么需要 reducer 去重？并行节点同时写同一状态字段会产生什么合并问题？
23. 主图 `validation → permission_check` 是无条件边；校验子图出错时如何避免带错误 DSL 继续进入权限和构建？请跟一次实际状态。
24. `route_after_validate` 用生成尝试数和校验失败数双重上限。若 `correct_dsl` 原样返回，循环何时终止？
25. SQL 执行失败后简化 DSL 最多一次。简化可能删除用户原本要求的过滤或维度吗？如何证明“重试成功”仍回答原问题？
26. `detect_complexity` 给出 simple/complex，但目前两个标签都去 `scan_sql`。复杂度字段还有什么作用？你会怎样避免误导性分支？
27. Engine 使用 `InMemorySaver`；进程重启、水平扩容、请求打到另一实例时，`/query/resume` 会怎样？【设计题】
28. 图在 `human_review` 前 interrupt；审批请求怎样防重复批准、越权批准和过期批准？现有 `/query/resume` 做了哪些校验？
29. `query_resume` 使用默认 `query_graph`。如果原请求属于非 ecommerce domain，恢复会命中哪张图？如何把 domain 放进可验证的恢复令牌？
30. 节点执行了数据库读或写审计后进程崩溃，恢复时是否会重复执行？为图节点设计怎样的幂等键？【设计题】

### D. RAG、语义层与优化器（10 题）

源码：`nl2dsl/rag/retriever.py`、`sync.py`、`nl2dsl/optimizer/engine.py`、`registry.py`、`nl2dsl/semantic/registry.py`。

31. 指标从 YAML 删除后向量集合仍有旧记录，LLM 召回旧指标会怎样？注册表校验是否能兜住，用户体验如何？
32. YAML 加载与向量 upsert 不是原子发布。索引版本新、注册表版本旧时，怎样给一次查询绑定同一版本？【设计题】
33. 关键词库在 `RAGRetriever` 初始化时从集合读取。新增术语同步后，已运行进程里的 jieba 词库何时更新？
34. 短实体走逐关键词 embedding 检索，用户一句话切出 10 个关键词时会有多少向量调用？缓存、批量化和候选去重如何优化？
35. Reranker 把正确指标排到第一，但语义注册表中同名指标属于另一数据源，哪个组件应该拒绝？
36. RAG 的召回率、精排质量和端到端答案正确率如何分别评估？TopK 增大为何可能让答案更差？
37. RuleEngine 按优先级执行规则，某条 Fix 改了 DSL 后，同优先级下一条规则读到哪个版本？这会造成什么规则顺序依赖？
38. RuleEngine 的 `dsl_before` 是浅拷贝；嵌套结构原地修改时 diff 可能怎样失真？如何写最小复现测试？
39. 优化规则异常被转成 `Warn` 继续运行。若该规则本应拦越权指标，这种失败策略是否合适？如何分类 fail-open/fail-closed？
40. LLM、规则生成器和语义优化器都能改 DSL。如何记录每一步的修改理由、前后版本与最终责任边界？

### E. 安全、租户与数据正确性（10 题）

源码：`nl2dsl/permission/`、`nl2dsl/sql_engine/builder.py`、`scanner.py`、`nl2dsl/query/sandbox.py`、`nl2dsl/graph/nodes.py`。

41. 未配置 `user_id` 时 `RowLevelSecurity.inject` 直接返回原 DSL。如何设计拒绝默认，同时兼容公开 Demo？
42. 用户过滤为 `(region=华东 OR region=华南)`，系统追加 `tenant_id=T1`。代码处理条件树时会保留 `(A OR B) AND T1` 吗？请用测试证明。
43. `ColumnLevelSecurity.check` 只遍历 `dimensions`。敏感字段出现在 filters、metrics、order_by、having 时分别会怎样？
44. 租户 ID 同时出现在 HTTP 请求和权限配置中；两者冲突时应该相信哪一个？跨领域子查询是否复用相同租户上下文？
45. SQLScanner 正则拦截 `UNION` 等模式，但未做完整 AST 白名单。举一个误报和一个无法证明安全的案例。
46. SQLBuilder 的 `text(expr)` 接收治理配置表达式。谁有权改配置？配置发布如何审查、签名、回滚和审计？【设计题】
47. 沙箱的 SQLite `EXPLAIN QUERY PLAN` 粗略返回“SCAN 数 ×1000”。为什么不能把它叫真实扫描行数？迁移 MySQL/PostgreSQL 怎么做？
48. `LIMIT 10` 预览可能先执行大 JOIN/聚合。怎样在数据库层真正限制资源和超时？【设计题】
49. 没有 WHERE 的合法全量小维表查询被判风险，怎样区分可接受的扫描和高风险事实表扫描？
50. 结果解释 LLM 可能泄露原始字段或敏感值。脱敏应在 SQL、结果、Prompt 注入、日志的哪些边界做？【设计题】

### F. SSE、审批与可观测性（8 题）

源码：`nl2dsl/api_factory.py:1150`、`:1175`、`:1279`、`:1355`；`nl2dsl/audit/`。

51. 复杂查询 SSE 用 `asyncio.create_task(run_agent())` 和无界 `asyncio.Queue`。客户端读得慢或断连时，内存和后台任务会怎样？【设计题】
52. 简单查询 `graph.astream(stream_mode="updates")` 与复杂查询 callback 事件，事件 schema 是否一致？前端如何兼容两种流？
53. SSE 中途抛异常后写审计并发送 error；如果审计写入也失败，客户端能收到什么？如何避免只看到 done？
54. 网络断线重连时如何避免重复执行 SQL？需要事件序号、query_id、结果缓存或检查点中的哪些信息？【设计题】
55. `/query/resume` 的 action 由谁提交？当前代码有无鉴权和审批人身份记录？怎样防止用户拿到 query_id 就批准自己的高风险查询？
56. 父请求、子查询和图节点分别用什么 ID 串起来？当前复杂路径里子查询 thread_id 与父 query_id 的关系是什么？
57. 审计日志、业务 trace、LLM token/耗时指标各解决什么问题？哪些字段需要脱敏或限制保留周期？
58. 同一请求的 `/query` 与 `/query/stream` 结果不一致时，先比对哪些构造参数、状态合并和路径分支？

### G. 评测、性能与工程变更（8 题）

源码：`nl2dsl/evaluation/`、`tests/evaluation/v2/`、`nl2dsl/quality/`、`tests/integration/`。

59. 如何把路由正确率、规划正确率、指标命中率、SQL 可执行率、结果正确率拆开评估？为什么一个总分不够？
60. 子查询部分失败返回 `warning`，评测时算成功、失败还是部分分？对比/占比意图应如何定不同规则？
61. 相对时间“上周”会随运行日期变化。评测如何固定 `reference_date`，并同时覆盖跨年和时区边界？
62. LLM 与规则生成两种模式都能运行。怎样做公平 A/B：固定数据、检索索引、配置、模型版本与成本统计？
63. 你说 rerank 从 800ms 到 80ms，计时从哪里到哪里？是否包含 embedding、候选检索、模型冷启动、批量大小、P95？
64. 一条规则提升整体分数却让高风险权限用例退化，回归门禁应以平均分还是关键场景零容忍为准？
65. 修改 SQLBuilder 的 JOIN 推导，最小回归集应覆盖哪些情况：多跳 JOIN、同名列、空维度、聚合、TopN、方言？
66. 怎样把线上失败 trace 脱敏后沉淀进评测集，同时避免相同模板样本导致过拟合？【设计题】

### H. 现场设计与代码题（6 题）

67. **设计题：** 支持一个新业务域。画出配置、DomainContext、RAG 集合、数据库连接、权限和评测集的隔离边界；说出灰度与回滚步骤。
68. **设计题：** 增加“连续三轮澄清”能力。定义会话状态、过期时间、用户身份绑定、恢复接口与重复提交语义。
69. **设计题：** 给 Agent 增加一个外部工具，例如指标解释服务。定义输入 schema、超时、重试、幂等、权限、审计与工具结果可信度。
70. **代码题：** 修复 `thread_id=f"sub_{sub_query.id}"` 潜在碰撞，同时保证一个父请求中多个子查询可追踪；补并发测试。
71. **代码题：** 给行级权限增加未知用户拒绝策略，并验证平铺过滤与条件树的布尔语义；写正常和越权用例。
72. **代码题：** 为 SSE 复杂路径加入有界队列和断连取消，保证完成、失败、取消都写最终审计且不会重复执行。

**练习方法：** 每组先抽 3 题，各用 2 分钟回答；面试官再追问“源码在哪里、当前缺口是什么、测试怎么写”。答不出具体分支就回到相应函数，不背通用框架定义。
