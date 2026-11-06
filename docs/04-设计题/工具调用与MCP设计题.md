# 工具调用与 MCP：从零开始（小白版）

> **证据边界**：NL2DSL 项目里**没有任何工具调用实现**（全仓无 `bind_tools` / `ToolNode` / `tool_call`）。
> 本文是**教学 + 设计方案**，面试口径是「我们没做，但我知道该怎么做、什么时候该做」。
> 已实现的编排能力见 [Agent 编排专项](../03-专项/Agent编排专项.md)。

---

## 这份文档怎么读

**前置知识**（会这些就够）：
- 会写 Python，知道函数和字典
- 知道什么是 API（发个 HTTP 请求，拿个 JSON 回来）
- 知道大模型的基本行为：给它一段话，它接着往下写

**不需要**：不需要会 LangChain、不需要懂 Agent、不需要会 MCP。

**读法**：按顺序读，**第 2 节的代码一定要自己跑一遍**。跑通之后，后面全是细节和工程化。

**目录**

| 节 | 解决什么问题 |
|---|---|
| 1 | 没有工具调用，大模型缺什么？ |
| 2 | 30 行代码跑通第一个工具调用 |
| 3 | 把第 2 节拆开：消息到底怎么流转 |
| 4 | 怎么"告诉"模型有哪些工具（schema 设计） |
| 5 | 一个工具不够用：三种调用形态 |
| 6 | 手写循环 vs 用框架（LangGraph） |
| 7 | 工具会失败，工程上怎么办 |
| 8 | MCP：工具要给很多应用用时怎么办 |
| 9 | 回到你的项目：该做哪个工具 |
| 10 | 自测 10 题 |
| 11 | 面试怎么答 |
| 12 | 一页速查 |

---

## 第 1 节：没有工具调用，大模型缺什么？

**本节要解决的问题**：搞清楚"工具调用"这个词到底在说什么，为什么需要它。

### 1.1 一个具体的尴尬

用户问：**"我们公司的 GMV 是怎么算的？"**

大模型脑子里有全网公开的知识，但**没有你公司的指标定义**。它只有三条路：

| 选择 | 结果 |
|---|---|
| 编一个 | 说"GMV 是年度总收入"——**错的，这是幻觉** |
| 说不知道 | "抱歉，我不知道贵公司的定义"——**没用** |
| **去查一下** | 调用一个能查指标定义的函数——**这就是工具调用** |

### 1.2 一句话定义

> **工具调用 = 让模型在回答之前，先"申请"执行一个函数，拿到结果再继续。**

注意两个关键词：

- **申请**：模型不执行函数，它只会说"我想调 `get_metric_definition`，参数是 `GMV`"
- **执行**：真正干活的是**你的代码**

### 1.3 用"点菜"理解整个过程

```
顾客（用户）："宫保鸡丁怎么做的？"
服务员（模型）：我不知道配方，但厨房有配方册
              → 在单子上写："查配方：宫保鸡丁"   ← 这叫 tool_call
厨房（你的代码）：翻配方册，把配方写出来        ← 这叫执行工具
服务员（模型）：把配方转述给顾客                ← 这叫最终回答
```

**关键理解**：服务员不会做菜，模型不会执行函数。**它只是"知道该找谁要什么"**。

### 1.4 谁负责什么（这张表要记住）

| 环节 | 谁做 | 具体动作 |
|---|---|---|
| 决定"要不要调工具、调哪个、传什么参数" | **模型** | 输出 `tool_calls` |
| 执行函数、访问数据库/API | **你的代码** | 调真实函数 |
| 把结果交回模型 | **你的代码** | 追加一条 `role: "tool"` 消息 |
| 根据结果生成最终答案 | **模型** | 输出自然语言 |

> **一句话：模型负责"决策"，你的代码负责"执行"。这个分工是所有工具调用框架的共同本质。**

**读完这节你应该能**：用一句话向别人解释什么是工具调用，并说清"模型不执行函数"。

---

## 第 2 节：30 行代码跑通第一个例子

**本节要解决的问题**：亲手跑一次，建立"原来就这么回事"的直觉。

### 2.1 完整代码（可直接运行）

```python
# 依赖：pip install openai
import json
from openai import OpenAI

client = OpenAI(
    api_key="你的key",
    base_url="https://api.deepseek.com",   # 任何 OpenAI 兼容接口都行
)

# ---------- ① 定义工具：告诉模型"你能调什么" ----------
tools = [{
    "type": "function",
    "function": {
        "name": "get_metric_definition",
        "description": "查询公司指标的定义和计算方式。当用户询问某个指标怎么算、口径是什么时使用。",
        "parameters": {
            "type": "object",
            "properties": {
                "metric_name": {
                    "type": "string",
                    "description": "指标名称，例如 GMV、销售额、客单价",
                }
            },
            "required": ["metric_name"],
        },
    },
}]

# ---------- ② 工具的真实实现（这就是普通函数） ----------
def get_metric_definition(metric_name: str) -> str:
    fake_db = {
        "GMV": "GMV = 所有已支付订单金额之和，含运费，不含退款",
        "客单价": "客单价 = GMV / 支付用户数",
    }
    return fake_db.get(metric_name, f"未找到指标「{metric_name}」")

# ---------- ③ 第一轮：把问题和工具清单一起发出去 ----------
messages = [{"role": "user", "content": "GMV 是怎么算的？"}]
resp = client.chat.completions.create(
    model="deepseek-chat", messages=messages, tools=tools,
)

msg = resp.choices[0].message
print("模型正文:", msg.content)       # None  ← 它还没回答
print("模型意图:", msg.tool_calls)    # [tool_call(name='get_metric_definition', ...)]

# ---------- ④ 执行模型要求的工具 ----------
call = msg.tool_calls[0]
args = json.loads(call.function.arguments)   # ⚠ arguments 是字符串，要转字典
result = get_metric_definition(**args)
print("工具结果:", result)

# ---------- ⑤ 第二轮：把执行结果塞回历史，再问一次 ----------
messages.append(msg)                                   # 模型的"申请单"
messages.append({                                      # 你的"执行回执"
    "role": "tool",
    "tool_call_id": call.id,                           # ⚠ 必须带上这个 id
    "content": result,
})

resp2 = client.chat.completions.create(
    model="deepseek-chat", messages=messages, tools=tools,
)
print("最终回答:", resp2.choices[0].message.content)
# 输出：GMV 是所有已支付订单金额之和，含运费，不含退款。
```

### 2.2 运行后你会看到的三个现象

| 现象 | 说明 |
|---|---|
| `msg.content` 是 `None` | 模型这轮**没有回答**，它只想调工具 |
| `msg.tool_calls` 是一个列表 | 里面写着"调哪个函数、参数是什么" |
| 要**请求两次** | 第一次拿意图，第二次才拿答案 |

**这就是工具调用最反直觉的地方**：它不是"一次请求得到一个答案"，而是"一来一回"。

### 2.3 三个马上会踩的坑

**坑 1：`arguments` 是字符串，不是字典**

```python
call.function.arguments       # '{"metric_name":"GMV"}'   ← 字符串
json.loads(call.function.arguments)   # {"metric_name": "GMV"}   ← 才是字典
```

因为模型是"生成文本"的，它生成的是一段 JSON 文本，不是对象。

**坑 2：`tool_call_id` 必须带**

```python
{"role": "tool", "tool_call_id": call.id, "content": result}
```

模型一次可能申请调多个工具，`id` 是它区分"这个结果对应哪个申请"的唯一凭证。漏了会报错。

**坑 3：模型可能一次申请调好几个工具**

```python
for call in msg.tool_calls:     # 用 for，不要写死 [0]
    ...
```

**读完这节你应该能**：跑通一遍，并说出"为什么需要请求两次"。

---

## 第 3 节：把第 2 节拆开——消息到底怎么流转

**本节要解决的问题**：把"两次请求"的内部结构看清楚，以后调试时能自己看懂日志。

### 3.1 第一次请求发出去的东西

```json
{
  "model": "deepseek-chat",
  "messages": [
    {"role": "user", "content": "GMV 是怎么算的？"}
  ],
  "tools": [ ...tool 定义... ]
}
```

注意：**工具清单每次请求都要带上**。模型不会"记住"上一次请求里的工具。

### 3.2 第一次回来的东西

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [{
    "id": "call_abc123",
    "type": "function",
    "function": {
      "name": "get_metric_definition",
      "arguments": "{\"metric_name\": \"GMV\"}"
    }
  }]
}
```

这条消息要**原样存回历史**。它不是"模型的回答"，而是"模型的申请单"。

### 3.3 第二次请求发出去的东西

```json
{
  "messages": [
    {"role": "user", "content": "GMV 是怎么算的？"},
    {"role": "assistant", "content": null, "tool_calls": [ ...申请单... ]},
    {"role": "tool", "tool_call_id": "call_abc123", "content": "GMV = 所有已支付订单金额之和…"}
  ],
  "tools": [ ...同一份工具清单... ]
}
```

**这里最容易出错的理解**：你可能以为只发"结果"就够了，其实要把**完整的对话历史**都发过去：

```
用户问题 → 模型的申请 → 工具的结果
```

模型靠这三段拼出上下文。少一段它就不知道发生了什么。

### 3.4 第二次回来的东西

```json
{
  "role": "assistant",
  "content": "GMV 是所有已支付订单金额之和，含运费，不含退款。"
}
```

这次 `tool_calls` 为空，`content` 有值——**这才是最终答案**。

### 3.5 一张图记住整个过程

```
┌─────────┐   ① 问题 + 工具清单    ┌─────────┐
│ 你的代码 │ ────────────────────► │  模型   │
└─────────┘                        └─────────┘
     ▲                                   │
     │                              ② tool_calls（申请单）
     │                                   ▼
     │  ┌──────────────────────────────────┐
     │  │ 你执行函数，得到结果              │
     │  └──────────────────────────────────┘
     │                                   │
     │   ③ 全历史 + 工具结果             ▼
     └────────────────────────────  ┌─────────┐
                                    │  模型   │
                                    └─────────┘
                                         │
                                    ④ 最终答案
```

**读完这节你应该能**：看懂任何工具调用的日志——知道哪条是申请单、哪条是执行结果、哪条是最终答案。

---

## 第 4 节：怎么"告诉"模型有哪些工具

**本节要解决的问题**：模型凭什么知道该调哪个工具？答案是 schema 写得清不清楚。

### 4.1 模型看到的信息，只有这三样

```json
{
  "name": "get_metric_definition",
  "description": "查询公司指标的定义…",
  "parameters": { ...参数结构... }
}
```

**没有函数体、没有代码、没有实现**。模型只能靠这三样判断"要不要调、传什么参数"。

> 所以 schema 本质上是 **prompt**——你写得多清楚，模型选得多准。

### 4.2 反面教材（这样写模型一定乱调）

```json
{
  "name": "metric_tool",
  "description": "指标工具",
  "parameters": {
    "type": "object",
    "properties": {"q": {"type": "string"}}
  }
}
```

三个问题：

| 问题 | 后果 |
|---|---|
| `name` 是 `metric_tool` | 看不出是"查"还是"改"还是"算" |
| `description` 是"指标工具" | 模型不知道**什么时候该用它** |
| 参数叫 `q`、类型是自由字符串 | 模型会瞎编指标名，比如 `"GMV金额"` |

### 4.3 正面教材（逐字段解释）

```json
{
  "name": "get_metric_definition",
  "description": "查询公司指标的定义和计算方式。当用户询问某个指标怎么算、口径是什么、包含哪些数据时使用。注意：只查定义，不查具体数值。",
  "parameters": {
    "type": "object",
    "properties": {
      "metric_name": {
        "type": "string",
        "enum": ["GMV", "销售额", "客单价", "客户数"],
        "description": "指标名称，必须从枚举中选择，不要自创"
      }
    },
    "required": ["metric_name"],
    "additionalProperties": false
  }
}
```

| 字段 | 为什么这么写 |
|---|---|
| `name` 用"动词_名词" | `get_metric_definition` 一眼看出是"获取定义" |
| `description` 写**"什么时候用"** | 模型靠这句做选择：问"怎么算"才用，问"是多少"不用 |
| `description` 写**边界** | "只查定义，不查具体数值"——防止模型拿它当查询工具 |
| 参数用 `enum` | **这是防幻觉最有效的一招**：模型物理上没法传一个不存在的指标名 |
| `required` | 告诉模型这个参数不能省 |
| `additionalProperties: false` | 防止模型自作聪明多塞字段，导致执行报错 |

### 4.4 一个自检方法

把你的 schema 给一个**不懂你系统的同事**看，问他：

> "如果用户问 X，你觉得该调这个工具吗？"

- 他答对了 → schema 及格
- 他答不出来 → 模型也答不出来

**这是一个很实用的技巧：把"模型能不能选对"转化成"人能不能看懂"。**

### 4.5 错误信息也要写给模型看

工具执行失败时，返回什么很关键：

```python
# ❌ 模型看不懂，只能重复同样的错误
return {"error": "KeyError"}

# ✅ 模型能看到"有什么可选"，下一轮能自我纠正
return {
    "error": f"指标「{metric_name}」不存在",
    "available": ["GMV", "销售额", "客单价", "客户数"],
    "hint": "请从 available 中选择一个"
}
```

> **错误信息本质上是给模型的 prompt。** 这条想通，工具调用的成功率会明显上升。

**读完这节你应该能**：写出一个"模型不会选错"的工具 schema，并说出 `enum` 的作用。

---

## 第 5 节：一个工具不够用时——三种调用形态

**本节要解决的问题**：真实任务往往要调好几个工具，怎么组织？

### 5.1 三种形态

| 形态 | 场景 | 模型行为 |
|---|---|---|
| **单次** | 调一个就够 | 一次返回 1 个 `tool_call` |
| **并行** | 多个互不依赖的查询 | 一次返回多个 `tool_call`（数组） |
| **串行（循环）** | 下一步依赖上一步的结果 | 调一个 → 看结果 → 再调下一个 |

### 5.2 并行：一次问多件事

用户："GMV 和客单价分别怎么算的？"

模型可以一次返回两个申请：

```json
"tool_calls": [
  {"id": "c1", "function": {"name": "get_metric_definition", "arguments": "{\"metric_name\":\"GMV\"}"}},
  {"id": "c2", "function": {"name": "get_metric_definition", "arguments": "{\"metric_name\":\"客单价\"}"}}
]
```

你的代码可以并发执行（更快），然后**一次性把两个结果都塞回历史**：

```python
for call in msg.tool_calls:
    result = run(call)
    messages.append({"role": "tool", "tool_call_id": call.id, "content": result})
```

### 5.3 串行循环（这就是 ReAct）

用户："**哪个地区的客单价最高？**"

这个问题**一步答不了**，因为：
1. 先得知道有哪些维度可用（是不是有"地区"）
2. 再得按地区分组算客单价，取第一

所以模型会这样走：

```
思考：我需要先知道有哪些维度
  ↓ 调用 list_dimensions()
观察：["地区", "渠道", "品牌"]
思考：有"地区"，那我按地区算客单价，降序取 1
  ↓ 调用 run_metric(metric="客单价", group_by="地区", order="desc", limit=1)
观察：[{"地区": "华东", "客单价": 1280}]
思考：可以回答了
  ↓
回答："华东地区客单价最高，1280 元"
```

**这个"思考 → 行动 → 观察 → 再思考"的循环，学名叫 ReAct**（Reason + Act）。

### 5.4 循环的最大风险：停不下来

模型可能：

- 反复调同一个工具（结果一样，但就是不结束）
- 在错误的方向上试 20 次
- 每次都说"再确认一下"

**所以循环必须有刹车**：

```python
MAX_STEPS = 8

for step in range(MAX_STEPS):
    resp = call_model(messages, tools)
    if not resp.tool_calls:          # 没有申请了 → 说明它要回答
        return resp.content
    execute_and_append(resp)

# 走完 8 步还没结束 → 强制收敛
return "抱歉，这个问题需要更多步骤才能完成，我先给你目前查到的：..."
```

> 记住这条：**没有步数上限的 Agent 循环，迟早会在线上卡死。**

**读完这节你应该能**：区分并行和串行，并说出为什么循环必须设上限。

---

## 第 6 节：手写循环 vs 用框架

**本节要解决的问题**：框架（LangGraph）到底帮你做了什么？为什么建议先手写？

### 6.1 手写一个 ReAct 循环（核心只有 20 行）

```python
def run_agent(question: str, tools: list, max_steps: int = 8) -> str:
    messages = [{"role": "user", "content": question}]

    for _ in range(max_steps):
        resp = client.chat.completions.create(
            model="deepseek-chat", messages=messages, tools=tools,
        )
        msg = resp.choices[0].message

        # 没有申请 → 它要回答了
        if not msg.tool_calls:
            return msg.content

        # 有申请 → 执行并回填
        messages.append(msg)
        for call in msg.tool_calls:
            result = TOOLS_MAP[call.function.name](**json.loads(call.function.arguments))
            messages.append({
                "role": "tool",
                "tool_call_id": call.id,
                "content": str(result),
            })

    return "超过最大步数，已中止"
```

**这 20 行就是"ReAct Agent"的全部核心。** 剩下的都是工程加固。

### 6.2 用 LangGraph 写（对照看框架替了什么）

```python
from langgraph.prebuilt import ToolNode, tools_condition

builder.add_node("agent", call_model)      # 调模型
builder.add_node("tools", ToolNode(tools)) # 执行工具（框架替你写）
builder.add_conditional_edges("agent", tools_condition)  # 有申请→tools，否则→END
builder.add_edge("tools", "agent")         # 执行完回到模型
```

| 你手写的部分 | 框架提供了什么 |
|---|---|
| 循环 | 图的条件边 + 循环边 |
| 判断"有没有 tool_calls" | `tools_condition` |
| 执行 + 回填消息 | `ToolNode` |
| 最大步数 | 需要自己加（`recursion_limit`） |
| 状态管理 | `State` 的 reducer |

### 6.3 ToolNode 内部其实就三步

```python
def tool_node(state):
    last_msg = state["messages"][-1]
    results = []
    for call in last_msg.tool_calls:
        try:
            output = TOOLS_MAP[call.function.name](**call.args)
            results.append(ToolMessage(content=str(output), tool_call_id=call.id))
        except Exception as e:
            # ⚠ 失败也必须返回 ToolMessage，否则消息历史不完整，下一轮 API 会报错
            results.append(ToolMessage(content=f"执行失败：{e}", tool_call_id=call.id))
    return {"messages": results}
```

### 6.4 建议的顺序

```
先手写 20 行跑通   →   理解每一步在干什么
再用框架重构       →   享受状态管理和可视化
```

反过来（先框架）就会变成你现在的困境：**会用但不知道黑盒里是什么**。

**读完这节你应该能**：白板写出一个 ReAct 循环的核心逻辑。

---

## 第 7 节：工具会失败，工程上怎么办

**本节要解决的问题**：demo 能跑不等于能上线。工具是外部依赖，一定会坏。

### 7.1 五个必须处理的情况

| 情况 | 会发生什么 | 怎么处理 |
|---|---|---|
| **超时** | 工具卡住，整个请求挂死 | 每个工具独立超时 |
| **临时失败** | 网络抖动、下游 5xx | 重试（只对幂等工具） |
| **重复执行** | 用户重试导致写两次 | 幂等键 |
| **参数错误** | 模型传了不存在的值 | 服务端二次校验 |
| **出问题查不到** | 谁在什么时候调了什么 | 审计日志 |

### 7.2 超时：返回可读错误，不要抛异常

```python
import asyncio

async def call_with_timeout(tool, args, timeout_s=5.0):
    try:
        return await asyncio.wait_for(asyncio.to_thread(tool.invoke, args),
                                      timeout=timeout_s)
    except asyncio.TimeoutError:
        return {"error": f"工具 {tool.name} 超时（{timeout_s}s），请换个方式"}
```

**为什么返回而不是抛异常**：抛异常会中断整个循环；返回错误信息则给模型一个"换个策略"的机会。

### 7.3 重试：只对幂等工具

```python
def call_with_retry(tool, args, max_retry=3):
    for attempt in range(max_retry):
        try:
            return tool.invoke(args)
        except TransientError:
            if not tool.idempotent:
                raise                    # 写操作不重试，避免重复写
            time.sleep(2 ** attempt)     # 1s、2s、4s
    raise
```

**什么叫幂等**：调 1 次和调 3 次结果一样。查指标定义是幂等的，`create_report` 不是。

### 7.4 参数校验：不信模型传的值

```python
def validate_args(schema: dict, args: dict) -> None:
    allowed = set(schema["properties"].keys())
    unknown = set(args) - allowed
    if unknown:
        raise ValidationError(f"未知参数：{unknown}")

    for key, spec in schema["properties"].items():
        if "enum" in spec and args.get(key) not in spec["enum"]:
            raise ValidationError(f"{key} 必须是 {spec['enum']} 之一")
```

**为什么要二次校验**：`enum` 只是"劝"模型别乱传，模型仍然可能传错。**服务端必须自己验一遍**。

### 7.5 审计：工具调用最容易成为盲区

```python
audit.log(
    query_id=query_id, user_id=user_id,
    event="tool_call", tool=name, args=args,
    status="ok" if ok else "error", elapsed_ms=elapsed,
)
```

> 你的查询链路有审计，但**工具调用不走那条链路**——不单独埋点，出事根本查不到。

**读完这节你应该能**：说出工具调用在生产上最容易出的 5 类问题及对策。

---

## 第 8 节：MCP——工具要给很多应用用时怎么办

**本节要解决的问题**：如果每个应用都要自己实现一遍工具，太浪费。MCP 就是解决这个。

### 8.1 先看没有 MCP 的痛点

```
你的 IDE 想查指标  → 自己实现一遍 explain_metric
你的 Chat 应用想查 → 再实现一遍
另一个 Agent 想查  → 再实现一遍
```

三份代码、三种行为、三处维护。

### 8.2 MCP 是什么（用一句话 + 一个类比）

> **MCP 是一个协议：把"工具"做成一个独立的服务，谁想用就按统一协议调。**

**类比：USB 接口。**

- 以前：每个设备一个专用接口（诺基亚圆口、索尼的、苹果的）
- 现在：都插 USB，电脑不用管是谁

MCP 就是 AI 工具界的 USB 接口。**只要你的工具按 MCP 实现，任何支持 MCP 的客户端（Cursor、Claude Desktop、你自己的 Agent）都能直接用。**

### 8.3 三个概念（用图书馆类比）

| 原语 | 是什么 | 类比 |
|---|---|---|
| **tools** | 可调用的函数 | 借书、还书、查书位置 |
| **resources** | 可读取的数据 | 书架上的书本身 |
| **prompts** | 可复用的模板 | 馆员准备好的"推荐书单模板" |

**做工具调用只需要关心 `tools`**，另外两个是给"喂上下文"用的。

### 8.4 两种传输方式

| 方式 | 怎么跑 | 用在哪 |
|---|---|---|
| **stdio** | 客户端启动一个子进程，通过标准输入输出通信 | 本地工具（IDE 插件） |
| **HTTP（SSE/Streamable）** | 远程服务 | 团队共享一个服务 |

### 8.5 和 Function Calling 什么关系（最容易搞混）

```
Function Calling  = 模型怎么"表达"我要调工具   （模型层的约定）
MCP               = 工具怎么"被发现和调用"     （协议层的标准）
```

**它们不是替代关系，是一上一下**：

```
模型输出 tool_call  ← Function Calling 负责
        ↓
你的代码执行这个调用
        ↓
这个工具恰好是通过 MCP 协议拿到/调用的  ← MCP 负责
```

> 一句话：**Function Calling 解决"模型怎么说话"，MCP 解决"工具怎么接进来"。**

### 8.6 一个最小 MCP server（示意）

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
```

### 8.7 什么时候需要 MCP，什么时候不需要

| 情况 | 要不要 MCP |
|---|---|
| 工具只在自己一个应用里用 | ❌ 不需要，直接函数调用 |
| 工具要给 IDE、多个 Agent 共用 | ✅ 需要 |
| 想对外提供能力（给别人接） | ✅ 需要 |
| 团队里多个项目重复实现同样的工具 | ✅ 需要 |

**读完这节你应该能**：用一句话说清 MCP 和 Function Calling 的区别，并判断自己的项目要不要上 MCP。

---

## 第 9 节：回到你的项目——该做哪个工具

**本节要解决的问题**：NL2DSL 里哪些能力适合做成工具？

### 9.1 三个判断条件（全中才做）

```
① 只读、幂等
② 不经过 DSL 校验链（不是查询执行）
③ 模型需要"动态决定要不要调"，不是每次都必调
```

### 9.2 为什么 SQL 执行**不能**做成工具

假设做了：

```
模型 → tool_call: run_sql("SELECT ...") → 直接执行
```

问题：

| 问题 | 后果 |
|---|---|
| SQL 是模型生成的自由文本 | 没有可校验的中间层 |
| 权限只能靠 SQL 层拦 | 不如 DSL 层注入精确 |
| 出错分不清是理解错、生成错、还是执行错 | 无法归因 |
| 模型可以反复试错 | 成本不可控 |

而固定链路是这样的：

```
DSL（可校验）→ 校验 → 权限注入 → 优化 → 构建 SQL → 扫描 → 沙箱 → 执行
```

**每一环都可以单独拦下来、单独审计。**

### 9.3 该做成工具的候选清单

**第一梯队：元数据类（零风险，先做）**

| 工具 | 作用 | 为什么适合 |
|---|---|---|
| `explain_metric` | 查指标口径 | 用户常问"怎么算的"，按需调用 |
| `list_dimensions` | 查可用维度 | 生成前探路，减少校验失败 |
| `sample_values` | 查维度取值 | 帮确认过滤值（"华东"对应什么编码） |

**共同点：查的是"元数据"，不是"数据"。** 不涉及行级权限，天然安全。

**第二梯队：数据状态 / 预检类（ROI 最高）**

| 工具 | 一句话价值 | 为什么值得排前面 |
|---|---|---|
| `check_data_freshness` | "数据到几号了" | 问数场景最高频的追问——用户看到空结果，第一反应就是这个 |
| `explain_term` | "流水和销售额有什么区别" | 术语歧义是问数场景最大的损失来源，而且配置现成（`terms.yaml`） |
| `search_columns` | "有没有跟金额相关的字段" | 用户不知道字段名时的救星，直接降低校验失败率 |
| `dry_run` | "这条查得通吗" | 让错误在生成阶段暴露，省掉一整轮修正 |

> 这四个加起来约 **2.5 天**，比做 SQL 执行工具安全得多，价值也高得多。**推荐从这里开始。**

**第三梯队：按需扩展**

| 方向 | 例子 |
|---|---|
| 表结构 | `describe_table`、`find_joins`、`list_tables` |
| 数据探查 | `profile_column`（空值率、Top N）、`check_row_count` |
| 历史经验 | `search_similar_queries`（带结构化过滤，比 RAG 召回更准） |
| 执行预检 | `estimate_query_cost`（估算扫描量，不真跑） |
| 主动澄清 | `ask_clarification`（让模型自己判断"我该不该问"） |

**不该做成工具的**

| 能力 | 原因 |
|---|---|
| SQL 执行 | 见 9.2：会绕开校验、权限、扫描、沙箱四道闸 |
| 任何写操作（建报表、发通知） | 必须人工审批，不能由模型自主决定 |

### 9.4 接进现有系统：旁路设计（重点，含完整改造方案）

#### 9.4.1 什么叫"旁路"

你现在的图（`graph/builder.py`）是一条直线：

```
clarification → plan → decompose → validation → permission_check
→ resolve_semantic → optimize_dsl → confidence → build_sql
→ scan_sql → sandbox_check → execute_sql → verify_dsl → explain → END
```

加工具之后，在 `plan` 后面**分出一条岔路**：

```
plan ─┬─ continue ────────────────────────────► decompose（原链路，一个字不动）
      │
      └─ explore ─► tool_agent ─┬─ 有 tool_call ─► tools ─┐
                                 │                        │
                                 └─ 没 tool_call ─► decompose
                                          ▲               │
                                          └───────────────┘（执行完回 model）
```

**"旁路"的含义有三层**：

| 含义 | 具体表现 |
|---|---|
| **不改主链路** | `decompose` 往后所有节点、边、校验逻辑都不动 |
| **可以随时关掉** | 一个开关，关掉后图退化成原来的样子 |
| **坏了不影响主流程** | 工具分支任何异常 → 回到 `decompose`，查询照常进行 |

> 这三条就是"旁路"和"替换"的根本区别：**你不用赌工具一定好用。**

#### 9.4.2 全景图：分叉、工具、循环、失败

上面那张图太简，展开画一遍。

**A. 完整分叉**

```
                         ┌──────────┐
                         │   plan   │
                         └────┬─────┘
                              │
      ┌───────────────────────┼────────────────────────────┐
      │                       │                            │
 intent ≠ single_query   single_query                single_query
      │                   ＋无元数据需求                ＋命中元数据关键词
      ▼                       │                            ▼
     END                      │                   ┌─────────────────┐
（Agent 层拆解）               │                   │   tool_agent    │◄──────┐
                              │                   └────────┬────────┘       │
                              │                            │                │
                              │        ┌───────────────────┼──────────────┐ │
                              │        │                   │              │ │
                              │   有 tool_calls      无 tool_calls    出错/超预算 │
                              │        │                   │              │ │
                              │        ▼                   │              │ │
                              │   ┌───────────┐            │              │ │
                              │   │   tools   │            │              │ │
                              │   └─────┬─────┘            │              │ │
                              │         │                  │              │ │
                              │         └──执行完回模型─────┼──────────────┘ │
                              │                            │                │
                              ▼                            ▼                ▼
                   ┌────────────────────────────────────────────────────────┐
                   │                      decompose                         │
                   │        （若查到元数据，先拼进上下文再进主链路）           │
                   └────────────────────────────────────────────────────────┘
```

**四个出口，只有一条进主链路**：

| 出口 | 何时 | 去哪 |
|---|---|---|
| `agent` | 意图不是单查询 | END（Agent 层拆解） |
| `continue` | 单查询，不需要元数据 | 直接 decompose |
| `explore` | 单查询，但可能要先查元数据 | tool_agent |
| `done` / `fallback` | 工具循环结束 / 出错 | 都回 decompose |

**B. 三个工具分别是什么**

| 工具 | 参数 | 返回 | 什么时候该调 | 读什么 |
|---|---|---|---|---|
| `explain_metric` | `metric_name`（enum：所有已注册指标） | `{expr, description}` | 用户问"怎么算的/口径/包含什么" | `registry_dict["metrics"]` |
| `list_dimensions` | 无 | 维度列表（name/column/description） | 用户问"能按什么看/有哪些维度" | `registry_dict["dimensions"]` |
| `sample_values` | `dimension`（enum）、`limit` | 该维度的可选值 | 需要确认过滤值（"华东"到底存什么） | `dimensions[dim].value_map` |

**三个工具的共同点**（这是它们能做成工具、而 SQL 不能的原因）：

```
✓ 只读        不写任何数据
✓ 幂等        调 1 次和 3 次结果一样
✓ 不越权      读的是元数据（表结构/口径/枚举），不是业务数据
✓ 极快        纯内存字典查询，毫秒级，不需要连数据库
```

**C. 一轮循环到底发生了什么**

以用户问"**有哪些维度可以按地区看？**"为例：

```
┌─────────────────────────────────────────────────────────────────┐
│ 第 1 次进 tool_agent（tool_steps = 0）                          │
│                                                                 │
│   检查：llm_client 有？注册表非空？tool_steps < 6？   → 都通过   │
│                                                                 │
│   messages = [{"role":"user", "content":"有哪些维度可以按地区看"}]│
│                                                                 │
│   llm.chat_with_tools(messages, 三个工具的 schema)              │
│        ↓                                                        │
│   返回：content=None，tool_calls=[{                             │
│            id:"call_1",                                         │
│            name:"list_dimensions",                              │
│            arguments:{} }]                                      │
│                                                                 │
│   → 存 pending_tool_calls，tool_steps 变成 1                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ tools 节点                                                      │
│                                                                 │
│   for call in pending_tool_calls:                               │
│     tool = registry.get("list_dimensions")                      │
│     out  = tool.handler()                  ← 真正执行            │
│             = {"dimensions":[{name:"region",column:"region"},…]}│
│                                                                 │
│     messages.append({                                           │
│       "role": "tool",                                           │
│       "tool_call_id": "call_1",                                 │
│       "content": '{"dimensions":[…]}"                           │
│     })                                                          │
│                                                                 │
│   写审计：tool=list_dimensions, status=ok, 耗时=2ms              │
│                                                                 │
│   → 清空 pending_tool_calls                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 第 2 次进 tool_agent（tool_steps = 1）                          │
│                                                                 │
│   messages 现在有 3 条：                                        │
│     ① user：有哪些维度可以按地区看                               │
│     ② assistant：tool_calls=[list_dimensions]                   │
│     ③ tool：{"dimensions":[…]}"                                 │
│                                                                 │
│   llm.chat_with_tools(messages, schemas)                        │
│        ↓                                                        │
│   返回：content="可以按 region（地区）、channel（渠道）… 这些维度",│
│         tool_calls=[]        ← 空了，说明它不需要再查了          │
│                                                                 │
│   → 结束旁路，生成 metadata_context                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                        decompose（主链路）
```

**state 在循环中怎么变**（这张表看懂了，循环就懂了）：

| 时刻 | `tool_steps` | `pending_tool_calls` | `tool_messages` 条数 |
|---|---|---|---|
| 刚进 tool_agent | 0 | — | 1（只有问题） |
| 模型要调工具 | 0 → **1** | 1 个 | 2（+assistant） |
| tools 执行完 | 1 | — →清空 | 3（+tool 结果） |
| 模型不再调工具 | 1 | — | 3 |
| 退出旁路 | 1 | — | **清空**（只留 metadata_context） |

> 最后一行的"清空"很重要：**中间消息不进主链路**，只把整理好的文本传下去。否则主链路的状态会被工具对话污染。

**D. 失败了怎么办（两级处理）**

失败分**两个层级**，处理方式完全不同——这是整个设计里最容易被忽略的地方：

```
工具级失败  →  把错误回填给模型，让它换个做法，循环继续
旁路级失败  →  放弃工具，回主链路，查询照常进行
```

| 失败场景 | 在哪检测 | 怎么处理 | 最终用户看到 |
|---|---|---|---|
| **模型编了个不存在的工具** | `registry.get()` 抛 KeyError | 回填 `{"error":"未知工具 xxx"}`，模型下一轮会换 | 正常（可能多绕一轮） |
| **工具内部抛异常**（数据格式不对等） | tools 节点 try/except | 回填 `{"error":"工具执行失败：…"}` | 正常 |
| **工具超时** | 单工具 `timeout_s` | 回填 `{"error":"超时，请换方式"}` | 正常 |
| **模型传了 enum 外的参数** | 参数校验 + 工具内兜底 | 回填 `{"error":…, "available":[…]}`,模型自我纠正 | 正常 |
| **LLM 调用失败** | tool_agent | 结束旁路，`reason=llm_error`，用已查到的收口 | 正常（直接走主链路） |
| **步数超过 6** | tool_agent 入口 | 强制收口，`reason=budget_exhausted` | 正常（用已查到的继续） |
| **旁路整体异常** | `with_error_handler` + `route_after_tool_agent` | 返回 `fallback` → decompose | 正常（用户完全无感） |

**一句话原则**：

> **工具失败要让模型知道（好让它纠正），但旁路失败不能让用户知道（因为查询本身没错）。**

对应的代码就两处：

```python
# tools 节点：工具失败 → 回填，不中断
try:
    content = json.dumps(tool.handler(**args))
except Exception as exc:
    content = json.dumps({"error": f"工具执行失败：{exc}"})   # ← 回填，循环继续

# 路由：旁路失败 → 回主链路，不报错
def route_after_tool_agent(state):
    if state.get("status") == "error":
        return "fallback"        # ← 去 decompose，不是 END
```

**E. 循环会不会转太久**

会。所以三层刹车：

| 刹车 | 值 | 触发后 |
|---|---|---|
| 单工具超时 | 5s | 回填超时错误 |
| 步数上限 | 6 步 | 强制收口 |
| 图递归上限 | LangGraph `recursion_limit`（默认 25） | 抛异常 → 被 error_handler 接住 → fallback |

> 三步上限加起来，最坏情况也就 6 次工具调用 + 6 次 LLM 调用，不会失控。

#### 9.4.3 改造点清单（文件级）

| 文件 | 类型 | 改什么 |
|---|---|---|
| `nl2dsl/llm/client.py` | 改 | 新增 `chat_with_tools(messages, tools)` —— 现有客户端只有 `generate(prompt)`，发不了消息列表 |
| `nl2dsl/graph/state.py` | 改 | 新增 3 个字段：`tool_messages` / `tool_steps` / `metadata_context` |
| `nl2dsl/tools/registry.py` | **新增** | 工具注册表 |
| `nl2dsl/tools/metadata_tools.py` | **新增** | 三个工具的实现 |
| `nl2dsl/graph/tool_nodes.py` | **新增** | `tool_agent_node`（调模型）+ `tool_exec_node`（执行工具） |
| `nl2dsl/graph/edges.py` | 改 | `route_after_plan` 加 `explore` 分支；新增 `route_after_tool_agent` |
| `nl2dsl/graph/builder.py` | 改 | 注册 2 个节点 + 3 条边（条件：注册表非空） |
| `nl2dsl/graph/nodes.py` | 改 | `create_node_functions` 里装配两个新节点 |
| `nl2dsl/config.py` | 改 | 新增开关 `tools_enabled` / `max_tool_steps` |
| `nl2dsl/engine.py` | 改 | 启动时构建注册表并注入 |

#### 9.4.4 依赖一：LLM 客户端要能"发消息 + 带工具"

这是第一个必须先解决的**硬依赖**。看你现在的能力：

```python
# nl2dsl/llm/client.py —— 现在只有这三种
generate(prompt, system_prompt)            # 一句话进，一句话出
generate_structured(prompt, sys, schema)   # 带 json_schema
generate_json_object(prompt, sys)          # 保证是 JSON
```

**它们都是"单轮 prompt"，发不了 `messages` 列表，也带不了 `tools` 参数。** 工具调用必须是多轮消息，所以要先加一个：

```python
# nl2dsl/llm/client.py 新增
from dataclasses import dataclass

@dataclass
class ChatResult:
    content: str | None
    tool_calls: list[dict]      # [{"id": ..., "name": ..., "arguments": {...}}]

def chat_with_tools(self, messages: list[dict], tools: list[dict]) -> ChatResult:
    """带工具的消息式调用。返回模型的回答或工具申请。"""
    resp = self._client.chat.completions.create(
        model=self._model,
        messages=messages,
        tools=tools,
        temperature=0,               # 工具选择要稳定
    )
    msg = resp.choices[0].message
    calls = []
    for c in (msg.tool_calls or []):
        calls.append({
            "id": c.id,
            "name": c.function.name,
            "arguments": json.loads(c.function.arguments or "{}"),
        })
    return ChatResult(content=msg.content, tool_calls=calls)
```

> 注意 `temperature=0`：选工具这件事不需要创造力，抖动了反而有害。

#### 9.4.5 依赖二：state 加三个字段

```python
# nl2dsl/graph/state.py
class QueryState(TypedDict):
    # ... 原有字段不动 ...

    # --- 工具旁路（新增）---
    tool_messages: Annotated[list[dict] | None, add_to_list]  # 工具对话历史
    tool_steps: int | None          # 已用步数，用于预算控制
    metadata_context: str | None    # 工具查到的元数据，给主链路用
```

**为什么要三个独立字段，而不是复用现有的？**

| 字段 | 为什么不复用 |
|---|---|
| `tool_messages` | 不能塞进 `trace`（trace 是给审计看的扁平结构），也不能塞进 `dsl_attempts` |
| `tool_steps` | 独立的计数器，方便上限判断 |
| `metadata_context` | 主链路消费的是**拼好的文本**，不是原始消息列表 |

> 这三个字段是旁路和主链路之间**唯一的接口**。旁路只写 `metadata_context`，主链路只读它——耦合降到最低。

#### 9.4.6 依赖三：工具注册表 + 三个工具

```python
# nl2dsl/tools/registry.py
from dataclasses import dataclass
from typing import Any, Callable

@dataclass(frozen=True)
class Tool:
    name: str
    description: str
    schema: dict
    handler: Callable[..., Any]
    timeout_s: float = 5.0
    idempotent: bool = True

class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        self._tools[tool.name] = tool

    def get(self, name: str) -> Tool:
        if name not in self._tools:
            raise KeyError(f"未知工具: {name}")
        return self._tools[name]

    def openai_schemas(self) -> list[dict]:
        """转成模型能看懂的格式（就是第 4 节讲的那三段）"""
        return [
            {"type": "function",
             "function": {"name": t.name,
                          "description": t.description,
                          "parameters": t.schema}}
            for t in self._tools.values()
        ]

    def __len__(self) -> int:
        return len(self._tools)
```

三个工具（注意：**全部基于已有的 `registry_dict`，不需要访问数据库**）：

```python
# nl2dsl/tools/metadata_tools.py
def build_metadata_tools(registry_dict: dict) -> list[Tool]:
    metrics = registry_dict.get("metrics", {})
    dimensions = registry_dict.get("dimensions", {})

    # ---------- 工具 1：查指标口径 ----------
    def explain_metric(metric_name: str) -> dict:
        cfg = metrics.get(metric_name)
        if not cfg:
            return {"error": f"指标「{metric_name}」不存在",
                    "available": sorted(metrics.keys())[:20]}
        return {"metric": metric_name,
                "expression": cfg.get("expr", ""),
                "description": cfg.get("description", "")}

    # ---------- 工具 2：查可用维度 ----------
    def list_dimensions() -> dict:
        return {"dimensions": [
            {"name": n,
             "column": c.get("column", n),
             "description": c.get("description", "")}
            for n, c in dimensions.items()
        ]}

    # ---------- 工具 3：查维度取值 ----------
    def sample_values(dimension: str, limit: int = 10) -> dict:
        cfg = dimensions.get(dimension)
        if not cfg:
            return {"error": f"维度「{dimension}」不存在",
                    "available": sorted(dimensions.keys())}
        value_map = cfg.get("value_map") or {}
        return {"dimension": dimension,
                "values": list(value_map.keys())[:limit]}

    return [
        Tool(name="explain_metric",
             description=("查询公司指标的口径和计算方式。"
                          "当用户问某个指标怎么算、口径是什么、包含哪些数据时使用。"
                          "只查定义，不查具体数值。"),
             schema={"type": "object",
                     "properties": {"metric_name": {
                         "type": "string",
                         "enum": sorted(metrics.keys()),
                         "description": "指标名，必须从枚举中选择"}},
                     "required": ["metric_name"],
                     "additionalProperties": False},
             handler=explain_metric),
        Tool(name="list_dimensions",
             description="列出所有可用的分析维度。当用户问'能按什么看'、或需要确认某个维度是否存在时使用。",
             schema={"type": "object", "properties": {},
                     "additionalProperties": False},
             handler=list_dimensions),
        Tool(name="sample_values",
             description="查询某个维度的可选值。当需要确认过滤条件里的值（如地区名、渠道名）时使用。",
             schema={"type": "object",
                     "properties": {"dimension": {"type": "string",
                                                  "enum": sorted(dimensions.keys())},
                                    "limit": {"type": "integer", "default": 10}},
                     "required": ["dimension"],
                     "additionalProperties": False},
             handler=sample_values),
    ]
```

#### 9.4.7 两个新节点

**节点 A：`tool_agent_node` —— 问模型"要不要调工具"**

```python
# nl2dsl/graph/tool_nodes.py
TOOL_SYSTEM_PROMPT = (
    "你是元数据查询助手。你的唯一职责是查询指标定义、维度和取值。"
    "不要生成 SQL，不要回答数据本身的问题。"
    "如果不需要查询元数据，直接回复「不需要」即可。"
)

def make_tool_agent_node(llm_client, registry, max_steps: int = 6):
    @with_error_handler("tool_agent")
    def tool_agent_node(state: QueryState) -> dict:
        # 保险 1：没有 LLM 或没有工具 → 直接跳过
        if llm_client is None or len(registry) == 0:
            return {"metadata_context": None,
                    "trace": {"step": "tool_agent", "status": "skipped",
                              "reason": "no_llm_or_tools"}}

        # 保险 2：预算耗尽 → 收口退出
        steps = state.get("tool_steps") or 0
        if steps >= max_steps:
            return {"metadata_context": _summarize(state),
                    "trace": {"step": "tool_agent", "status": "skipped",
                              "reason": "budget_exhausted", "steps": steps}}

        messages = state.get("tool_messages") or [
            {"role": "user", "content": state["question"]}
        ]

        result = llm_client.chat_with_tools(messages, registry.openai_schemas())

        # 追加模型这条消息（有 tool_calls 时必须原样保留）
        messages.append(_to_assistant_message(result))

        if not result.tool_calls:
            # 模型认为不需要再查了 → 结束旁路
            return {"tool_messages": None,      # 清空，别把中间态带进主链路
                    "metadata_context": _summarize_text(messages),
                    "trace": {"step": "tool_agent", "status": "success",
                              "steps": steps, "tool_calls": 0}}

        # 有待执行的工具 → 交给 tools 节点
        return {"tool_messages": messages,
                "tool_steps": steps + 1,
                "pending_tool_calls": result.tool_calls,
                "trace": {"step": "tool_agent", "status": "tool_call",
                          "count": len(result.tool_calls)}}
    return tool_agent_node
```

**节点 B：`tool_exec_node` —— 执行工具并把结果写回消息**

```python
def make_tool_exec_node(registry, audit_logger=None):
    @with_error_handler("tools")
    def tool_exec_node(state: QueryState) -> dict:
        messages = list(state.get("tool_messages") or [])
        calls = state.get("pending_tool_calls") or []

        for call in calls:
            tool = registry.get(call["name"])          # 未知工具会抛 KeyError → 被 error_handler 接住
            started = time.time()
            try:
                output = tool.handler(**call["arguments"])
                content = json.dumps(output, ensure_ascii=False, default=str)
                status = "ok"
            except Exception as exc:
                # ★ 失败也要写回消息，否则下一轮请求会因为历史不完整而报错
                content = json.dumps({"error": f"工具执行失败：{exc}"},
                                     ensure_ascii=False)
                status = "error"

            messages.append({"role": "tool",
                             "tool_call_id": call["id"],
                             "content": content})

            # ★ 工具调用是审计盲区，必须单独埋点
            if audit_logger:
                audit_logger.log(
                    query_id=state["query_id"], user_id=state["user_id"],
                    event="tool_call", tool=call["name"],
                    args=call["arguments"], status=status,
                    elapsed_ms=int((time.time() - started) * 1000),
                )

        return {"tool_messages": messages,
                "pending_tool_calls": None,
                "trace": {"step": "tools", "status": "success",
                          "executed": len(calls)}}
    return tool_exec_node
```

#### 9.4.8 路由与图改造

**第一步：决定"要不要走旁路"**

三种触发方式，按成本排：

| 方式 | 怎么做 | 优点 | 缺点 |
|---|---|---|---|
| **A. 关键词规则** | 问题含"怎么算/口径/定义/包含哪些/有哪些维度/取值" | 零成本、可控 | 覆盖不全 |
| **B. planner 判断** | 让 planner 输出一个 `needs_metadata` 字段 | 更准 | 要改 planner 的 prompt 和 schema |
| **C. 永远都进** | 无条件进 tool_agent，它自己判断 | 最全 | 每次多一次 LLM 调用（费钱） |

**推荐先做 A**（第一版够用），跑一段时间看命中率，再升级到 B。

```python
# nl2dsl/graph/edges.py

_METADATA_HINTS = ("怎么算", "口径", "定义", "包含哪些", "有哪些维度",
                   "维度有哪些", "取值", "可选值", "字段")

def route_after_plan(state: QueryState) -> str:
    """在原有逻辑上加一个 explore 分支。"""
    plan = state.get("plan")

    if plan is None:
        return "continue"

    # 原有逻辑：非单查询交给 Agent 层
    if plan.intent != "single_query":
        return "agent"

    # 新增：需要查元数据 → 先走旁路
    question = state.get("question") or ""
    if state.get("tools_enabled") and any(k in question for k in _METADATA_HINTS):
        return "explore"

    return "continue"
```

**第二步：工具循环怎么收口**

```python
def route_after_tool_agent(state: QueryState) -> str:
    # 出错 → 回主链路（这才是"旁路"的意义）
    if state.get("status") == "error":
        return "fallback"

    # 还有待执行的工具 → 去执行
    if state.get("pending_tool_calls"):
        return "tools"

    # 没得调了 → 回主链路
    return "done"
```

**第三步：改 `builder.py`**

```python
    # ---- 旁路节点：仅当注册表非空时加入（关掉就和原来一模一样）----
    has_tools = tool_registry is not None and len(tool_registry) > 0

    if has_tools:
        builder.add_node("tool_agent", nodes["tool_agent_node"])
        builder.add_node("tools", nodes["tool_exec_node"])

        builder.add_conditional_edges("plan", route_after_plan, {
            "continue": "decompose",
            "agent": END,
            "explore": "tool_agent",          # ← 新增分支
        })

        builder.add_conditional_edges("tool_agent", route_after_tool_agent, {
            "tools": "tools",
            "done": "decompose",
            "fallback": "decompose",          # ← 出错也回主链路
        })

        builder.add_edge("tools", "tool_agent")   # ← 循环边
    else:
        # 原来的边，一个字不动
        builder.add_conditional_edges("plan", route_after_plan, {
            "continue": "decompose",
            "agent": END,
        })
```

> **注意 `if/else` 的结构**：关掉工具时，图的结构和现在**完全一致**。这样回归测试可以保证"没开这个功能时行为不变"。

#### 9.4.9 主链路怎么消费工具查到的内容

旁路只产出一样东西：`metadata_context`（一段拼好的文本）。主链路在**生成 DSL 之前**把它拼进上下文：

```python
# nl2dsl/graph/nodes.py —— generate_dsl_node 里
rag_context = rag_retriever.build_context(question)

# 新增三行：把工具查到的元数据拼在前面
meta = state.get("metadata_context")
if meta:
    rag_context = f"【工具查询到的元数据】\n{meta}\n\n{rag_context}"
```

**为什么拼在 RAG 上下文前面**：它比向量召回更精确（是模型主动要的），优先级更高。

#### 9.4.10 四道保险（旁路的本质）

| 保险 | 触发条件 | 行为 |
|---|---|---|
| **开关** | `tools_enabled=False`（默认） | 图退化成原样，零影响 |
| **无依赖** | `llm_client is None` 或注册表为空 | 直接跳过，不报错 |
| **预算** | `tool_steps >= max_steps`（建议 6） | 强制收口，用已查到的内容继续 |
| **降级** | 旁路任何异常 | `route_after_tool_agent` 返回 `fallback` → 回主链路 |

> **核心思想：工具是"锦上添花"，不是"雪中送炭"。它坏了，查询必须照常能跑。**

#### 9.4.11 上线顺序（重要，别一次改完）

```
第 1 步：加 state 字段 + llm.chat_with_tools     ← 不动图，跑全量回归，必须全绿
第 2 步：加 tools/ 注册表和三个工具               ← 不接图，单测工具本身
第 3 步：加两个节点 + 路由，但默认开关关闭         ← 图结构没变，回归必须全绿
第 4 步：打开开关，跑端到端                        ← 观察 trace
第 5 步：线上灰度（10% 流量）→ 全量
```

**每步都要能回滚**。第 1、2 步即使全部保留，也不影响现有行为。

#### 9.4.12 测试怎么写

```python
# 1. 工具本身的单测（不涉及模型）
def test_explain_metric_returns_definition():
    tools = build_metadata_tools(registry_dict)
    tool = next(t for t in tools if t.name == "explain_metric")
    assert tool.handler("GMV")["expression"] == "SUM(pay_amount)"

def test_explain_metric_unknown_returns_available():
    result = tool.handler("不存在的指标")
    assert "available" in result        # 错误信息要能指导模型

# 2. 路由的单测（不涉及模型）
def test_route_explore_when_metadata_hint():
    state = {"question": "GMV 怎么算的", "tools_enabled": True,
             "plan": Plan(intent="single_query")}
    assert route_after_plan(state) == "explore"

def test_route_continue_when_no_hint():
    state = {"question": "华东销售额", "tools_enabled": True,
             "plan": Plan(intent="single_query")}
    assert route_after_plan(state) == "continue"

# 3. 旁路错不影响主链路（最关键的回归）
def test_tool_branch_error_falls_back():
    state = {"status": "error"}
    assert route_after_tool_agent(state) == "fallback"

# 4. 开关关闭时行为不变
def test_graph_unchanged_when_tools_disabled():
    graph = build_graph(..., tool_registry=None)
    assert "tool_agent" not in graph.get_graph().nodes
```

#### 9.4.13 五个坑

| 坑 | 后果 | 避免方法 |
|---|---|---|
| **循环没上限** | 模型反复调工具，请求永远不返回 | `tool_steps` + `max_steps`；LangGraph 还有 `recursion_limit` |
| **工具抛异常没接住** | 消息历史不完整，下一轮 API 直接报错 | `tool_exec_node` 里 try/except，失败也写 `role: tool` 消息 |
| **工具结果太长** | 撑爆上下文，主链路反而变差 | 工具返回摘要（`available` 只给前 20 个） |
| **旁路污染主链路状态** | 复用 `dsl` 字段导致主链路拿到半个 DSL | 旁路**只写** `metadata_context`，不碰 `dsl` / `sql` / `status` |
| **没有审计** | 出事查不到"模型调了什么工具、传了什么参数" | 每次调用单独埋点（不依赖查询链路的审计） |

**读完这节你应该能**：说出改造要动哪 5 个文件、为什么要先加 `chat_with_tools`、四道保险分别防什么、以及为什么出错要回主链路而不是直接失败。

### 9.5 工具该嵌在哪：三种位置，别都塞进旁路

#### 9.5.1 判断标准

| 位置 | 谁决定执行 | 确定性 | 适用 |
|---|---|---|---|
| **A. 工具旁路** | 模型决定调不调 | 低（可能不调） | 只在**部分场景**需要的信息 |
| **B. 系统节点内** | 代码决定，必然执行 | 高 | **每次都需要**，或某条件下必须有的信息 |
| **C. 规则守卫** | 规则触发，零 LLM 成本 | 最高 | 能纯规则判断的场景 |

**核心判断就一句**：

```
这条信息是不是"每次都需要"？
├─ 是 → B 或 C（写成代码，别指望模型每次都调）
└─ 否 → A（交给模型判断）
```

#### 9.5.2 四个高价值工具分别该放哪

| 工具 | 每次都需要吗 | 最佳位置 |
|---|---|---|
| `check_data_freshness` | 否，但**结果为空时必须解释** | **B（空结果兜底）+ A（模型按需调）** |
| `search_columns` | 否，但**校验失败时必须给候选** | **B（错误修复）+ A** |
| `dry_run` | **是**，每次生成后都该验 | **B（修正回路内）+ A（模型自查）** |
| `explain_term` | 否，只在术语相关时 | **A（工具）+ B（澄清升级）** |

#### 9.5.3 逐个说清嵌入点

**① `check_data_freshness` → `verify_dsl` 节点（空结果兜底）**

```python
# graph/nodes.py 的 verify_dsl_node 里
data = state.get("data") or []
if len(data) == 0:
    freshness = check_data_freshness(state.get("data_source"))
    verify_reason = (
        f"查询结果为空。数据范围为 {freshness['earliest']} ~ {freshness['latest']}"
        f"（延迟 {freshness['delay_days']} 天），"
        f"请确认用户查询的时间范围是否在数据范围内。"
    )
```

**为什么放这里而不是旁路**：用户看到空结果时必须得到解释。放旁路的话，模型可能不调，用户就又一次看到"0 行，不知道为什么"。

**② `search_columns` → `correct_dsl` 节点（校验失败自动修复）**

你的 `correct_dsl` 现在是"LLM 挑关键词 → RAG 召回 → 重新生成"，**RAG 这一步精准度最低**。换成 `search_columns`：

```python
except ValidationError as e:
    missing = _extract_missing_name(str(e))     # 从"维度 'brand_x' 不存在"抠出名字
    candidates = search_columns(missing, limit=5)
    prompt = f"""
【错误】{e}
【可能的正确字段】
{json.dumps(candidates, ensure_ascii=False, indent=2)}
请从中选择正确字段重新生成 DSL。"""
```

价值：把"模型瞎猜 + RAG 召回"变成"模型看着候选挑"。

**③ `dry_run` → `correct_dsl` 生成之后（快速预演）**

```python
new_dsl = llm_generate(...)
result = dry_run(new_dsl, user_id)
if not result["ok"]:
    return {"error_hint": result["errors"]}     # 不回主链路，当场再改一次
```

**注意**：`dry_run` 和 `validation` 节点**必须复用同一个 validator 实例**，否则会出现"预演通过但真校验失败"的打脸。

**④ `explain_term` → `clarification` 节点（替换硬编码关键词表）**

现在的 `clarification.py` 是写死的：

```python
_METRIC_AMBIGUOUS = {"销量": [...], "客户数": [...]}   # 只覆盖两个词
```

改成配置驱动：

```python
def detect(self, question: str) -> list[ClarificationItem]:
    items = []
    for term in _extract_candidate_terms(question):
        info = explain_term(term)
        if "error" in info or not info.get("confusable_with"):
            continue
        items.append(ClarificationItem(
            type="term_ambiguous",
            question=f'您说的「{term}」是指 {info["standard_name"]} 吗？',
            options=[info["standard_name"]] + [c["name"] for c in info["confusable_with"]],
        ))
    return items
```

价值：澄清从"硬编码几个词"变成"**加一个术语就自动获得歧义检测**"，不用改代码。

#### 9.5.4 图上标出来

```
                    ┌──────────────────┐
                    │  clarification   │◄── B: explain_term 升级歧义检测
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │       plan       │
                    └────────┬─────────┘
                             ▼
              ┌──────────────────────────────┐
              │      tool_agent（旁路）       │◄── A: 四个工具都可被模型调
              └──────────────┬───────────────┘
                             ▼
                    ┌──────────────────┐
                    │    decompose     │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────────────────┐
                    │  validation / correct_dsl     │◄── B: search_columns 给候选
                    │                              │◄── B: dry_run 快速预演
                    └──────────────┬───────────────┘
                                   ▼
                          ... build_sql / scan ...
                                   ▼
                    ┌──────────────────┐
                    │   execute_sql    │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────────────────┐
                    │       verify_dsl             │◄── C: 0 行 → check_data_freshness
                    └──────────────┬───────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │        explain               │◄── B: 新鲜度/术语差异写进回答
                    └──────────────────────────────┘
```

#### 9.5.5 还有个维度：不只在图内

同一个工具函数，可以在**三个层次**暴露：

| 层次 | 形态 | 用途 |
|---|---|---|
| **图内** | 节点 / 旁路 | 服务自带 Agent |
| **API 层** | `GET /api/v1/tools/{name}` | 前端直接调（用户点"数据到几号了"） |
| **MCP 服务** | MCP server | 给 Cursor / 其他 Agent 复用 |

**代码只写一次，三种方式暴露。**

#### 9.5.6 反模式：别把所有工具都塞进旁路

```
❌ 四个工具全交给 tool_agent，让模型决定用哪个
```

| 后果 | 说明 |
|---|---|
| **行为不一致** | 同一个问题，这次调了新鲜度检查、下次没调，用户看到的结果不一样 |
| **成本不可控** | 每次多一次 LLM 决策 |
| **关键能力失效** | 空结果必须解释——模型可能就是不调，用户照样懵 |

**正确做法**：

```
必须发生的 → 写成节点逻辑（B / C）
可能发生的 → 交给模型（A）
```

#### 9.5.7 落地顺序

| 顺序 | 做什么 | 工作量 |
|---|---|---|
| 1 | `check_data_freshness` 塞进 `verify_dsl` | 半天 |
| 2 | `explain_term` 替换 `clarification` 的硬编码表 | 半天 |
| 3 | `search_columns` 接进 `correct_dsl` | 半天 |
| 4 | `dry_run` 接进 `correct_dsl` | 1 天 |

**读完 9.5 你应该能**：说出四种嵌入位置的取舍，并解释为什么"空结果兜底"必须写成代码而不是交给模型。

---

## 第 10 节：自测 10 题

**本节要解决的问题**：验证你真的看懂了，而不是"看过了"。

<details>
<summary>1. 模型会执行函数吗？</summary>

不会。模型只输出"我想调哪个函数、参数是什么"（`tool_calls`），执行的是你的代码。
</details>

<details>
<summary>2. 为什么工具调用要请求两次？</summary>

第一次拿"申请单"（tool_calls），执行后第二次带上结果才拿答案。它是个循环，不是单次调用。
</details>

<details>
<summary>3. `arguments` 为什么是字符串？</summary>

因为模型在"生成文本"，它生成的是一段 JSON 文本。你需要 `json.loads` 转成字典。
</details>

<details>
<summary>4. `tool_call_id` 是干什么的？</summary>

把"工具结果"和"模型的申请"对应起来。一次可能申请多个，靠 id 区分。漏了会报错。
</details>

<details>
<summary>5. 参数用 `enum` 的好处是什么？</summary>

物理上限制模型只能传合法值，是**防幻觉最有效的手段**。但服务端仍要二次校验。
</details>

<details>
<summary>6. 并行和串行调用怎么选？</summary>

互不依赖 → 并行（省时间）；下一步依赖上一步结果 → 串行（ReAct 循环）。
</details>

<details>
<summary>7. 自主循环为什么必须有步数上限？</summary>

模型可能反复调同一工具或在错误方向试很多次，没有上限会在线上卡死。一般 5–10 步。
</details>

<details>
<summary>8. 工具执行失败时该返回什么？</summary>

返回**给模型看懂的**错误信息（包含"有哪些可选"），不要裸抛异常。错误信息本质是 prompt。
</details>

<details>
<summary>9. MCP 和 Function Calling 什么关系？</summary>

上下两层：Function Calling 是模型表达"要调工具"的方式；MCP 是工具被发现和调用的协议标准。不冲突。
</details>

<details>
<summary>10. 为什么 SQL 执行不该做成工具？</summary>

会让"决策"和"执行"耦合，失去可校验的中间层（DSL）、精确的权限注入、以及分环节的错误归因。
</details>

---

## 第 11 节：面试怎么答

**Q：你们 Agent 支持工具调用吗？**
> "目前没有，是编排型 Agent——LLM 只做规划和生成 DSL，执行走固定 Graph。这不是能力缺失，是安全选择：SQL 要过校验、权限、扫描、沙箱四道闸。元数据类工具（查指标口径、查维度取值）我设计了方案：注册表 + ToolNode + 审计限流，只是还没落地。"

**Q：Function Calling 的原理？**
> "模型不执行函数，只输出 tool_call——函数名加一段 JSON 字符串参数。应用执行后把结果作为 tool 消息回传，模型再据此生成答案。所以它是个循环：每轮都要带完整历史，token 成本随轮次增长。"

**Q：tool_call 参数解析失败怎么办？**
> "三层：strict schema 保证合法 JSON；解析失败把错误信息回灌让模型重试，最多两次；仍失败就降级。参数用 enum 能把大部分失败提前消灭。"

**Q：怎么防工具越权？**
> "两道关：工具级 RBAC，控制这个角色能调哪些工具；参数级白名单，enum 加服务端二次校验。不信模型传进来的任何值。"

**Q：MCP 了解吗？**
> "了解。它把工具做成独立服务，用统一协议暴露，任何支持 MCP 的客户端都能调。和 Function Calling 是上下两层：FC 管模型怎么表达调用意图，MCP 管工具怎么被发现和接入。"

---

## 第 12 节：一页速查

| 问题 | 答案 |
|---|---|
| 本质 | 模型输出调用意图，你的代码执行，结果回传，循环 |
| 谁执行函数 | **你的代码**，不是模型 |
| 消息结构 | user → assistant(tool_calls) → tool(tool_call_id) → assistant |
| 三个坑 | arguments 是字符串 / 必须带 tool_call_id / 可能多个调用 |
| schema 三要素 | name（动词+名词）、description（写何时用）、parameters（enum 收敛） |
| 防幻觉最有效 | 参数用 enum + 服务端二次校验 |
| 三种形态 | 单次 / 并行 / 串行（ReAct 循环） |
| 循环必须有 | 步数上限（5–10） |
| 手写核心 | 20 行：调模型 → 有 tool_calls 就执行回填 → 没有就返回 |
| 五个工程问题 | 超时 / 重试（仅幂等） / 幂等键 / 参数校验 / 审计 |
| MCP 地位 | 工具即服务的协议标准；与 FC 是上下层关系 |
| 该做工具的能力 | 只读、幂等、不越权的**元数据查询** |
| 不该做工具 | SQL 执行（要过校验链） |
| 一句话 | **模型负责想调什么，你的代码负责安全地执行。** |
