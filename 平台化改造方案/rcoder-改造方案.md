# rcoder 改造方案（单容器 + Docker socket 挂载）

## 1. 背景与目标

`rcoder` 是平台的 Agent Computer 主控层，负责把 AI 任务请求转化为容器化的执行环境。它通过 Docker API（bollard）创建和管理 Agent 子容器，通过 gRPC 下发任务，通过 Pingora 反向代理透传 VNC/音频/IME 端口。

原 `nuwax_deploy/docker/docker-compose.yml` 中，`rcoder` 以 compose 服务方式启动，加入 `agent-network`，挂载 Docker socket、配置、日志和项目工作目录。

目标：

- `rcoder` 可由 Jenkins 单独构建镜像。
- `rcoder` 可通过 `docker run` 单容器启动。
- 不依赖 Docker Compose，不加入 `docker_agent-network`。
- 继续挂载 Docker socket、配置、日志和项目工作目录。
- 对齐后端默认跨机器参数。

## 2. 当前代码与配置

真实源码仓：

```bash
/Users/atan/Desktop/work/vs_code_nuwax/rcoder
```

关键文件：

- `docker/rcoder-master/Dockerfile`：最终镜像构建（基于 `master-rcoder-base`）
- `docker/rcoder-master/Dockerfile.base`：基础镜像（Node.js + Rust + Chromium + nginx + npm 工具链）
- `docker/config.yml`：Docker 环境配置
- `docker/docker-compose.yml`：源码仓内 compose 示例
- `docker/start-rcoder.sh`：容器启动脚本
- `crates/rcoder/src/config.rs`：配置加载，已支持大量环境变量覆盖
- `config.yml`：默认配置文件

已支持的环境变量：

| 变量 | 说明 |
|------|------|
| `RCODER_PORT` | 覆盖服务端口 |
| `RCODER_PROJECTS_DIR` | 覆盖项目工作目录 |
| `RCODER_NETWORK_MODE` | 覆盖 Docker 网络模式 |
| `RCODER_NETWORK_BASE_NAME` | 覆盖网络基础名称 |
| `RCODER_WORK_DIR` | 覆盖工作目录 |
| `RCODER_AUTO_CLEANUP` | 覆盖自动清理开关 |
| `RCODER_CONTAINER_TTL` | 覆盖容器存活时间 |
| `RCODER_API_TIMEOUT_SECONDS` | 覆盖 API 超时 |
| `RCODER_API_KEY_ENABLED` | 覆盖 API Key 认证开关 |
| `RCODER_API_KEY` | 覆盖 API Key |
| `RCODER_AGENT_IDLE_TIMEOUT_SECS` | 覆盖 Agent 空闲超时 |
| `RCODER_AGENT_CLEANUP_INTERVAL_SECS` | 覆盖清理间隔 |
| `RCODER_AGENT_CONCURRENCY_LIMIT` | 覆盖并发限制 |
| `DOCKER_SOCKET_PATH` | Docker socket 路径 |

## 3. 改造设计

### 3.1 端口规范

| 项 | 端口 | 说明 |
|----|------|------|
| 主服务端口 | `8086` | HTTP API（Chat、Progress、Pod 管理） |
| Pingora 代理端口 | `8088` | VNC/音频/IME 反向代理 |
| 构建服务端口 | `60000` | 文件构建服务 |
| Nginx 端口 | `80` | 子应用静态服务 |

后端连接参数：

```bash
AI_AGENT_URL=http://host.docker.internal:8086
DOCKER_PROXY_URL=http://host.docker.internal:8088
BUILD_SERVER_URL=http://host.docker.internal:60000/api
```

### 3.2 挂载目录

| 容器路径 | 宿主机路径 | 说明 | 模式 |
|----------|-----------|------|------|
| `/var/run/docker.sock` | `/var/run/docker.sock` | Docker socket（核心） | 只读 |
| `/app/config.yml` | `nuwax_deploy/docker/config/rcoder/config.yml` | 配置文件 | 只读 |
| `/app/logs` | `nuwax_deploy/docker/logs/rcoder` | 日志 | 读写 |
| `/app/project_workspace` | `nuwax_deploy/docker/project_workspace` | 普通 Agent 工作目录 | 读写 |
| `/app/computer-project-workspace` | `nuwax_deploy/docker/computer-project-workspace` | Computer Agent 工作目录 | 读写 |
| `/root/.cache` | `nuwax_deploy/docker/computer-cache` | npm/uv/pnpm 包缓存 | 读写 |

### 3.3 网络模式

关键决策：rcoder 需要与它创建的 Agent 子容器通信（gRPC on port 50051），因此必须确保网络可达。

两种方案：

**方案 A：bridge 网络 + 容器 IP 发现**（推荐）
- rcoder 使用默认 bridge 网络
- Agent 子容器也使用 bridge 网络
- rcoder 通过 Docker API 获取 Agent 容器的 IP 地址，直接 gRPC 连接
- 配置：`RCODER_NETWORK_MODE=bridge`

**方案 B：自定义 Docker 网络**
- 创建自定义网络 `nuwax-agent-network`
- rcoder 和 Agent 子容器都加入该网络
- 通过容器名互通
- 配置：`RCODER_NETWORK_BASE_NAME=nuwax-agent-network`

当前 rcoder 代码已支持通过 `RCODER_NETWORK_MODE` 和 `RCODER_NETWORK_BASE_NAME` 环境变量控制网络行为。

### 3.4 单容器启动模式

不使用 compose 网络，不使用容器别名。rcoder 通过 Docker socket 直接管理 Agent 容器。

## 4. 改造文件清单

| 文件 | 改造内容 |
|------|----------|
| `scripts/build-runtime-image.sh` | 新增 Jenkins/本地镜像构建脚本 |
| `scripts/deploy-runtime-container.sh` | 新增单容器启动脚本 |
| `docker/config.yml` | 确认端口和网络配置可被环境变量覆盖 |

## 5. Jenkins 命令

构建镜像：

```bash
cd /Users/atan/Desktop/work/vs_code_nuwax/rcoder
IMAGE_NAME=rcoder IMAGE_TAG=${BUILD_NUMBER:-local} scripts/build-runtime-image.sh
```

启动容器：

```bash
cd /Users/atan/Desktop/work/vs_code_nuwax/rcoder
IMAGE_NAME=rcoder \
IMAGE_TAG=${BUILD_NUMBER:-local} \
CONTAINER_NAME=nuwax-rcoder \
RCODER_PORT=8086 \
CONFIG_FILE=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/config/rcoder/config.yml \
LOG_DIR=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/logs/rcoder \
PROJECT_WORKSPACE=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/project_workspace \
COMPUTER_WORKSPACE=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/computer-project-workspace \
COMPUTER_CACHE=/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/computer-cache \
scripts/deploy-runtime-container.sh
```

## 6. 验证命令

```bash
# 健康检查
curl -f http://localhost:8086/health

# Pingora 代理
curl -f http://localhost:8088/health

# 容器状态
docker inspect -f '{{.Name}} {{range $name, $_ := .NetworkSettings.Networks}}{{$name}} {{end}}' nuwax-rcoder

# 确认 Docker socket 可访问
docker exec nuwax-rcoder ls -la /var/run/docker.sock
```

## 7. 风险与注意事项

1. **Docker socket 权限**：挂载 Docker socket 意味着容器内可以执行宿主机上的 Docker 命令。建议使用只读模式（`:ro`）。
2. **镜像较大**：base 镜像包含 Node.js、Rust、Chromium、nginx、大量 npm 包，镜像可能超过 2GB。
3. **Agent 容器网络**：rcoder 创建的 Agent 子容器需要与 rcoder 主容器在同一网络中，否则 gRPC 通信会失败。
4. **Dockerfile.base 依赖外网**：安装 Chromium、npm 包等依赖外网，Jenkins 弱网环境可能失败。
5. **不执行 `docker compose down -v`**：不删除原 `project_workspace`、`computer-project-workspace`、`computer-cache`、`logs/rcoder` 目录。
6. **start-rcoder.sh**：当前 nuwax_deploy 中使用 `start-services.sh` 作为 entrypoint，需要确认单容器模式下使用哪个启动脚本。
7. **config.yml 中的镜像地址**：`multi_image_config` 中的 `registry_prefix` 指向阿里云仓库，单容器部署时可能需要调整为本地镜像或可访问的仓库。
