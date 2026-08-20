# LangGraph Notebook

一套面向初学者的 LangGraph 中文学习笔记，以 Jupyter Notebook 为载体：每个概念都配「定义 → 特点 → 可运行代码 → 预期输出 → 常见坑」的可执行案例，由浅入深、层层递进。

> 环境基于 langgraph 1.2.x + Python 3.14，全部案例已实机验证。

## 学习路径

按顺序学习，每章只依赖前面的章节：

| 顺序 | 章节 | 一句话简介 | 核心主题 |
|---|---|---|---|
| 0 | [环境搭建](环境搭建/环境搭建.ipynb) | 用 uv 从零搭好可复现的学习环境 | uv、虚拟环境、Jupyter |
| 1 | [LangGraph 总览](LangGraph/1_LangGraph总览.ipynb) | LangGraph 是什么、Graph 由什么构成、怎么跑起来 | LangChain vs LangGraph、State / Node / Edge / START-END、最小 Graph、invoke 过程 |
| 2 | [State 管理](LangGraph/2_State管理.ipynb) | 数据在 Graph 上怎么流动、怎么合并、怎么隔离 | reducer（内置/自定义/Overwrite）、Pydantic State、Node 读写 State、Multi Schema（输入/输出/私有） |
| 3 | [控制流](LangGraph/3_控制流.ipynb) | 执行路径不再固定：并行、分支、循环与动态路由 | super-step 执行模型、静态/条件/Send 动态分支、Defer 汇合、循环与 recursion_limit、Command、stream 调试 |

## 快速开始

```bash
# 1. 克隆并恢复依赖（需要已安装 uv，见环境搭建章节）
git clone https://github.com/aihaipeng/LangGraph-Notebook.git
cd LangGraph-Notebook
uv sync

# 2. 配置 API Key（LLM 案例使用 DeepSeek）
cp .env.example .env   # 或手动创建 .env
# 编辑 .env 填入 DEEPSEEK_API_KEY

# 3. 启动 Jupyter
uv run jupyter lab
```

没有 API Key 也能学：每章的控制流案例都有「纯 Python、不依赖 LLM」的版本。

## 后续章节规划

- **持久化与人工介入**：checkpointer、thread、interrupt 暂停等用户确认（第 1 章预告的「人工介入」在此展开）
- **子图（Subgraph）与多 Agent**：把编译好的 Graph 作为 Node 组合复用、supervisor 编排

## 目录结构

```
├── 环境搭建/环境搭建.ipynb   # 第 0 章
├── LangGraph/
│   ├── 1_LangGraph总览.ipynb # 第 1 章
│   ├── 2_State管理.ipynb     # 第 2 章
│   └── 3_控制流.ipynb        # 第 3 章
├── pyproject.toml            # 依赖清单（uv 管理）
└── uv.lock                   # 锁定的依赖版本
```
