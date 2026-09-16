# 工具节点 CheatSheet

> 沉淀工具调用相关的提问与解答：模型接入（init_chat_model）、工具调用协议、工具节点审批、
> ToolNode 预构建节点、tools_condition 路由、wrap_tool_call 审批钩子（对齐官方 HumanInTheLoopMiddleware 协议）。
> 中断主题的问答见 [../6_中断/CheatSheet.md](../6_中断/CheatSheet.md)。流程可视化：[工具调用流程图.drawio](工具调用流程图.drawio)

## 一、模型接入（init_chat_model）

### 1.1 `init_chat_model()` 可以填哪些参数？

显式参数只有 4 个，其余全部走 `**kwargs` 透传给底层 ChatModel 的 `__init__`：

| 参数 | 作用 |
|---|---|
| `model`（位置参数） | 模型名。带前缀 `'openai:gpt-4o'`（推荐）或裸名 `'deepseek-flash'`——按前缀**尽力推断** provider（`deepseek...` → deepseek、`gpt-...` → openai、`claude...` → anthropic）。官方建议用固定版本号而非别名，防止别名上游重指向 |
| `model_provider` | 单独指定 provider，等价于前缀写法。可选值：openai / anthropic / azure_openai / google_vertexai / google_genai / bedrock / cohere / deepseek / xai / mistralai / perplexity 等（各自需要对应 integration 包） |
| `configurable_fields` | 把字段做成**运行时可配**（如 `"model,temperature"` 或 `"any"`），invoke 时经 `config["configurable"]` 切换 |
| `config_prefix` | 可配字段的键前缀，多模型共存避免键冲突；空字符串 = 无前缀 |
| `**kwargs` | 透传底层 ChatModel：`temperature` / `max_tokens` / `timeout` / `max_retries` / `base_url` / `api_key` / `rate_limiter`，及 provider 私有参数 |

存在意义：**换 provider 只改字符串，图代码零改动**。

### 1.2 DeepSeek 私有参数（如 `thinking`）填在哪里？

填 **`extra_body`**——provider 私有参数不是 OpenAI SDK 标准字段，SDK 规定非标准字段经 `extra_body` **合并进 HTTP 请求的 JSON body**。两种写法实测等效：

```python
# init_chat_model：kwargs 原样透传给 ChatDeepSeek.__init__
model = init_chat_model("deepseek-flash", extra_body={"thinking": {"type": "disabled"}})

# 直接用 ChatDeepSeek（第 2/3 章写法）
model = ChatDeepSeek(model="deepseek-v4-flash", extra_body={"thinking": {"type": "enabled"}})
```

区分两类参数：**标准参数**（`temperature` / `max_tokens` / `timeout`）直接作顶层 kwargs 传；**provider 私有参数**（`thinking`、`thinking_budget` 等）进 `extra_body`。

## 二、工具调用协议

### 2.1 节点调用工具的固定写法（三段式）

```python
# 1. 绑定：告诉模型有哪些工具可用（只影响模型，不执行）
tools = [get_weather, get_news]
model_with_tools = model.bind_tools(tools)

# 2. llm_node：读完整历史 -> 调模型 -> 返回消息增量
def llm_node(state) -> ...:
    ai_msg = model_with_tools.invoke(state["messages"])
    return {"messages": [ai_msg]}        # 路由方式见下表

# 3. tool_node：遍历 tool_calls -> 逐个执行 -> 每个 call 一条 ToolMessage
def tool_node(state) -> dict:
    last = state["messages"][-1]                     # 最后一条 = 带 tool_calls 的 AIMessage
    tool_map = {t.name: t for t in tools}            # 名字 -> 工具对象
    messages = []
    for tc in last.tool_calls:                       # 一条 AIMessage 可带多个调用
        selected = tool_map.get(tc["name"])
        result = (selected.invoke(tc["args"])
                  if selected else f"未知工具 {tc['name']}")
        messages.append(ToolMessage(content=str(result), tool_call_id=tc["id"]))
    return {"messages": messages}
```

**4. 组装与路由：把"结果返还模型"闭合起来**

```python
builder = StateGraph(state_schema=State)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", tool_node)
builder.add_edge(START, "llm_node")
builder.add_edge("tool_node", "llm_node")     # 工具结果回传模型（固定下一跳，静态边）

# llm_node 的下一跳（二选一）：
# A. 条件边版：llm_node 返回普通 dict，由 route 读 state 决定去向
def route(state) -> str:
    return "tool_node" if state["messages"][-1].tool_calls else END
builder.add_conditional_edges("llm_node", route, ["tool_node", END])

# B. Command 版：llm_node 自己决定去向（不再需要条件边）
def llm_node(state) -> Command[Literal["tool_node", END]]:
    ai_msg = model_with_tools.invoke(state["messages"])
    goto = "tool_node" if ai_msg.tool_calls else END
    return Command(goto=goto, update={"messages": [ai_msg]})
```

**"结果返还模型"的语义**：不是特殊通道，而是两件事的组合——`ToolMessage` 经 `add_messages` **追加进 messages**（数据回流），路由把控制流**交回 llm_node**（控制回流）；下一轮 `invoke(state["messages"])` 时模型自然看到调用与应答。缺一不可。

**三条铁律**：

1. `tool_calls` 结构固定：`[{'name': 工具名, 'args': 参数字典, 'id': 调用id}]`——执行时要靠 `名字 → 工具对象` 映射表找回工具；
2. **每个 `tool_call_id` 必须有一条对应 ToolMessage**（协议要求，拒绝也要回填）；
3. 循环闭合：`tool_node` 执行完必须回到 `llm_node`，由模型判断是否继续调工具。

**变体对照**（骨架不变，变的是三处）：

| 变化点 | 写法 | 出处 |
|---|---|---|
| 路由 | llm_node 返回 `Command(goto="tool_node"/END)` | 第 7 节、3_控制流 工具审批案例 |
| | 条件边 `add_conditional_edges` + route 函数 | 3_控制流 模拟工具调用案例 |
| tool_node 实现 | 手写循环（教学透明，见 2.1）/ 预构建 `ToolNode(tools)`（第五节，支持 `wrap_tool_call` 审批钩子） | 第五节 |
| tool_node 增强 | 两阶段审批、失败重试（ToolMessage 带失败文案让模型自行重调） | 第 7 节 |

不想手写：`create_react_agent(model, tools, checkpointer=...)` 一行内置全部三段。

### 2.2 状态更新的形状规则：`{"messages": messages}` 与 `{"messages": [ai_msg]}` 为什么一个不包一个包

`update` 交给 `add_messages` 的值必须是"**一批消息的集合**"（可迭代、逐条合并）。两处代码手里拿的东西形状不同：

| 位置 | 手里拿的 | 直接给的结果 | 正确写法 |
|---|---|---|---|
| `tool_node` 的 `messages` | **列表**（循环攒的多条 ToolMessage） | 本身就是集合 ✅ | `update={"messages": messages}` |
| `llm_node` 的 `ai_msg` | **单条** AIMessage（pydantic 对象，不可迭代） | 遍历它 → `TypeError` ❌ | 包一层 `update={"messages": [ai_msg]}` ✅ |

两种报错对应两种错误：

1. **单条不装箱**：`update={"messages": ai_msg}` → `TypeError: 'AIMessage' object is not iterable`；
2. **整箱套箱**：`update={"messages": [messages]}` → 外层遍历拿到的是"一整箱"，拆开不是消息 → `ValueError: MESSAGE_COERCION_FAILURE`（实测）。

一句话规律：**给的就是集合要求的形状——手里一批直接给，手里一条装进箱，绝不整箱套箱**。

### 2.3 ToolMessage 的两个 id

```python
ToolMessage(content='...', id='4a99e497-...', tool_call_id='call_00_...')
```

| 字段 | 谁生成 | 作用 | 要不要手动填 |
|---|---|---|---|
| `tool_call_id` | **模型服务端**（call_00_... 调用编号） | 配对键：告诉模型这条结果回应哪次调用 | **必须填**，与 AIMessage 里对应 `tool_call` 的 id 一致 |
| `id` | **LangChain 客户端**（不传自动生成 uuid） | 去重键：`add_messages` 按 id 同 id 替换、新 id 追加 | 一般**不填**——随机 uuid 保证正常追加 |

手动写 `id` 的唯一场景：想**替换**某条已有消息——用相同 id 重写即可覆盖（HITL 编辑工具参数的 update 路径、修复对话历史）。

**案例：用同 id 覆盖实现"编辑历史里的 tool_calls"**（审核与编辑模式的 update 路径骨架）：

```python
# 用户把模型生成的 delete_path("/tmp/a") 改成 delete_path("/backup/a")
# 构造一条同 id 的 AIMessage，add_messages 按 id 覆盖原消息，tool_call_id 绑定不丢
fixed = AIMessage(
    content=orig.content, id=orig.id,                # ← 同 id：覆盖而非追加
    tool_calls=[{"name": "delete_path",
                 "args": {"path": "/backup/a"},
                 "id": orig.tool_calls[0]["id"]}],    # ← tool_call_id 原样保留
)
return Command(goto="tool_node", update={"messages": [fixed]})
```

对比第七节 wrap_tool_call 的 edit 决策——那条路用 `request.override(tool_call=...)` 只改执行参数（不碰历史消息），这条路直接改写历史。两者取舍：要让模型**看到**被修正的调用用后者（覆盖历史），只想修正执行且保留原始痕迹用前者。

一句话：`tool_call_id` 是**对话协议的配对键**（对模型负责），`id` 是**消息列表的去重键**（对 reducer 负责）。

## 三、工具节点内审批（多工具）

> 手写版实现（教学透明）。同一需求的 ToolNode 钩子版（对齐官方协议）见第七节，两者演进关系：手写两阶段 → wrap_tool_call 钩子。

### 3.1 标准写法：策略表 + 两阶段

```python
NEED_APPROVAL = {"get_weather": False, "get_news": True}   # 未登记的工具默认要审批

def tool_node(state: State) -> dict:
    last = state["messages"][-1]

    # 阶段一：先收齐所有审批决策（不做任何有副作用的动作）
    decisions = {}
    for tc in last.tool_calls:
        if NEED_APPROVAL.get(tc["name"], True):
            decisions[tc["id"]] = interrupt(f"允许调用 {tc['name']}({tc['args']})?")
        else:
            decisions[tc["id"]] = True

    # 阶段二：决策齐了才统一执行（重放时阶段一的 interrupt 依次返回缓存值，不再暂停）
    messages = []
    for tc in last.tool_calls:
        if decisions.get(tc["id"]):
            messages.append(ToolMessage(content=str(tools[tc["name"]].invoke(tc["args"])), tool_call_id=tc["id"]))
        else:
            messages.append(ToolMessage(content=f"用户拒绝执行 {tc['name']}，请勿重试，直接回答", tool_call_id=tc["id"]))
    return {"messages": messages}
```

实测（计数器验证）：同意场景 weather=1/news=1；拒绝场景 weather=1/**news=0**（拒绝的工具从未执行），模型如实告知查询被拒。

### 3.2 两个实测踩到的坑

**坑 1：恢复值必须显式转 bool。** 传字符串 `'False'` —— 非空字符串是真值，`if not approved` 判成了"同意"，被拒工具照样执行。恢复值就是传入的原样数据，input 拿到的字符串要先转换。

**坑 2：审批与执行不能混在同一个循环里。** "边审批边执行"时，免审批工具在 interrupt 之前就执行了；恢复时整个节点从头重跑，**它会再执行一次**（第一次的产物随本地变量丢失）——读接口无害，免审批工具若是发邮件就是事故。修正：先收齐全部决策、再统一执行，执行阶段没有任何 interrupt，重放时不会重复干活。

### 3.3 循环里调用 interrupt() 为什么这里是安全的？

官方反对的是"断点**数量/顺序不确定**"，不是循环本身。官方要求：

> ensure the number and order of interrupt calls within the node is **deterministic**

判定标准：**循环的数据源在"暂停 → 恢复"之间会不会变**。

| 循环数据源 | 判定 |
|---|---|
| `state["messages"][-1].tool_calls`（已提交进 checkpoint 的 AIMessage） | ✅ 安全——重跑读到同一批写入，逐位一致 |
| `fetch_live_data()` / 重新调用 LLM 推理 | ❌ 每次执行结果可能变 |
| 模块级可变变量、运行时可改的配置 | ❌ 实测错配：暂停时数据源删掉一条，"给B的答案"被塞给了A |

附带纪律：`NEED_APPROVAL` 策略表也是循环结构的一部分，暂停期间修改（Jupyter 改 cell 重跑）同样会错配——生产中策略表应是**部署期固定配置**。

另见 [../6_中断/CheatSheet.md](../6_中断/CheatSheet.md) Q15：tool_node 下一跳固定时用静态边 + 普通返回值，`Command(goto=...)` 留给运行时决定去向的场景；以及 `update={"messages": [messages]}` 嵌套列表的 MESSAGE_COERCION_FAILURE 坑。

## 四、审批逻辑放在哪里？（三种位置对比）

同一个"逐个审批"需求，代码可以放在三个位置：

| 维度 | ① 独立 review_node（外置） | ② tool_node 内（两阶段） | ③ @tool 函数体内 |
|---|---|---|---|
| 职责 | 决策、执行分离，单一职责 ✓ | 混合（决策+执行一个函数） | 决策执行同处，粒度最细 |
| 重放副作用 | **天然安全**：决策与执行物理分离在两个节点 | 靠"两阶段纪律"规避——阶段一加行副作用代码就埋雷 | 天然安全（工具体内先 interrupt 再动作） |
| 复用性 | review_node 每个图要复制，且与 tool_node 耦合 | ✓ 整块 tool_node 搬走即可 | **最强**：工具挂到任何图自动带审批 |
| 拓扑可见 | ✓ 审批环节在图上一目了然 | ✗ 拓扑只见 tool_node | ✗ 完全隐藏 |
| 决策依据 | 基于 state（已提交数据） | 基于 state | **可基于前序工具的输出**（如读完文件再决定是否批准写入） |
| 排查 | get_state 可见 review_node 挂起 ✓ | 任务内挂起，外部看不清 | 同左 |

③ 的最小实现（`@tool` 函数体内直接 `interrupt()`，`GraphInterrupt` 会从 ToolNode 冒泡暂停整图）：

```python
@tool(parse_docstring=True)
def send_email(to: str, subject: str, body: str) -> str:
    """发送邮件..."""
    decision = interrupt(f"是否允许发送邮件给 {to}？")
    if decision is True or (isinstance(decision, dict) and decision.get("approved")):
        ...  # 真正发送
        return "邮件已发送"
    return "用户取消了本次发送"      # 拒绝也要 return 文案，让模型知情
```

**选择建议**：

- 生产业务 → **①外置**：决策与执行分离从结构上消灭"重放副作用"错误（官方 HITL 中间件在模型输出与工具执行之间拦截，同为决策/执行分离形态，见 6_中断/CheatSheet Q4）。
- 工具本身是产品、要随工具分发 → **③工具体内**。
- 理解底层 / 深度定制 → **②内置两阶段**（对纪律要求最高，教学跳板；ToolNode 化形态见第七节）。

三种形态通用原则：**决策（问人）与执行（干活）分离，副作用只在拿到许可后发生；拒绝也回填 ToolMessage**。

## 五、ToolNode 替代手写工具节点

### 5.1 手写五步与 ToolNode 的对照

| 手写步骤（见 2.1） | ToolNode 内部实现 |
|---|---|
| ① 取最后一条 AIMessage | 读 `state[messages_key][-1]`，提取 `tool_calls` |
| ② 遍历 tool_calls | 内部循环，多个调用用线程池**并行执行**（源码 `executor.map`） |
| ③ 查映射表 | 构造时 `ToolNode([tool1, tool2])` 自动按 `name` 建映射 |
| ④ `tool.invoke(args)` | 自动调用，异常按 `handle_tool_errors` 策略处理 |
| ⑤ 构造 ToolMessage | 自动生成并返回 `{"messages": [...]}` 交给 reducer 追加 |

```python
from langgraph.prebuilt import ToolNode, tools_condition

tool_node = ToolNode(tools, handle_tool_errors=...)   # 直接当节点用

builder.add_node("llm_node", llm_node)                # llm_node 返回普通 dict 即可
builder.add_node("tool_node", tool_node)
builder.add_conditional_edges("llm_node", tools_condition,
                              {"tools": "tool_node", "__end__": END})
builder.add_edge("tool_node", "llm_node")             # 静态边，路由责任全在 tools_condition
```

其他参数（签名实测）：`messages_key="messages"`（State 消息字段改名时用）、
`wrap_tool_call` / `awrap_tool_call`（干预钩子，审批最佳实践见第七节）。

### 5.2 `handle_tool_errors` 三种取值的实测行为（坑）

| 取值 | 行为 | 实测结果 |
|---|---|---|
| 默认（不传） | 只包裹 `ToolException`，其余异常**原样抛出炸图** | `ValueError('不支持的地区')` 直接 `GraphRecursionError` 前崩掉 |
| `True` | 包裹**所有**异常为 `"Error: ...\n Please fix your mistakes."` ToolMessage | 模型能自纠 ✓；但 **`GraphInterrupt` 也被吞**（见第七节坑 1）|
| `Callable[[Exception], str]` | 逐异常决定文案，handler 内可 `raise` 放行 | 最佳：放行中断 + 包裹业务异常（写法见 7.3）|

### 5.3 初学者：先手写，再换封装

- 手写才能踩到 `tool_call_id` 双向绑定、拒绝也回填、未知工具兜底这些细节——将来 ToolNode 出问题才知道去哪查；
- 手写的边界处理永远比官方封装少（异常包裹、并行、参数校验），生产环境用 ToolNode；
- 一句话：**手写是为了看懂循环，封装是为了少踩坑**。

## 六、tools_condition：官方预置的路由函数

### 6.1 本质（源码就这几行，无魔法）

```python
def tools_condition(state, messages_key="messages") -> Literal["tools", "__end__"]:
    ai_message = messages[-1]                # 兼容 dict/list/BaseModel 三种 state
    if hasattr(ai_message, "tool_calls") and len(ai_message.tool_calls) > 0:
        return "tools"                       # 返回值是约定的节点名，不是 END 常量
    return "__end__"
```

它和手写路由函数遵循**同一个路由契约**（读 state → 返回节点名），只是官方预写、判断写死。

### 6.2 与普通路由函数的对比

| | 手写路由函数 | `tools_condition` |
|---|---|---|
| 返回值 | 任意节点名/END，可多路分支 | 写死 `"tools"` / `"__end__"` 两种 |
| 判断依据 | 自定（审批结果、轮数、工具类型……） | 只看最后一条消息有没有 `tool_calls` |
| 适用 | 一切复杂路由 | 标准 ReAct 两分支的快捷方式 |

使用细节：节点名不叫 `tools` 时用映射 `{"tools": "tool_node", "__end__": END}`；
消息字段改名时传 `messages_key`。

### 6.3 使用建议

初学者**手写路由函数更好**：它只省 3 行 if，却引入"返回值 `tools` ≠ 节点名 → 需要 mapping"
的额外概念；且实际项目路由需求一变（加审批、限轮数、分流）就得换回手写。
理解后当作标准模板的快捷方式即可。

## 七、wrap_tool_call 审批钩子（对齐官方 HumanInTheLoopMiddleware）

### 7.1 官方调研结论：两条路线

| 路线 | 写法 | 适合 |
|---|---|---|
| 官方全托管 | `create_agent(model, tools, middleware=[HumanInTheLoopMiddleware(interrupts={...})])` | 生产快速接入，零样板 |
| 自建图 + 钩子 | `ToolNode(tools, wrap_tool_call=review_tool_call)` | 教学/图结构需定制 |

`HumanInTheLoopMiddleware` 源码（`langchain/agents/middleware/human_in_the_loop.py`）
定义了权威的**四决策协议**，自建图钩子应对齐它：

| 决策 | resume 值 | 官方语义 |
|---|---|---|
| approve | `{"type": "approve"}` | 执行原工具 |
| edit | `{"type": "edit", "edited_action": {"name":..., "args":...}}` | **保留原 tool_call_id**，只换 name+args |
| reject | `{"type": "reject"}` 或带 `message` | ToolMessage `status="error"`，默认文案含 *"Do not retry this tool call unless the user explicitly requests it."* |
| respond | `{"type": "respond", "message": ...}` | 人代替工具回答，ToolMessage `status="success"` |

中断 payload 官方结构：`{"action_request": {"action": 工具名, "args": 参数}, "allowed_decisions": [...], "tool_call_id": ...}`。

### 7.2 钩子签名与核心 API

```python
from langgraph.prebuilt.tool_node import ToolCallRequest  # 注意：不在 prebuilt 顶层导出

def review_tool_call(request: ToolCallRequest, execute) -> ToolMessage | ...:
    tc = request.tool_call                 # {'name':..., 'args':..., 'id':...}
    request.state / request.runtime        # 也能读 state、runtime
    execute(request)                       # 执行（原/改后的）工具，返回 ToolMessage
    request.override(tool_call=new_tc)     # 不可变替换，返回新 request
```

### 7.3 完整实现（四场景实测全过：批准/编辑/拒绝/代答）

```python
REVIEW_TOOLS: dict[str, list[str]] = {
    "get_weather": ["approve", "edit", "reject", "respond"],   # 每种工具允许的决策
}

def review_tool_call(request: ToolCallRequest, execute) -> ToolMessage | object:
    tc = request.tool_call
    allowed = REVIEW_TOOLS.get(tc["name"])
    if allowed is None:
        return execute(request)                     # 不在审批名单 → 放行

    decision = interrupt({                          # ===== 挂起等审批 =====
        "action_request": {"action": tc["name"], "args": tc["args"]},
        "allowed_decisions": allowed,
        "tool_call_id": tc["id"],
    })

    if decision["type"] == "approve" and "approve" in allowed:
        return execute(request)
    if decision["type"] == "edit" and "edit" in allowed:      # 官方语义：保留 id
        edited = decision["edited_action"]
        new_tc = {**tc, "name": edited["name"], "args": edited["args"]}
        return execute(request.override(tool_call=new_tc))
    if decision["type"] == "reject" and "reject" in allowed:
        content = decision.get("message") or (
            f"User rejected the tool call for `{tc['name']}` with id {tc['id']}. "
            "The tool was not executed. Do not retry this tool call unless the user "
            "explicitly requests it."
        )
        return ToolMessage(content=content, name=tc["name"],
                           tool_call_id=tc["id"], status="error")
    if decision["type"] == "respond" and "respond" in allowed:
        return ToolMessage(content=decision["message"], name=tc["name"],
                           tool_call_id=tc["id"], status="success")
    raise ValueError(f"不支持的决策: {decision}，允许: {allowed}")


def handle_errors(e: Exception) -> str:
    if isinstance(e, GraphInterrupt):   # 坑 1：中断是控制流信号，必须放行
        raise e
    return f"工具执行出错: {e}"

tool_node = ToolNode(tools, wrap_tool_call=review_tool_call,
                     handle_tool_errors=handle_errors)
# 图组装同第五节；compile(checkpointer=InMemorySaver())
```

恢复侧：`graph.invoke(Command(resume={"type": "approve"}), config)`，四种决策值见 7.1 表。

**调用侧审批循环案例**（`4_wrap_tool_call处理审批逻辑.ipynb` 的模式，四场景实测全过）：

```python
def run_scenario(name: str, question: str, resume_value):
    config = {"configurable": {"thread_id": name}}
    res = graph.invoke({"messages": [HumanMessage(content=question)]}, config)
    print("审批请求 ->", res["__interrupt__"][0].value)   # action_request + allowed_decisions
    res = graph.invoke(Command(resume=resume_value), config)  # 决策注入，钩子按协议处理
    for m in res["messages"][2:]:
        print(type(m).__name__, "|", m.content if m.content else m.tool_calls)

run_scenario("批准", "查一下上海的天气", {"type": "approve"})
run_scenario("编辑", "查一下上海的天气",
             {"type": "edit", "edited_action": {"name": "get_weather", "args": {"city": "北京"}}})
run_scenario("拒绝", "查一下上海的天气", {"type": "reject"})
run_scenario("代答", "查一下上海的天气",
             {"type": "respond", "message": "天气服务正在维护，请直接告诉用户：暂时无法查询"})
```

### 7.4 两个实测踩到的坑

**坑 1：`handle_tool_errors=True` 会吞掉 `GraphInterrupt`。** 钩子里的 `interrupt()`
抛出的中断被当成工具异常包成 `Error: GraphInterrupt(...)` 的 ToolMessage 回给模型，
审批直接失效（模型反而替它道歉）。必须用自定义 handler 把 `GraphInterrupt` `raise` 放行（见 7.3）。

**坑 2：恢复必须挂 checkpointer。** `Command(resume=...)` 没有 checkpointer 直接
`RuntimeError: Cannot use Command(resume=...) without checkpointer`。生产换 `PostgresSaver`。

### 7.5 实测行为补充

- 官方 reject 默认文案能明显降低模型盲目重试，**但不保证**——拒绝场景模型仍说"我可以再次尝试"，只是没有真的再调工具；
- edit 场景模型会发现"我请求的是上海，返回的却是北京"——参数确实被换了，模型如实报告（好的行为）；
- 与第四节三种位置的对应：`wrap_tool_call` 属于"工具节点内"位置的官方化形态，兼具 ② 的内聚和钩子的纪律性。
