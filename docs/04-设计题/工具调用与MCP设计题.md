# 工具调用与 MCP 设计题（未实现）

> **证据边界**：NL2DSL 项目里**没有任何工具调用实现**（全仓无 `bind_tools` / `ToolNode` / `tool_call`）。
> 本文是**设计方案**，面试口径必须是「我们没做，但我知道该怎么做、什么时候该做」。
> 已实现的编排能力见 [Agent 编排专项](../03-专项/Agent编排专项.md)。

---

## 0. 定位

一句话：**"工具调用这一层我是空的，但我清楚两件事——为什么现在不做，以及如果要做先做哪三个。"**

---

## 一、缺口清单

| 能力 | 现状 |
|---|---|
| `bind_tools` 注入工具 schema | ❌ 无 |
| 工具注册表（name / description / JSON Schema） | ❌ 无 |
| `ToolNode` 执行并回填结果 | ❌ 无 |
| tool_call 解析 + 参数校验 + 失败重试 | ❌ 无 |
| 工具调用审计 | ❌ 无（审计只覆盖查询链路） |
| MCP 接入 | ❌ 无 |

---

## 二、未采用的技术

### 2.1 Function Calling 原理

**模型不执行函数，只输出结构化意图**：

```json
{"role": "assistant", "tool_calls": [
  {"id": "call_1", "type": "function",
   "function": {"name": "explain_metric", "arguments": "{\"metric_alias\": \"gmv\"}"}}
]}
```

应用执行后把结果作为 `role: tool` 消息回传，模型再据此生成最终回答。

| 模式 | 说明 | 适用 |
|---|---|---|
| 并行工具调用 | 一次返回多个 tool_call，应用并发执行 | 多个独立查询 |
| 串行（ReAct 循环） | 调一个 → 看结果 → 再决定下一个 | 依赖前序结果 |

### 2.2 LangGraph 集成

```python
model_with_tools = llm.bind_tools([...])

builder.add_node("agent", call_model)
builder.add_node("tools", ToolNode(tools))
builder.add_conditional_edges("agent", tools_condition)  # 有 tool_call → tools
builder.add_edge("tools", "agent")                       # 结果回填后回到模型
```

关键理解：**ToolNode 只是"读 state 里的 tool_calls → 执行 → 写回 ToolMessage"的节点**，智能在模型决策里。

### 2.3 MCP

| 概念 | 说明 |
|---|---|
| 定位 | 工具/数据源变成可复用服务，被多个 LLM 客户端共享 |
| 角色 | Server 暴露能力；Client 发现并调用 |
| 三原语 | `tools` / `resources` / `prompts` |
| 传输 | `stdio`（本地） / SSE·Streamable HTTP（远程） |
| 与 FC 关系 | MCP 是协议层，底层仍可映射成 Function Calling |

> 一句话区别：**Function Calling 是单应用内嵌函数，MCP 是跨应用的标准协议。**

### 2.4 工具设计原则

| 原则 | 说明 |
|---|---|
| name 用动词+名词 | `explain_metric` 而不是 `metric_tool` |
| description 写"什么时候用" | 模型靠 description 决策 |
| 参数用 enum | 能收敛就收敛，防幻觉 |
| 错误信息写给模型看 | "指标不存在，可用：…" 比 `KeyError` 有用一万倍 |
| 返回值小而结构化 | 给摘要 + 关键字段，别塞整张表 |

### 2.5 五个工程问题

| 问题 | 做法 |
|---|---|
| 超时 | 每工具独立超时，超时返回可读错误 |
| 重试 | 只对幂等工具重试，带退避、有上限 |
| 幂等 | 写操作带 `idempotency_key` |
| 权限 | 工具级 RBAC + 参数级校验 |
| 审计 | 谁、何时、调了什么、参数、结果、耗时 |

### 2.6 何时用工具调用 vs 固定工作流

| 场景 | 该用 |
|---|---|
| 流程确定、强治理、强安全（SQL 查数） | 固定工作流 |
| 工具种类多且会变、需模型灵活决策 | 工具调用 |
| 步骤依赖由结果决定（探索式） | 工具调用 |
| 每步都要审计和权限校验 | 固定工作流 |

### 2.7 安全风险

| 风险 | 缓解 |
|---|---|
| 参数注入 | 参数 enum + 服务端二次校验 |
| 越权调用 | 工具级 RBAC + 最小工具集 |
| 间接注入（工具返回值夹带指令） | 返回值当数据处理，输出侧校验 |
| 工具链爆炸 | 调用次数上限 + 预算熔断 |
| 不可逆动作 | HITL + 幂等键 + 可回滚 |

---

## 三、怎么补

### 3.1 先定边界：哪些能力配做成工具

三条全中才做：

```
① 只读、幂等
② 不需要经过 DSL 校验链（不是查询执行）
③ 模型需要动态决定要不要调——不是每次都必调
```

按这个标准，**SQL 执行不该做工具**，而这三个非常适合：

| 工具 | 用途 | 为什么适合 |
|---|---|---|
| `explain_metric(metric_alias)` | 查指标口径与计算方式 | 用户常问"这个数怎么算的"，按需调用 |
| `list_dimensions(data_source)` | 查可用维度 | 生成前探索，减少校验失败 |
| `sample_values(dimension, limit)` | 查维度取值分布 | 帮确认过滤值编码 |

共同点：**它们是"查元数据"，不是"查数据"**——不涉及行级权限，天然安全。

### 3.2 方案 A：最小可用（2~3 天）

**Step 1 工具注册表**

```python
@dataclass
class Tool:
    name: str
    description: str          # 写给模型的"什么时候用"
    schema: dict
    handler: Callable
    timeout_s: float = 5.0
    idempotent: bool = True

class ToolRegistry:
    def schemas(self) -> list[dict]:
        return [{"type": "function",
                 "function": {"name": t.name, "description": t.description,
                              "parameters": t.schema}}
                for t in self._tools.values()]
```

**Step 2 指标解释工具**

```python
schema = {
    "type": "object",
    "properties": {
        "metric_alias": {"type": "string", "enum": sorted(metrics.keys()),
                         "description": "必须是已注册指标"}
    },
    "required": ["metric_alias"],
    "additionalProperties": False,   # ★ 防模型塞多余参数
}

def handler(metric_alias: str) -> dict:
    cfg = metrics.get(metric_alias)
    if not cfg:
        return {"error": f"指标 '{metric_alias}' 不存在",
                "available": sorted(metrics.keys())[:20]}   # ★ 写给模型看
    return {"metric": metric_alias, "expr": cfg.get("expr"),
            "description": cfg.get("description")}
```

**Step 3 接入图（与固定链路并存）**

```python
builder.add_conditional_edges("plan", route_after_plan, {
    "continue": "decompose",
    "agent": END,
    "explore": "tool_agent",     # ← 需要查元数据时才进
})
builder.add_conditional_edges("tool_agent", tools_condition)
builder.add_edge("tools", "tool_agent")
builder.add_edge("tool_agent", "decompose")   # 探索完回主链路
```

**Step 4 审计与限流**

```python
for call in state["messages"][-1].tool_calls:
    audit.log(query_id=..., user_id=..., event="tool_call",
              tool=call["name"], args=call["args"], ts=time.time())
if len(state["tool_calls_used"]) > MAX_TOOL_CALLS:
    raise Error("工具调用次数超限")
```

### 3.3 方案 B：MCP 化（1 周）

```
nl2dsl-mcp/
├── server.py
└── tools/{explain_metric,list_dimensions,sample_values}.py
```

```python
app = Server("nl2dsl-metadata")

@app.list_tools()
async def list_tools():
    return [{"name": "explain_metric", "description": "...",
             "inputSchema": METRIC_SCHEMA}]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "explain_metric":
        return await explain_metric(**arguments)
```

价值：IDE（Cursor/Claude Desktop）和其他 Agent 都能复用同一套元数据能力。

### 3.4 方案 C：完整（1 个月）

1. **工具分层**：只读元数据（无限制）→ 只读数据（走后端服务，不直连库）→ 写类（HITL + 幂等键）。
2. **工具网关**：统一鉴权、限流、审计、超时、熔断。
3. **工具评估**：选择准确率、参数正确率、调用次数分布，纳入评测。
4. **预算熔断**：单会话工具调用次数 + token 成本双上限。
5. **回归用例**：每加一个工具补三条（正确调用 / 不该调用 / 参数错误）。

---

## 四、设计类追问

**Q1：你们 Agent 支持工具调用吗？**
> "不支持，是编排型 Agent。但这不是能力缺失，是场景选择：SQL 执行必须固定链路走校验和权限。元数据查询这类能力我设计了工具化方案，只是还没落地。"

**Q2：Function Calling 原理？**
> "模型不执行函数，只输出 tool_call（函数名 + 参数 JSON）；应用执行后把结果作为 tool 消息回传，模型再生成最终回答。核心是'模型负责决策，应用负责执行'。"

**Q3：参数解析失败怎么办？**
> "三层：strict schema 保证合法 JSON；解析失败把错误回灌重试（最多 2 次）；仍失败就降级——当没调工具或转澄清。参数用 enum 能提前消灭大部分失败。"

**Q4：工具怎么保证幂等？**
> "读类天然幂等；写类必须带幂等键（客户端生成、服务端去重），同样 key 第二次调用返回第一次结果，而不是再写一次。"

**Q5：工具会不会越权？**
> "两道关：工具级 RBAC（这个角色能调哪些工具）+ 参数级白名单（服务端二次校验，不信模型传入值）。"

**Q6：MCP 和 Function Calling 什么关系？**
> "FC 是单应用内函数调用；MCP 把工具做成服务、用标准协议暴露，可被多客户端复用。MCP 底层仍可映射到 FC。"

**Q7：为什么不全用自主调用，省得写工作流？**
> "自主调用的代价是确定性和可审计性。查数要求同样问题给同样 SQL、同样权限、同样审计记录，固定工作流能保证。探索型任务才适合自主。"

**Q8：怎么评测工具调用能力？**
> "三个指标：选择准确率（该调没调 / 不该调却调）、参数正确率、任务完成率。用例必须包含'不该调'的反例，否则模型会倾向乱调。"

---

## 五、速查

| 问题 | 答案 |
|---|---|
| 有 tool call 吗 | 没有（`bind_tools`/`ToolNode`/`tool_call` 全仓无） |
| 为什么 SQL 不做工具 | 会耦合决策与执行，失去校验/权限/审计/归因 |
| 什么该做成工具 | 只读、幂等、不越权的**元数据查询** |
| 三个候选工具 | explain_metric / list_dimensions / sample_values |
| 设计要点 | 动词命名、description 写"何时用"、参数 enum、错误给模型看 |
| 五个工程问题 | 超时 / 重试 / 幂等 / 权限 / 审计 |
| 落地顺序 | 注册表 → ToolNode → 审计限流 → MCP 化 → 工具评估 |
| 一句话 | **"模型负责想查什么，系统负责怎么安全地查——这条线不能因工具调用而模糊。"** |
