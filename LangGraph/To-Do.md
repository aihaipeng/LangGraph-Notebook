# 待补知识点 To-Do

> 基于各章节内容的差距分析，按优先级排列；已完成的条目标注对应文件。

## ★★★ 高优先级

- [x] **编辑工具参数（approve/edit/reject 中的 edit）**（已覆盖：`7_工具节点/CheatSheet.md` 第七节 wrap_tool_call 审批钩子，官方四决策协议）
  - 场景：模型生成的 SQL 表名、删除路径、金额有误，不想原样执行也不想直接拒绝，想改完参数再放行
  - 实现逻辑：对齐官方 HumanInTheLoopMiddleware 的 edit 语义——interrupt 恢复值 `{"type": "edit", "edited_action": {...}}`，钩子内 `request.override(tool_call=...)` 换参数执行，**保留原 tool_call_id**（双向绑定不丢）；review_node 版则构造带相同 id 的 AIMessage 经 `add_messages` 按 id 覆盖

- [x] **节点重放语义**（已覆盖：`4_串行中断.ipynb` 横幅双打印演示 + `3_同一超步部分任务触发中断.ipynb` 被中断任务重放/已完成任务不重放对比）
  - 场景：恢复执行后发现下单/发邮件跑了两次——interrupt() 之前的代码在恢复时从头重跑
  - 实现逻辑：LangGraph 恢复时重新执行**被中断的任务**（已完成任务不重跑），interrupt() 直接返回恢复值；副作用代码必须放在 interrupt() 之后

## ★★ 中优先级

- [x] **静态中断 `interrupt_before` / `interrupt_after`**（已覆盖：`8_静态中断.ipynb`，按官方定位实现为调试断点单步案例）
  - 官方定位：调试断点（逐节点单步执行、检查中间状态）；**不推荐用于业务 HITL**，人机协同用动态 `interrupt()`
  - 实现逻辑：`compile(checkpointer=..., interrupt_before=[...])`；恢复是 `graph.invoke(None, config)` 而非 `Command(resume=值)`；暂停信号不在结果里，用 `get_state(config).next` 查看

- [ ] **持久化 checkpointer**
  - 场景：审批人第二天才点同意，期间服务重启/换进程，对话恢复不了
  - 实现逻辑：`InMemorySaver` 换 `PostgresSaver.from_conn_string(DB_URL)`（依赖已装：langgraph-checkpoint-postgres）；thread 状态落库后，恢复只依赖同一 `thread_id`，与进程无关

- [x] **工具内 interrupt**（已覆盖：审批三位置对比见 `7_工具节点/CheatSheet.md` 第四节，含 ③ 的最小实现）
  - 场景：审批逻辑跟着工具走——换一个图挂上这个工具，自动带审批，不用改图结构
  - 实现逻辑：在 `@tool` 函数体内直接调 `interrupt(payload)`，根据恢复值决定执行还是 return 取消文案；`GraphInterrupt` 会从 ToolNode 冒泡暂停整图。位置取舍（外置节点 / tool_node 内 / 工具体内）见对比表

## ★ 低优先级

- [x] **单节点多 interrupt（索引匹配）**（已覆盖：`4_串行中断.ipynb` 单节点案例）
  - 场景：一个节点里依次问多个问题，或「输入 → 校验 → 不合格重输」循环
  - 实现逻辑：同一节点内多次调用 `interrupt()`，恢复值与 interrupt 按出现顺序严格索引匹配；与 `2_并行中断.ipynb` 的并行节点 interrupt（按 id 映射恢复）是两种不同形态

- [ ] **stream 中处理中断**
  - 场景：生产 UI 边流式输出 token 边弹审批框，不能靠一次 invoke 拿结果
  - 实现逻辑：`graph.stream(..., stream_mode="updates")` 返回的 chunk 里带 `__interrupt__` 信息驱动前端弹窗，用户决策后 `Command(resume=...)` 继续流式执行

- [x] **子图中断传播**（已覆盖：机制沉淀见 `9_子图/CheatSheet.md` 第二节 ③；原 `03_checkpointer与中断传播.ipynb` 实测结论已并入，notebook 随第 9 章重构移除）
  - 场景：多 agent 场景——子图/子 agent 里 interrupt，暂停要传导到父图统一处理
  - 实现逻辑：子图内 `interrupt()` 冒泡到顶层 `__interrupt__`，`Command(resume=...)` 的值送回子图内 `interrupt()` 返回值；1.2.11 实测"直接作节点"与"节点内 invoke"两种集成方式均支持；resume 后父图节点从头重跑

## 建议实施顺序

1. ~~编辑工具参数~~（已覆盖） → 2. 持久化 checkpointer → 3~5（工具内 interrupt 已覆盖 / stream / 子图传播）按教程定位取舍
