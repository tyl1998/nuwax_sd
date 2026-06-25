# mcp-proxy 改造内容

## 1. 背景

`mcp-proxy` 是 MCP 工具代理服务，后端通过 `MCP_PROXY_URL` 和 `CODE_EXECUTE_URL` 调用它完成 MCP 注册、工具调用与代码执行能力。

原 `nuwax_deploy/docker/docker-compose.yml` 中，`mcp-proxy` 以 compose 服务方式启动，容器内端口 `8089`，宿主机映射端口 `8020`，并挂载日志、配置、`uv` 缓存和 `npm` 缓存。

平台化后，`mcp-proxy` 需要能作为单容器独立启动，不依赖 compose 网络和容器别名。

实际代码改造落点为上层真实仓库：`../mcp-proxy`。

## 2. 改造目标

- `mcp-proxy` 可由 Jenkins 单独构建镜像。
- `mcp-proxy` 可通过 `docker run` 单容器启动。
- 不依赖 Docker Compose，不加入 `docker_agent-network`。
- 继续挂载原 `nuwax_deploy/docker` 下的配置、日志和缓存目录。
- 对齐后端默认跨机器参数：`MCP_PROXY_URL=http://host.docker.internal:8020`。

## 3. 主要改造内容

### 3.1 Dockerfile 端口对齐

修改文件：

- `docker/Dockerfile.mcp-proxy`

改造内容：

- EXPOSE 端口从 `8080` 对齐为 `8089`（容器内端口）
- 健康检查端口从 `8080` 改为 `8089`：`curl -f http://localhost:8089/health`
- 多阶段构建保持不变：`rust:1.92` builder → `rust:1.92` runtime
- runtime 阶段包含 Rust、Node.js、Python、Deno、uv，兼容动态 MCP 工具

### 3.2 Docker 配置文件

修改文件：

- `docker/config.yml`

改造内容：

- 默认端口为 `8089`（已是正确值，无需修改）
- 日志路径 `/app/logs`
- 日志级别 `info`

### 3.3 环境变量覆盖（已有）

无需代码改造，`mcp-proxy/src/config.rs` 已支持环境变量覆盖：

| 变量 | 说明 |
|------|------|
| `MCP_PROXY_PORT` | 覆盖服务端口 |
| `MCP_PROXY_LOG_DIR` | 覆盖日志目录 |
| `MCP_PROXY_LOG_LEVEL` | 覆盖日志级别 |
| `RUST_LOG` | Rust 日志过滤级别 |

优先级：环境变量 > 配置文件 > 默认值。

### 3.4 构建与部署脚本

新增文件：

- `scripts/build-runtime-image.sh` — Jenkins/本地统一构建镜像
- `scripts/deploy-runtime-container.sh` — 单容器启动脚本

构建脚本支持参数：

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `IMAGE_NAME` | `mcp-proxy` | 镜像名 |
| `IMAGE_TAG` | `local` | 镜像 tag |
| `PUSH_IMAGE` | `false` | 是否推送镜像 |
| `PLATFORM` | 空 | Docker build 平台 |

部署脚本支持参数：

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `CONTAINER_NAME` | `nuwax-mcp-proxy` | 容器名 |
| `HOST_PORT` | `8020` | 宿主机端口 |
| `MCP_PROXY_PORT` | `8089` | 容器内端口 |
| `CONFIG_FILE` | `nuwax_deploy/docker/config/mcp_config.yml` | 配置文件路径 |
| `LOG_DIR` | `nuwax_deploy/docker/logs/mcp_proxy` | 日志目录 |
| `UV_CACHE_DIR` | `nuwax_deploy/docker/data/uv_cache/uv` | uv 包缓存 |
| `NPM_CACHE_DIR` | `nuwax_deploy/docker/data/npx_cache/.npm` | npm 包缓存 |

挂载目录：

| 容器路径 | 宿主机路径 | 说明 |
|----------|-----------|------|
| `/app/config.yml` | `nuwax_deploy/docker/config/mcp_config.yml` | 配置文件（只读） |
| `/app/logs` | `nuwax_deploy/docker/logs/mcp_proxy` | 日志持久化 |
| `/root/.cache/uv` | `nuwax_deploy/docker/data/uv_cache/uv` | uv 包缓存 |
| `/root/.npm` | `nuwax_deploy/docker/data/npx_cache/.npm` | npm 包缓存 |

## 4. 当前运行结果

当前本地验证环境：

- 镜像：`mcp-proxy:local`
- 容器名：`nuwax-mcp-proxy`
- 端口：`0.0.0.0:8020->8089/tcp`
- 网络：`bridge`（默认，不依赖 compose 网络）
- 状态：`healthy`

已验证：

- 脚本语法通过：`bash -n scripts/build-runtime-image.sh`、`bash -n scripts/deploy-runtime-container.sh`
- Docker 镜像构建成功（Rust release 编译，约 4 分钟）
- 单容器启动成功
- `GET /health` 返回 200

后端连接参数：

- `MCP_PROXY_URL=http://host.docker.internal:8020`
- `CODE_EXECUTE_URL=http://host.docker.internal:8020/api/run_code_with_log`

## 5. 验证命令

构建镜像：

```bash
cd ../mcp-proxy
scripts/build-runtime-image.sh
```

启动容器：

```bash
scripts/deploy-runtime-container.sh
```

健康检查：

```bash
curl -fsS http://localhost:8020/health
curl -fsS http://localhost:8020/mcp/list
```

容器状态：

```bash
docker ps --filter name=nuwax-mcp-proxy
docker inspect -f '{{.Name}} {{range $name, $_ := .NetworkSettings.Networks}}{{$name}} {{end}}' nuwax-mcp-proxy
```

## 6. 注意事项

- Dockerfile runtime 阶段包含 Rust、Node、Python、Deno、uv，镜像较大，但有利于兼容动态 MCP 工具。
- Dockerfile 中 Deno 和 uv 安装依赖外网脚本，Jenkins 弱网环境可能失败，后续可沉淀内部基础镜像。
- `uv` / `npm` 缓存目录必须保留挂载，否则首次工具调用会变慢，离线环境也更容易失败。
- 不执行 `docker compose down -v`，不删除原 `data/uv_cache`、`data/npx_cache`、`logs/mcp_proxy` 目录。
- 跨机器部署时，把 `host.docker.internal` 替换为 `mcp-proxy` 所在机器 IP、域名或负载均衡地址。

## 7. 版本记录

- 2026-06-25：新增 mcp-proxy 单容器改造记录。Dockerfile 端口对齐、构建/部署脚本、镜像构建和容器启动验证。
