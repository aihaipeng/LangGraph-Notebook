# 子图 CheatSheet

> 子图 = 把一张编译后的图整体作为父图的一个节点复用。核心机制只有两条：**必须先 `compile()`**、父子图靠**同名 key 自动同步**或**手动转换**通信。
> 实测环境：langgraph 1.2.11 / Python 3.14。官方文档：https://docs.langchain.com/oss/python/langgraph/use-subgraphs
> 本目录案例：[01_explore工具化委派.ipynb](01_explore工具化委派.ipynb)（cell0 工具集 → cell1 explore 只读子图 → cell2 `research` 工具 + 父图 agent，全流程实测）。

## 一、用法与适用场景（核心）

| 用法 | 前提 | 写法 | 适用场景 |
|---|---|---|---|
| ① 编译后直接作节点 | 父/子图**共享 ≥1 个 state key**（如 `messages`） | `builder.add_node("x", subgraph)` | 多 agent **共享对话历史**协作（总控+专员）；子图全程轨迹对父图可见、无隔离需求 |
| ② 节点内 invoke + 状态转换 | schema 不同或需**隔离** | 普通函数里 `sub.invoke(输入映射)`，返回前映射回父图 | 探索/测试/日志等**大输出任务**：内部轨迹不出子图、只回传结论；私有消息历史；可复用独立组件 |
| ②+ 工具化委派（② 的业界主流变体） | 父图是 ReAct 循环 | 子图包装成 `@tool`，与全量工具平级 | 通用 coding agent：**委派时机由模型决定**（Claude Code / OpenHands / opencode 同构形态） |
| ③ 子图 + checkpointer/中断 | 需要持久化或人工介入 | 父图挂 checkpointer，子图默认继承 | 高危操作**审批**（interrupt 冒泡）、长任务断点续跑、独立记账审计 |

一句话：**能共享 key 且想全程透明 → ①；要隔离/转换 → ②；委派时机交给模型 → ②+；审批/持久化 → ③。**

## 二、各用法最小代码

### ① 共享 state，直接作节点（实测）

```python
subgraph = sub_builder.compile()             # 未编译直接 add_node 会 TypeError
builder.add_node("refund_agent", subgraph)   # 同名 key（messages）自动双向读写
```

- 子图 schema 可带私有 key（如 `ticket_id`），只在子图内流转，父图不可见（实测）
- 默认 stream 中子图节点以父图节点名平铺；`stream(subgraphs=True)` 看内部，chunk 变 `(namespace, chunk)`——运营展示/排查利器

### ② 节点内 invoke：隔离 + 只回传摘要（实测）

```python
class TaskState(TypedDict):
    task: str
    findings: str  # 父图唯一可见的结果

def explore_node(state: TaskState) -> dict:
    result = explore_graph.invoke({
        "messages": [SystemMessage(EXPLORE_PROMPT), HumanMessage(state["task"])]
    })
    return {"findings": result["messages"][-1].content}  # 内部轨迹全留在子图
```

实测证据：explore 子图内部自主 ls/grep/read 十几轮，父图 messages 仅 16 条。

### ②+ 工具化委派：sub-agent as tool（实测）

```python
@tool(parse_docstring=True)
def research(task: str) -> str:
    """只读调研，返回结构化摘要。适合多步骤深度调查；零散小问题自己查即可。"""
    result = explore_graph.invoke({"messages": [SystemMessage(EXPLORE_PROMPT), HumanMessage(task)]})
    return result["messages"][-1].content

agent_tools = [research, read, ls, grep, write, shell, ...]  # 委派与落盘平级
```

- 与 ② 机制相同（invoke 编译图 + 手动映射），差别只在**触发方式**：固定边必经 → 模型 `tool_calls` 按需
- docstring 必须写清适用边界，否则模型可能跳过委派直接干
- 父图保留 read/ls 等轻量工具："小事自己查、大事派 explore"——委派是选项不是必经

### ③ checkpointer 三种取值（审批组件，实测）

| checkpointer | 行为 |
|---|---|
| 不传 / `None` | **继承父图**（默认，官方称 "stateless"）：interrupt 正常冒泡，但子图状态**每次调用重置**；官方推荐场景——agent 在 tool 里调另一个 agent |
| `True` | 官方称 "stateful"：同一 `thread_id` 下**跨调用累积状态**；`get_state(config, subgraphs=True)` 的 `tasks[0].state` 非空，审计可查"审批卡在哪步"（实测 `next=('approve',)`） |
| `False` | 即使父图有 checkpointer，子图也不留自己的 checkpoint |

子图内 `interrupt()` 会**冒泡**到顶层 `__interrupt__`；resume 后**父图节点从头重跑**（官方文档明确），副作用要放在 `interrupt()` 之后。

## 三、explore 子图实战要点（01_explore工具化委派.ipynb）

```
父图 AgentState{messages}（ReAct 循环：llm_agent ⇄ agent_tool_node）
└── research @tool ── 子图 ExploreState{messages}（私有）
    └── llm_explore ⇄ explore_tool_node（只读工具循环）
```

- **工具集收窄**：explore 只给只读工具（read/ls/grep/tavily_search），不给 write——物理隔离优于 prompt 自觉（opencode 同款思路）
- **pi 式防御（实测有效）**：read 分页（offset/limit + 续读提示，防大文件撑爆上下文）；shell 输出截断 + 非零退出码回显；grep 跳过 .git/node_modules
- **循环退出条件**：prompt 写明"完成后输出结构化摘要、不再调用工具"

## 四、选型

**固定节点 vs 工具化委派（谁决定何时委派）：**

| | 固定节点（② 原版） | 工具化委派（②+） |
|---|---|---|
| 委派时机 | 图结构写死，每次必经 | 模型自主决定 |
| 适合 | 流程确定的管线（每次必先调研/审批） | 通用 agent，任务不可预测 |
| 业界主流 | LangGraph 流水线教程（plan-and-execute 等） | Claude Code / OpenHands / opencode |

**子图合适的功能全景（六类）**：①上下文隔离的重活 ②多 agent 专业 worker ③带内部循环的流程（自修复/打磨，带轮数上限防死循环）④可复用组件 ⑤`Send` 并行 fan-out（map-reduce）⑥独立记账审批。
反例：单步无循环、线性固定步骤——普通节点即可，别为省代码行数包子图。

## 五、实测坑

| 坑 | 实测结果 |
|---|---|
| 未编译 builder 直接 `add_node` | `TypeError: Expected a Runnable...`——**必须先 `compile()`**（老教材已过时） |
| 零共享 key 的子图直接作节点 | 数据悄悄断线：子图收到空输入 `{}`；读缺失 key 则**子图内部 `KeyError`**，不读则输出被**静默丢弃** |
| 中断传播方式 | 官方文档称 HITL 需"节点内直接调用子图"；实测 **直接作节点同样冒泡 + resume 正常**，两种方式都可用 |
| Interrupt 的 `id` | 每次运行会变，按 id 匹配恢复，别写死 |
| 工具改名后 `agent_tools` 引用旧名 | `NameError`（实测：`web_search` 改名 `tavily_search` 后未同步） |
| cell 合并时丢 `load_dotenv()` 调用 | `KeyError: 'TAVILY_API_KEY'`（import 在、调用丢了，最易漏） |
| deepseek thinking 参数 | `extra_body={"thinking": {"type": ...}}` 只收 `enabled`/`adaptive`/`disabled`，写 `"enable"` 报 422（实测） |
| 并行子图写同一文件 | 互相覆盖；探索并行、写码串行，或 git worktree 隔离 |
| `Literal` 注解里写 `END` 变量 | 类型检查不过——`Literal` 只收字面量，写 `"__end__"`；运行时 `END == "__end__"` 等价（实测） |
| `add_conditional_edges` 的 `path_map` | 路由函数直接返回节点名时**不用传**；仅返回自造标签时才用 dict 翻译，list 形式是恒等映射基本没用（实测 `path_map=["llm"]` 导致 KeyError） |

## 六、业界对照速记（2026-09 调研）

| 派系 | 子图形态 |
|---|---|
| Claude Code / OpenHands / opencode | sub-agent as tool（独立 context/会话 + 权限收窄），只回传摘要；三大动机：上下文隔离（第一）、专业化、并行 |
| LangGraph 系 | 子图 = 显式图结构：可持久化、可恢复、可观测 |
| pi（badlogic） | 明确不内置（"No sub-agents"）：用会话级 artifact 工作流替代；警告委派是"黑箱里的黑箱"，若每个任务都先委派说明规划有问题 |

来源：code.claude.com/docs/en/sub-agents、OpenHands SDK 论文（arXiv 2511.03690）、opencode.ai/docs/agents + DeepWiki、mariozechner.at/posts/2025-11-30-pi-coding-agent、pi.dev。
