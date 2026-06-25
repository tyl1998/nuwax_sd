# nuwax 改造方案（单容器 + 配置可注入）

## 1. 背景与目标

### 背景

当前 `nuwax` 采用前端静态构建产物 + Nginx 托管模型，适合容器化，但现有部署方式偏向 compose 整体编排。

### 目标

1. `nuwax` 可作为单系统独立部署，不依赖其他业务容器启动顺序。
2. 运行时配置可注入，同一镜像可部署到多环境（dev/test/prod）。
3. 保持 SPA 路由与既有页面能力不受影响。

## 2. 改造范围

### 包含

- 前端运行时配置注入机制
- Docker 单容器启动流程增强
- Nginx 配置模板参数化
- 健康检查与部署验证

### 不包含

- 后端接口改造
- 业务功能逻辑重写
- 多容器编排（compose/k8s）模板细化

## 3. 配置注入设计

## 3.1 设计原则

1. 优先读取运行时配置（容器启动时注入）
2. 兜底读取构建期默认值
3. 未配置时有安全默认值（如空字符串或同源路径）

## 3.2 建议环境变量清单（首批）

| 变量名 | 说明 | 示例 |
| --- | --- | --- |
| `BASE_URL` | 后端 API 基地址 | `https://api.example.com` |
| `WS_BASE_URL` | SSE/WS 基地址（如有） | `wss://api.example.com` |
| `ROUTER_BASENAME` | 子路径部署前缀 | `/nuwax` |
| `PUBLIC_PATH` | 静态资源前缀 | `/` 或 `https://cdn.example.com/nuwax/` |
| `ENABLE_MOBILE_REDIRECT` | PC/H5 跳转开关 | `true` |
| `APP_ENV` | 运行环境标识 | `prod` |
| `APP_VERSION` | 版本标识 | `1.2.3` |

## 3.3 配置优先级

运行时注入配置 > 构建期默认配置 > 代码内默认值

## 4. 部署形态（单容器）

1. 多阶段镜像构建：Node 构建 + Nginx 运行。
2. 容器启动时由入口脚本渲染 `runtime-config.js`。
3. Nginx 托管 `dist`，并配置 SPA 回退策略（`index.html`）。
4. 提供健康检查端点：`/healthz`。

## 5. 改造步骤（文件级）

1. 新增运行时配置模板文件（如 `public/runtime-config.template.js`）。
2. 新增前端配置读取模块（统一读取 `window` 运行时配置）。
3. 改造请求封装入口，统一使用运行时 `BASE_URL`。
4. 新增容器入口脚本（渲染运行时配置并启动 Nginx）。
5. 调整 Dockerfile，复制模板和入口脚本。
6. 调整 Nginx 模板，增加 `/healthz` 与参数化能力。
7. 增加部署说明文档，给出变量与示例。

## 6. 回滚方案

1. 保留原构建期配置读取路径作为兜底。
2. 若运行时注入异常，可关闭入口脚本渲染逻辑并回到静态配置。
3. 回滚镜像到前一版本，保留旧部署参数集。

## 7. 验证方案

## 7.1 启动验证

目标：单容器独立可启动。

检查项：

1. 容器进程正常运行。
2. 首页可访问，静态资源返回 200。
3. `/healthz` 返回 200。

通过标准：连续 5 分钟无异常重启。

## 7.2 配置注入验证

目标：同一镜像仅改环境变量即可切换环境。

检查项：

1. 使用镜像 A + 配置集 DEV，页面请求指向 DEV 后端。
2. 使用同一镜像 A + 配置集 PROD，页面请求指向 PROD 后端。
3. 不重新构建镜像。

通过标准：两套环境请求地址与预期一致。

## 7.3 功能验证

目标：核心主链路不受影响。

检查项：

1. 登录流程正常。
2. 会话页面可发送消息。
3. 关键页面路由刷新正常（SPA 回退生效）。

通过标准：核心路径可用，无阻断问题。

## 7.4 容错验证

目标：后端异常时前端行为可预期。

检查项：

1. 后端不可达时，前端容器仍可启动。
2. 前端展示错误提示，不出现白屏崩溃。

通过标准：可观测到明确错误提示，前端服务进程稳定。

## 7.5 回归验证

目标：防止改造破坏原行为。

检查项：

1. 历史主要页面可访问。
2. 静态资源缓存策略未异常。
3. 浏览器控制台无新增高频错误。

通过标准：关键回归用例通过率 100%。

## 8. 验收标准

1. `nuwax` 可在公司节点以单容器独立部署。
2. 同一镜像可通过环境变量切换不同后端环境。
3. 健康检查、路由刷新、核心业务链路均通过验证。
4. 形成可复用的部署参数模板与验证记录。

## 9. 版本记录

- 2026-06-18：初版方案建立（目录化 + 验证标准化）。

## 10. 具体改动文件清单

以下清单基于真实源码仓：`/Users/atan/Desktop/work/vs_code_nuwax/nuwax`。

## 10.1 必须改

| 文件 | 改动内容 | 原因 | 验证点 |
| --- | --- | --- | --- |
| `src/utils/runtimeConfig.ts`（新增） | 封装运行时配置读取，提供 `getRuntimeConfig()`、`getBaseUrl()`、`withBaseUrl()` 等方法 | 避免业务代码继续分散直读 `process.env.BASE_URL`，形成单一配置入口 | 单测验证运行时配置优先级：`window.__NUWAX_RUNTIME_CONFIG__` > `process.env` > 默认值 |
| `src/types/runtime-config.d.ts`（新增或合并到现有声明文件） | 声明 `window.__NUWAX_RUNTIME_CONFIG__` 类型 | 保证 TypeScript 编译通过，避免 `window` 扩展属性报错 | `pnpm test` / `pnpm build:prod` 不出现类型错误 |
| `public/runtime-config.template.js`（新增） | 定义运行时配置模板，例如 `BASE_URL`、`WS_BASE_URL`、`APP_ENV`、`APP_VERSION` | 容器启动时由环境变量渲染为 `runtime-config.js` | 构建产物中存在模板，容器启动后生成真实配置文件 |
| `config/config.ts` | 在 `scripts` 或 `headScripts` 中注入 `/runtime-config.js`，必要时保留不存在时不阻塞页面加载 | 确保 React 应用初始化前运行时配置已挂载到 `window` | 浏览器控制台可读取 `window.__NUWAX_RUNTIME_CONFIG__` |
| `src/services/common.ts` | 请求拦截器中用 `withBaseUrl(url)` 替换 `process.env.BASE_URL + url` | 全局接口请求是最核心链路，必须支持运行时注入后端地址 | 登录、菜单、普通 API 请求实际打到注入的 `BASE_URL` |
| `src/constants/common.constants.ts` | 上传、会话 SSE、提示词优化等 URL 常量改为函数或基于 `withBaseUrl()` 动态生成 | 该文件在模块加载期固化 URL，直接常量会导致运行时配置不生效或不易测试 | 上传地址、会话 SSE 地址、优化接口地址随运行时配置变化 |
| `src/models/appDevSseConnection.ts` | SSE URL 使用统一配置方法生成 | AppDev 长连接必须跟随后端地址配置 | AppDev SSE 请求地址正确 |
| `src/utils/chatUtils.ts` | `generateSSEUrl()`、`generateAIChatSSEUrl()` 使用统一配置方法 | AI 聊天 / 页面开发 SSE 是主链路，不能继续绑定构建期 BASE_URL | SSE 地址随容器环境变量变化 |
| `Dockerfile` | 改为 Node 构建 + Nginx 运行 + 复制入口脚本与运行时配置模板，启动时渲染配置 | 当前 Dockerfile 在 build 阶段执行 `envsubst`，不支持同镜像多环境 | `docker run -e BASE_URL=...` 不重新构建即可切换后端 |
| `docker/entrypoint.sh`（新增） | 容器启动时生成 `/usr/share/nginx/html/runtime-config.js`，再启动 Nginx | 实现运行时配置注入的关键入口 | 容器内 `runtime-config.js` 内容与环境变量一致 |
| `k8s/default.conf.template` | 增加 `/healthz`，保留 SPA `try_files`，可选增加 `/api` 代理模板 | 公司节点需要健康检查；SPA 刷新不能破坏 | `/healthz` 返回 200，任意前端路由刷新返回 `index.html` |

## 10.2 建议改

| 文件 | 改动内容 | 原因 | 验证点 |
| --- | --- | --- | --- |
| `k8s/config/frontend-deployment.yaml` | 增加 `env`、`readinessProbe`、`livenessProbe` 示例 | 为公司节点部署提供可参考模板 | 探针命中 `/healthz`，环境变量能注入 |
| `README.md` 或新增 `docs/deploy-single-container.md` | 补充单容器部署说明、环境变量清单、`docker run` 示例 | 后续交付/平台部署需要标准化说明 | 文档命令可直接跑通 |
| `config/config.production.ts` | 保留构建期默认 `BASE_URL=''`，并说明生产推荐走运行时配置 | 避免旧逻辑断裂，同时明确新旧优先级 | 无运行时配置时仍按原行为同源请求 |
| `src/utils/fileTree.tsx`、`src/components/FileTreeView/index.tsx` 等散落 URL 拼接点 | 分阶段替换为 `withBaseUrl()` | 降低后续维护成本，避免局部功能仍绑定构建期配置 | 文件预览、下载、代理 URL 正常 |
| `src/pages/AppDev/**`、`src/pages/AppDevDesign/**` | 分阶段替换页面级 `process.env.BASE_URL` | AppDev 页面内 URL 拼接较多，建议集中清理 | AppDev 预览、生产地址跳转正常 |

## 10.3 暂不改

| 文件/范围 | 暂不改原因 |
| --- | --- |
| 后端接口协议 | 本轮只做前端单容器与配置注入，不改变 API 协议 |
| 移动端构建下载逻辑 `scripts/download-mobile-build.js` | 不属于 PC 前端单容器启动主链路，除非部署要求同时内置 H5 包 |
| 业务页面逻辑 | 本轮只替换配置来源，不重构业务交互 |
| compose 编排文件 | `nuwax` 仓当前无 compose 文件，单系统部署不依赖 compose |

## 10.4 最小首批改造范围

如果要降低风险，第一批只改下面这些文件即可完成核心目标：

1. `src/utils/runtimeConfig.ts`
2. `src/types/runtime-config.d.ts`
3. `public/runtime-config.template.js`
4. `config/config.ts`
5. `src/services/common.ts`
6. `src/constants/common.constants.ts`
7. `src/models/appDevSseConnection.ts`
8. `src/utils/chatUtils.ts`
9. `Dockerfile`
10. `docker/entrypoint.sh`
11. `k8s/default.conf.template`

这 11 个文件覆盖：运行时配置注入、普通 API、上传地址、SSE 主链路、容器启动、健康检查。

## 10.5 改造后的验证命令建议

本地源码验证：

```bash
cd /Users/atan/Desktop/work/vs_code_nuwax/nuwax
pnpm install
pnpm test
pnpm build:prod
```

镜像验证：

```bash
docker build -t nuwax-frontend:runtime-config .
docker run --rm -p 8080:80 \
	-e BASE_URL=https://testagent.xspaceagi.com \
	-e APP_ENV=test \
	nuwax-frontend:runtime-config
```

接口验证：

```bash
curl -i http://localhost:8080/healthz
curl -s http://localhost:8080/runtime-config.js
```

浏览器验证：

1. 打开 `http://localhost:8080`。
2. 控制台执行 `window.__NUWAX_RUNTIME_CONFIG__`。
3. Network 面板确认 `/api/**`、SSE、上传接口均指向注入的 `BASE_URL`。
4. 刷新任意前端路由，确认仍返回页面而不是 404。