# log-platform 改造方案（单容器 + Quickwit 外部化）

## 1. 背景与目标

`log-platform` 是 Rust 编写的日志服务，后端通过 `LOG_SERVICE_URL` 调用它完成智能体日志新增、批量新增、搜索和详情查询。

原 `nuwax_deploy/docker/docker-compose.yml` 中，`log_platform` 通过 compose 服务名访问 `quickwit:7280`，并依赖 `quickwit` healthy 后启动。平台化后，`log-platform` 需要能作为单容器独立启动，不依赖 compose 网络和容器别名。

目标：

- `log-platform` 可由 Jenkins 单独构建镜像。
- `log-platform` 可通过 `docker run` 单容器启动。
- Quickwit 地址通过运行时环境变量注入，支持跨机器部署。
- 继续挂载原 `nuwax_deploy/docker/logs/log_platform` 和 `data/log_platform`，避免数据丢失。
- 对齐后端默认 `LOG_SERVICE_URL=http://host.docker.internal:8097`。

## 2. 当前代码与配置

真实源码仓：

```bash
/Users/atan/Desktop/work/vs_code_nuwax/log-platform
```

关键文件：

- `src/config/mod.rs`：加载配置文件。
- `src/main.rs`：读取配置、初始化日志、启动 HTTP 服务、异步创建 Quickwit 索引。
- `docker/Dockerfile`：构建 Rust 二进制并打运行镜像。
- `docker/config.docker.yml`：容器内默认配置。
- `config.yml`：本地开发默认配置。

当前配置加载顺序：

1. `LOG_PLATFORM_CONFIG` 指定的配置文件。
2. `/app/config.yml`。
3. 当前目录 `config.yml`。

当前问题：

- `docker/Dockerfile` 中 `COPY config.docker.yml /app/config.yml` 路径与 build context 不匹配。
- Dockerfile 暴露端口为 `8098`，但 `nuwax_deploy` 和后端默认使用 `8097`。
- `docker/config.docker.yml` 的 Quickwit 地址为 `http://quickwit:7280`，跨机器模式不可用。
- 配置主要依赖文件，不支持直接通过环境变量覆盖关键项。

## 3. 改造设计

### 3.1 配置优先级

改造后配置优先级：

运行时环境变量 > 配置文件 > 默认值。

支持的运行时变量：

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `LOG_PLATFORM_PORT` | 配置文件 `server.port` | 服务端口，平台统一建议 `8097` |
| `LOG_PLATFORM_LOG_PATH` | 配置文件 `server.log_path` | 日志目录，容器内建议 `/app/logs` |
| `QUICKWIT_URL` | 配置文件 `quickwit.url` | Quickwit 地址，跨机器用 IP / 域名 |

### 3.2 单容器启动模式

单容器启动时不加入 `docker_agent-network`，通过外部地址访问 Quickwit：

```bash
QUICKWIT_URL=http://host.docker.internal:7280
```

跨机器部署时替换为：

```bash
QUICKWIT_URL=http://<quickwit-host>:7280
```

### 3.3 挂载目录

继续复用原目录：

- 日志目录：`/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/logs/log_platform:/app/logs`
- 数据目录：`/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/data/log_platform:/app/data`

## 4. 改造文件清单

| 文件 | 改造内容 |
| --- | --- |
| `src/config/mod.rs` | 增加环境变量覆盖 `server.port`、`server.log_path`、`quickwit.url` |
| `docker/Dockerfile` | 修正配置文件复制路径，端口改为 `8097`，健康检查改为 `8097` |
| `docker/config.docker.yml` | 默认端口改为 `8097`，Quickwit 默认地址改为 `host.docker.internal:7280` |
| `scripts/build-runtime-image.sh` | 新增 Jenkins/本地镜像构建脚本 |
| `scripts/deploy-runtime-container.sh` | 新增单容器启动脚本 |

## 5. Jenkins 命令

构建镜像：

```bash
cd /Users/atan/Desktop/work/vs_code_nuwax/log-platform
IMAGE_NAME=log-platform IMAGE_TAG=${BUILD_NUMBER:-local} scripts/build-runtime-image.sh
```

启动容器：

```bash
cd /Users/atan/Desktop/work/vs_code_nuwax/log-platform
IMAGE_NAME=log-platform \
IMAGE_TAG=${BUILD_NUMBER:-local} \
CONTAINER_NAME=nuwax-log-platform \
HOST_PORT=8097 \
LOG_PLATFORM_PORT=8097 \
QUICKWIT_URL=http://host.docker.internal:7280 \
LOG_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/logs/log_platform \
DATA_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/data/log_platform \
scripts/deploy-runtime-container.sh
```

## 6. 验证命令

```bash
curl -f http://localhost:8097/health
curl -f http://localhost:8097/ready
curl -f http://localhost:8097/api-docs/openapi.json
```

后端联调参数：

```bash
LOG_SERVICE_URL=http://host.docker.internal:8097
```

## 7. 风险与注意事项

- Quickwit 必须先启动并可访问，否则 `log-platform` 启动后异步索引创建会重试并记录失败日志。
- 单容器模式不要使用 `http://quickwit:7280`，除非明确加入了同一个 Docker 网络。
- 不删除原 `logs/log_platform` 和 `data/log_platform` 目录。
- Jenkins 多机器部署时，`QUICKWIT_URL` 必须配置为 Quickwit 实际机器 IP、域名或负载均衡地址。
