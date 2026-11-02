# 工具调用与 MCP 设计题（未实现 · 教学版）

> **证据边界**：NL2DSL 项目里**没有任何工具调用实现**（全仓无 `bind_tools` / `ToolNode` / `tool_call`）。
> 本文是**设计方案与教学**，面试口径必须是「我们没做，但我知道该怎么做、什么时候该做」。
> 已实现的编排能力见 [Agent 编排专项](../03-专项/Agent编排专项.md)。
>
> **怎么读**：从"一次工具调用到底发生了什么"开始，逐层到 schema 设计、错误处理、MCP、落地代码。

---

## 0. 一句话先讲透

**模型不会执行函数，它只会输出一段"我想调用某个函数、参数是这些"的 JSON。真正执行的是你的代码。**

```
模型产出意图（tool_call） → 你的代码执行 → 结果回传模型 → 模型继续
```

所有工具调用框架（OpenAI FC、LangChain ToolNode、MCP）都是这一句话的不同包装。

---

## 1. 一次工具调用，消息是怎么流转的

假设用户问："GMV 这个指标是怎么算的？"

### 第 1 步：发请求，带上工具清单

```json
{
  "model": "deepseek-chat",
  "messages": [
    {"role": "system", "content": "你是数据助手。"},
    {"role": "user", "content": "GMV 是怎么算的？"}
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "explain_metric",
        "description": "查询某个业务指标的口径和计算方式。当用户询问指标怎么算、口径是什么时使用。",
        "parameters": {
          "type": "object",
          "properties": {
            "metric_alias": {"type": "string", "enum": ["gmv", "sales_amount", "customer_count"]}
          },
          "required": ["metric_alias"],
          "additionalProperties": false
        }
      }
    }
  ]
}
```

### 第 2 步：模型返回的不是答案，是"调用意图"

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {"name": "explain_metric", "arguments": "{\"metric_alias\":\"gmv\"}"}
    }
  ]
}
```

注意三点：

- `content` 是 `null`——模型这轮没说话，只想调工具；
- `arguments` 是**字符串**，不是对象——所以你要 `json.loads` 一次；
- `id` 是这次调用的唯一标识，回传结果时必须带上。

### 第 3 步：你的代码执行，把结果作为 `tool` 消息追加

```python
result = explain_metric(metric_alias="gmv")
# {"metric": "gmv", "expr": "SUM(pay_amount)", "description": "含税成交总额"}
```

```json
{"role": "tool", "tool_call_id": "call_abc123", "content": "{\"metric\":\"gmv\",\"expr\":\"SUM(pay_amount)\"}"}
```

### 第 4 步：带着完整消息历史再请求一次

```json
{
  "messages": [
    {"role": "system", ...},
    {"role": "user", "content": "GMV 是怎么算的？"},
    {"role": "assistant", "tool_calls": [...]},
    {"role": "tool", "tool_call_id": "call_abc123", "content": "..."}
  ],
  "tools": [...]
}
```

这次模型返回自然语言答案。

> **关键理解**：工具调用是一个**循环**，不是一次请求。每轮都要把全部历史带上，所以 token 会随工具调用次数增长——这是成本控制的重点。

---

## 2. 三种模式

### 2.1 单次调用

模型一次只返回一个 tool_call，执行后直接出答案。最简单，适合"查到就行"。

### 2.2 并行调用

模型一次返回多个 tool_call（`tool_calls` 是数组）：

```json
"tool_calls": [
  {"id": "c1", "function": {"name": "explain_metric", "arguments": "{\"metric_alias\":\"gmv\"}"}},
  {"id": "c2", "function": {"name": "explain_metric", "arguments": "{\"metric_alias\":\"sales_amount\"}"}}
]
```

你的代码可以 `asyncio.gather` 并发执行，把多个 tool 消息一次性回传。

> 你的 `dispatcher.py` 做的就是同一件事（并行 + 依赖），只是触发方是 planner 而不是模型。

### 2.3 串行（ReAct 循环）

调一个 → 看结果 → 再决定下一个。适合"下一步依赖上一步结果"：

```
用户：哪个地区的 AOV 最高？
→ tool_call: list_dimensions(data_source="orders")
→ 结果：region, channel, brand...
→ tool_call: run_metric(metric="avg_order_value", group_by="region", order="desc", limit=1)
→ 答案
```

**风险**：模型可能无限试错。必须加**调用次数上限**。

---

## 3. Tool schema 怎么设计（决定模型选得对不对）

模型**完全靠 name + description + parameter description** 来决定调哪个、传什么。所以这是 prompt engineering 的一个分支。

### 3.1 好的 schema vs 差的 schema

```python
# ❌ 差
{
  "name": "metric_tool",
  "description": "指标工具",
  "parameters": {"type": "object", "properties": {"q": {"type": "string"}}}
}
```

问题：名字没信息量、description 没说"什么时候用"、参数 `q` 是自由文本（模型会瞎编指标名）。

```python
# ✅ 好
{
  "name": "explain_metric",
  "description": (
      "查询某个业务指标的口径、计算公式和来源表。"
      "当用户询问'这个指标怎么算的''口径是什么''包含哪些数据'时使用。"
      "注意：只查指标定义，不查具体数值。"
  ),
  "parameters": {
      "type": "object",
      "properties": {
          "metric_alias": {
              "type": "string",
              "enum": ["gmv", "sales_amount", "customer_count", "avg_order_value"],
              "description": "指标别名，必须从枚举中选择，不要自创"
          }
      },
      "required": ["metric_alias"],
      "additionalProperties": False
  }
}
```

### 3.2 五条设计规则

| 规则 | 说明 |
|---|---|
| **name 用动词+名词** | `explain_metric` > `metric_query`；模型对动作类比名词类理解准 |
| **description 写"何时用"，不只写"是什么"** | 模型靠这个做选择，不说清就会乱调 |
| **参数能枚举就枚举** | `enum` 是防幻觉最有效的手段——模型没法传一个不存在的指标名 |
| **`additionalProperties: false`** | 防止模型塞多余字段导致执行报错 |
| **description 里写清边界** | "只查定义，不查数值"——避免模型拿它当查询工具用 |

### 3.3 错误信息也要写给模型看

```python
def handler(metric_alias: str) -> dict:
    cfg = metrics.get(metric_alias)
    if not cfg:
        # ❌ return {"error": "KeyError"}          → 模型不知道怎么办
        # ✅
        return {"error": f"指标 '{metric_alias}' 不存在",
                "available": sorted(metrics.keys())[:20]}
    return {"metric": metric_alias, "expr": cfg["expr"]}
```

**为什么**：模型看到"可用指标有这些"，下一轮就能自我纠正。错误信息本质上是**给模型的 prompt**。

---

## 4. LangGraph 的 ToolNode 内部做了什么

很多人把 ToolNode 当成黑盒，其实它只有三步（示意实现）：

```python
def tool_node(state):
    # 1. 从最后一条 AIMessage 里读 tool_calls
    last_message = state["messages"][-1]
    results = []

    # 2. 逐个执行
    for call in last_message.tool_calls:
        tool = TOOLS[call["name"]]
        try:
            output = tool.invoke(call["args"])     # 参数已由框架解析成 dict
            results.append(ToolMessage(content=str(output),
                                       tool_call_id=call["id"]))
        except Exception as e:
            # ★ 失败也要返回 ToolMessage，否则消息历史不完整，下一轮会报错
            results.append(ToolMessage(content=f"工具执行失败：{e}",
                                       tool_call_id=call["id"]))

    # 3. 写回 state
    return {"messages": results}
```

**两个容易忽略的点**：

1. **失败必须返回 ToolMessage**。如果抛异常直接中断，消息历史里就会出现"assistant 说要调工具，但没有 tool 回复"，下一次请求会被 API 拒绝。
2. **ToolNode 不含智能**。它只是执行器，选哪个工具是模型的事。

配套的路由：

```python
from langgraph.prebuilt import tools_condition

builder.add_conditional_edges(
    "agent",
    tools_condition,          # 最后一条消息有 tool_calls → "tools"，否则 → END
)
builder.add_edge("tools", "agent")   # 执行完回到模型
```

---

## 5. 五个工程问题，每个都要落到代码

### 5.1 超时

```python
async def call_tool_with_timeout(tool, args, timeout_s=5.0):
    try:
        return await asyncio.wait_for(asyncio.to_thread(tool.invoke, args),
                                      timeout=timeout_s)
    except asyncio.TimeoutError:
        return {"error": f"工具 {tool.name} 超时（{timeout_s}s）"}
```

超时要**返回可读错误**而不是抛异常，让模型有机会换策略。

### 5.2 重试

**只对幂等工具重试**，且必须带退避：

```python
for attempt in range(3):
    try:
        return tool.invoke(args)
    except TransientError:
        if not tool.idempotent:
            raise                             # 非幂等，直接失败
        time.sleep(2 ** attempt)              # 1s, 2s, 4s
```

### 5.3 幂等

读类工具天然幂等（查指标定义、查维度），不用处理。
写类工具必须带幂等键：

```python
def create_report(month: str, idempotency_key: str) -> dict:
    if cached := cache.get(idempotency_key):
        return cached                          # 重复调用返回第一次结果
    result = do_create(month)
    cache.set(idempotency_key, result)
    return result
```

### 5.4 参数校验（不信模型传的值）

```python
def validate_args(schema: dict, args: dict) -> None:
    allowed = set(schema["properties"].keys())
    unknown = set(args) - allowed
    if unknown:
        raise ValidationError(f"未知参数：{unknown}")
    # enum 校验
    for key, spec in schema["properties"].items():
        if "enum" in spec and args.get(key) not in spec["enum"]:
            raise ValidationError(f"{key} 必须是 {spec['enum']} 之一")
```

> `enum` 已经拦掉了大部分注入，但**服务端必须再校验一次**——不能假设模型一定守规矩。

### 5.5 审计

```python
audit.log(
    query_id=state["query_id"], user_id=state["user_id"],
    event="tool_call", tool=name, args=args,
    status="ok" if ok else "error", elapsed_ms=ms,
)
```

面试时这条能加分：**"工具调用是最容易成为审计盲区的地方——它不经过我原来的查询链路，必须单独埋点。"**

---

## 6. MCP：把工具变成"服务"

### 6.1 它解决什么问题

Function Calling 的问题是：**每个应用都要自己实现一遍工具**。你的 IDE 想查指标，得接一遍；另一个 Agent 想查，再接一遍。

MCP（Model Context Protocol）就是把这层标准化：**工具做成 Server，客户端发现并调用**。

### 6.2 三个原语

| 原语 | 是什么 | 例子 |
|---|---|---|
| **tools** | 可调用的函数（和 FC 一样） | `explain_metric` |
| **resources** | 可读取的数据（像文件/URL） | 某张表的 schema |
| **prompts** | 可复用的模板 | "生成销售报告的模板" |

工具调用只需要关心 `tools`，另两个是给"给模型喂上下文"用的。

### 6.3 传输方式

| 传输 | 场景 |
|---|---|
| **stdio** | 本地进程（IDE 启动一个子进程） |
| **SSE / Streamable HTTP** | 远程服务（团队共享一个 MCP server） |

底层是 **JSON-RPC 2.0**：客户端发 `tools/list` 拿工具清单，发 `tools/call` 执行。

### 6.4 和 Function Calling 的关系

```
MCP  = 协议层：怎么发现工具、怎么调用、怎么传结果
FC   = 模型层：模型怎么表达"我要调这个工具"
两者可以叠加：MCP server 提供的 tools，最终仍以 FC 的形式喂给模型
```

### 6.5 最小 server（示意）

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server

app = Server("nl2dsl-metadata")

@app.list_tools()
async def list_tools():
    return [{
        "name": "explain_metric",
        "description": "查询指标口径",
        "inputSchema": METRIC_SCHEMA,
    }]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "explain_metric":
        return await explain_metric(**arguments)

async def main():
    async with stdio_server() as (r, w):
        await app.run(r, w, app.create_initialization_options())
```

**为什么值得做**：做完之后，这个能力就能被 Cursor、Claude Desktop、其他 Agent 直接复用，而不是锁死在你的应用里。

---

## 7. 给 NL2DSL 加第一个工具（完整设计）

### 7.1 边界：什么该做成工具

三条全中才做：

```
① 只读、幂等
② 不需要经过 DSL 校验链（不是查询执行）
③ 模型需要动态决定要不要调——不是每次都必调
```

**SQL 执行永远不该做成工具**——那等于把决策和执行耦合，失去校验、权限、审计、归因。

### 7.2 三个候选工具

| 工具 | 参数 | 返回 | 为什么适合 |
|---|---|---|---|
| `explain_metric` | metric_alias (enum) | 口径、表达式、来源表 | 用户常问"怎么算的"，按需调用 |
| `list_dimensions` | data_source (enum) | 维度列表 | 生成前探索，减少校验失败 |
| `sample_values` | dimension, limit=10 | 枚举值分布 | 帮确认过滤值编码（"华东"→"HD"） |

共同点：**查元数据，不查数据**——不涉及行级权限。

### 7.3 接入方式：旁路，不替代主链路

```python
# plan 之后加一个分支
builder.add_conditional_edges("plan", route_after_plan, {
    "continue":  "decompose",
    "agent":     END,
    "explore":   "tool_agent",      # ← 需要查元数据才进
})
builder.add_conditional_edges("tool_agent", tools_condition)
builder.add_edge("tools", "tool_agent")
builder.add_edge("tool_agent", "decompose")   # 探索完回主链路
```

**关键设计**：工具分支是**旁路**，探索完必须回到原来的固定链路。这样即使模型调错工具，也不会破坏查询主流程。

### 7.4 工具注册表

```python
@dataclass
class Tool:
    name: str
    description: str
    schema: dict
    handler: Callable
    timeout_s: float = 5.0
    idempotent: bool = True

class ToolRegistry:
    def __init__(self): self._tools = {}
    def register(self, t: Tool): self._tools[t.name] = t
    def schemas(self):
        return [{"type": "function",
                 "function": {"name": t.name, "description": t.description,
                              "parameters": t.schema}}
                for t in self._tools.values()]
```

---

## 8. 什么时候**不该**用工具调用

| 场景 | 该用 |
|---|---|
| 流程确定、强治理、强安全（SQL 查数） | 固定工作流 |
| 每步都要审计和权限校验 | 固定工作流 |
| 工具种类多且会变、需灵活决策 | 工具调用 |
| 步骤依赖由结果决定（探索式） | 工具调用 |
| 只有 2-3 个固定工具、每次都必调 | 直接写死在流程里 |

> 最后一条最容易被忽略：**如果每个请求都要调这个工具，那它不是"工具"，是流程的一步。**

---

## 9. 工具调用怎么评测

| 指标 | 定义 | 怎么测 |
|---|---|---|
| **选择准确率** | 该调的时候调了、不该调的时候没调 | 标注"这个 query 该不该调工具" |
| **参数正确率** | 传的参数对不对 | 比对 args 与期望 |
| **任务完成率** | 调完是否解决了问题 | 端到端判分 |
| **调用次数分布** | 平均几轮、最长几轮 | 统计 + 熔断阈值 |

**必须包含"不该调"的反例**，否则模型会倾向乱调工具（因为调了至少不扣分）。

---

## 10. 面试怎么答

**"你们 Agent 支持工具调用吗？"**
> "目前不支持，是编排型 Agent。但这不是能力缺失，是场景选择：SQL 执行必须走固定链路过校验和权限，交给模型自主调用就等于放弃可治理性。元数据查询这类只读、幂等的能力我设计了工具化方案——注册表 + ToolNode + 审计限流，只是还没落地。"

**"Function Calling 的原理？"**
> "模型不执行函数，只输出 tool_call（函数名 + JSON 字符串参数）；应用执行后把结果作为 tool 消息回传，模型再据此生成最终答案。所以它是个循环，不是一次调用，token 成本会随轮次增长。"

**"tool_call 参数解析失败怎么办？"**
> "三层：strict schema 保证合法 JSON；解析失败把错误信息回灌让模型重试（最多 2 次）；仍失败就降级。参数用 enum 能把大部分失败提前消灭。"

**"工具调用怎么防越权？"**
> "两道关：工具级 RBAC（这个角色能调哪些工具）+ 参数级白名单（enum + 服务端二次校验）。不信模型传进来的任何值。"

---

## 11. 速查

| 问题 | 答案 |
|---|---|
| 本质 | 模型输出调用意图，你的代码执行，结果回传，循环 |
| 消息结构 | assistant.tool_calls → tool(tool_call_id) → assistant |
| 三种模式 | 单次 / 并行 / 串行 ReAct |
| schema 五规则 | 动词命名、description 写何时用、参数 enum、禁止多余字段、写清边界 |
| ToolNode | 读 tool_calls → 执行 → 写 ToolMessage；失败也必须返回消息 |
| 五个工程问题 | 超时 / 重试（仅幂等） / 幂等键 / 参数校验 / 审计 |
| MCP | 协议层；三原语 tools/resources/prompts；stdio 或 HTTP；底层 JSON-RPC |
| 该做工具的能力 | 只读、幂等、不越权的元数据查询 |
| 不该做工具 | SQL 执行（要过校验链） |
| 评测 | 选择准确率 / 参数正确率 / 任务完成率，必须有"不该调"反例 |
| 一句话 | **模型负责想查什么，系统负责怎么安全地查——这条线不能因为工具调用而模糊。** |
