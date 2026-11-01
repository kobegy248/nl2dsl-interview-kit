# 高宇 NL2DSL 面试总结速查

> 本文是《面试官追问清单》《面试参考答案》两份材料的提炼速查，用于面试前快速回忆链路、口径和边界。原文档逐题展开，本文只保留**必背骨架**和**容易被问穿的口径红线**。
>
> 核心心态：越被追问，越先界定口径——哪些是公司项目里的业务实践，哪些是个人开源项目里的抽象复现，哪些是评测/本地验证结果。不硬编生产规模、核心职责或无法证明的数据。

---

## 0. 一句话定位

8 年后端+数据工程，最近做企业级**NL2DSL**智能问数：让业务用自然语言查数，但 LLM 不直接生成 SQL，而是先生成结构化 **DSL**，由系统负责校验、权限注入、语义解析、SQL 构建、安全扫描、执行和审计，把查询变成**可治理、可审计、可回溯**的链路。

- 公司项目：业务落地的智能问数/数据治理实践，内部细节不展开。
- 开源项目 `nl2dsl-engine`：基于同类问题的个人抽象复现，展示架构与工程能力，不含公司数据。
- 两者不是代码一比一搬运。

---

## 1. 项目边界（最先被追问，必须先想清楚）

| 维度 | 公司项目 | 开源项目 |
|---|---|---|
| 性质 | 真实业务场景智能问数 | 个人抽象复现/沉淀 |
| 重点 | 业务落地、治理、权限、审计 | 架构展示、代码可讲 |
| 包含 | 内部数据/表/配置（保密） | 不含公司内部细节 |
| 指标 | 以内部实际为准，不展开 | 评测集/本地链路验证 |

**证明“真做过”的三类东西**：① 讲清抽象后的模块边界（谁产生/校验/消费）；② 讲清异常路径（指标不存在、字段越权、召回不足、执行失败怎么处理）；③ 打开开源项目讲对应实现。

**不要说**：公司项目和开源项目“基本一样”；开源能力=公司生产已全部上线；用“这个不能讲”结束回答。

---

## 2. NL2DSL 架构

### 为什么不直接 NL2SQL，要先 DSL
SQL 是自由文本：不好校验、权限难注入（子查询/UNION 易漏）、方言复杂、审计难还原业务意图。DSL 是**结构化语义契约**，LLM 只生成受约束结构，系统负责校验/权限/优化/SQL 构建，把不确定性限制在可控范围。

### DSL 核心字段
`data_source`（数据源）、`metrics`（被聚合的值）、`dimensions`（切分/筛选维度）、`filters`（过滤条件）、`order_by`、`limit`、`post_process`（TopN/占比/趋势/同环比）。

### DSL → SQL 完整链路
`自然语言 → RAG 召回 → LLM 生成 DSL → 结构校验 → 指标/字段语义校验 → 权限注入 → 语义解析 → SQLAlchemy 构建 SQL → sqlglot 方言处理 → SQL 安全扫描 → 沙箱预检 → 执行 → 结果验证 → 审计日志`

### LangGraph 10+ 节点（顺序）
`clarification → decompose → rag_retrieve → generate_dsl → validate_dsl → correct_dsl → permission_check → resolve_semantic → build_sql → scan_sql → sandbox_check → execute_sql → verify_dsl → audit_log`

- 条件分支：`correct_dsl`（校验失败进修正）、`sandbox_check`（风险高走人工审核）、`execute_sql`（失败可走简化 DSL 重试）。
- State 字段：question、domain/user_id/tenant_id、检索上下文、当前 DSL、SQL、结果、权限、trace、错误、重试次数。节点只读写自己负责字段，避免强耦合。

### Agent 层 vs Graph 层
- **Agent 层**：查什么——意图识别、复杂查询拆解、子查询调度、结果聚合、自然语言解释。
- **Graph 层**：怎么查——单查询从 DSL 到 SQL 执行的稳定链路。
- 分开后，复杂查询编排与底层执行独立演进。无依赖子查询并行、有依赖串行，Aggregator 按意图（compare/trend/ranking/proportion）做 diff/growth_rate 等聚合。

### 失败处理原则
- **可重试/可修正**：DSL 生成失败、校验失败、召回不足、SQL 执行失败。
- **必须终止**：权限拒绝、危险 SQL、安全扫描失败、敏感字段越权、达最大重试次数。
- **原则**：语义不确定可澄清/修正，安全和权限问题不能靠重试绕过。

### correct_dsl 与死循环控制
读 DSL 校验错误 → 提取关键词做**定向 RAG 召回** → 带错误原因和补充上下文让 LLM 重新生成（非盲目重试）。死循环控制：`dsl_attempts` 递增超限终止；新 DSL 与旧 DSL 无变化视为无进展提前终止。

### 关键分工
- **指标口径**：由系统注册表决定，LLM 只选 alias（如 `sales_amount`），不能自发明 `SUM(pay_amount)`。SemanticResolver 把 alias 展开成表达式。
- **SQLAlchemy**：结构化构建 SQL，避免字符串拼接。
- **sqlglot**：方言解析/转换（MySQL/PG/ClickHouse/SQLite 日期、分页、引号差异）。

---

## 3. RAG / 检索 / Rerank

### 召回内容（五类）
Schema（表/字段/描述/Join）、Metrics（指标名/口径/表达式）、Terms（业务别名：流水→GMV）、History（历史问题+标准 DSL few-shot）、Permissions（必要时给权限校验上下文）。

### 三层检索分工
| 层 | 解决 | 工具 |
|---|---|---|
| 向量检索 | 语义相似（“业绩/流水/成交金额”→销售额） | BGE Embedding + Milvus Lite |
| 关键词检索 | 短词/精确匹配（GMV、销售额、华东） | jieba 分词 |
| Rerank | 召回候选精排 | BGE-Reranker（Cross-Encoder） |

- **Embedding 快但粗，Reranker 准但贵** → 先召回 TopK 再精排。
- **Reranker 替代 LLM 粗排**：LLM 粗排成本高/延迟大/输出不稳定；Reranker 专做相关性排序，延迟成本可控。

### 向量库选型口径（重点）
- **开源项目用 Milvus Lite**：轻量、部署简单、适合本地演示。
- **开源项目没有 Elasticsearch**。简历技能清单里的 Qdrant/ES 来自其他实践或技术覆盖，**不和当前项目技术栈混说**。
- 生产可混合：ES 做字段名/表名/枚举值/日志精确检索，向量库做语义召回，reranker 统一排序。

### 向量库记录设计
分类型存：schema（表/字段/描述/业务域）、metric（名/别名/口径/数据源）、term（术语/同义词/映射对象）、history（问题+标准 DSL）。metadata 至少含 `domain/type/name/data_source/version/enabled`，不能只存自然语言文本。**Schema、metrics、terms、history 分 collection**（长度/策略/更新频率/排序权重不同）。

### 性能口径（最容易被质疑，务必讲清）
- **800ms→80ms、成本降 90%**：指**重排序环节**，不是端到端请求。同候选规模、同问题集下，BGE-Reranker 替代 LLM 粗排的平均耗时和调用成本对比。
- **不是生产全链路 P95**。补充口径：候选集大小、是否缓存 embedding、是否含冷启动、机器环境、平均值/P95。
- **成本下降限重排序环节**，不是整套系统成本下降。
- **准确率 90%**：拆四维（Semantic/Planning/Execution/Governance）说，是评测集/开源样例集综合结果，不代表所有业务域。重点是有评测集和回归门禁，不是单看数字。

### 字段取值召回（避免扫大表）
优先字典表/维表/枚举配置 → 离线 DISTINCT/TopN 采样 → 前缀/关键词检索 → 分层索引。高基数字段和敏感字段加黑名单或采样限制；值域过大触发澄清。缓存 key：`domain + normalized_keyword + embedding_model_version`，TTL/版本号失效。

### RAG 召回不是唯一防线
召回错了会在：DSL 校验失败、SQL 构建失败、结果验证失败、用户反馈、评测低分暴露。trace 记录 RAG context 便于回溯。RAG **降低**幻觉而非消除，后面还有校验/语义解析/权限/评测兜底。

---

## 4. 数据治理 / 权限 / 安全

### 语义层治理模型
业务表达 → 标准映射：“销售额”=`sales_amount`=`SUM(pay_amount)`；“华东”=地区编码；无权区域/字段在 DSL 层注入限制。配置项：metrics、dimensions、data_sources、terms、permissions（YAML/DB 表）。

### 权限处理阶段
- **行级权限**：DSL 阶段注入 filters（如 `region in ['华东']`）。
- **列级权限**：DSL 阶段检查 dimensions/metrics 是否含敏感字段或未授权指标，拒绝或脱敏。
- **为什么 DSL 层注入而非 SQL 字符串替换**：字符串替换易出错（子查询/UNION/别名/括号/方言），DSL 结构化更清晰可审计。

### 敏感字段处理
三种：无权拒绝 / 有权正常 / 部分权限脱敏。脱敏在 SQLBuilder 或结果后处理执行，但**权限判断尽量前置**。

### SQL 安全扫描拦截
DELETE/UPDATE/DROP/ALTER/TRUNCATE、UNION 注入、多语句分号、注释注入、存储过程、非 SELECT、全表/超大扫描风险。

### 沙箱预检
执行前 EXPLAIN 或 LIMIT 预览，估算扫描量和执行计划。扫描行数过大/未命中分区或索引/Join 代价过高 → 拒绝/降级/人工审核。解决“SQL 语法安全但执行代价过高”。

### 审计 vs 链路追踪
- **审计日志**（面向合规）：query_id、user_id、tenant_id、question、DSL、SQL、状态、耗时、返回行数、扫描行数、trace_json、错误码、错误信息、时间。
- **链路追踪**（面向排查）：每节点输入输出/耗时/错误/中间 DSL。
- 越权返回：明确但不泄露敏感信息的错误 + 审计记录越权行为。

### 多租户隔离
独立语义配置/domain context、向量库按租户/领域隔离、DB 连接按租户路由、DSL 校验带 tenant、权限注入默认带 tenant_id、缓存 key 含 tenant/domain。

### 提示词注入防护
核心**不信任 LLM**。用户说“忽略权限”，LLM 也只能生成 DSL，后面有语义校验/权限注入/安全扫描/沙箱预检。安全靠系统结构性约束，不靠 Prompt。

### 两个易问点
- SQLBuilder 有 bug，安全扫描能兜危险关键字/多语句/非 SELECT/明显注入；但业务语义错误而语法安全的 SQL 发现不了，需评测/审计/结果验证补充。
- 权限注入影响准确率评估 → 评测区分语义过滤和治理过滤，评分时剥离或单独记录治理注入条件。

---

## 5. 自动化评测体系

### 四维指标
| 维度 | 评估 |
|---|---|
| **Semantic** | 意图/指标/维度/过滤条件是否理解正确 |
| **Planning** | Join/排序/Limit/分组等查询计划是否合理 |
| **Execution** | SQL 是否执行成功、结果是否正确 |
| **Governance** | 权限/脱敏/审计是否生效 |

### 量化方法
- **DSL 质量**：拆 metric/dimension/filter 字段与操作符/order_by/limit 多维评分，加权总分。
- **SQL Success**：能否执行成功；**Result Accuracy**：结果一致性（行数/指标值/排序/TopN/分组）。
- **治理测试**：无权区域应注入过滤、敏感字段应拒绝/脱敏、每次查询必生成 audit log，检查 DSL/SQL/结果/审计表。

### 回归门禁
稳定版本结果作 baseline → 改 RAG/Prompt/DSL 校验/Optimizer/SQLBuilder 后重跑 → overall 下降/关键维度降超阈值/新增失败用例/执行错误 → 阻断合并。新增规则看新增通过/失败用例和维度 diff。

### 测试集构造
人工核心场景 + 线上历史问题脱敏抽样 + 典型失败案例沉淀，每例只测一个主要能力。

### 评测失败定位（看 trace）
RAG context 错→召回；LLM 原始 DSL 错→Prompt/上下文；validate_dsl 失败→语义层/生成；build_sql 失败→SQLBuilder/语义解析；execution 错→DB/方言/数据；governance 错→权限配置/注入。

---

## 6. 数据资产与语义层元数据加工平台（NL2DSL 上游底座）

### 与 NL2DSL 的关系
NL2DSL 要准确理解自然语言，需知道有哪些表/字段/指标/维度/枚举/术语/权限，这些资产不能靠 LLM 猜，需元数据加工链路提前沉淀。产出供：Schema 召回、指标召回、术语召回、字段值召回、DSL 校验、权限注入、审计分析、评测集构造。

### 输入 → 加工 → 输出
- **输入**：DB 元数据、表字段 comment、指标配置、维表/字典表、业务术语、权限策略、历史查询样例。
- **加工**：字段标准化、质量统计（空值率/重复率/值域）、枚举值抽取、指标口径聚合、业务域归属、敏感字段标记、质量评分。
- **输出**：元数据宽表、指标元数据表、术语表、字段值样例表、权限配置表、同步到向量库的可检索文本。

### 元数据宽表核心字段
`domain、table_name、column_name、column_type、column_comment、business_desc、metric_alias、dimension_alias、value_samples、is_sensitive、data_source、update_time、quality_score`

### 指标元数据表
`metric_id、metric_name、metric_alias、business_desc、expr、unit、data_source、time_field、owner、enabled、version、created_at、updated_at`（关键是唯一 alias + 标准表达式 + 版本管理）。

### 维度枚举采集优先级
字典表/维表同步 → 人工配置 → 离线 DISTINCT → 高频 TopN 采样。高基数/敏感字段不全量枚举。

### 字段注释质量差
质量评分（过短/拼音缩写/纯英文缩写/无业务含义标低质量）→ 人工补充或 AI 辅助改写生成业务描述，**生产使用前必须人工审核**。

### 同步给向量库
转可嵌入文本 → embedding → 写 Milvus collection，保留 domain/type/name metadata，支持全量重建和增量更新。

### 数据质量校验规则
空值率、重复率、枚举合法性、数据量波动、字段覆盖率、主键唯一性、分区完整性、指标结果波动、源表字段变更。

### Spark 调优口径（不硬编数字）
- **大表 Join**：看 Spark UI stage 耗时/shuffle read-write/task 倾斜/输入分区；判断广播小表、join key 倾斜、分区裁剪、历史分区扫描过多。
- **分区扫描过大**：增加时间分区过滤、分区裁剪、避免函数包裹分区字段、按业务域拆任务、只处理增量。
- **数据倾斜**：某些 task 明显慢/shuffle read 特别大 → 加盐、两阶段聚合、广播小表、过滤热点 key、热点单独处理。
- **历史重复计算**：按分区增量、记录任务水位、幂等写入，口径变更或修复才重跑。
- **失败重跑幂等**：按业务日期+批次重跑，先写临时表/临时分区，校验通过再覆盖正式分区，避免重复 append。
- **源表字段变化**：定时抽 DB 元数据对比上次 schema 快照，生成变更报告触发语义层同步。
- **指标口径变化**：版本管理，新口径上线前跑评测确认影响，历史审计记录当时版本。

### 数仓与 Spark 基础（速记，简历离线为主）

- **数仓分层**：ODS（贴源/按天分区）→ DWD（清洗明细/粒度统一）→ DWS（主题预聚合）→ ADS（应用/报表）。目的：复用、解耦、可控、性能。元数据宽表 ≈ DWS/ADS 性质。
- **建模**：维度建模（Kimball，星型/雪花，OLAP 友好）vs 范式建模（Inmon，三范式，规范但 join 多）。实际多混合：底层规范、上层星型宽表。
- **事实/维度/SCD**：事实表存度量+外键（事务型/周期快照/累积快照）；维度表存属性；SCD2 最常用（起止时间保留历史），SCD1 覆盖、SCD3 加历史列。
- **指标体系**：原子指标（度量+业务过程，如下单金额）→ 派生指标（+修饰词+时间周期，华东近7天下单金额）→ 复合指标（转化率/同环比）。统一 alias+表达式+版本，和 NL2DSL 注册表同一套。
- **Spark 抽象**：RDD（底层/强类型/无 schema）< DataFrame（schema/Catalyst 优化）< DataSet（强类型）。平时用 DataFrame/Spark SQL。
- **执行流程**：Driver 构建 DAG → DAGScheduler 按 shuffle 切 Stage → TaskScheduler 调度 Task 到 Executor。窄依赖同 Stage 流水线，宽依赖切 Stage。
- **Shuffle**：上游按 key 写本地盘（write）→ 下游拉取（read）。瓶颈：磁盘 IO+网络+序列化+排序+易倾斜。减 shuffle（广播 join）、减数据量（先过滤）、治倾斜。
- **资源**：num-executors / executor-cores / executor-memory / shuffle 分区数。每 task 128MB~1GB；按 Spark UI（task 数/利用率/shuffle/GC）调，不盲目堆内存。动态分配适合资源波动。
- **广播变量**：小表/配置广播到 Executor，broadcast hash join 避免 shuffle。**累加器**：分布式只增计数（脏数据条数），action 后才更新。
- **小文件**：分区过多/分区数过大/流式写入 → NN 压力+调度开销+seek。解：coalesce（缩分区不 shuffle）/ repartition（扩/重分布 shuffle）/ Hive merge / 控分区粒度 / ORC+Parquet 列存。coalesce 不能可靠扩。
- **文件格式**：ORC（Hive 生态/压缩比高/带索引）vs Parquet（Spark 生态/通用）。列存做列裁剪+谓词下推。压缩 Snappy（快/中）平衡首选，Zlib/Gzip 高比慢，LZO 可切片。
- **SQL 调优**：分区裁剪（WHERE 用分区字段别包函数）、谓词下推（过滤靠近源）、列裁剪（别 select *）、mapjoin（小表广播免 shuffle）。加：避免笛卡尔积、join on 等值、先过滤再聚合。
- **数据质量**：准确性/完整性/一致性/及时性/唯一性。规则：空值率/重复率/枚举合法/数据量波动/主键唯一/分区完整/指标波动。校验不达标告警或阻断下游。
- **调度**：DolphinScheduler/Airflow/Azkaban。管定时+依赖+重试+SLA。避免环形依赖、跨层依赖、长链路单点。
- **AQE**（Spark3）：运行时合并小 shuffle 分区、自动转 broadcast join、自动处理倾斜。`spark.sql.adaptive.enabled=true`。
- **Spark 内存**：reserved（预留）+ user memory + spark memory（execution/storage 动态借还，`spark.memory.fraction` 控制）。理解能调 OOM/spill/cache 占用。
- **Kryo**：比 Java 序列化快且紧凑，需注册类；shuffle 默认 Kryo；closure 可能仍用 Java。
- **倾斜方案**：加盐/两阶段聚合/广播小表/过滤热点 key/增 shuffle 分区（对单 key 无效）/自定义 Partitioner/AQE 自动倾斜/热点单独处理再 union。加盐副作用：改 key 语义需两阶段还原。
- **分桶 vs 分区**：分区按目录裁剪（粗粒度过滤），分桶按 hash 拆固定文件（bucket map join/采样/均匀分布）。可叠加：按天分区+按 user_id 分桶。
- **大表 join 大表**：先过滤/列裁剪/提前聚合、分桶表 bucket map join、SMB join、Skew Join、热点拆分 union、维度退化预计算宽表（最稳）。
- **血缘**：SQL AST 解析（sqlglot/Calcite）/调度集成/Hook 采集。用途：影响分析、根因定位、合规、下线评估。帮 NL2DSL 知道指标物理来源，辅助 Schema 召回+审计。
- **一致性维度/总线矩阵**：跨主题域统一维度保证口径一致；总线矩阵规划业务过程×维度，保证可扩展、复用。
- **实时口径（不硬讲）**：Structured Streaming 微批+watermark+exactly-once；Lambda 批流双链路（两套代码口径难统一）、Kappa 纯流（重算成本高）。简历离线为主，倾向批为主按需补流，不强行上 Kappa。

---

## 7. Vibe Coding / AI 辅助开发

### 人机分工
- **AI 做**：代码初稿、测试样例、文档整理、错误排查思路、样板代码、重复性实现。
- **必须自己把关**：架构边界、业务口径、安全策略、数据权限、异常处理、性能取舍、最终代码质量。
- **定位**：AI 辅助工程实践，不是“AI 替我做项目”。你负责定义问题/拆任务/定接口/Review 关键逻辑/补测试/回归验证。

### 质量控制
任务拆小（如“给 DSL Validator 增加未知指标校验”“行级权限注入”“Rerank 耗时统计”）→ 给 AI 明确输入输出/已有文件路径/不能改的边界/验收标准 → Review（接口契约/破坏已有逻辑/异常路径/权限安全绕过/测试覆盖/过度抽象）→ 跑单测+集成+E2E+评测回归。

### 典型 AI 写错例子（建议准备）
AI 初稿只校验 DSL 的 JSON 结构（Pydantic），忽略业务语义校验（metric 是否注册、dimension 是否属当前 data_source、filter 操作符与字段类型匹配）。处理：拆两层——结构校验 + 语义校验（基于注册表）；失败不盲目重试，错误原因带回 correct_dsl 带补充召回修正。

### 现场改需求示例
加 `time_range` 字段：先改 DSL Schema → 改 Prompt 约束和 few-shot → 改 Validator 检查时间格式和字段归属 → 看 SemanticResolver/SQLBuilder 是否展开成时间过滤 → 补评测样例和回归用例。靠对链路理解，不靠 AI 记忆。

---

## 8. Java 后端基本盘（不要让面试官觉得在追热点）

8 年 Java，AI Agent 链路用 Python，但后端经验是基本盘：接口、MQ、DB、缓存、并发、部署、线上排查迁移到 AI 项目很直接（超时控制、审计、异常分类、权限、缓存、降级）。

- **RabbitMQ**：高吞吐抓拍接入、异步解耦、削峰填谷。不丢：生产 confirm + 队列/消息持久化 + 消费手动 ack + 失败重试/死信 + 业务幂等。积压排查：生产/消费速率、队列堆积、消费者异常、DB 写入耗时、下游延迟 → 扩容/批量写入/预取控制/限流。
- **幂等**：业务唯一键/消息 ID/DB 唯一索引/状态机，重复消息先查是否已处理。
- **WebSocket 权限推送**：连接绑 user_id/角色/权限范围，告警按区域/设备/组织过滤推送，连接信息存内存或 Redis，断线清理。
- **2000 万底库导入**：分批读取、批量写入、多线程、控制事务大小、避免一次加载内存，失败批次可重试，记录进度。线程数按 CPU/IO/DB 连接池定，控制 batch size 避免长事务。
- **Zookeeper**：SDK 算法服务注册/发现/健康检查/负载均衡，扩容下线自动感知。
- **Arthas**：dashboard 看 JVM、thread 看线程、jad 反编译、watch 看入参返回、trace 看耗时、stack 看调用栈、heapdump 内存分析。
- **CPU 飙高排查**：top 找进程 → `top -H`/Arthas thread 找热点线程 → 看栈（业务循环/序列化/正则/DB 轮询/GC）→ trace 看耗时 → 临时止血（限流/重启/扩容）+ 长期修复。
- **JVM 排查**：CPU/内存/GC/线程；jstack/jmap/GC 日志；CPU 高看热点线程，内存高看对象分布，接口慢看 trace，死锁看 thread。
- **降级**：超时控制/重试/熔断/切备用节点/消息暂存/告警延迟处理，不让 SDK 不可用拖垮接入链路。

---

## 9. 重点准备的 10 个答案

1. **3 分钟介绍 NL2DSL**：业务问题→架构方案→我的职责→结果沉淀，不堆技术栈。
2. **为什么 NL2DSL 比 NL2SQL 适合企业治理**：可校验、权限可注入、口径可控、SQL 可构建、审计可追踪。
3. **自然语言查询完整链路**：问题→Agent/Graph→RAG→DSL→校验→权限→语义解析→SQL→安全→执行→审计。
4. **DSL Schema 与校验**：data_source/metrics/dimensions/filters/order_by/limit/post_process 例子+校验项。
5. **RAG 召回方式**：schema/metrics/terms/history 四类用途与示例。
6. **权限与安全协同**：行级注入 filters、列级查敏感字段、SQL 安全扫描、沙箱预检控制扫描量。
7. **自动化评测**：四维评分 + baseline + 回归门禁。
8. **元数据加工如何支撑 NL2DSL**：提供 Schema/指标/维度/术语/枚举/权限，是 RAG、DSL 校验、权限注入上游。
9. **Vibe Coding 质量控制**：AI 提效，你负责架构/边界/Review/测试/质量，准备具体案例。
10. **亲手解决的复杂问题**（四选一）：Rerank 优化 / DSL 修正闭环 / 权限注入 / 评测回归。结构：问题背景→根因→方案→验证→复盘。

---

## 10. 口径红线（“不要这么说”汇总）

- ❌ 公司项目和开源项目“基本一样”。
- ❌ 把开源完整能力说成公司生产已全部上线。
- ❌ 把 Demo/开源验证/内部工具/生产系统混成一个说法。
- ❌ 说“生产环境稳定 80ms”，除非有完整压测数据。
- ❌ 把局部 rerank 成本下降说成整套系统成本下降。
- ❌ 只报准确率数字，不讲评测集/评分维度/失败样本。
- ❌ 把 Milvus Lite、Qdrant、Elasticsearch 说成同一项目里都用了。
- ❌ 说“RAG 能解决幻觉”（应说降低幻觉，后面还有校验和评测）。
- ❌ 把所有失败都说成“重试一下”（权限/安全类必须终止）。
- ❌ 说“基本都是 AI 写的，我负责调 Prompt”。
- ❌ 把 AI 生成代码天然当正确代码（尤其权限/安全/SQL 构建）。
- ❌ 只背目录结构，讲不出请求流和数据流。
- ❌ 硬编接入业务域/用户/QPS/数据量/优化百分比。
- ❌ 只说“做过 Spark SQL 清洗”，讲不出输入表/输出表/字段/下游用途。
- ❌ Java 经验只背八股，不结合消息积压/底库导入/WebSocket 权限推送/现场部署。
- ❌ 只背 Spark/数仓八股，说不出 RDD/Stage/Shuffle/资源调优和自己元数据加工任务的关联。
- ❌ 把实时流/Flink/Structured Streaming 包装成主力方向（简历是离线为主，按“了解原理”讲，不硬上 Kappa）。

---

## 11. 现场讲代码三条路线

1. **API → Graph**：FastAPI 查询接口接收问题/身份/业务域 → Agent（意图/拆解/调度/解释）→ Graph（clarification…audit_log 全节点围绕同一 State 读写）。
2. **DSL → SQL**：generate_dsl → validate_dsl（结构+语义两层）→ correct_dsl（带错误定向召回）→ permission_check → resolve_semantic（alias 展开表达式）→ build_sql（SQLAlchemy）→ scan_sql → sandbox_check → execute_sql。
3. **RAG → Rerank**：rag_retrieve 召回 schema/metrics/terms/history → 分块注入 Prompt（每类 TopK，token 预算内，注册表为权威）→ Reranker 精排 → 不足时触发 correct_dsl 补充召回。

开源项目目录：`nl2dsl/agent`、`nl2dsl/graph`、`nl2dsl/dsl`、`nl2dsl/rag`、`nl2dsl/permission`、`nl2dsl/sql_engine`、`nl2dsl/evaluation`、`web`。
