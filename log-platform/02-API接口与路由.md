# log-platform API 接口与路由

## 1. 路由总入口

所有路由在 `src/api/routes.rs` 的 `init_routes()` 函数中集中定义。路由按业务域分三大组，加上健康检查和文档入口。

```rust
pub fn init_routes(app_states: Arc<AppStates>) -> Router {
    // 健康检查 + OpenAPI 文档
    // Agent 日志域
    // 知识库片段域
    // 通用日志域
}
```

## 2. 完整路由表

### 基础端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/health` | 健康检查（就绪检查） |
| GET | `/ready` | 就绪检查（别名） |
| GET | `/swagger-ui` | Swagger UI 交互式文档 |
| GET | `/api-docs/openapi.json` | OpenAPI 3.0 规范文件 |

### A. Agent 日志域（`/api/agent/log/*`）

| 方法 | 路径 | Handler | 说明 |
|------|------|---------|------|
| POST | `/api/agent/log/ingest` | `agent_ingest_log` | 写入单条 Agent 日志 |
| POST | `/api/agent/log/batch` | `agent_batch_ingest_logs` | 批量写入 Agent 日志 |
| GET | `/api/agent/log/search` | `agent_search_logs` | 搜索 Agent 日志（分页、排序、时间过滤） |
| GET | `/api/agent/log/detail` | `agent_query_detail_log` | 查询单条日志详情 |
| POST | `/api/agent/log/index` | `agent_create_index` | 创建 v1 索引 |
| POST | `/api/agent/log/index/v2` | `agent_create_v2_index` | 创建 v2 索引 |
| POST | `/api/agent/log/migrate` | `agent_migrate_data` | 触发 v1→v2 数据迁移 |
| GET | `/api/agent/log/migration/status` | `agent_migration_status` | 查询迁移状态 |
| POST | `/api/agent/log/migration/reset` | `agent_reset_migration` | 重置迁移检查点 |
| DELETE | `/api/agent/log/index/{index_name}` | `agent_delete_logs` | 删除指定索引 |

### B. 知识库片段域（`/api/knowledge/*`）

| 方法 | 路径 | Handler | 说明 |
|------|------|---------|------|
| POST | `/api/knowledge/segments` | — | 写入知识库片段（单条/批量） |
| GET | `/api/knowledge/segments/search` | — | 搜索知识库片段（BM25 评分） |
| PUT | `/api/knowledge/segments/{id}` | — | 更新指定片段 |
| DELETE | `/api/knowledge/segments/{id}` | — | 删除指定片段 |
| POST | `/api/knowledge/segments/delete` | — | 按条件批量删除（同步） |
| POST | `/api/knowledge/segments/delete-async` | — | 按条件批量删除（异步，返回任务 ID） |
| GET | `/api/knowledge/segments/delete-tasks` | — | 查询所有异步删除任务 |
| GET | `/api/knowledge/segments/delete-tasks/{task_id}` | — | 查询指定任务状态 |
| GET | `/api/knowledge/segments/stats` | — | 获取知识库统计信息 |
| POST | `/api/knowledge/segments/query-ids` | — | 按条件查询片段 ID 列表 |
| DELETE | `/api/knowledge/segments/clear` | — | 清空全部知识库片段 |
| POST | `/api/knowledge/index` | — | 创建知识库索引 |
| DELETE | `/api/knowledge/index` | — | 删除知识库索引 |

### C. 通用日志域（`/api/logs/*`）

| 方法 | 路径 | Handler | 说明 |
|------|------|---------|------|
| POST | `/api/logs/ingest` | — | 写入通用日志 |
| POST | `/api/logs/batch` | — | 批量写入通用日志 |
| GET | `/api/logs/search` | — | 搜索通用日志 |
| POST | `/api/logs/index` | — | 创建通用日志索引 |
| DELETE | `/api/logs/index` | — | 删除通用日志索引 |

## 3. Agent 日志域详细说明

### 写入单条日志

```http
POST /api/agent/log/ingest
Content-Type: application/json

{
  "timestamp": "2025-01-01T00:00:00Z",
  "level": "INFO",
  "message": "用户发起对话",
  "agent_id": "agent-001",
  "session_id": "sess-001",
  "extra": {}
}
```

服务层会将 JSON 转为 NDJSON 格式发送给 Quickwit 的 `_ingest` API。

### 搜索 Agent 日志

```http
GET /api/agent/log/search?query=用户&agent_id=agent-001&from=0&size=20&sort_by=timestamp&sort_order=desc&start_time=2025-01-01T00:00:00Z&end_time=2025-01-02T00:00:00Z
```

支持的查询参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| `query` | string | 全文搜索关键词 |
| `agent_id` | string | 按 Agent 过滤 |
| `session_id` | string | 按会话过滤 |
| `level` | string | 按日志级别过滤 |
| `from` | int | 分页偏移量 |
| `size` | int | 每页数量 |
| `sort_by` | string | 排序字段 |
| `sort_order` | string | 排序方向（asc/desc） |
| `start_time` | datetime | 起始时间 |
| `end_time` | datetime | 结束时间 |

### 批量写入

```http
POST /api/agent/log/batch
Content-Type: application/json

[
  { "timestamp": "...", "level": "INFO", "message": "..." },
  { "timestamp": "...", "level": "ERROR", "message": "..." }
]
```

批量写入会在服务层拼接为 NDJSON 格式后一次性提交给 Quickwit。

## 4. 知识库片段域详细说明

### 搜索知识库片段

```http
GET /api/knowledge/segments/search?query=机器学习&knowledge_base_id=kb-001&from=0&size=10
```

搜索使用 Quickwit 的 **BM25 评分算法**，返回结果按相关性排序。支持按 `knowledge_base_id` 过滤特定知识库。

### 异步删除机制

对于大批量删除场景，提供异步删除接口：

1. 调用 `POST /api/knowledge/segments/delete-async` 返回任务 ID
2. 通过 `GET /api/knowledge/segments/delete-tasks/{task_id}` 轮询任务状态
3. 任务完成后获取删除数量

删除任务使用 Quickwit 的 **clear API** 执行，比逐条删除高效得多。

### 统计信息

```http
GET /api/knowledge/segments/stats?knowledge_base_id=kb-001
```

返回知识库的片段数量、存储大小等统计信息。

## 5. 请求与响应约定

### 统一错误响应

所有 API 在出错时返回统一格式：

```json
{
  "error": "错误描述信息"
}
```

HTTP 状态码遵循标准语义：
- `200` — 成功
- `400` — 请求参数错误
- `404` — 资源不存在
- `500` — 服务端内部错误

### 请求体大小限制

CORS 中间件配置了 **20 MB** 的请求体上限，适配批量日志写入场景。

### Content-Type

- 请求：`application/json`
- 写入 Quickwit：`application/json`（NDJSON 格式）

## 6. 从接口层角度如何理解这个服务

### 三大域的分工
- **Agent 日志**：面向对话场景，支持会话级过滤和时间范围查询，有索引版本迁移能力
- **知识库片段**：面向 RAG 场景，支持 BM25 评分搜索、异步批量删除、统计分析
- **通用日志**：面向运维场景，最简单的写入+搜索

### 统一下沉
所有域的底层都是同一个模式：**HTTP API → Service 层 → Quickwit HTTP API**。业务方不需要知道 Quickwit 的索引名、查询语法、NDJSON 格式，log-platform 全部封装好了。

### 扩展性
新增日志域只需：定义新 Service → 新 API handler → 在 `routes.rs` 中注册路由，模式完全一致。
