模型生成多个工具调用（并行调用）的机制，本质上是在一次响应中同时输出多个**结构化工具调用请求**，让客户端可以并发执行这些调用，从而提高效率。下面通过一个典型示例，详细拆解其原理和流程。

---

## 1. 典型场景示例：一次询问多个信息

用户可能在一个问题中询问多个地点或多个类型的信息，例如：
> “旧金山、巴黎和北京的天气怎么样？顺便查一下德国和印度的首都是哪里？”

这个问题需要调用 **5 次工具**（3 次天气查询 + 2 次首都查询）。在支持并行调用的模型中，一次响应就可以返回全部 5 个调用请求。

### 请求示例（工具定义）
客户端在请求中提供两个工具：`get_current_weather` 和 `get_capital`。
```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_current_weather",
            "description": "获取指定城市的天气",
            "parameters": {...}  # 包含 location, unit
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_capital",
            "description": "获取指定国家的首都",
            "parameters": {...}  # 包含 country
        }
    }
]
```

### 模型响应（多个 `tool_calls`）
模型判断需要调用这些工具，于是返回如下响应（核心部分）：
```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [
        {
          "id": "call_1",
          "type": "function",
          "function": {
            "name": "get_current_weather",
            "arguments": "{\"location\":\"San Francisco, CA\",\"unit\":\"fahrenheit\"}"
          }
        },
        {
          "id": "call_2",
          "type": "function",
          "function": {
            "name": "get_current_weather",
            "arguments": "{\"location\":\"Paris, France\",\"unit\":\"celsius\"}"
          }
        },
        {
          "id": "call_3",
          "type": "function",
          "function": {
            "name": "get_current_weather",
            "arguments": "{\"location\":\"Beijing, China\",\"unit\":\"celsius\"}"
          }
        },
        {
          "id": "call_4",
          "type": "function",
          "function": {
            "name": "get_capital",
            "arguments": "{\"country\":\"Germany\"}"
          }
        },
        {
          "id": "call_5",
          "type": "function",
          "function": {
            "name": "get_capital",
            "arguments": "{\"country\":\"India\"}"
          }
        }
      ]
    },
    "finish_reason": "tool_calls"
  }]
}
```
可以看到，`tool_calls` 数组包含了 **5 个独立的调用请求**，每个都有唯一的 `id`、工具名称和参数。`finish_reason` 为 `"tool_calls"`，表明生成结束是因为模型发出了工具调用。

---

## 2. 机制与原理详解

### 2.1 模型层面的生成原理
- **训练阶段**：模型在微调时，学习到了一种能力——当用户问题涉及多个独立子任务时，可以在一次生成中输出**多个结构化的 JSON 对象**。这些对象通常用特殊标记分隔（如 `<|tool_call|>` 和 `<|/tool_call|>`），以便服务端解析。
- **推理阶段**：在自回归生成过程中，模型逐个预测 token。当它判断需要调用工具时，会输出类似以下的 token 序列（简化示意）：
  ```
  [特殊标记开始] 
  {"id": "call_1", "type": "function", "function": {"name": "get_current_weather", "arguments": {...}}}
  [分隔标记]
  {"id": "call_2", "type": "function", "function": {"name": "get_capital", "arguments": {...}}}
  [特殊标记结束]
  ```
  模型可以一次性生成多个这样的 JSON 对象，而不必等待外部循环逐个触发。

### 2.2 服务端解析与封装
- **检测与提取**：服务端在接收到模型输出的 token 流后，识别出其中的工具调用标记，将这些 JSON 对象从 `content` 字段中剥离，并放入 `tool_calls` 数组。
- **分配唯一 ID**：为每个调用生成一个唯一的 `id`（如 `call_1`、`call_2`），以便客户端后续将执行结果与调用对应起来。
- **设置 finish_reason**：将 `finish_reason` 设为 `"tool_calls"`，告知客户端本次响应不是普通文本，而是工具调用请求。

### 2.3 客户端处理流程
客户端收到包含多个 `tool_calls` 的响应后，应执行以下步骤：

1. **遍历所有工具调用**，而不是只处理第一个。
2. **并发执行**：可以并行调用多个工具（如同时查询天气和首都），提高效率。
3. **收集结果**：每个工具执行后，将结果与对应的 `tool_call_id` 关联。
4. **构造后续消息**：将所有工具的结果一次性作为多个 `tool` 角色的消息返回给模型，以便模型整合信息并生成最终回答。

代码示例（伪代码）：
```python
# 收到包含多个 tool_calls 的响应
tool_calls = response.choices[0].message.tool_calls

# 并发执行所有工具调用（例如使用 asyncio.gather）
results = await asyncio.gather(*[execute_tool(tc) for tc in tool_calls])

# 将所有结果以 tool 角色消息追加到对话历史
for tool_call, result in zip(tool_calls, results):
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": json.dumps(result)
    })

# 再次调用模型，让其整合结果并生成最终回答
final_response = client.chat.completions.create(messages=messages, tools=tools)
print(final_response.choices[0].message.content)
```
最终模型会输出类似：
> “旧金山当前气温 72°F，巴黎 22°C，北京 22°C。德国首都柏林，印度首都新德里。”

---

## 3. 并行调用的控制与优化

### 3.1 默认行为与配置
- **默认并行**：OpenAI 及多数兼容 API 默认启用并行工具调用，即只要模型认为合适，就可以在一次响应中返回多个 `tool_calls`。
- **禁用并行**：某些场景需要顺序执行（如后续调用依赖前一个结果），可以通过参数 `parallel_tool_calls=False` 强制模型每次只返回一个工具调用。例如在 OpenAI Python SDK 中：
  ```python
  llm_with_tools = llm.bind_tools(tools, parallel_tool_calls=False)
  ```
  此时即使问题需要多个调用，模型也会分多次响应完成。

### 3.2 并行调用的优势
- **减少往返次数**：原本需要多次请求才能完成的多个工具调用，现在一次完成，大幅降低延迟。
- **提升用户体验**：模型可以一次性获取所有必要信息，然后整合输出，避免用户等待多次交互。
- **支持复杂查询**：适合“多地点、多维度”的查询，如天气+汇率+新闻等组合。

---

## 4. 总结：多个工具调用的核心原理

| 层面 | 机制与原理 |
|------|------------|
| **模型生成** | 通过训练学会在一次自回归生成中输出多个结构化的 JSON 对象，用特殊标记分隔。 |
| **服务端封装** | 识别标记，将多个 JSON 对象提取到 `tool_calls` 数组，为每个调用分配唯一 ID。 |
| **客户端执行** | 并发执行所有调用，收集结果后一次性返回给模型。 |
| **控制机制** | 默认并行，可通过 `parallel_tool_calls=False` 强制顺序执行。 |

这种机制使得模型能够像“多任务处理”一样，在一个思维周期内同时规划多个外部操作，大幅提升了智能体处理复杂查询的效率和能力。