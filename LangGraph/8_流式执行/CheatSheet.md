# 流式执行 CheatSheet

> 沉淀流式 API 的版本差异、官方口径与选型建议。实测环境：langgraph 1.2.9 / langchain 1.3.13 / Python 3.14。
> 基础机制（stream vs astream 同步/异步对比实测）见 [01_stream and astream.ipynb](01_stream and astream.ipynb)。

## 一、两套 API 是什么？

| API | 定位 | 消费方式 |
|---|---|---|
| `graph.stream()` / `graph.astream()` | **底层**：按 `stream_mode` 吐原始执行事件（`for` / `async for`） | 教材与存量代码的主体 |
| `graph.stream_events()` / `graph.astream_events()` | **应用层**：把原始事件规范化后给出"投影"（类型化视图） | v1.2 起官方主推的方向 |

分层关系（官方原话）：*"Event streaming sits one level above streaming… Use streaming when you need low-level access to those modes; use event streaming when application code benefits from typed projections."*
即：stream_events 建立在 stream 的模式之上，两层长期共存、定位不同，**不是替代关系**。

## 二、stream/astream 各版本差异

`stream_mode` 共 7 种：`values`（逐步全量状态）/ `updates`（逐步节点更新）/ `messages`（LLM token，节点内用 `invoke` 也能流出）/ `custom`（节点内 `get_stream_writer()` 自定义上报）/ `checkpoints` / `tasks` / `debug`（后三种需 checkpointer，调试用）。

### v1（默认，不传 version）

输出形状**随参数组合变化**——这正是各教材讲法不一的根源：

| 调用参数 | 输出形状 |
|---|---|
| 单个 `stream_mode` | 裸数据：`{"node_a": {"log": ["A 完成"]}}` |
| `stream_mode=[...]` 多模式 | `(mode, data)` 元组：`("updates", {...})` |
| 加 `subgraphs=True` | 再叠一层命名空间：`(ns, data)` / `(ns, mode, data)` |

### v2（`version="v2"`，langgraph ≥ 1.1）

无论什么参数组合，输出**永远**是统一 `StreamPart` dict，按 `chunk["type"]` 分流，编辑器有完整类型收窄（`UpdatesStreamPart` 等 TypedDict）：

```python
for chunk in graph.stream(inputs, stream_mode="updates", version="v2"):
    chunk["type"]   # "updates"
    chunk["ns"]     # 子图命名空间元组，根图为 ()
    chunk["data"]   # {"node_a": {"log": ["A 完成"]}}
```

另：`invoke(..., version="v2")` 返回 `GraphOutput`（`.value` + `.interrupts`），中断不再混在 `result["__interrupt__"]` 里。

## 三、astream_events 各版本差异

### v1 / v2（默认 v2）——兼容层

吐 langchain-core 老式 `StreamEvent` 字典（`on_chat_model_stream` / `on_chain_start` 等），
是给**存量 langchain Runnable 链代码**做兼容的，官方 event streaming 文档对这两档只字未提。
**没有独立的使用价值**：要 token 流，`stream_mode="messages"`（配 v2 格式）是稳定路径；
要类型化投影，只有 v3 有。这两档"认识即可"——读老代码时能反应过来。

### v3——类型化投影（beta，官方主推方向）

返回 `GraphRunStream` 对象，不再按 mode 分支，直接给类型化投影：

```python
stream = graph.stream_events(input, version="v3")

for message in stream.messages:      # token 流（每个 LLM 调用一个 ChatModelStream）
    for token in message.text:
        print(token, end="", flush=True)
print(stream.interrupted, stream.interrupts)   # HITL 状态
final = stream.output                # 最终结果
```

- 内置投影：`stream.messages / values / subgraphs / output / interrupts / interrupted`；
  `updates/custom/checkpoints/tasks/debug` 需注册 transformer 开启
- **接管** `stream_mode` 和 `subgraphs` 参数（传了直接 TypeError，由 transformer 声明决定）
- 投影默认单消费者：`tee(n)` 分流、`interleave("messages", "values")` 按到达顺序合并多投影
- 自定义投影：实现 `StreamTransformer`（声明 `required_stream_modes`），暴露在 `stream.extensions`
- ⚠️ 源码 docstring：*"The version='v3' API is experimental and may change."*；Release 1.2.0 标注 **beta**

## 四、官方口径（原文）

1. streaming 页 Tip：*"For new applications, we recommend **event streaming** — the typed-projection API introduced in LangGraph v1.2."*（新应用推荐 event streaming）
2. event streaming 页：*"Event streaming is the **recommended in-process streaming model** for most LangGraph application code."*；*"Use streaming when you need **low-level access** to those modes; use event streaming when application code benefits from **typed projections**."*
3. GitHub Release 1.2.0（2026-05-12）：*"New event streaming API **(beta)**… may change before stabilization."*；*"recommended path for token-level streaming to a UI"*；*"v3 is opt-in; v1 and v2 are **unchanged**."*
4. 时间线：1.0.0（2025-10）stream_mode v1 格式 → 1.1.0（2026-03）`version="v2"` 统一格式 → 1.2.0（2026-05）event streaming v3（beta）

**解读**：方向上主推 v3；但 beta 标注明确；`stream/astream` + `stream_mode` 定位为底层访问层，**未被弃用**，v1/v2 格式承诺不变。

## 五、抉择

按需求选，**当前主力只有一行**：

| 需求 | 选择 |
|---|---|
| 节点进度 / 状态流（脚本、进度条） | ✅ `stream/astream` + `stream_mode="updates"` + `version="v2"` |
| LLM token 流（UI 打字机） | ✅ `stream/astream` + `stream_mode="messages"` + `version="v2"` |
| 子图内 token / 事件 | 同上 + `subgraphs=True`（v3 里由 `stream.subgraphs` 投影接管） |
| 多投影同时消费、结构化协议（新项目、敢尝鲜） | `astream_events(version="v3")`——beta，API 可能变，生产自担风险 |
| 读存量代码（裸 dict、`(mode, data)` 元组） | 认识 v1 格式即可 |
| 读老 langchain 代码（`on_chat_model_stream` 事件字典） | 认识 `astream_events` v1/v2 即可 |
| ❌ 常见误区 | 不要用 `astream_events` v1/v2 当"稳定替代 v3"——它是兼容层，token 需求走 `stream_mode="messages"` 更稳 |

一句话：**主力写 `stream/astream + version="v2"`；`astream_events` 整个 API 停在"认识"档，v3 稳定后再升级为"使用"。**

## 六、学法建议

| 内容 | 怎么学 |
|---|---|
| `stream/astream` 机制 + `stream_mode` 概念 | **必学**——`updates/values/messages/custom` 四个常用，`checkpoints/tasks/debug` 调试时查 |
| `version="v2"` 统一格式 | **当默认写法**——官方示例全部用它；无新概念，只是输出形状稳定 |
| event streaming v3 | **当方向学**——能读懂官方示例即可；beta，生产采用自担 API 变动风险 |
| v1 裸格式、`astream_events` v1/v2 | **只需认识**——读存量代码用 |
