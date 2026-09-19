# 子图 CheatSheet

> 子图（subgraph）= 把一张编译后的图整体作为父图的一个节点复用。
> 实测环境：langgraph 1.2.11 / Python 3.14，全部结论均实测。
> 三个用法各对应一个真实场景 notebook：[01 多智能体客服（共享 messages）](01_共享State：子图直接作节点.ipynb) / [02 检索组件嵌入周报生成（不同 State）](02_不同State：节点内调用与转换.ipynb) / [03 可复用审批组件（checkpointer 与中断）](03_checkpointer与中断传播.ipynb)。

## 一、三种用法总览

| 用法 | 前提 | 写法 | 真实场景 |
|---|---|---|---|
| ① 编译后子图直接作节点 | 父/子图**共享至少一个 state key** | `builder.add_node("x", subgraph)` | 多智能体共享 `messages`：总控路由 + 退款专员子图（内部生成回复→合规审查） |
| ② 节点内 `invoke` + 手动转换 | schema 不同或需显式控制数据流 | 节点函数内 `sub.invoke(映射)`，返回时再映射回父图 | 组件复用：检索子图（`query`/`hits` 自有 schema）嵌入周报生成父图（`topic`/`report`） |
| ③ 子图 + checkpointer/中断 | 需要持久化或 human-in-the-loop | 子图 `compile()` 不传 checkpointer 默认**继承父图** | 可复用审批组件：报销"起草→审批"子图插进任何父流程，interrupt 在子图内冒泡 |

## 二、各用法最小代码

### ① 共享 State，直接作节点（多智能体客服）

```python
refund_agent = refund_builder.compile()     # 1.2.11 必须先编译
builder.add_node("refund_agent", refund_agent)  # 共享 messages，自动双向读写
```

- 子图 schema 可带**私有 key**（如工单号 `ticket_id`），只在子图内部流转，父图不可见（实测）。
- 默认 `stream` 中子图节点以父图节点名**平铺**出现；要看子图内部用 `stream(..., subgraphs=True)`，
  chunk 变成 `(namespace, chunk)`，namespace 形如 `('refund_agent:<uuid>',)`——运营展示/排查利器。

### ② 不同 State，节点内调用（检索组件嵌入）

```python
def retrieve(state: ReportState):
    resp = retriever.invoke({"query": state["topic"]})   # 父 topic -> 子 query
    return {"report": "本周要点：\n- " + "\n- ".join(resp["hits"])}   # 子 hits -> 父 report
```

附带收益：子图 schema 自洽，可独立测试（`retriever.invoke({"query": ...})`）、多父图复用。

### ③ checkpointer 三种取值（审批组件）

| checkpointer | 行为 |
|---|---|
| 不传 / `None` | **继承父图**（默认） |
| `True` | 子图独立持久化；`get_state(config, subgraphs=True)` 的 `tasks[0].state` 非空，审计可查"审批卡在哪步"（实测 `next=('approve',)`） |
| `False` | 子图不留自己的 checkpoint |

子图内 `interrupt()`（审批请求）会**冒泡**到顶层 `__interrupt__`，`Command(resume=True)` 的审批结果送回子图内 `interrupt()` 返回值；resume 后**父图节点从头重跑**（官方文档明确行为），副作用要放在 `interrupt()` 之后。

## 三、实测坑（langgraph 1.2.11）

| 坑 | 实测结果 |
|---|---|
| 未编译 builder 直接 `add_node` | `TypeError: Expected a Runnable, callable or dict`——**必须先 `compile()`**（老教材若这么写已过时） |
| 零共享 key 的子图直接作节点 | 数据悄悄断线：子图收到空输入 `{}`；读缺失 key 则**子图内部 `KeyError`**（易误判为子图自身 bug），不读则输出被**静默丢弃**、父图字段不更新 |
| 中断传播方式 | 官方文档称 HITL 需"节点内直接调用子图"；实测 1.2.11 **直接作节点同样冒泡+resume 正常**，两种方式都可用 |
| Interrupt 的 `id` | 每次运行会变，按 id 匹配恢复，别写死 |

## 四、选型

| 场景 | 选择 |
|---|---|
| 各 Agent 共享 `messages`，无额外逻辑 | ✅ ① 直接作节点 |
| schema 不同（独立组件）/ 要转换、校验、日志 | ✅ ② 节点内调用 |
| 子图内要审批/人工介入（interrupt） | ✅ ③：父图挂 checkpointer，子图默认继承即可 |
| 需要单独回溯子图内部历史 | 子图 `compile(checkpointer=True)` |

一句话：**能共享 key 就直接作节点（①）；共享不了或要加工数据就节点内调用（②）；需要中断/持久化时确认 checkpointer 继承关系（③）。**
