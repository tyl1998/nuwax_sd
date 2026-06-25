# log-platform 改造内容

## 1. 背景

`log-platform` 是平台的日志管理中间层，基于 Rust + Axum 构建，通过 HTTP API 对接 Quickwit 全文搜索引擎。原 `nuwax_deploy/docker/docker-compose.yml` 中，`log_platform` 通过 compose 服务名访问 `quickwit:7280`，并依赖 `quickwit` healthy 后启动。

平台化后，`log-platform` 需要能作为单容器独立启动，不依赖 compose 网络和容器别名。

实际代码改造落点为上层真实仓库：`../log-platform`。

## 2. 改造目标

- `log-platform` 可由 Jenkins 单独构建镜像。
- `log-platform` 可通过 `docker run` 单容器启动。
- 不依赖 Docker Compose，不加入 `docker_agent-network`。
- 通过环境变量覆盖配置文件中的端口、日志路径和 Quickwit 地址。
- 对齐后端默认跨机器参数：Quickwit 地址改为 `host.docker.internal:7280`。

## 3. 主要改造内容

### 3.1 配置层改造（环境变量覆盖）

修改文件：

- `src/config/mod.rs`

改造内容：

在已有配置文件加载逻辑之后，新增环境变量覆盖：

- `LOG_PLATFORM_PORT` — 覆盖服务监听端口
- `LOG_PLATFORM_LOG_PATH` — 覆盖日志存储路径
- `QUICKWIT_URL` — 覆盖 Quickwit 服务地址

优先级：环境变量 > 配置文件 > 默认值。

原有逻辑保留不变，环境变量覆盖发生在配置文件加载之后。

### 3.2 Dockerfile 修正

修改文件：

- `docker/Dockerfile`

改造内容：

- 修正 `FROM ... AS builder` 语法（原为 `FROM rust:1.90-slim AS builder`，保持不变）
- 删除不存在的 `assets` 复制指令
- 修正配置复制路径为 `COPY docker/config.docker.yml /app/config.yml`
- 端口从 `8098` 对齐为 `8097`（生产环境）
- 健康检查改为 `http://localhost:8097/health`

### 3.3 生产配置对齐

修改文件：

- `docker/config.docker.yml`

改造内容：

- 默认端口改为 `8097`
- 默认日志目录改为 `/app/logs`
- Quickwit 默认地址改为 `http://host.docker.internal:7280`

### 3.4 构建与部署脚本

新增文件：

- `scripts/build-runtime-image.sh` — Jenkins/本地统一构建镜像
- `scripts/deploy-runtime-container.sh` — 单容器启动脚本

构建脚本支持参数：

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `IMAGE_NAME` | `log-platform` | 镜像名 |
| `IMAGE_TAG` | `local` | 镜像 tag |
| `PUSH_IMAGE` | `false` | 是否推送镜像 |
| `PLATFORM` | 空 | Docker build 平台 |

部署脚本支持参数：

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `CONTAINER_NAME` | `nuwax-log-platform` | 容器名 |
| `HOST_PORT` | `8097` | 宿主机端口 |
| `QUICKWIT_URL` | `http://host.docker.internal:7280` | Quickwit 地址 |

挂载目录：

| 容器路径 | 宿主机路径 | 说明 |
|----------|-----------|------|
| `/app/logs` | `nuwax_deploy/docker/logs/log_platform` | 日志持久化 |
| `/app/data` | `nuwax_deploy/docker/data/log_platform` | 迁移检查点（redb） |

## 4. 当前运行结果

当前本地验证环境：

- 镜像：`log-platform:local`
- 容器名：`nuwax-log-platform`
- 端口：`0.0.0.0:8097->8097/tcp`
- 网络：`bridge`（默认，不依赖 compose 网络）
- 状态：`healthy`

已验证：

- 脚本语法通过：`bash -n scripts/build-runtime-image.sh`、`bash -n scripts/deploy-runtime-container.sh`
- Docker 镜像构建成功
- 单容器启动成功
- `GET /health` 返回 200
- `GET /ready` 返回 200

## 5. 验证命令

构建镜像：

```bash
cd ../log-platform
scripts/build-runtime-image.sh
```

启动容器：

```bash
scripts/deploy-runtime-container.sh
```

健康检查：

```bash
curl -fsS http://localhost:8097/health
curl -fsS http://localhost:8097/ready
```

容器状态：

```bash
docker ps --filter name=nuwax-log-platform
docker inspect -f '{{.Name}} {{range $name, $_ := .NetworkSettings.Networks}}{{$name}} {{end}}' nuwax-log-platform
```

## 6. 注意事项

- 本机无 `cargo` 命令，未执行本机 `cargo check`。Docker 构建阶段已完成 Rust release 编译，可作为编译验证。
- Quickwit 服务需要独立部署，log-platform 启动后会异步创建索引（5 次重试，间隔 3 秒），不会阻塞服务启动。
- 日志和数据目录必须保留挂载，迁移检查点使用 redb 嵌入式数据库持久化。
- 不执行 `docker compose down -v`，不删除原 `logs/log_platform`、`data/log_platform` 目录。

## 7. 版本记录

- 2026-06-25：新增 log-platform 单容器改造记录。配置层环境变量覆盖、Dockerfile 修正、生产配置对齐、构建/部署脚本、镜像构建和容器启动验证。
