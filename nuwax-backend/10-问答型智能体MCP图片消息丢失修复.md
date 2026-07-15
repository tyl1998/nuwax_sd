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

## 3. 根因（修正版）

**图片工具执行成功，但助手消息在图片返回前就被判定 `finished` 并关闭了 SSE**，导致：

1. 工具结果没写进消息 → `componentExecutedList=[]`；
2. 模型“工具后第二轮补全”（把图片写进回答）无法推给前端；
3. 用户只看到前置话术。

代码落点：`app-platform-agent-core-infra` 的
`.../component/model/ModelInvoker.java`。该模型走原生流式函数调用分支 `streamCall()`，
且 `internalToolExecutionEnabled=true`（`ModelClientFactory`，由 Spring AI 内部执行工具并做后续补全）。

> ⚠️ 第一版修复（仅在 `doMessage` 内对 finish 判断加 `getToolCalls()` 护栏）**无效**：
> 因为触发提前关闭的不是 `doMessage`，而是 `streamCall` 的 `doOnComplete` 回调。

### 3.1 真正的触发点：`doOnComplete`

`streamCall()` 在订阅 `chatClientRequestSpec.stream().chatResponse()` 时注册了：

```java
// 修复前
.doOnComplete(() -> {
    if (!finished.get()) {
        handleFinish(modelContext, sink, messageId, finalMsgSb.toString(), "stop");
        // -> sink.tryEmitComplete() 提前关闭 SSE
    }
})
```

`doOnComplete` **只判断 `finished` 是否为 false，不感知是否有待执行的工具调用**。
当 Hy3 把「前置文本 + tool_call」放在同一段返回、且 `finish_reason` 上报为 `stop`（而非 `tool_calls`）时：

- 第一流片段（首轮，含 tool_call）在 Spring AI 内部执行工具 + 第二轮补全**之前**就结束，
- `doOnComplete` 在 `finished==false` 时立即 `handleFinish` 并 `sink.tryEmitComplete()`，
- 此时图片尚未生成（日志佐证：executing `04:52:16.966` → executed `04:52:21.573`，而 SSE 在 `04:52:16.973` 已 completed），
- 后续工具结果与第二轮补全的 `tryEmitNext` 全部被丢弃，消息在 finish 时刻定稿，`componentExecutedList` 仍为空。

**核心缺陷**：首轮流结束 ≠ 整个对话结束。`doOnComplete` 没有校验“是否仍有工具在 Spring AI 内部执行 / 还有后续补全轮次”，只信任 `finished` 标志。

## 4. 修复（修正版）

文件：`app-platform-modules/app-platform-agent/app-platform-agent-core-infra/src/main/java/com/xspaceagi/agent/core/infra/component/model/ModelInvoker.java`
两处：新增 `pendingToolCalls` 标志、`doOnComplete` 跳过、`doMessage` 写入该标志。

### 4.1 `streamCall`：新增 pendingToolCalls 标志，并让 doOnComplete 跳过

```java
AtomicBoolean finished = new AtomicBoolean(false);
// 当前流式 assistant 消息仍带待执行 tool_calls 时置位。
// 某些模型（如 Tencent Hy3）会把 tool_calls 与 finishReason="stop" 同段返回，
// 首轮流片段在 Spring AI 内部执行工具 + 第二轮补全之前就结束，此时不能 finalize。
AtomicBoolean pendingToolCalls = new AtomicBoolean(false);

chatClientRequestSpec.stream().chatResponse()
  ...
  .doOnComplete(() -> {
      if (finished.get() || pendingToolCalls.get()) {
          // finished==true：已在 doMessage 内 finalize。
          // pendingToolCalls==true：流结束时尚有工具在 Spring AI 内部执行，
          //   真正的结束是工具后补全轮次完成，由它驱动 finalize。
          log.info("stream doOnComplete skipped: finished={}, pendingToolCalls={}", finished.get(), pendingToolCalls.get());
          return;
      }
      log.info("stream doOnComplete finalize: finished={}, pendingToolCalls={}", finished.get(), pendingToolCalls.get());
      handleFinish(modelContext, sink, messageId, finalMsgSb.toString(), "stop");
  })
  .subscribe(chatResponse -> {
      ...
      doMessage(modelContext, sink, messageId, assistantMessage, assistantMessage.getText(),
                finalMsgSb, thinking, finished, null, pendingToolCalls);
  }, ...);
```

### 4.2 `doMessage`：写入 pendingToolCalls

方法签名新增 `AtomicBoolean pendingToolCalls` 参数（同步更新 `customReactStreamCall` 调用处传 `null`）。
在 finish 判断处：

```java
boolean hasPendingToolCalls = CollectionUtils.isNotEmpty(assistantMessage.getToolCalls());
if (pendingToolCalls != null) {
    pendingToolCalls.set(hasPendingToolCalls);
}
if (!hasPendingToolCalls && finishReason != null && !"".equals(finishReason.toString())
        && !finishReason.equals("tool_call") && !finishReason.equals("tool_calls")) {
    handleFinish(modelContext, sink, messageId, msgSb.toString(), finishReason.toString());
    finished.set(true);
}
```

效果：首轮若带 tool_calls，则 `pendingToolCalls=true`；首轮流结束时 `doOnComplete` 跳过 finalize，
等待 Spring AI 内部执行工具并流式第二轮补全；第二轮以 `finishReason="stop"` 且无 tool_calls 到达时，
`doMessage` 内正常 `handleFinish` 并 `sink.tryEmitComplete()`，消息包含工具结果与图片 URL。

说明：`customReactStreamCall` 分支（`functionCall != StreamCallSupported`，手动 react 调工具）不走 Spring AI 内部执行，
调用处传 `pendingToolCalls=null`，行为不受影响。

## 5. 验证

- 编译（模块）：`mvn -o -pl app-platform-modules/app-platform-agent/app-platform-agent-core-infra -am compile` → **BUILD SUCCESS**。
- 打包（镜像）：`mvn -o -pl app-platform-bootstrap/app-platform-web-bootstrap -am package -DskipTests -Pprod`
  → 产出 `app-platform-bootstrap/app-platform-web-bootstrap/target/app-platform-web-bootstrap-*.jar`。
- **尚未部署**：本环境堡垒（lg.sh）仅允许命令执行、无法传输文件，280MB jar 无法经堡垒推送。
  需在能访问后端机器的环境用仓库自带脚本部署：
  `scripts/build-runtime-image.sh` + `scripts/deploy-runtime-container.sh`
  （或直接将 jar 拷到 `172.17.54.139` 容器 `/app/app.jar` 并 `docker restart nuwax-backend`）。
- 回归：问答型智能体 + 图像 MCP，发送「生成一张宠物图」，确认：
  - 日志出现 `stream doOnComplete skipped: finished=false, pendingToolCalls=true`；
  - 工具执行结束后出现 `stream doOnComplete finalize`（或 doMessage 内 finalize），SSE 不再提前 complete；
  - DB `conversation_message` 该条 `componentExecutedList` 含 image_generation 结果、`finished=true`；
  - 最终回答包含图片。

## 6. 后续待确认（非本次代码改动范围）

- `image_generation` 的 MCP 返回为 `McpContent(type=TEXT, data={"url":...})`。时序修复后，“图片能否渲染”还取决于：
  - 第二轮模型是否把该 URL 写进正文（markdown 图片），或
  - 前端是否消费 `componentExecutedList` 中的图片结果并渲染。
- 若发现修复时序后图片仍不显示，应从上述两点继续排查（模型 prompt 引导 / 前端渲染逻辑）。
