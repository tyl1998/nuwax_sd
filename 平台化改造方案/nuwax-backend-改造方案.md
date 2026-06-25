# nuwax-backend 改造方案（单容器 + 配置可注入 + 数据库依赖编排）

> 真实源码仓：`/Users/atan/Desktop/work/vs_code_nuwax/nuwax-backend`
> 本方案与前端方案 `nuwax-改造方案.md` 配套，前端已完成运行时配置注入改造，本轮聚焦后端。

## 1. 背景与目标

### 背景

`nuwax-backend` 是一个 Spring Boot 3 / WebFlux 的 Maven 多模块单体（DDD 分层，18+ 业务模块），默认端口 `8081`，通过 `spring.profiles.active` 选择 `application-{env}.yml`。

当前部署形态（见 `nuwax_deploy/docker/docker-compose.yml`）有三个特征：

1. **不是自构建镜像**：compose 把外部预构建的 `app.jar`、`application-external.yml`、`docker-entrypoint.sh` 以 volume 挂进通用 JRE 容器运行，镜像内不含应用产物。
2. **配置入口分散**：Spring 侧读 `DB_HOST/DB_NAME/REDIS_HOST/MILVUS_URI` 等细粒度变量（见 `application-prod.sample.yml`），而 compose 注入的是 `DATABASE_URL/REDIS_URL/MILVUS_HOST` 等聚合变量，中间靠 `docker-entrypoint.sh` 手工桥接，命名不一致、易错。
3. **强基础设施依赖**：启动即连 MySQL、Redis、Milvus，compose 用 `depends_on: service_healthy` 串起来；Elasticsearch、Doris、COS/OSS/S3、rcoder、mcp-proxy 为按需依赖。

### 目标

1. `nuwax-backend` 可作为单系统独立部署：**自构建可分发镜像**（应用产物进镜像，不再依赖外部挂载 jar）。
2. **运行时配置可注入**：同一镜像通过环境变量切换 dev/test/prod，无需重新构建。
3. **数据库依赖显式化、可编排**：MySQL/Redis/Milvus 为硬依赖并做就绪等待；ES/Doris/对象存储/沙箱等为可选依赖，缺失时行为可预期（启动不崩、降级或明确报错）。
4. 统一健康检查（`/health` 存活、`/ready` 就绪），对接平台节点探针。
5. 保持 DDD 分层、SSE 流式推流、鉴权链路等既有能力不受影响。

## 2. 改造范围

### 包含

- 后端配置入口统一化（环境变量命名对齐，去掉 entrypoint 里的手工桥接）
- 自构建多阶段 Dockerfile（Maven 构建 + JRE 运行）
- 容器启动入口脚本（渲染/校验配置、等待依赖、启动 JVM）
- 数据库与中间件依赖编排（硬依赖就绪等待 + 可选依赖降级）
- SQL 初始化与版本化迁移策略（`sql/init.sql` + `update-*.sql`）
- 健康检查 / 就绪检查标准化与探针配置
- 跨域（CORS）与凭证策略对齐前端单容器跨域场景

### 不包含

- 业务 API 协议与领域逻辑重写
- DDD 分层结构调整
- 模型适配 / Agent 执行引擎内部改造
- 多区域 / 高可用集群拓扑（本轮只做单容器可独立部署）

## 3. 配置注入设计

### 3.1 设计原则

1. 优先读取容器启动时注入的环境变量。
2. 兜底读取镜像内打包的默认 profile（`application.yml` 公共段 + `application-{env}.yml`）。
3. 未配置时有安全默认值；**密钥类无默认值或为占位值，缺失即显式失败**，不允许静默用弱默认密钥启动生产。
4. **环境变量命名以 Spring 配置实际读取的为准**（`application-prod.sample.yml` 已是规范来源），compose / k8s 必须对齐，删除 entrypoint 内的变量改名桥接。

### 3.2 环境变量清单（以源码实际读取为准）

来源：`app-platform-bootstrap/app-platform-web-bootstrap/src/main/resources/application-prod.sample.yml`。

#### 硬依赖（缺失则不可启动）

| 变量名 | 说明 | 示例 / 默认 |
| --- | --- | --- |
| `APP_PROFILE` / `spring.profiles.active` | 运行环境 profile | `prod` |
| `DB_HOST` | MySQL 主库地址 | `mysql` |
| `DB_NAME` | 主库库名 | `agent_platform` |
| `DB_USERNAME` | 主库用户名 | `root` |
| `DB_PASSWORD` | 主库密码（**生产必填，无弱默认**） | — |
| `REDIS_HOST` | Redis 地址 | `redis` |
| `REDIS_PORT` | Redis 端口 | `6379` |
| `REDIS_PASSWORD` | Redis 密码 | — |
| `REDIS_DB` | Redis 库号 | `1` |
| `MILVUS_URI` | 向量库地址 | `http://milvus:19530` |
| `MILVUS_USER` / `MILVUS_PASSWORD` | 向量库凭证 | `root` / — |
| `JWT_SECRET_KEY` | JWT 签名密钥（**≥32 字符，生产必填**） | — |

#### 可选依赖（缺失时降级 / 该能力不可用，但不阻塞启动）

| 变量名 | 说明 | 默认 |
| --- | --- | --- |
| `DORIS_HOST` / `DORIS_DB_NAME` / `DORIS_USERNAME` / `DORIS_PASSWORD` | 自定义数据表（compose 模块） | localhost / agent_custom_table |
| `ES_URL` / `ES_USERNAME` / `ES_PASSWORD` / `ES_API_KEY` | 知识库全文检索、模型代理日志 | `http://localhost:9200` |
| `STORAGE_TYPE` | `local` / `cos` / `oss` / `s3` | `local` |
| `COS_*` / `OSS_*` / `S3_*` | 对应对象存储凭证 | 空 |
| `FILE_UPLOAD_FOLDER` / `FILE_BASE_URL` | 本地上传目录与外链前缀 | `/tmp/uploads` |
| `MCP_PROXY_URL` | mcp-proxy 地址 | `http://localhost:8020` |
| `CODE_EXECUTE_URL` | 代码执行（rcoder） | `http://localhost:8020/api/run_code_with_log` |
| `LOG_SERVICE_URL` | Rust 日志服务 | `http://localhost:8097` |
| `BUILD_SERVER_URL` / `AI_AGENT_URL` / `DOCKER_PROXY_URL` | 无代码页面构建链路 | localhost:* |
| `ECO_MARKET_SERVER_URL` | 生态市场 server | `https://agent-market-api.xspaceagi.com` |
| `MODEL_API_BASE_URL` / `MODEL_PROXY_ENABLE` / `MODEL_PROXY_PORT` | 模型代理 | enable=true / 18086 |
| `SERVICE_HOST` / `BIND_HOST` / `REVERSE_PORTS` / `OUTER_HOST` / `OUTER_PORT` | 内外穿透 | 见 sample |

#### 跨域与前端联调相关（配合前端单容器跨域）

| 变量名 | 说明 | 建议 |
| --- | --- | --- |
| `access.control.allow-origin`（现为 `"*"`） | CORS 允许源 | 改为可注入 `CORS_ALLOW_ORIGIN`，跨域携带凭证时**必须精确域名，不能用 `*`** |
| `CORS_ALLOW_CREDENTIALS`（新增） | 是否允许携带凭证 | 前端 `WITH_CREDENTIALS=true` 时后端须为 `true` |

> 关键约束（与前端方案一致）：前端单容器跨域访问后端时，后端 CORS 必须精确域名 + 允许 credentials；跨站 Cookie 需 `SameSite=None; Secure`（HTTPS）；SSO/CAS 回调域与前端实际访问域一致。详见前端 `sd/README.md` 2026-06-22 联调结论。

### 3.3 配置优先级

运行时环境变量 > 镜像内 `application-{env}.yml` > `application.yml` 公共默认 > 代码内默认值。

### 3.4 命名对齐改造（重点）

当前 compose 注入 `DATABASE_URL/REDIS_URL/MILVUS_HOST/MYSQL_*`，与 Spring 读取的 `DB_HOST/DB_NAME/REDIS_HOST/MILVUS_URI` 不一致，靠 `docker-entrypoint.sh` 桥接。本轮**以 sample.yml 的变量名为唯一标准**，compose / k8s 直接注入这些变量，删除 entrypoint 内的改名逻辑，消除两套命名。

## 4. 部署形态（单容器）

1. **多阶段镜像**：阶段一 `maven` 构建 `mvn -pl app-platform-bootstrap/app-platform-web-bootstrap -am clean package -DskipTests`；阶段二 `eclipse-temurin:JRE` 仅复制最终 jar。
2. **端口**：应用 `8081`（容器内），对外按节点策略映射；内外穿透端口（`OUTER_PORT`、`REVERSE_PORTS`）按需开放。
3. **入口脚本** `docker/entrypoint.sh`：① 校验硬依赖变量与密钥非空；② 等待 MySQL/Redis/Milvus 就绪（TCP 探测，带超时）；③ 以 `JAVA_OPTS` 启动 jar。
4. **探针**：存活 `/health`，就绪 `/ready`（就绪应反映 DB/Redis 连接可用）。
5. **持久化卷**：`/app/upload`（本地存储时）、`/app/logs`、`/app/config/jwt`（JWT 密钥文件，如沿用文件挂载）。
6. **数据库**：MySQL/Redis/Milvus 作为外部服务或同编排服务；镜像本身**不内置数据库**，仅内置 SQL 初始化脚本供首次初始化使用。

## 5. 改造步骤（按文件与模块）

### 5.1 必须改 / 新增

| 文件 | 改动内容 | 原因 | 验证点 |
| --- | --- | --- | --- |
| `Dockerfile`（新增于 `nuwax-backend/`） | 多阶段：Maven 构建 + JRE 运行，产物入镜像 | 当前无自构建镜像，依赖外部挂载 jar，无法独立分发 | `docker build` 产出可独立 `docker run` 的镜像 |
| `docker/entrypoint.sh`（新增） | 校验必填变量与密钥、等待硬依赖就绪、启动 JVM | 取代现有挂载式 `docker-entrypoint.sh` 的变量桥接逻辑 | 依赖未就绪时等待并最终明确报错，而非半启动 |
| `application-prod.sample.yml` → 落地为 `application-prod.yml` 注入路径 | `access.control.allow-origin` 改为 `${CORS_ALLOW_ORIGIN:*}`；新增 `CORS_ALLOW_CREDENTIALS` | 支撑前端跨域携带凭证；生产禁止 `*`+credentials 组合 | 跨域请求带 cookie 能通过且仅放行精确域名 |
| CORS 配置类（`app-platform-bootstrap` 内 WebMvc/CorsFilter） | 读取 `CORS_ALLOW_ORIGIN` / `CORS_ALLOW_CREDENTIALS`，支持多域名逗号分隔 | 现为硬编码 `*`，与 credentials 互斥 | 多域名精确匹配，凭证可携带 |
| `docker-compose`（后端段，落到 `nuwax-backend/deploy/` 或更新 `nuwax_deploy`） | 改注入 `DB_HOST/DB_NAME/DB_USERNAME/DB_PASSWORD/REDIS_*/MILVUS_URI/...`，对齐 sample 变量名 | 消除 `DATABASE_URL` 等与 Spring 读取名不一致问题 | 不经桥接即可启动 |
| `sql/README.md`（新增） | 说明 `init.sql` 全量初始化 + `update-YYYYMMDD.sql` 增量迁移的执行顺序与幂等约定 | 当前迁移脚本散落、无执行规范，新环境初始化易漏 | 按文档可从空库初始化到最新结构 |

### 5.2 建议改

| 文件 | 改动内容 | 原因 |
| --- | --- | --- |
| `HealthController` | `/ready` 增加 DB / Redis / Milvus 轻量连通性检查 | 就绪探针应真实反映依赖可用，而非常量 200 |
| `application.yml` | `knife4j.production` 由 `CORS`/`APP_ENV` 控制，生产关闭 API 文档 | 生产不暴露 `/doc.html` |
| `deploy/.env.sample`（新增） | 汇总全部环境变量与分级（必填/可选） | 交付标准化，降低漏配 |
| `k8s/`（新增模板，可选） | Deployment + readiness/liveness 探针 + env 注入 | 平台节点编排参考 |

### 5.3 暂不改

| 范围 | 原因 |
| --- | --- |
| DDD 分层与模块拆分 | 本轮只做部署与配置，不动架构 |
| Agent 执行引擎 / 模型适配 | 非部署主链路 |
| 业务 API 协议 | 保持前后端兼容 |
| Doris/ES 内部用法 | 仅做"可选依赖降级"包装，不改查询逻辑 |

## 6. 数据库与中间件依赖编排

### 6.1 依赖分级

| 依赖 | 级别 | 缺失行为 | 就绪策略 |
| --- | --- | --- | --- |
| MySQL（master，库 `agent_platform`） | **硬** | 启动失败 | entrypoint TCP 等待 + `depends_on: service_healthy` |
| Redis | **硬** | 启动失败 | 同上 |
| Milvus | **硬** | 启动失败 | 同上 |
| Doris（库 `agent_custom_table`） | 可选 | compose 数据表模块不可用，主服务正常 | 懒连接，调用时报明确错误 |
| Elasticsearch | 可选 | 全文检索 / 模型代理日志降级 | 懒连接 |
| 对象存储 COS/OSS/S3 | 可选 | 回退 `local` 存储 | `STORAGE_TYPE` 选择 |
| rcoder / mcp-proxy / build-server | 可选 | 对应能力不可用 | 调用时报错，不阻塞启动 |

### 6.2 SQL 初始化与迁移

- 全量：`sql/init.sql`（含 `agent_platform` 全部表，40+ 张）。
- 增量：`sql/update-YYYYMMDD.sql`（已有 `20260320 / 20260331 / 20260418 / 20260510 / 20260524`），按日期顺序执行。
- 多租户：`mybatis-plus.tenant.ignoreTenantTables` 列出的表为系统级，不带租户隔离；迁移时注意这些表的结构变更影响全局。
- 约定：迁移脚本应幂等（`CREATE TABLE IF NOT EXISTS` / `ALTER ... IF NOT EXISTS` 或前置存在性判断），便于重复执行。
- Doris 库（`agent_custom_table`）由数据表模块按需建表（`replication-num` / `bucket-num` 等在 sample 中配置），不在 `init.sql` 范围内。

### 6.3 首次初始化流程

1. 创建 MySQL 库 `agent_platform`，导入 `init.sql`。
2. 按日期顺序导入所有 `update-*.sql`。
3. 启动 Redis、Milvus（可选 ES/Doris）。
4. 注入环境变量启动后端，`/ready` 通过即初始化完成。

## 7. 回滚方案

1. 保留现有"挂载式 jar + 外部 application-external.yml"部署路径作为兜底，可不切自构建镜像直接回退。
2. CORS 改造保留 `${CORS_ALLOW_ORIGIN:*}` 默认值，异常时回到原 `*` 行为（仅限不带 credentials 场景）。
3. 镜像版本化（`agent-platform-backend:<tag>`），回滚即切回上一个 tag。
4. SQL 迁移按版本回退：仅执行结构新增/兼容性变更，破坏性变更需配套 down 脚本或备份。
5. entrypoint 等待逻辑可通过 `WAIT_DEPENDENCIES=false` 关闭，回到"立即启动"老行为。

## 8. 验证方案

### 8.1 启动验证
单容器（外接 MySQL/Redis/Milvus）可独立启动；进程稳定 5 分钟无异常重启；`/health` 返回 200。

### 8.2 配置验证
同一镜像分别注入 DEV / PROD 变量集，连接到各自数据库与中间件，无需重新构建。检查日志中实际连接的 `DB_HOST` / `MILVUS_URI` 与预期一致。

### 8.3 健康验证
- `/health` 始终 200（存活）。
- `/ready` 在 DB/Redis/Milvus 全部可用时 200；任一硬依赖断开时非 200（探针应能摘流量）。

### 8.4 功能验证
- 登录（`/api/user/login`）签发 JWT 正常。
- 会话 SSE 推流（`/api/agent/conversation`）正常流式返回。
- 知识库检索、文件上传按 `STORAGE_TYPE` 正常。
- 前端单容器跨域 + `WITH_CREDENTIALS=true` 场景：`/api/user/getLoginInfo` 带 cookie 正常，无持续重定向（对齐前端 `4011` 联调结论）。

### 8.5 容错验证
- Doris/ES/对象存储/mcp-proxy 不可用时，主服务仍可启动，仅对应能力降级并给出明确错误码，不白屏不崩溃。
- 硬依赖（MySQL/Redis/Milvus）不可用时，容器在等待超时后明确失败退出，日志可定位。

### 8.6 回归验证
- 既有 API 协议不变，前端无需改动即可联调。
- 多租户隔离（`ignoreTenantTables` 之外的表）行为不变。
- 鉴权链路（AuthInterceptor / ApiKeyInterceptor / 白名单）不变。

## 9. 验收标准

1. `nuwax-backend` 可由自构建镜像在公司节点单容器独立部署，不依赖外部挂载 jar。
2. 同一镜像仅改环境变量即可切换 dev/test/prod 后端环境。
3. 环境变量命名与 Spring 配置实际读取一致，删除 entrypoint 桥接改名逻辑。
4. `/health`、`/ready` 探针生效；硬依赖缺失时就绪探针能正确摘流量。
5. 数据库初始化（`init.sql` + `update-*.sql`）可从空库一键到最新结构。
6. 前端单容器跨域携带凭证场景联调通过（CORS 精确域名 + credentials）。
7. 可选依赖缺失时主服务可用、行为可预期。

## 10. 最小首批改造范围

1. `Dockerfile`（自构建多阶段）
2. `docker/entrypoint.sh`（依赖等待 + 变量校验）
3. CORS 配置参数化（`CORS_ALLOW_ORIGIN` / `CORS_ALLOW_CREDENTIALS`）
4. compose 后端段变量名对齐 sample
5. `/ready` 增加硬依赖连通性检查
6. `sql/README.md` + 迁移执行规范

覆盖：自构建分发、运行时配置注入、数据库依赖编排、健康检查、前端跨域联调。

## 11. 版本记录

- 2026-06-23：初版建立。基于真实仓 `nuwax-backend`（`application-prod.sample.yml`、`sql/`、现有 `nuwax_deploy` compose）梳理数据库依赖与配置注入，产出后端单容器改造方案。
