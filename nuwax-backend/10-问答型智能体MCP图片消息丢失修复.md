# 问答型智能体 MCP 生图消息丢失修复

生成时间：2026-07-15
关联请求：`requestId=7a1da0f202c3438e9d39c7cc78111994`，`conversationId=19`

## 1. 现象

问答型（Default）智能体接入图像 MCP 服务，用户发送「生成一张宠物图看看」。

- 后端确实调用了 MCP 工具 `image_generation` 并**成功生成图片**（返回了合法图片 URL）。
- 但前端最终 message **只有前置话术、没有图片**，会话里图片“丢失”。

## 2. 链路取证（只读）

- 基础服务机器：`172.17.54.139`，容器 `nuwax-backend`，日志 `/app/logs/app.log`。
- 数据库机器：`172.17.52.69`，容器 `nuwax-mysql`，库 `agent_platform`。

### 2.1 后端日志时间线（UTC）

| 时间 | 事件 |
|---|---|
| 03:35:30.175 | 收到 chat 请求，message=`生成一张宠物图看看`，模型 `Tencent: Hy3 (free)`（`tencent/hy3:free`，`functionCall=StreamCallSupported`，`streamCall=true`） |
| 03:35:34.800 | `tool-call component executing: image_generation`（size=1024*1024） |
| **03:35:34.804** | **落库 ASSISTANT 消息**：text=`好的！我来为您生成一张可爱的宠物图。我选择正方形尺寸（1024×1024）…`，`finished:true`，`componentExecutedList:[]` |
| **03:35:34.811** | **Agent execution completed / SSE channel completed, cid 19**（此刻图片尚未生成） |
| **03:35:39.364** | `tool-call component executed: image_generation`，**success=true**，返回 `McpExecuteOutput → McpContent(data={"url":"https://dashscope-…png"}, type=TEXT)` |
| 03:35:40.6 | Calculate bill `errorCode=0001` |

### 2.2 数据库佐证

`agent_platform.conversation_message` 中本次请求（`message_id=7a1da0f202c3438e9d39c7cc78111994`，id=108）：

- `content.text` 仅为前置话术，无图片；
- `componentExecutedList` 为空数组；
- `finished=true`。

上一条「风景图吧」（id=106）现象完全一致——**稳定复现**。

## 3. 根因

**图片工具执行成功，但助手消息在图片返回前约 4.5 秒就被判定 `finished` 并关闭了 SSE**，导致：

1. 工具结果没写进消息 → `componentExecutedList=[]`；
2. 模型“工具后第二轮补全”（把图片写进回答）无法推给前端；
3. 用户只看到前置话术。

代码落点：`app-platform-agent-core-infra` 的
`.../component/model/ModelInvoker.java`。

- 该模型走原生流式函数调用分支 `streamCall()`，且 `internalToolExecutionEnabled=true`（`ModelClientFactory`，由 Spring AI 内部执行工具并做后续补全）。
- `doMessage()` 的 finish 判断**只比较 `finishReason` 字符串**：

```java
// 修复前
Object finishReason = assistantMessage.getMetadata().get("finishReason");
if (finishReason != null && !"".equals(finishReason.toString())
        && !finishReason.equals("tool_call") && !finishReason.equals("tool_calls")) {
    handleFinish(...);        // -> sink.tryEmitComplete() 提前关闭流
    finished.set(true);
}
```

- 该模型/OpenAI 兼容网关把「前置文本 + tool_call」放在同一段返回，但 `finish_reason` 上报为 `stop`（而非 `tool_calls`），于是此处**误判为已结束**，`handleFinish()` 内 `sink.tryEmitComplete()` 提前完成流。
- 随后 Spring AI 内部执行 `image_generation`（拿到 URL）并进行第二轮补全，但 sink 已 complete，后续 `tryEmitNext` 全被丢弃；消息也已在 finish 时刻定稿，`componentExecutedList` 仍为空。

**核心缺陷**：finish 判断没有校验 `assistantMessage.getToolCalls()` 是否仍有待执行的工具调用，只信任了 `finishReason` 文本。

## 4. 修复

文件：`app-platform-modules/app-platform-agent/app-platform-agent-core-infra/src/main/java/com/xspaceagi/agent/core/infra/component/model/ModelInvoker.java`
方法：`doMessage(...)`

在 finish 判断中增加“当前 chunk 仍带 tool_calls 则不 finish”的护栏，让真正的结束交由 `doOnComplete`（整段含工具后补全结束时）处理：

```java
// 修复后
Object finishReason = assistantMessage.getMetadata().get("finishReason");
// 某些模型/OpenAI 兼容网关会把 assistant 文本与 tool_call 放在同一段返回，
// 却上报非工具类的 finishReason（如 "stop"）。若此处 finish，会在 Spring AI
// 内部工具执行 + 工具后补全之前就 complete 掉 SSE，导致工具结果（如 image_generation
// 的图片 URL）永远进不了消息。只要当前 chunk 仍带待执行 tool_calls，就不 finish；
// 真正的结束交由 doOnComplete 处理。
boolean hasPendingToolCalls = CollectionUtils.isNotEmpty(assistantMessage.getToolCalls());
if (!hasPendingToolCalls && finishReason != null && !"".equals(finishReason.toString())
        && !finishReason.equals("tool_call") && !finishReason.equals("tool_calls")) {
    handleFinish(modelContext, sink, messageId, msgSb.toString(), finishReason.toString());
    finished.set(true);
}
```

说明：`CollectionUtils` 为 `org.apache.commons.collections4.CollectionUtils`（该文件已有引用），与上文 L443 的既有用法一致。

## 5. 验证

- 编译：`mvn -o -pl app-platform-modules/app-platform-agent/app-platform-agent-core-infra -am compile` → **BUILD SUCCESS**。
- 建议回归：问答型智能体 + 图像 MCP，发送「生成一张宠物图」，确认：
  - SSE 不再在工具执行前 finish；
  - 消息 `componentExecutedList` 含 image_generation 结果；
  - 最终回答包含图片。

## 6. 后续待确认（非本次代码改动范围）

- `image_generation` 的 MCP 返回为 `McpContent(type=TEXT, data={"url":...})`。时序修复后，“图片能否渲染”还取决于：
  - 第二轮模型是否把该 URL 写进正文（markdown 图片），或
  - 前端是否消费 `componentExecutedList` 中的图片结果并渲染。
- 若发现修复时序后图片仍不显示，应从上述两点继续排查（模型 prompt 引导 / 前端渲染逻辑）。
