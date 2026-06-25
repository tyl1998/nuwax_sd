# backend 改造完成

## 完成时间

2026-06-23

## 真实改动仓库

`../nuwax-backend`

## 已完成改造

1. 新增单容器多阶段构建 `Dockerfile`，保留为 CI 全量构建备用方案。
2. 新增 `Dockerfile.runtime`，用于 Jenkins 先 Maven 构建 jar、再打 runtime 镜像的推荐发布链路。
3. 新增 `.dockerignore`，减少 Docker 构建上下文并只保留 runtime 镜像所需 jar。
4. 新增 `docker/entrypoint.sh`，对齐旧 compose 启动方式：
   - 支持 `APP_PORT=8080`，保持历史端口一致。
   - 支持旧变量名 `MYSQL_HOST/MYSQL_PORT/MYSQL_DATABASE/MYSQL_USER/MYSQL_PASSWORD`。
   - 支持 `DB_*` 新变量名。
   - 支持 MySQL、Redis、Milvus 启动前 TCP 等待。
   - 支持 `/app/config/jwt/jwt_secret_key.txt` JWT 文件。
   - 加载 `classpath:/` 与 `file:/app/config/`，配置名为 `application,application-external`。
5. 新增 `docker/application-external.yml`，对齐旧 `nuwax_deploy/docker/config/application-external.yml` 的关键配置。
6. 调整 CORS 配置，支持 `CORS_ALLOW_ORIGIN` 与 `CORS_ALLOW_CREDENTIALS` 运行时注入。
7. 调整生产配置示例，支持 `DB_PORT`、`DORIS_PORT` 等环境变量。
8. 将健康检查收口到 bootstrap：
   - `/health`：存活检查
   - `/ready`：MySQL、Redis、Milvus TCP 就绪检查
   - `/ready` 已兼容 `MYSQL_HOST/MYSQL_PORT`。
9. 避免旧 `ContainerCheckController` 与 bootstrap 健康检查重复映射。
10. 新增部署样例：
- `scripts/build-runtime-image.sh`
- `scripts/deploy-runtime-container.sh`
- `deploy/.env.sample`

   - `deploy/docker-compose.yml`
   - `deploy/README.md`
11. 新增 SQL 初始化与迁移说明：`sql/README.md`。
12. 调整 `RedissonConfig`，Redis 密码为空时不再发送 AUTH。
13. 调整 prod 日志输出，控制台可看到请求 INFO 日志。
14. 新增仓库内本地 JWT 目录：`docker/jwt/`。
15. 新增构建脚本 `scripts/build-runtime-image.sh`，用于 Jenkins/本地统一构建 jar 与 runtime 镜像。
16. 新增部署脚本 `scripts/deploy-runtime-container.sh`，用于停止旧容器、启动新镜像并执行 `/ready` 健康检查。

## 已修改文件清单

- `Dockerfile`
- `Dockerfile.runtime`
- `.dockerignore`
- `docker/entrypoint.sh`
- `docker/application-external.yml`
- `docker/jwt/.gitkeep`
- `docker/jwt/.gitignore`
- `app-platform-bootstrap/app-platform-web-bootstrap/src/main/resources/application.yml`
- `app-platform-bootstrap/app-platform-web-bootstrap/src/main/resources/application-prod.sample.yml`
- `app-platform-bootstrap/app-platform-web-bootstrap/src/main/resources/logback-custom.xml`
- `app-platform-bootstrap/app-platform-web-bootstrap/src/main/java/com/xspaceagi/config/WebMvcConfig.java`
- `app-platform-bootstrap/app-platform-web-bootstrap/src/main/java/com/xspaceagi/ctl/HealthController.java`
- `app-platform-modules/platform-system/system-infra/src/main/java/com/xspaceagi/system/infra/health/ContainerCheckController.java`
- `app-platform-modules/platform-system/system-infra/src/main/java/com/xspaceagi/system/infra/redis/config/RedissonConfig.java`
- `deploy/.env.sample`
- `deploy/docker-compose.yml`
- `deploy/README.md`
- `sql/README.md`

## 执行过的关键 Docker / 构建命令

### 推荐构建脚本

```bash
IMAGE_NAME=${IMAGE_NAME} IMAGE_TAG=${IMAGE_TAG} PUSH_IMAGE=true scripts/build-runtime-image.sh
```

本地验证：

```bash
scripts/build-runtime-image.sh
```

只重新打镜像：

```bash
SKIP_MAVEN_BUILD=true scripts/build-runtime-image.sh
```

脚本支持参数：

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `IMAGE_NAME` | `nuwax-backend` | 镜像名 |
| `IMAGE_TAG` | `local` | 镜像 tag |
| `MAVEN_PROFILE` | `prod` | Maven profile |
| `SKIP_TESTS` | `true` | 是否跳过测试 |
| `SKIP_MAVEN_BUILD` | `false` | 是否跳过 Maven 构建 |
| `PUSH_IMAGE` | `false` | 是否推送镜像 |
| `PLATFORM` | 空 | Docker build 平台，例如 `linux/amd64` |

### Maven 构建 jar

```bash
mvn -pl app-platform-bootstrap/app-platform-web-bootstrap -am package -DskipTests -Pprod
```

结果：`BUILD SUCCESS`。

### 构建 runtime 镜像

```bash
docker build -f Dockerfile.runtime -t nuwax-backend:test .
```

结果：镜像构建成功，生成 `nuwax-backend:test`。

### 构建脚本验证

```bash
SKIP_MAVEN_BUILD=true IMAGE_NAME=nuwax-backend IMAGE_TAG=script-test scripts/build-runtime-image.sh
```

结果：镜像构建成功，生成 `nuwax-backend:script-test`。

### 部署脚本

脚本会执行：

1. 可选 `docker pull`。
2. 停止并删除同名旧容器。
3. 使用指定镜像启动新容器。
4. 等待 `http://localhost:${HOST_APP_PORT}/ready` 通过。
5. 健康检查失败时打印容器尾部日志并返回非 0。

#### 新模式：外部依赖地址注入

适用于公司平台把 MySQL、Redis、Milvus、ES、MCP、rcoder 等基础服务拆成独立服务后部署。该模式不依赖 compose 网络。

```bash
IMAGE_NAME=nuwax-backend \
IMAGE_TAG=${IMAGE_TAG} \
CONTAINER_NAME=nuwax-backend \
MYSQL_HOST=${MYSQL_HOST} \
MYSQL_PORT=${MYSQL_PORT} \
MYSQL_DATABASE=agent_platform \
MYSQL_USER=${MYSQL_USER} \
MYSQL_PASSWORD=${MYSQL_PASSWORD} \
REDIS_HOST=${REDIS_HOST} \
REDIS_PORT=${REDIS_PORT} \
REDIS_PASSWORD=${REDIS_PASSWORD} \
MILVUS_URI=${MILVUS_URI} \
ES_URL=${ES_URL} \
MCP_PROXY_URL=${MCP_PROXY_URL} \
CODE_EXECUTE_URL=${CODE_EXECUTE_URL} \
BUILD_SERVER_URL=${BUILD_SERVER_URL} \
AI_AGENT_URL=${AI_AGENT_URL} \
DOCKER_PROXY_URL=${DOCKER_PROXY_URL} \
LOG_SERVICE_URL=${LOG_SERVICE_URL} \
CORS_ALLOW_ORIGIN=${CORS_ALLOW_ORIGIN} \
scripts/deploy-runtime-container.sh
```

本地使用已映射到宿主机端口的基础服务时，可使用默认值直接启动：

```bash
IMAGE_NAME=nuwax-backend IMAGE_TAG=local CONTAINER_NAME=nuwax-backend scripts/deploy-runtime-container.sh
```

默认依赖地址：

- MySQL：`host.docker.internal:13306`
- Redis：`host.docker.internal:16379`
- Milvus：`http://host.docker.internal:19530`
- Elasticsearch：`http://host.docker.internal:9200`
- log-platform：`http://host.docker.internal:8097`
- mcp-proxy：`http://host.docker.internal:8020`
- rcoder：`http://host.docker.internal:60000/8086/8088/8099`

#### 旧模式：兼容 compose 网络

仅用于本地横向对比或过渡期调试。该模式显式加入旧 compose 网络，并使用 compose 服务名访问依赖。

```bash
IMAGE_NAME=nuwax-backend \
IMAGE_TAG=test \
CONTAINER_NAME=nuwax-backend-test \
NETWORK_NAME=docker_agent-network \
NETWORK_ALIAS=backend \
MYSQL_HOST=mysql \
MYSQL_PORT=3306 \
MYSQL_DATABASE=agent_platform \
MYSQL_USER=agent_platform \
MYSQL_PASSWORD=admin123 \
REDIS_HOST=redis \
REDIS_PORT=6379 \
REDIS_PASSWORD=123456 \
REDIS_DB=0 \
MILVUS_HOST=milvus \
MILVUS_PORT=19530 \
MILVUS_URI=http://milvus:19530 \
MILVUS_USER=root \
MILVUS_PASSWORD=Milvus \
DORIS_HOST=mysql \
DORIS_PORT=3306 \
DORIS_DB=agent_custom_table \
DORIS_USERNAME=agent_platform \
DORIS_PASSWORD=admin123 \
ES_URL=http://elasticsearch:9200 \
ES_USERNAME=elastic \
ES_PASSWORD=elastic123 \
LOG_SERVICE_URL=http://log_platform:8097 \
MCP_PROXY_URL=http://mcp-proxy:8089 \
CODE_EXECUTE_URL=http://mcp-proxy:8089/api/run_code_with_log \
BUILD_SERVER_URL=http://rcoder:60000/api \
AI_AGENT_URL=http://rcoder:8086 \
DOCKER_PROXY_URL=http://rcoder:8088 \
DEV_SERVER_HOST=http://rcoder \
PROD_SERVER_HOST=http://rcoder \
CORS_ALLOW_ORIGIN=http://localhost,http://localhost:80,http://localhost:8080 \
scripts/deploy-runtime-container.sh
```

### 停止旧 compose 后端

```bash
docker stop docker-backend-1
```

用途：释放旧 compose 后端占用的 `8080` 与 `6443`。

### 启动单容器后端（本地对齐旧 compose）

```bash
docker rm -f nuwax-backend-test || true

docker run -d \
  --name nuwax-backend-test \
  --network docker_agent-network \
  --network-alias backend \
  -p 8080:8080 \
  -p 6443:6443 \
  -v "/Users/atan/Desktop/work/vs_code_nuwax/nuwax-backend/docker/jwt:/app/config/jwt" \
  -v "/Users/atan/Desktop/work/vs_code_nuwax/nuwax_deploy/docker/upload:/app/upload" \
  -e APP_PROFILE=prod \
  -e APP_PORT=8080 \
  -e MYSQL_HOST=mysql \
  -e MYSQL_PORT=3306 \
  -e MYSQL_DATABASE=agent_platform \
  -e MYSQL_USER=agent_platform \
  -e MYSQL_PASSWORD=admin123 \
  -e MYSQL_ROOT_PASSWORD=root \
  -e REDIS_HOST=redis \
  -e REDIS_PORT=6379 \
  -e REDIS_PASSWORD=123456 \
  -e REDIS_DB=0 \
  -e MILVUS_HOST=milvus \
  -e MILVUS_PORT=19530 \
  -e MILVUS_URI=http://milvus:19530 \
  -e MILVUS_USER=root \
  -e MILVUS_PASSWORD=Milvus \
  -e DORIS_HOST=mysql \
  -e DORIS_PORT=3306 \
  -e DORIS_DB=agent_custom_table \
  -e DORIS_USERNAME=agent_platform \
  -e DORIS_PASSWORD=admin123 \
  -e ES_URL=http://elasticsearch:9200 \
  -e ES_USERNAME=elastic \
  -e ES_PASSWORD=elastic123 \
  -e LOG_SERVICE_URL=http://log_platform:8097 \
  -e MCP_PROXY_URL=http://mcp-proxy:8089 \
  -e CODE_EXECUTE_URL=http://mcp-proxy:8089/api/run_code_with_log \
  -e BUILD_SERVER_URL=http://rcoder:60000/api \
  -e AI_AGENT_URL=http://rcoder:8086 \
  -e DOCKER_PROXY_URL=http://rcoder:8088 \
  -e DEV_SERVER_HOST=http://rcoder \
  -e PROD_SERVER_HOST=http://rcoder \
  -e CORS_ALLOW_ORIGIN=http://localhost,http://localhost:80,http://localhost:8080 \
  -e CORS_ALLOW_CREDENTIALS=true \
  -e OUTER_PORT=6443 \
  -e LOG_LEVEL=INFO \
  -e WAIT_TIMEOUT_SECONDS=30 \
  nuwax-backend:test
```

### 数据库增量迁移命令

本地验证时，因新后端代码依赖较新的表结构，执行过增量 SQL：

```bash
docker exec -i docker-mysql-1 mysql --force -u root -proot agent_platform < sql/update-20260320.sql

docker exec -i docker-mysql-1 mysql --force -u root -proot agent_platform < sql/update-20260331.sql

docker exec -i docker-mysql-1 mysql --force -u root -proot agent_platform < sql/update-20260418.sql

docker exec -i docker-mysql-1 mysql --force -u root -proot agent_platform < sql/update-20260510.sql

docker exec -i docker-mysql-1 mysql --force -u root -proot agent_platform < sql/update-20260524.sql
```

其中重复对象会报 duplicate，使用 `--force` 跳过；关键修复包括 `sandbox_config.bind_info/type/isolation`、`i18n_lang`、支付/订阅相关表等。

## 验证结果

### 容器状态

```bash
docker ps --filter name=nuwax-backend-test --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

结果：`nuwax-backend-test` 持续运行，端口为 `8080:8080`、`6443:6443`。

### 健康检查

```bash
curl http://localhost:8080/health
curl http://localhost:8080/ready
```

结果：均返回 `success`。

### 前端代理接口

```bash
curl http://localhost/api/tenant/config
curl http://localhost/api/i18n/lang/list
```

结果：均返回 `code=0000`。

### 登录验证

```bash
curl -i -c /tmp/nuwax-cookie.txt \
  -H 'Content-Type: application/json' \
  -d '{"phoneOrEmail":"admin@nuwax.com","password":"123456","captchaVerifyParam":""}' \
  http://localhost/api/user/passwordLogin

curl -i -b /tmp/nuwax-cookie.txt http://localhost/api/user/getLoginInfo
```

结果：`passwordLogin` 与登录后 `getLoginInfo` 均返回 `code=0000`。

### 控制台请求日志

```bash
docker logs --since=2m nuwax-backend-test
```

结果：容器控制台可看到请求 INFO 日志，例如 `Request URI /api/tenant/config`、`HTTP API call log`、`Response ...`。

## 最终构建方案

正式发布采用 Jenkins 两阶段构建：

1. Jenkins 拉取代码。
2. Jenkins 使用固定 JDK 17 / Maven 执行 `mvn -pl app-platform-bootstrap/app-platform-web-bootstrap -am package -DskipTests -Pprod`。
3. Jenkins 使用 `Dockerfile.runtime` 将 jar 打进镜像。
4. Jenkins 推送镜像到镜像仓库。
5. 部署平台或节点拉取镜像替换容器。

`Dockerfile` 只作为 CI 全量 Docker 构建备用方案；`deploy/docker-compose.yml` 只作为本地验证样例，不作为正式部署方式。

## 后续部署注意事项

- `JWT_SECRET_KEY` 必须至少 32 字符，且不能使用占位值。
- `DB_PASSWORD` 不能使用占位值。
- 跨域携带凭证时，`CORS_ALLOW_ORIGIN` 必须配置为精确前端域名，不能使用 `*`。
- `/ready` 依赖 MySQL、Redis、Milvus 均可 TCP 连通后才返回 200。

## 后续 TODO

- 当前临时将 JWT 文件放到仓库 `docker/jwt/jwt_secret_key.txt`，后续应改为公司平台 Secret / 环境变量注入 `JWT_SECRET_KEY`，避免多机器部署时密钥不一致。
