# 中断章节待补知识点 To-Do List

> 基于现有 5 节内容（interrupt 基础 / 并行中断 / 审批模式 / 审核与编辑 / 工具审批）的差距分析，按优先级排列。

## ★★★ 高优先级

- [ ] **编辑工具参数（approve/edit/reject 中的 edit）**
  - 场景：模型生成的 SQL 表名、删除路径、金额有误，不想原样执行也不想直接拒绝，想改完参数再放行
  - 实现逻辑：review_node 的 interrupt 恢复值改为 `{tool_call_id: {"action": "edit", "args": {...}}}`；edit 决策时构造带**相同 id** 的 AIMessage（`add_messages` 按 id 覆盖原 tool_calls）写入 `Command(update=...)` 再 goto tool_node；官方文档的 update 路径就是这个写法

- [ ] **节点重放语义**
  - 场景：恢复执行后发现下单/发邮件跑了两次——interrupt() 之前的代码在恢复时从头重跑
  - 实现逻辑：LangGraph 恢复时重新执行整个节点，interrupt() 直接返回恢复值；副作用代码必须放在 interrupt() 之后，或「先 interrupt 拿许可 → 再执行动作」；第 1 节 `node_a` 加一行 print 就能现场演示这个坑

## ★★ 中优先级

- [ ] **静态中断 `interrupt_before` / `interrupt_after`**
  - 场景：固定卡点——工具节点前无条件暂停（如支付/部署步骤），不想在每个节点里手写 interrupt
  - 实现逻辑：`compile(checkpointer=..., interrupt_before=["tool_node"])`；恢复**不是** `Command(resume=...)` 而是 `graph.invoke(None, config)`；`graph.get_state(config).next` 可查看当前停在哪个节点

- [ ] **持久化 checkpointer**
  - 场景：审批人第二天才点同意，期间服务重启/换进程，对话恢复不了
  - 实现逻辑：`InMemorySaver` 换 `PostgresSaver.from_conn_string(DB_URL)`（依赖已装：langgraph-checkpoint-postgres）；thread 状态落库后，恢复只依赖同一 `thread_id`，与进程无关

- [ ] **工具内 interrupt**
  - 场景：审批逻辑跟着工具走——换一个图挂上这个工具，自动带审批，不用改图结构
  - 实现逻辑：在 `@tool` 函数体内直接调 `interrupt(payload)`，根据恢复值决定执行还是 return 取消文案；`GraphInterrupt` 会从 ToolNode 冒泡暂停整图

## ★ 低优先级

- [ ] **单节点多 interrupt（索引匹配）**
  - 场景：一个节点里依次问多个问题，或「输入 → 校验 → 不合格重输」循环
  - 实现逻辑：同一节点内多次调用 `interrupt()`，恢复值与 interrupt 按出现顺序严格索引匹配；与第 2 节的并行节点 interrupt（按 id 映射恢复）是两种不同形态

- [ ] **stream 中处理中断**
  - 场景：生产 UI 边流式输出 token 边弹审批框，不能靠一次 invoke 拿结果
  - 实现逻辑：`graph.stream(..., stream_mode="updates")` 返回的 chunk 里带 `__interrupt__` 信息驱动前端弹窗，用户决策后 `Command(resume=...)` 继续流式执行

- [ ] **子图中断传播**
  - 场景：多 agent 场景——子图/子 agent 里 interrupt，暂停要传导到父图统一处理
  - 实现逻辑：子图节点的 interrupt 会以 `GraphInterrupt` 冒泡到父图挂起；多个子图并行中断时，恢复值按 interrupt id 组成 dict 映射（与第 2 节同构）

## 建议实施顺序

1. 编辑工具参数 → 2. 节点重放语义 → 3. 静态中断 → 4. 持久化 checkpointer → 5. 工具内 interrupt → 6~8 按教程定位取舍
