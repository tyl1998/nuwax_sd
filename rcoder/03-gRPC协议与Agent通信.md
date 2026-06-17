# gRPC 协议与 Agent 通信

rcoder 与容器内 agent_runner 之间的所有通信都通过 gRPC 完成。这篇文档覆盖 Proto 定义、连接池、类型转换链、以及事件系统的设计。

## 1. 为什么用 gRPC

最初 rcoder 通过 HTTP/JSON 调用容器内 agent_runner，2025-12-06 完成迁移。原因：

- JSON 解析开销：序列化慢 5x、消息体大 2.5x
- 无类型安全：json_payload 字段在运行时才发现类型错误
- 轮询低效：HTTP 需要反复轮询进度，改用 gRPC Server Streaming 实时推送
- 连接复用：HTTP/2 多路复用避免重复 TCP 握手

## 2. agent.proto 核心定义

```protobuf
service AgentService {
  rpc Chat (ChatRequest) returns (ChatResponse);
  rpc SubscribeProgress (ProgressRequest) returns (stream ProgressEvent);
  rpc CancelSession (CancelRequest) returns (CancelResponse);
  rpc GetStatus (GetStatusRequest) returns (GetStatusResponse);
  rpc StopAgent (StopAgentRequest) returns (StopAgentResponse);
  rpc GetContainerStatus (GetContainerStatusRequest) returns (GetContainerStatusResponse);
  rpc GetVncStatus (GetVncStatusRequest) returns (GetVncStatusResponse);
}
```

### ProgressEvent（oneof 设计）

```protobuf
message ProgressEvent {
  oneof event {
    LogEvent                  log                  = 1;
    ThinkingEvent             thinking             = 2;
    ChunkEvent                chunk                = 3;
    CompletionEvent           completion           = 4;
    ErrorEvent                error                = 5;
    AskConfirmationEvent      ask_confirmation     = 6;
    ProgressNotificationEvent progress_notification = 7;
    ToolUseEvent              tool_use             = 8;
  }
  int64 timestamp = 11;
}
```

`oneof` 的优势：编译时类型检查 + 完全消除 JSON 序列化 + 二进制编码（Protobuf 比 JSON 快 5x）。

## 3. 类型转换链

```
UnifiedSessionMessage（agent_runner 内部类型）
    ↓ unified_message_to_progress_event()   [agent_runner 侧]
ProgressEvent（gRPC Protobuf，二进制传输）
    ↓ from_grpc_progress_event()            [rcoder grpc/converters.rs]
UnifiedSessionMessage（rcoder 内部类型）
    ↓ progress_event_to_sse()               [rcoder grpc/sse_stream.rs]
SSE Event（HTTP text/event-stream → nuwax-backend）
```

### 关键映射（agent_runner → ProgressEvent）

| UnifiedSessionMessage sub_type | ProgressEvent 类型 |
|-------------------------------|-------------------|
| `agent_thought_chunk` | `ThinkingEvent { content, is_complete }` |
| `agent_message_chunk` | `ChunkEvent { content, index }` |
| `tool_call` | `ToolUseEvent { tool_name, tool_input, tool_output, is_error }` |
| `end_turn` | `CompletionEvent { result, total_tokens, duration_ms }` |
| `cancelled` | `ErrorEvent { error_code, error_message }` |

### rcoder SSE 转换（ProgressEvent → SSE）

| ProgressEvent | SSE event 字段 | SSE data |
|--------------|---------------|---------|
| `LogEvent` | `log` | `{ level, message }` |
| `ThinkingEvent` | `thinking` | `{ content, is_complete }` |
| `ChunkEvent` | `chunk` | `{ content, index }` |
| `CompletionEvent` | `completion` | `{ result, total_tokens, duration_ms }` |
| `ErrorEvent` | `error` | `{ error_code, error_message }` |
| `AskConfirmationEvent` | `ask_confirmation` | `{ message, options, default_option }` |
| `ProgressNotificationEvent` | `progress_notification` | `{ status, percentage, details }` |
| `ToolUseEvent` | `tool_use` | `{ tool_name, tool_input, tool_output, is_error }` |
| `None`（空）| — | heartbeat comment |

## 4. GrpcChannelPool（连接池）

```
连接池: DashMap<String, Channel>
key = "{container_ip}:50051"

get_client(addr):
  ├── 快速路径：DashMap.get(addr) → clone Channel → AgentServiceClient::new(channel)
  └── 慢速路径：Channel::from_shared("http://{addr}")
                .connect_timeout(5s)
                .timeout(30s)
                .connect().await
                → DashMap.insert(addr, channel)
                → AgentServiceClient::new(channel)
```

**HTTP/2 多路复用**：同一 addr 的并发 gRPC 请求共用一个 TCP 连接，无需重复握手。

**连接清理**：
- `pool.remove(addr)` — 连接失败时主动清理，下次请求重建
- 容器销毁时（`cleanup_resources`）同步移除对应 Channel

## 5. SubscribeProgress 流处理

```rust
create_grpc_sse_stream(grpc_addr, session_id, project_id, pool)

1. 2 次重试获取 gRPC 客户端
2. GetStatus(session_id) → 检查 Agent 是否 idle
   └── idle → 直接发 SessionPromptEnd SSE → return（不建立流）
3. SubscribeProgress(ProgressRequest { session_id })
   └── Server Streaming → 持续接收 ProgressEvent
4. mpsc::channel(100) 解耦接收与 SSE 推送
   ├── tokio::spawn → 后台任务逐条接收 → tx.send(progress_event_to_sse(event))
   └── ReceiverStream(rx) → axum SSE 响应
```

**先检查状态的原因**：客户端可能在任务已完成后才建立 SSE 连接（网络延迟、页面刷新），此时 Agent 已 idle，直接推送结束事件避免客户端一直等待。

## 6. Chat RPC 与 HTTP 回退

```rust
// gRPC 优先
match grpc_chat_with_pool(pool, grpc_addr, ChatRequest { ... }).await {
    Ok(resp) → 转换为 ChatResponse 返回
    Err(e)   → warn! → forward_request_via_http(request, container_info).await
}
```

HTTP 回退发送 `POST {container_ip}:{port}/chat`，保证 gRPC 不可用时服务仍可用。

## 7. ChatRequest 字段

```protobuf
message ChatRequest {
  string project_id = 1;
  string session_id = 2;
  string prompt     = 3;
  optional ModelProviderConfig model_config = 4;
  repeated Attachment attachments = 5;
  optional string request_id = 6;
  repeated string data_source_attachments = 7;
}
```

`model_config` 允许调用方覆盖 Agent 默认模型配置（provider、model_id、temperature 等），支持多模型切换。

## 8. 代码文件速查

| 文件 | 说明 |
|------|------|
| `shared_types/proto/agent.proto` | Proto 服务定义 |
| `shared_types/build.rs` | tonic-build 编译 Proto |
| `rcoder/src/grpc/channel_pool.rs` | GrpcChannelPool 实现 |
| `rcoder/src/grpc/chat_client.rs` | Chat / CancelSession gRPC 客户端 |
| `rcoder/src/grpc/sse_stream.rs` | SubscribeProgress → SSE 桥接 |
| `rcoder/src/grpc/converters.rs` | ProgressEvent → UnifiedSessionMessage 转换 |
| `agent_runner/src/grpc/agent_service_impl.rs` | AgentService gRPC 服务端实现 |

## 一句话总结

rcoder 通过 Protobuf `oneof` ProgressEvent 定义 8 种类型化进度事件，使用 GrpcChannelPool（DashMap + HTTP/2 复用）发起 gRPC 调用，由 `sse_stream.rs` 把 Server Streaming 进度流桥接为 SSE 推给调用方，整条链路类型安全、无 JSON 开销，并通过 HTTP 回退保证可用性。
