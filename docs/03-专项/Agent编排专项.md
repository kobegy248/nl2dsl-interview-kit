# Agent 编排专项（已实现）

> **证据边界**：本文只讲**已实现**的编排能力（意图识别、任务拆解、子查询调度、插件扩展），有 `file:line` 证据。
> 「工具调用」「Function Calling」「MCP」等**未实现**内容，见 [工具调用与 MCP 设计题](../04-设计题/工具调用与MCP设计题.md)。

---

## 0. 三句话（开场口径）

1. **我的 Agent 是"编排型"，不是"自主型"**：LLM 负责意图识别、查询规划和 DSL 生成，**执行由系统组件完成，模型不直接调用任何工具**。
2. **这不是能力缺失，是场景选择**：SQL 要过校验、权限、扫描、沙箱四道闸，交给模型自主调用就等于放弃可治理性。
3. **并行 + 依赖 + 限流这套调度我做过**（dispatcher），只是触发方是 planner 产出的 Plan，不是模型的 tool_call。

---

## 一、分工：模型做什么，系统做什么

```
用户问题
  ├─ Agent 层（LLM 参与）
  │   ├─ plan：意图识别（7 类，配置化）+ 任务拆解
  │   ├─ decompose：复杂问题改写
  │   └─ dispatch：子查询调度（并行 + 依赖串行）
  │
  └─ Graph 层（LLM 不参与执行）
      ├─ validation / permission_check / optimize_dsl
      ├─ confidence（LLM 只打分，不做决定）
      ├─ build_sql → scan_sql → sandbox_check → execute_sql
      └─ verify_dsl / explain（LLM 只自检和解释）
```

**模型出现的位置**：生成 DSL、打分、自检、解释。
**模型不出现的位置**：校验、权限、构建 SQL、执行、审计。

> 一句话：**"模型负责'想查什么'，系统负责'怎么安全地查'。"**

---

## 二、意图识别：7 类，配置化

`configs/intents.yaml` + `agent/strategies.py`

| 意图 | 关键词示例 | 拆解策略 | 聚合方式 |
|---|---|---|---|
| compare | 对比 / 比较 / vs / 同比 / 环比 | split_by_objects | diff |
| trend | 趋势 / 走势 / 变化 / 增长 / 下降 | single_with_time_grouping | trend_direction |
| correlation | 关联 / 影响 / 相关 / 关系 | split_by_objects | pearson |
| proportion | 占比 / 构成 / 贡献度 | total_plus_groups | proportion |
| sequential | 先查 / 然后 / 再查 / 接着 | sequential | sequential_filter |
| ranking | 排名 / Top / 第几 | single_with_ordering | ranking |
| single_query | （兜底） | passthrough | passthrough |

**关键设计**：意图是 **YAML 配置驱动的**（`IntentRegistry.load`，`strategies.py:31-48`）——加一类意图不用改代码，改配置即可。

> 面试口径：**"意图识别不是硬编码 if-else，是配置表 + 关键词匹配。新增'归因分析'这类意图，只需要在 YAML 里加一段。"**

---

## 三、任务拆解与调度

### 3.1 拆解：Plan + SubQuery

planner 把复杂问题拆成多个 `SubQuery`，每个带 `depends_on` 依赖声明。

### 3.2 调度：并行 + 依赖串行 + 限流

`agent/dispatcher.py:136` `dispatch_sub_queries`：

```
Phase 1：无依赖的子查询 → asyncio.gather 并行（Semaphore 限流）
Phase 2：有依赖的子查询 → 依赖满足后才执行
         依赖失败 → 跳过并标记 error，不阻塞其他分支
```

三个工程细节：

1. **信号量限流**（`MAX_PARALLEL_SUB_QUERIES`），避免一次拆 20 个子查询打爆数据库；
2. **依赖失败传播**（`dispatcher.py:199` "deps_satisfied"），下游跳过而不是报错崩溃；
3. **每个子查询独立跑图**（`_execute_sub_query`），状态隔离。

> 这套调度和工具调用框架里的"并行 tool call"是同一类问题——**只是触发方从模型换成了 planner**。这句话能直接化解"你没做过工具调用"的质疑。

### 3.3 聚合与解释

- `agent/aggregator.py`（294 行）：按聚合方式（diff / trend_direction / proportion / ranking）合并子查询结果；
- `agent/explainer.py`（285 行）：把结果转成自然语言结论；
- `agent/resolver.py`：解析子查询之间的引用关系；
- `agent/confidence.py`：质量评分（syntax + semantic），决定继续/告警/澄清。

---

## 四、扩展机制：插件与协议

### 4.1 Plugin / Pipeline（`plugin.py`）——节点级扩展

```python
class Pipeline:
    def before(self, node, handler)   # 节点前插一段
    def after(self, node, handler)    # 节点后插一段
    def replace(self, node, handler)  # 替换节点
    def add_node(self, name, handler, after)  # 新增节点
```

这是**给开发者扩展系统用的**，不是给模型调用的。区分点很清楚：

> **插件由开发者在编译期注册，工具由模型在运行期决定调用。**

### 4.2 Protocol 组件契约（`protocols.py`）

`LLMBackend` / `DSLGenerator` / `SQLBuilder` / `SQLExecutor` / `SQLScanner` / `Validator` / `SemanticResolver` / `QuerySandbox` / `ClarificationDetector`——**内部依赖注入的接口**，不是暴露给模型的函数描述（没有 name/description/JSON Schema）。

> 面试时这条能防一个坑：如果面试官问"你有工具注册表吗"，要分清"组件注册表（Registry）"和"工具注册表（Tool Registry）"不是一回事。

---

## 五、这个实现的已知边界

| 边界 | 说明 |
|---|---|
| 无工具调用 | `bind_tools` / `ToolNode` / `tool_call` 全仓无 |
| 意图识别靠关键词 | 没有 LLM 兜底，覆盖不到的说法会落回 `single_query` |
| 拆解质量依赖 LLM | planner 拆错了没有二次校验 |
| 子查询无缓存 | 相同语义重复检索 |
| 聚合规则固定 | 只支持 6 种聚合方式，新的分析类型要改代码 |

---

## 六、相关追问（只回答"你怎么做的"）

**Q1：为什么 SQL 不做成 tool call？**
> "会让决策和执行耦合。做成固定 Graph 后：生成的是可校验的 DSL；权限在 DSL 层精确注入；每步都有 trace 能定位；执行前必过扫描、沙箱、人审。工具调用把这三段揉成一段，出错没法归因。"

**Q2：7 类意图是怎么识别的？**
> "配置驱动。`intents.yaml` 给每类意图配关键词、拆解策略、聚合方式，查询时关键词匹配，匹配不到落回 single_query。"

**Q3：复杂查询怎么并行？**
> "planner 拆成带依赖声明的子查询，无依赖的用 asyncio.gather 并行、信号量限流；有依赖的等前置完成再跑；依赖失败直接跳过并标记。"

**Q4：为什么子查询不共享上下文？**
> "子查询语义可能完全不同（'对比华东和华南'），共享召回上下文会互相污染。各自独立检索更干净，代价是一次重复检索。"

**Q5：插件和工具调用的区别？**
> "插件是开发者在编译期注册的节点扩展，模型看不见；工具调用是模型在运行期决定调什么。前者改系统行为，后者改模型的行动能力。"

---

## 七、速查

| 项 | 值 |
|---|---|
| Agent 类型 | 编排型（LLM 规划 + 系统执行） |
| 意图 | 7 类，`intents.yaml` 配置驱动 |
| 拆解 | Plan → SubQuery（带 depends_on） |
| 调度 | 无依赖并行（Semaphore 限流）+ 依赖串行 + 失败传播 |
| 聚合 | diff / trend_direction / proportion / ranking 等 |
| 扩展 | Plugin/Pipeline（节点级）、Protocol（组件契约） |
| 已知边界 | 无 tool call / 意图靠关键词 / 拆解无二次校验 |
| 下一步 | 见 [工具调用与 MCP 设计题](../04-设计题/工具调用与MCP设计题.md) |
