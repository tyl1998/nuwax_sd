# mcp-proxy 改造方案（单容器 + 运行时缓存挂载）

## 1. 背景与目标

`mcp-proxy` 是 MCP 工具代理服务，后端通过 `MCP_PROXY_URL` 和 `CODE_EXECUTE_URL` 调用它完成 MCP 注册、工具调用与代码执行能力。

原 `nuwax_deploy/docker/docker-compose.yml` 中，`mcp-proxy` 以 compose 服务方式启动，容器内端口 `8089`，宿主机映射端口 `8020`，并挂载日志、配置、`uv` 缓存和 `npm` 缓存。

目标：

- `mcp-proxy` 可由 Jenkins 单独构建镜像。
- `mcp-proxy` 可通过 `docker run` 单容器启动。
- 不依赖 Docker Compose，不加入 `docker_agent-network`。
- 继续挂载原 `nuwax_deploy/docker` 下的配置、日志和缓存目录。
- 对齐后端默认跨机器参数：`MCP_PROXY_URL=http://host.docker.internal:8020`。

## 2. 当前代码与配置

真实源码仓：

```bash
/Users/atan/Desktop/work/vs_code_nuwax/mcp-proxy
```

关键文件：

- `mcp-proxy/src/config.rs`：加载配置，已支持环境变量覆盖端口、日志目录、日志级别。
- `mcp-proxy/src/main.rs`：服务入口，读取配置并监听端口。
- `docker/Dockerfile.mcp-proxy`：构建并运行 mcp-proxy。
- `docker/config.yml`：Docker 环境默认配置。
- `docker/docker-compose.yml`：源码仓内本地 compose 示例。

已支持的环境变量：

| 变量 | 说明 |
| --- | --- |
| `MCP_PROXY_PORT` | 覆盖服务端口 |
| `MCP_PROXY_LOG_DIR` | 覆盖日志目录 |
| `MCP_PROXY_LOG_LEVEL` | 覆盖日志级别 |
| `RUST_LOG` | Rust 日志过滤级别 |

## 3. 改造设计

### 3.1 端口规范

平台化后沿用原 `nuwax_deploy` 端口：

| 项 | 端口 |
| --- | --- |
| 容器内端口 | `8089` |
| 宿主机端口 | `8020` |

后端连接参数：

```bash
MCP_PROXY_URL=http://host.docker.internal:8020
CODE_EXECUTE_URL=http://host.docker.internal:8020/api/run_code_with_log
```

跨机器部署时，把 `host.docker.internal` 替换为 `mcp-proxy` 所在机器 IP、域名或负载均衡地址。

### 3.2 挂载目录

继续复用原目录：

- 配置：`/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/config/mcp_config.yml:/app/config.yml:ro`
- 日志：`/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/logs/mcp_proxy:/app/logs`
- uv 缓存：`/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/data/uv_cache/uv:/root/.cache/uv`
- npm 缓存：`/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/data/npx_cache/.npm:/root/.npm`

### 3.3 单容器启动模式

不使用 compose 网络，不使用容器别名，所有外部依赖通过 URL / IP 注入给后端或调用方。

## 4. 改造文件清单

| 文件 | 改造内容 |
| --- | --- |
| `docker/config.yml` | 默认端口从 `8080` 改为 `8089`，日志路径保持 `/app/logs` |
| `docker/Dockerfile.mcp-proxy` | `EXPOSE` 和健康检查端口从 `8080` 改为 `8089` |
| `scripts/build-runtime-image.sh` | 新增 Jenkins/本地镜像构建脚本 |
| `scripts/deploy-runtime-container.sh` | 新增单容器启动脚本 |

## 5. Jenkins 命令

构建镜像：

```bash
cd /Users/atan/Desktop/work/vs_code_nuwax/mcp-proxy
IMAGE_NAME=mcp-proxy IMAGE_TAG=${BUILD_NUMBER:-local} scripts/build-runtime-image.sh
```

启动容器：

```bash
cd /Users/atan/Desktop/work/vs_code_nuwax/mcp-proxy
IMAGE_NAME=mcp-proxy \
IMAGE_TAG=${BUILD_NUMBER:-local} \
CONTAINER_NAME=nuwax-mcp-proxy \
HOST_PORT=8020 \
MCP_PROXY_PORT=8089 \
CONFIG_FILE=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/config/mcp_config.yml \
LOG_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/logs/mcp_proxy \
UV_CACHE_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/data/uv_cache/uv \
NPM_CACHE_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/data/npx_cache/.npm \
scripts/deploy-runtime-container.sh
```

## 6. 验证命令

```bash
curl -f http://localhost:8020/health
curl -f http://localhost:8020/mcp/list
docker inspect -f '{{.Name}} {{range $name, $_ := .NetworkSettings.Networks}}{{$name}} {{end}}' nuwax-mcp-proxy
```

期望网络为默认 `bridge`。

## 7. 风险与注意事项

- Dockerfile runtime 阶段包含 Rust、Node、Python、Deno、uv，镜像较大，但有利于兼容动态 MCP 工具。
- Dockerfile 中 Deno 和 uv 安装依赖外网脚本，Jenkins 弱网环境可能失败，后续可沉淀内部基础镜像。
- `uv` / `npm` 缓存目录必须保留挂载，否则首次工具调用会变慢，离线环境也更容易失败。
- 不执行 `docker compose down -v`，不删除原 `data/uv_cache`、`data/npx_cache`、`logs/mcp_proxy` 目录。
