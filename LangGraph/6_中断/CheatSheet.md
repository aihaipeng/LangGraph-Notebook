# 中断 CheatSheet

> 本文件沉淀中断章节学习过程中的提问与详细解答，按主题分组。配套待办见 [../To-Do.md](../To-Do.md)。
> 工具审批的完整实现（手写两阶段 / ToolNode 钩子 / 官方四决策协议）见 [../7_工具节点/CheatSheet.md](../7_工具节点/CheatSheet.md)。

## 一、工具审批

### Q1. 用户输入 y，传回图中为什么变成了 true？

转换发生在**图外的恢复代码**里，LangGraph 不做任何转换：

```python
user_in = input(...)                                          # 字符串 "y"
approved = user_in.strip().lower() in ("y", "yes", "是", "1")  # 布尔表达式 → True/False
res = graph.invoke(Command(resume=approved), config=config)   # 传进图的就是 bool
```

`resume` 的值类型由图内外两端约定；完整的格式分档规则见 Q14。

### Q2. 拒绝时为每个 tool_call 回填 ToolMessage，这个做法对吗？

**回填是对的且必须的，简化掉的是审批粒度**——两件事要分开：

1. 对话协议（OpenAI/DeepSeek）要求：AIMessage 的每个 `tool_call_id` 都必须有对应 ToolMessage 才能继续调模型。拒绝的调用若不回填，下一轮 `model.invoke(messages)` 直接报 400。LangChain 官方 HITL 中间件的拒绝路径同样回填 ToolMessage。
2. 教程初版是"整体审批"（一次中断全部同意/全部拒绝）。多工具场景的标准做法是**逐个审批**：恢复值 `{tool_call_id: 是否同意}`，同意的执行、拒绝的回填——两种机制配合使用，不是二选一。

### Q3. 成熟开源 Agent 项目怎么做多工具审批？

| 项目 | 做法 |
|---|---|
| opencode `permission.ts` | 每个调用生成带 `callID` 的独立 Request，逐个 ask/reply；规则 allow/deny/ask 三态，只有 ask 挂起；拒绝可带反馈回传模型（CorrectedError） |
| LangChain `HumanInTheLoopMiddleware` | 一次 interrupt 打包全部待审批 `action_requests`，resume 的 `decisions` 与请求**按顺序一一对应**；approve 执行、reject 合成拒绝 ToolMessage |
| 共识 | 粒度 = 每个 tool_call；一次中断批量打包 + 逐个决策；拒绝回填是协议强制 |

当前 `7_工具审批模式.ipynb` 即按此实现：单次 interrupt 列出全部调用 + `{tool_call_id: bool}` 决策字典；ToolNode 钩子版（官方四决策协议）见 [../7_工具节点/CheatSheet.md](../7_工具节点/CheatSheet.md) 第 7.1 节。

### Q4. 审批逻辑放在独立审批节点还是工具节点里？

成熟实现都不用"独立审批节点"，而是嵌在**工具执行边界**上：

| 实现 | 位置 |
|---|---|
| opencode | 工具执行路径上的守卫/中间件（无图概念） |
| LangChain HITL 中间件 | `after_model` 钩子：模型输出与工具执行之间拦截 |
| LangGraph prebuilt `human_approval()` | `ToolNode(wrap_tool_call=...)` 每次调用的包装器，官方明确"no new node class" |

手写层面的三种合法形态（独立节点 / 工具内 interrupt / 静态中断+钩子）与逐维度对比表，
统一收敛在 [../7_工具节点/CheatSheet.md](../7_工具节点/CheatSheet.md) 第四节，此处不重复维护。

### Q5. `interrupt()` 的参数可以传列表吗？

可以。payload 只要求**可 JSON 序列化**：str / dict / list / 数字都行，原样出现在 `res["__interrupt__"][0].value` 并存入 checkpoint。本章节三种都用过：字符串（第 1 节）、dict（审核意见）、`list[dict]`（工具审批）。payload 与恢复值类型互不约束（恢复值方向的格式规则见 Q14）。

### Q6. 用列表推导构造 interrupt payload，是否踩了"循环内设断点"的禁忌？

**没有**。判断标准是 **`interrupt()` 的调用次数是否依赖运行时数据**：

```python
decisions = interrupt([... for tc in tool_calls])   # 安全：断点恒为 1 个，循环只构造 payload
for tc in tool_calls:
    ok = interrupt(...)                             # 危险：N 条 tool_call = N 个断点
```

危险写法的机理：同一节点内多个 `interrupt()` 的恢复值按**出现顺序严格索引匹配**；恢复时节点从头重跑，若循环次数/顺序与暂停时不同（数据变化），恢复值就会配错断点（索引匹配的源码级证据见 Q13）。官方 HITL 中间件正是"单断点批量打包"模式。

### Q7. 弹了两次确认弹窗，为什么只算一次中断？

**弹窗来自客户端代码的 for 循环，不是图里的断点**：

- 图：`review_node` 调 1 次 `interrupt()` → 挂起 1 次 → `__interrupt__` 长度为 1。
- 弹窗：恢复 cell 遍历 payload 里的 2 条工具调用 → `input()` 调 2 次 → 组装 decisions → 一次 `Command(resume=decisions)` 恢复。

弹窗次数 = 客户端循环次数；中断次数 = `__interrupt__` 列表长度。两层独立。

### Q8. 为什么没加 END 节点图也能正常结束？

`review_node` 全部分支走 `Command(goto=...)` 路由，`goto=END` 是合法目标，不需要静态边。补 `add_edge("review_node", END)` 后 Command 路由仍优先，只是为了与教程其他节的显式风格保持一致。

## 二、串行/并行中断

### Q9. 查 `get_state_history` 只看到第一个中断？如何看到完整过程？

**只看到第一个的三个原因**：

1. **中断是"执行到才产生"，不是预注册的**。第一次 invoke 后，第二个 interrupt 的代码从未运行，checkpoint 里没有它的任何记录——此刻查历史只有第一问，符合预期。
2. **单节点案例的特殊性**：恢复时节点在同一 super-step 内重跑，挂起快照被**原地覆盖**——"名字"的快照会被"年龄"的替换，历史里永远无法同时看到这两个中断（详见 Q12）。
3. 操作顺序问题：在第一次 invoke 后、恢复循环之前就查了历史；或跑完后**重跑了构建 cell**（新建 `InMemorySaver`，同 thread_id 历史从零开始）。

**跨节点案例看到完整过程的方法**：按 构建 → 恢复循环（输完所有答案）→ 查询 的顺序执行，不重跑构建 cell。跑完后每个节点的挂起各留一条独立快照（实测：`final → summarize → (ask_age, 年龄) → (ask_name, 名字) → __start__`）。可读化查询：

```python
for snap in graph.get_state_history(config=config):
    ints = [(t.name, i.value) for t in (snap.tasks or ()) for i in (t.interrupts or [])]
    print(f'next={tuple(snap.next or ())} state={snap.values} interrupts={ints}')
```

## 三、检查点（Checkpoint）机制

### Q10. `next` 说 ask_name 是下一个节点，`tasks` 却有它的执行结果，矛盾吗？

不矛盾——**同一条 checkpoint 被写了两次**（实测证实：恢复前后同 checkpoint_id，`result` 从 None 变为有值）：

| 时刻 | 发生了什么 | 账本变化 |
|---|---|---|
| T1 invoke | 写下这条 checkpoint，ask_name 开跑 → interrupt 挂起 | 只有 interrupts，`result=None` |
| T2 resume | 节点在**同一 super-step 内从头重跑**，interrupt() 返回恢复值，节点跑完 | 返回值作为 pending writes **原地回填** → `result` 出现 |
| T3 步进提交 | step 0 完成，写新 checkpoint | `values` 才合并该结果，`next` 指向下一节点 |

三个字段的准确含义与写入时机：

| 字段 | 含义 | 写入时机 |
|---|---|---|
| `next` | 本轮待跑的任务名清单（不是"尚未启动"，是这一页的任务归属） | 开页时写，之后封存不改 |
| `values` | 前面所有轮次的结果，**经 reducer 合并后**的已提交状态 | 翻页时更新（本条的 values 是开工前的家底） |
| `tasks` | 本轮每个任务的执行档案：`result`（收工）/ `error`（异常）/ `interrupts`（挂起） | 执行中按事件**陆续补挂** |

存储上分家：values/next 在 checkpoint 主记录（一次成型）；result/interrupts 在 pending writes 表，按 `(checkpoint_id, task_id)` 挂靠，查询时拼装成 StateSnapshot。终态快照 `next=()`、`tasks=()`。

### Q11. 为什么要设计成"一页一轮、收齐再翻页"？

本质：**每个 super-step 是一个事务**（durable execution）。四个目标倒逼：

1. **原子性**：一轮可能并行多个节点，必须全部收工才提交，否则并行分支会读到半成品状态；中途宕机丢弃未完成轮次的草稿，从本页重跑，状态不撕裂。
2. **可重放**：values/next 开页封存 → "本轮输入"永远完好 → 恢复/重启后从头重跑结果确定。HITL 能暂停数小时、跨进程恢复的物理基础。
3. **确定性合并**：并行任务完成顺序不确定；写同一 key 靠 reducer 攒到翻页时一次性合并，结果与完成顺序无关、可复现。
4. **并发与幂等**：pending writes 按任务分行 upsert，并行互不加锁，任务重跑覆盖自己那行。

同构于数据库事务/WAL 与 Google Pregel 的 super-step 同步屏障。

### Q12. 单节点串行中断，为什么后一个中断覆盖前一个的记录？

**两条中断记录写进了同一个存储槽位**。中断记录的键是 `(checkpoint_id, task_id)`（同一 super-step 内重跑、checkpoint_id 为何不变的账本细节见 Q10 的 T2 时刻）：

```
单节点：挂起#1 → (ckpt-step0, task:ask_user_info)
       恢复重跑（节点未收工 → 步未提交 → 页未翻 → checkpoint_id 不变）
       挂起#2 → (ckpt-step0, task:ask_user_info)   ← 同槽位，覆盖 #1

跨节点：挂起#1 → (ckpt-step0, task:ask_name)
       恢复 → ask_name 收工 → 翻页（新 checkpoint_id）
       挂起#2 → (ckpt-step1, task:ask_age)          ← 不同槽位，并存
```

根本原因：**checkpoint 的身份由 super-step 决定，中断不结束一个步骤**——节点内中断多少次都发生在同一页。

覆盖是正确语义而非缺陷：pending writes 槽位的含义是"该任务本轮**最新**执行状态"。恢复即整个任务从头重跑，上次执行是被废弃的半成品（从未收工、产出从未提交），记录无保留价值；若改成追加，重跑 N 次攒 N 条过期记录，提交时无法判断哪条算数。这也是幂等重跑特性的自然代价：同一节点内的历史中断不留痕，只能看到任务最近一次挂在哪里。

## 四、恢复机制（resume）

### Q13. `Command(resume=...)` 传进图后，是怎么把值填到 `"username": username` 里的？（含官方源码验证）

关键在 `interrupt()` 的**双重身份**——暂停时是"抛异常的点"，恢复时是"普通的返回值"：

```
暂停轮:  username = interrupt(...)  → 抛 GraphInterrupt，节点死在赋值处，return 从未执行
恢复轮:  username = interrupt(...)  → 命中排队的恢复值 → return '张三'，赋值完成，节点继续跑完
```

完整链路（以并行中断为例）：

1. 暂停时 interrupt 记录（payload + 确定性 id）挂到 checkpoint；
2. `invoke(Command(resume=resume_map))` → 引擎加载 checkpoint，按 interrupt id 把各应答配给对应断点排队；
3. 被中断的任务**从头重跑**：重跑再次调用 `interrupt()` 时发现本位置有排队的恢复值 → 不抛异常，直接 `return` 该值；
4. `username = '张三'` 就是普通函数返回值赋值，节点执行到 `return {"username": username}` 走正常状态写入。

节点代码对暂停/恢复无感知，只是被从头重新执行了一遍——这也是"副作用必须放在 interrupt() 之后"的原因（重跑时之前的代码会再执行一次）。

**官方源码验证**（langgraph 1.2.11，`langgraph/types.py` 文档字符串原话）：

> "The first invocation of this function **raises a `GraphInterrupt` exception**, halting execution... The graph **resumes from the start of the node, re-executing all logic**... matches resume values to interrupts **based on their order** in the node."

实现体就是两条路径：

```python
if scratchpad.resume:
    if idx < len(scratchpad.resume):
        return scratchpad.resume[idx]       # 恢复路径：返回排队的恢复值（按索引配对）
...
raise GraphInterrupt(Interrupt.from_ns(...))  # 暂停路径：抛异常，payload 随异常带给客户端
```

`GraphInterrupt → GraphBubbleUp → Exception`：冒泡基类让异常穿透 ToolNode/子图，被 Pregel 循环捕获后挂起任务。

### Q14. `resume_map = {}  # { id : msg }` 什么时候用？恢复格式是写死的吗？

**只有"多个未决断点"才需要 id 字典**，且这是引擎强制的协议——实测多未决时传裸值直接报错：

> `RuntimeError: When there are multiple pending interrupts, you must specify the interrupt id when resuming.`

多未决只出现在**并行分支**（fan-out 节点各自 interrupt、并行子图同时中断）。单节点内多个 interrupt 是依次触发的，任意时刻只有一个未决，不需要 map。

**格式并非全局写死，按未决断点数分档**（源码分发逻辑）：

| 未决断点数 | 恢复值格式 | 引擎行为 |
|---|---|---|
| 1 个 | **任意 JSON 值**（裸值/字典/列表） | 原样作为该断点的恢复值 |
| ≥2 个 | **必须** `{interrupt_id: value}` 字典 | 按 id 路由，否则 RuntimeError |

两个细节：

1. **判定是内容式的**：引擎检查"是 dict 且所有 key 都是哈希摘要格式（`is_xxh3_128_hexdigest`）"才当 id 路由表。工具审批的 decisions 字典（key 是 `call_00_...` 的 tool_call_id）不会被误判，在单未决场景就是普通值。
2. **信封写死、载荷自由**：多未决时外层必须 id→value 字典，但每个 value 仍可为任意 JSON——并行分支一次传"字符串 + 字典 + 列表"完全可以。

`resume_map` 的键来源：`res['__interrupt__']` 里每个 `Interrupt` 对象的 `.id`（确定性哈希 id），即 2_并行中断 恢复 cell 的 `resume_map[i.id] = ask_msg` 写法。

**多未决恢复的完整案例**（2_并行中断 的模式，一问一答逐个恢复）：

```python
res = graph.invoke(inputs, config)          # 并行两个分支各自 interrupt → 两个未决断点
ints = res["__interrupt__"]                 # [Interrupt(id='is_xx..', value='问名字'), Interrupt(id='is_yy..', value='问年龄')]

resume_map = {}
for i in ints:                              # 逐个问用户，按 id 配对
    resume_map[i.id] = input(f"{i.value}: ")

res = graph.invoke(Command(resume=resume_map), config)   # 必须 {id: value}，传裸值直接 RuntimeError
```

## 五、节点路由与状态更新

### Q15. tool_node 里 `return {"messages": messages}` 和 `return Command(goto="llm_node", update={"messages": [messages]})` 效果一样吗？

路由效果一样（同样的下一跳、同样的状态追加），但这么写有两个问题：

**① `[messages]` 多包一层会直接报错**（实测 ValueError）。`messages` 本身已是 ToolMessage 列表，外面再套 `[]` 就成了"列表的列表"，`add_messages` 会把内层列表当成**一条消息**去转换，直接崩：

```python
return Command(goto="llm_node", update={"messages": messages})   # 正确：列表直接给
return Command(goto="llm_node", update={"messages": [messages]}) # 错误：嵌套列表 → MESSAGE_COERCION_FAILURE
```

对照 llm_node 里的写法 `{"messages": [resp]}`——那是因为 `resp` 是**单个** AIMessage，`add_messages` 要求可迭代消息集合，单条才需要包一层。规律：**给的就是集合要求的形状**，列表不包、单条包。

**② 改用 Command 后，静态边成了冗余**。`add_edge("tool_node", "llm_node")` 与 `Command(goto="llm_node")` 同时存在时 Command 优先，边应删掉；类型注解也换成 `Command[Literal["llm_node"]]`。

**选择标准**（第 3 章 5.2 的分工）：

| 场景 | 写法 |
|---|---|
| 下一跳固定（本例：工具跑完永远回 llm_node） | 静态边 + 普通 `return {"messages": ...}`，拓扑一目了然 |
| 下一跳由运行时状态决定（如异常时 goto=END、审批通过走 A 拒绝走 B） | `Command(goto=..., update=...)` 路由与更新一步完成 |

第 3 章 cell 24/26 的 tool_node 用 Command，是因为它有两个去向（工具循环回 llm / 异常直接 END）；第 7 节的 tool_node 没有分支，维持静态边即可。

> 同一主题（状态更新的形状规则："列表不包、单条包"）在 [../7_工具节点/CheatSheet.md](../7_工具节点/CheatSheet.md) 2.2 节有更完整的两种报错对照，两处结论一致。
