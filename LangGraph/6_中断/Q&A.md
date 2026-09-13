# 中断章节问答记录

> 本文件沉淀中断章节学习过程中的提问与详细解答，按主题分组。配套待办见 [To-Do List.md](To-Do List.md)。

## 一、工具审批

### Q1. 用户输入 y，传回图中为什么变成了 true？

转换发生在**图外的恢复代码**里，LangGraph 不做任何转换：

```python
user_in = input(...)                                          # 字符串 "y"
approved = user_in.strip().lower() in ("y", "yes", "是", "1")  # 布尔表达式 → True/False
res = graph.invoke(Command(resume=approved), config=config)   # 传进图的就是 bool
```

`resume` 的值是任意的（只要可 JSON 序列化），类型由两端约定：第 1 节传用户名字符串，工具审批传 bool，多工具逐个审批传 `{tool_call_id: bool}` 字典。判断放图外还是图内只是风格差异。

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

当前 `6_工具审批模式.ipynb` 即按此实现：单次 interrupt 列出全部调用 + `{tool_call_id: bool}` 决策字典。

### Q4. 审批逻辑放在独立审批节点还是工具节点里？

成熟实现都不用"独立审批节点"，而是嵌在**工具执行边界**上：

| 实现 | 位置 |
|---|---|
| opencode | 工具执行路径上的守卫/中间件（无图概念） |
| LangChain HITL 中间件 | `after_model` 钩子：模型输出与工具执行之间拦截 |
| LangGraph prebuilt `human_approval()` | `ToolNode(wrap_tool_call=...)` 每次调用的包装器，官方明确"no new node class" |

LangGraph 手写层面的三种合法形态及取舍：

- **独立节点**（教程用法）：审批路由在图拓扑里可见，直观；决策需经 state 传递。
- **工具内 `interrupt()`**：审批跟工具走、可复用；拓扑不可见。
- **静态中断**：`create_react_agent(..., interrupt_before="tools")` 一行实现粗粒度卡点；细粒度用 `wrap_tool_call` 自写包装器（约 10 行，1.2.11 未导出现成工厂）。

### Q5. `interrupt()` 的参数可以传列表吗？

可以。payload 只要求**可 JSON 序列化**：str / dict / list / 数字都行，原样出现在 `res["__interrupt__"][0].value` 并存入 checkpoint。本章节三种都用过：字符串（第 1 节）、dict（审核意见）、`list[dict]`（工具审批）。payload 与恢复值类型互不约束。

### Q6. 用列表推导构造 interrupt payload，是否踩了"循环内设断点"的禁忌？

**没有**。判断标准是 **`interrupt()` 的调用次数是否依赖运行时数据**：

```python
decisions = interrupt([... for tc in tool_calls])   # 安全：断点恒为 1 个，循环只构造 payload
for tc in tool_calls:
    ok = interrupt(...)                             # 危险：N 条 tool_call = N 个断点
```

危险写法的机理：同一节点内多个 `interrupt()` 的恢复值按**出现顺序严格索引匹配**；恢复时节点从头重跑，若循环次数/顺序与暂停时不同（数据变化），恢复值就会配错断点。官方 HITL 中间件正是"单断点批量打包"模式。

### Q7. 弹了两次确认弹窗，为什么只算一次中断？

**弹窗来自客户端代码的 for 循环，不是图里的断点**：

- 图：`review_node` 调 1 次 `interrupt()` → 挂起 1 次 → `__interrupt__` 长度为 1。
- 弹窗：恢复 cell 遍历 payload 里的 2 条工具调用 → `input()` 调 2 次 → 组装 decisions → 一次 `Command(resume=decisions)` 恢复。

弹窗次数 = 客户端循环次数；中断次数 = `__interrupt__` 列表长度。两层独立。

### Q8. 为什么没加 END 节点图也能正常结束？

`review_node` 全部分支走 `Command(goto=...)` 路由，`goto=END` 是合法目标，不需要静态边。补 `add_edge("review_node", END)` 后 Command 路由仍优先，只是为了与教程其他节的显式风格保持一致。

## 二、串行/并行中断

### Q9. 两种串行中断查 `get_state_history` 都只看到第一个中断？

分两个层面：

1. **中断是"执行到才产生"，不是预注册的**。第一次 invoke 后，第二个 interrupt 的代码从未运行，checkpoint 里没有它的任何记录——此刻查历史只有第一问，符合预期。
2. **单节点案例的特殊性**：恢复时节点在同一 super-step 内重跑，挂起快照被**原地覆盖**——"名字"的快照会被"年龄"的替换，历史里永远无法同时看到这两个中断（详见 Q14）。
3. **跨节点案例跑完后两个都能看到**：每个节点的挂起是独立快照（实测：`final → summarize → (ask_age, 年龄) → (ask_name, 名字) → __start__`）。

只看到第一个的常见原因：在第一次 invoke 后、恢复循环之前就查了历史；或跑完后**重跑了构建 cell**（新建 `InMemorySaver`，同 thread_id 历史从零开始）。

### Q10. 跨节点串行如何看到完整过程？

按 构建 → 恢复循环（输完所有答案）→ 查询 的顺序执行，且不重跑构建 cell。可读化查询：

```python
for snap in graph.get_state_history(config=config):
    ints = [(t.name, i.value) for t in (snap.tasks or ()) for i in (t.interrupts or [])]
    print(f'next={tuple(snap.next or ())} state={snap.values} interrupts={ints}')
```

## 三、检查点（Checkpoint）机制

### Q11. `next` 说 ask_name 是下一个节点，`tasks` 却有它的执行结果，矛盾吗？

不矛盾——**同一条 checkpoint 被写了两次**（实测证实：恢复前后同 checkpoint_id，`result` 从 None 变为有值）：

| 时刻 | 发生了什么 | 账本变化 |
|---|---|---|
| T1 invoke | 写下这条 checkpoint，ask_name 开跑 → interrupt 挂起 | 只有 interrupts，`result=None` |
| T2 resume | 节点在**同一 super-step 内从头重跑**，interrupt() 返回恢复值，节点跑完 | 返回值作为 pending writes **原地回填** → `result` 出现 |
| T3 步进提交 | step 0 完成，写新 checkpoint | `values` 才合并该结果，`next` 指向下一节点 |

字段语义：`next` = 这一步轮到谁（开页时写，不是"尚未启动"）；`tasks[].result` = 本轮任务的写入缓冲（收工后补记）；`values` = 截至本条的已提交状态（任务输出记在**下一条**快照）。

### Q12. `next` / `values` / `tasks` 的准确含义？

| 字段 | 含义 | 写入时机 |
|---|---|---|
| `next` | 本轮待跑的任务名清单 | 开页时写，之后封存不改 |
| `values` | 前面所有轮次的结果，**经 reducer 合并后**的已提交状态 | 翻页时更新（本条的 values 是开工前的家底） |
| `tasks` | 本轮每个任务的执行档案：`result`（收工）/ `error`（异常）/ `interrupts`（挂起） | 执行中按事件**陆续补挂** |

存储上分家：values/next 在 checkpoint 主记录（一次成型）；result/interrupts 在 pending writes 表，按 `(checkpoint_id, task_id)` 挂靠，查询时拼装成 StateSnapshot。终态快照 `next=()`、`tasks=()`。

### Q13. 为什么要设计成"一页一轮、收齐再翻页"？

本质：**每个 super-step 是一个事务**（durable execution）。四个目标倒逼：

1. **原子性**：一轮可能并行多个节点，必须全部收工才提交，否则并行分支会读到半成品状态；中途宕机丢弃未完成轮次的草稿，从本页重跑，状态不撕裂。
2. **可重放**：values/next 开页封存 → "本轮输入"永远完好 → 恢复/重启后从头重跑结果确定。HITL 能暂停数小时、跨进程恢复的物理基础。
3. **确定性合并**：并行任务完成顺序不确定；写同一 key 靠 reducer 攒到翻页时一次性合并，结果与完成顺序无关、可复现。
4. **并发与幂等**：pending writes 按任务分行 upsert，并行互不加锁，任务重跑覆盖自己那行。

同构于数据库事务/WAL 与 Google Pregel 的 super-step 同步屏障。

### Q14. 单节点串行中断，为什么后一个中断覆盖前一个的记录？

**两条中断记录写进了同一个存储槽位**。中断记录的键是 `(checkpoint_id, task_id)`：

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
