# T01: 补全 LangGraph 教程第 3 章（控制流）并修复系列笔记已发现问题

Task objective: 将 `LangGraph/3_控制流.ipynb` 补充为完整、层层递进、重点突出的控制流章节（super-step 执行模型、分支、循环、Command、stream 调试、小结），同时修复第 1 章代码位置错乱、第 2 章缺 Pydantic State、README 空白等问题，并实机验证全部代码案例。

Acceptance criteria:
1. 第 3 章包含 6 节：super-step 执行模型 / 分支 / 循环 / Command / stream 调试 / 小结，编号连续、交叉引用一致
2. 原第 1 节中「路由函数可返回 Command」的错误说法被修正（实测 langgraph 1.2.11 中该写法被静默忽略）
3. 全部纯 Python 案例脚本化执行通过，注释中的预期输出与实际输出一致
4. 全部 LLM 案例实际运行通过（deepseek-v4-flash），真实输出摘录进注释（标注「输出示例」）
5. 第 1 章最小 Graph 代码单元格位于 §3 markdown 之前，无指代错乱
6. 第 2 章含 Pydantic State 小节且案例可运行
7. README.md 含学习路径目录与各章简介
8. `uv run ruff check` 通过

Out of scope: interrupt 人工介入（留后续章节）、子图 Subgraph（留后续章节）、持久化/多 agent 编排。

| Subtask | Objective | Input/Output | Dependencies | Verification | Status |
|---|---|---|---|---|---|
| 1 | 补全第 3 章（T01） | Input: 现有 19 cell + 实测结论; Output: 约 34 cell 的完整章节 | None | 纯 Python 案例脚本重跑 + LLM 案例实跑 | pending |
| 2 | 运行 LLM 单元格验证并记录输出（T04） | Input: 第 2/3 章 LLM 案例; Output: 验证结果 + 输出摘录 | 1 | 脚本提取源码执行，输出与注释一致 | pending |
| 3 | 修第 1 章代码位置错乱（T02） | Input: 1_LangGraph总览.ipynb; Output: cell 顺序修正 | None | 人工核对 markdown 指代 | pending |
| 4 | 第 2 章补 Pydantic State（T03） | Input: 2_State管理.ipynb; Output: 新增 1.4 小节 | None | 案例脚本重跑通过 | pending |
| 5 | 补 README 学习目录（T05） | Input: 全部章节; Output: README.md | 1,3,4 | 链接/命令有效性核对 | pending |
| 6 | 终验：全量重跑 + ruff | Input: 全部改动; Output: 验证报告 | 1-5 | 纯 Python 案例全量重跑 + `uv run ruff check` | pending |
