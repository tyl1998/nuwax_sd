# log-platform 服务层与 Quickwit 交互

## 1. 服务层整体结构

服务层是 log-platform 与 Quickwit 之间的**唯一桥梁**。所有对 Quickwit 的 HTTP 调用都封装在三个 Service 结构体中，API handler 不直接接触 Quickwit。

```
API Handler
    │
    ├── AgentLogQuickwitService      → Quickwit Agent 日志索引
    ├── KnowledgeQuickwitService     → Quickwit 知识库索引
    └── RecordCommonLogQuickwitService → Quickwit 通用日志索引
```

每个 Service 内部持有 `Arc<AppStates>`，通过共享的 `reqwest::Client` 发起 HTTP 请求。

## 2. 三个 Service 的统一模式

所有 Service 遵循相同的设计模式：

| 方法 | 职责 | 典型签名 |
|------|------|----------|
| `new()` | 默认构造，使用默认索引名 | `new(app_states: Arc<AppStates>)` |
| `new_with_index_name()` | 自定义索引名构造 | `new_with_index_name(app_states, index_name)` |
| `check_*_index_exists()` | 检查索引是否存在 | `→ Result<bool, AppError>` |
| `create_*_index()` | 创建索引 | `→ Result<(), AppError>` |
| `ensure_*_index_exists()` | 确保索引存在（检查+创建） | `→ Result<(), AppError>` |
| `ingest_*()` / `batch_ingest_*()` | 写入数据 | `→ Result<(), AppError>` |
| `search_*()` | 搜索数据 | `→ Result<SearchResult, AppError>` |
| `delete_*()` | 删除数据 | `→ Result<u64, AppError>` |

## 3. 索引管理

### 索引创建流程

```rust
pub async fn ensure_agent_index_exists(&self) -> Result<(), AppError> {
    // 1. 检查索引是否已存在
    match self.check_agent_index_exists().await {
        Ok(true) => return Ok(()),   // 已存在，跳过
        Ok(false) => { /* 继续创建 */ }
        Err(_) => { /* 连接失败，记录日志，继续尝试创建 */ }
    }
    // 2. 创建索引
    self.create_agent_index().await
}
```

关键设计：**先检查再创建**。即使检查失败（Quickwit 暂时不可用），也会尝试创建，避免因短暂网络问题导致索引缺失。

### Quickwit 索引检查 API

```
GET {quickwit_url}/api/v1/indexes/{index_id}
```

- 返回 200 → 索引存在
- 返回 404 → 索引不存在

### Quickwit 索引创建 API

```
POST {quickwit_url}/api/v1/indexes
Content-Type: application/json

{
  "version": "0.8",
  "index_id": "agent_logs",
  "doc_mapping": { ... },
  "search_settings": { ... },
  "retention": { ... }
}
```

## 4. 数据写入：NDJSON 格式

所有数据写入都使用 Quickwit 的 **NDJSON ingest API**：

```
POST {quickwit_url}/api/v1/{index_id}/ingest
Content-Type: application/json

{"field1": "value1", "field2": "value2"}
{"field1": "value3", "field2": "value4"}
```

### 单条写入流程

```rust
pub async fn ingest_agent_log(&self, log: &AgentLogEntry) -> Result<(), AppError> {
    // 1. 确保索引存在
    self.ensure_agent_index_exists().await?;
    // 2. 序列化为 JSON
    let json = serde_json::to_string(log)?;
    // 3. POST 到 Quickwit ingest API
    let response = self.app_states.http_client
        .post(format!("{}/api/v1/{}/ingest", quickwit_url, index_id))
        .header("Content-Type", "application/json")
        .body(json)
        .send()
        .await?;
    // 4. 检查响应状态
    ...
}
```

### 批量写入流程

```rust
pub async fn batch_ingest_agent_logs(&self, logs: &[AgentLogEntry]) -> Result<(), AppError> {
    // 1. 确保索引存在
    self.ensure_agent_index_exists().await?;
    // 2. 将所有日志拼接为 NDJSON（每行一个 JSON 对象）
    let ndjson: String = logs.iter()
        .map(|log| serde_json::to_string(log).unwrap())
        .collect::<Vec<_>>()
        .join("\n");
    // 3. 一次性 POST 到 Quickwit
    let response = self.app_states.http_client
        .post(format!("{}/api/v1/{}/ingest", quickwit_url, index_id))
        .header("Content-Type", "application/json")
        .body(ndjson)
        .send()
        .await?;
    ...
}
```

关键点：**批量写入是拼接 NDJSON 字符串后一次性提交**，不是逐条发送，性能远优于循环单条写入。

## 5. 数据搜索：Quickwit Search API

### Agent 日志搜索

```rust
pub async fn search_agent_logs(&self, params: SearchParams) -> Result<SearchResult, AppError> {
    // 1. 构建 Quickwit 查询体
    let query_body = serde_json::json!({
        "query": {
            "bool": {
                "must": [
                    { "query_string": params.query }
                ],
                "filter": [
                    { "term": { "agent_id": params.agent_id } },
                    { "range": { "timestamp": { "gte": params.start_time, "lte": params.end_time } } }
                ]
            }
        },
        "from": params.from,
        "size": params.size,
        "sort": [{ params.sort_by: { "order": params.sort_order } }]
    });
    // 2. POST 到 Quickwit search API
    let response = self.app_states.http_client
        .post(format!("{}/api/v1/{}/search", quickwit_url, index_id))
        .json(&query_body)
        .send()
        .await?;
    // 3. 解析响应，提取 hits
    ...
}
```

### 知识库片段搜索（BM25 评分）

知识库搜索使用 Quickwit 的全文检索能力，返回结果按 **BM25 评分**排序：

```rust
pub async fn search_knowledge_segments(&self, params: KnowledgeSearchParams) -> Result<...> {
    let query_body = serde_json::json!({
        "query": {
            "bool": {
                "must": [
                    { "query_string": params.query }
                ],
                "filter": [
                    { "term": { "knowledge_base_id": params.knowledge_base_id } }
                ]
            }
        },
        "from": params.from,
        "size": params.size
    });
    ...
}
```

### 搜索参数校验

所有搜索方法都会对参数做前置校验：
- `from` 不能为负数
- `size` 有上限限制
- `query` 不能为空（部分场景）
- 时间范围 `start_time` 必须早于 `end_time`

## 6. 数据删除机制

### 同步删除

直接调用 Quickwit 的 delete API，适用于小规模删除：

```rust
pub async fn delete_agent_logs(&self, index_name: String) -> Result<(), AppError> {
    let response = self.app_states.http_client
        .delete(format!("{}/api/v1/indexes/{}", quickwit_url, index_name))
        .send()
        .await?;
    ...
}
```

### 异步删除（知识库专用）

大批量删除场景使用异步任务：

1. 创建 `DeleteTask`，分配 `task_id`
2. 使用 `tokio::spawn` 在后台执行删除
3. 任务状态存储在内存中（`Vec<DeleteTask>`）
4. 通过 API 查询任务进度

```rust
pub async fn delete_knowledge_segments_async(&self, params: DeleteParams) -> Result<String, AppError> {
    let task_id = uuid::Uuid::new_v4().to_string();
    let task = DeleteTask { id: task_id.clone(), status: "pending", ... };
    // 存储任务
    self.delete_tasks.lock().push(task);
    // 异步执行
    tokio::spawn(async move {
        // 执行删除逻辑
        // 更新任务状态为 "completed"
    });
    Ok(task_id)
}
```

### 清空全部（Clear API）

知识库支持通过 Quickwit 的 **clear API** 一次性清空索引中所有数据：

```rust
pub async fn clear_all_knowledge_segments(&self) -> Result<ClearResult, AppError> {
    let response = self.app_states.http_client
        .post(format!("{}/api/v1/{}/clear", quickwit_url, index_id))
        .send()
        .await?;
    ...
}
```

## 7. 知识库片段更新

更新采用**先删后写**策略（Quickwit 不支持原地更新）：

```rust
pub async fn update_knowledge_segment(&self, id: &str, segment: KnowledgeRawSegment) -> Result<UpdateResult, AppError> {
    // 1. 查询该 ID 是否存在
    let existing = self.query_segment_ids(&[id.to_string()]).await?;
    // 2. 存在 → 删除旧文档
    if !existing.is_empty() {
        // 使用 Quickwit delete API 删除匹配的文档
    }
    // 3. 写入新文档
    self.batch_ingest_knowledge_segments(&[segment]).await?;
}
```

## 8. 统计查询

知识库统计通过 Quickwit 的 **aggregation API** 实现：

```rust
pub async fn get_knowledge_stats(&self, params: StatsParams) -> Result<StatsResult, AppError> {
    let agg_body = serde_json::json!({
        "query": { "term": { "knowledge_base_id": params.knowledge_base_id } },
        "aggs": {
            "total_count": { "value_count": { "field": "_id" } }
        }
    });
    ...
}
```

## 9. 辅助方法

### truncate_string

用于日志输出时截断过长字符串，防止日志爆炸：

```rust
fn truncate_string(s: &str, max_len: usize) -> String {
    if s.len() <= max_len {
        s.to_string()
    } else {
        format!("{}...", &s[..max_len])
    }
}
```

### add_vector_field_query

知识库搜索的辅助方法，用于在查询中添加向量字段相关条件。

## 10. 从服务层角度如何理解这个服务

### 一切都是 HTTP
log-platform 与 Quickwit 之间**没有任何 SDK 或 gRPC 依赖**，全部通过 `reqwest` 发起 HTTP 请求。这使得：
- Quickwit 版本升级对 log-platform 几乎无影响
- 可以用 curl 手动调试任何交互
- 切换到其他兼容 Quickwit API 的搜索引擎（如 Elasticsearch）成本低

### NDJSON 是写入的通用格式
无论单条还是批量，写入 Quickwit 都走 NDJSON。批量场景通过字符串拼接实现，避免了逐条 HTTP 请求的开销。

### 搜索走 Quickwit 原生查询语法
搜索请求直接透传 Quickwit 的 JSON 查询体，支持 `query_string`（全文）、`term`（精确）、`range`（范围）等查询类型。log-platform 负责参数校验和结果解析，不改变查询语义。

### 错误处理统一
所有 Service 方法返回 `Result<T, AppError>`，handler 层统一转换为 HTTP 错误响应。
