# nuwax 改造内容

## 1. 背景

本次改造针对 `nuwax` 前端在单容器部署下的访问问题：页面访问后持续重定向、运行时无法正确识别后端环境、登录页语言显示不符合预期，以及 `/api` 请求在静态 Nginx 容器中无法稳定转发到后端。

实际代码改造落点为上层真实仓库：`../nuwax`。

## 2. 改造目标

- 前端容器可独立通过 80 端口启动。
- 浏览器请求同源 `/api`，由 Nginx 转发到后端。
- 运行时配置通过容器环境变量注入，不依赖重新构建切换环境。
- 登录页不再出现 `/login?redirect=...` 无限嵌套。
- 新环境首次访问默认展示中文，并保留租户语言配置读取能力。
- 登录页和首页关键主链路可通过浏览器实际验证。

## 3. 主要改造内容

### 3.1 Docker 与运行时配置

修改文件：

- `Dockerfile`
- `docker/entrypoint.sh`
- `public/runtime-config.template.js`
- `src/utils/runtimeConfig.ts`

改造内容：

- 将前端镜像调整为构建阶段与运行阶段分离。
- 构建阶段使用 `pnpm build:prod` 产出 `dist`。
- 运行阶段使用 Nginx 托管静态资源。
- 容器启动时渲染 `/usr/share/nginx/html/runtime-config.js`。
- 新增统一运行时配置读取工具，集中处理：
  - `BASE_URL`
  - `WS_BASE_URL`
  - `ROUTER_BASENAME`
  - `PUBLIC_PATH`
  - `ENABLE_MOBILE_REDIRECT`
  - `WITH_CREDENTIALS`
  - `APP_ENV`
  - `APP_VERSION`

当前推荐部署模式：

- `BASE_URL=''`
- `WITH_CREDENTIALS=''`
- 浏览器请求相对路径 `/api`
- Nginx 代理到后端，例如 `http://host.docker.internal:8080`

### 3.2 Nginx 同源代理

修改文件：

- `k8s/default.conf.template`

改造内容：

- 新增 `/healthz` 健康检查。
- 新增 `/api/` 反向代理。
- 保留 SPA fallback：前端路由仍回退到 `index.html`。
- 补齐代理头：`Host`、`X-Real-IP`、`X-Forwarded-For`、`X-Forwarded-Proto`、`Upgrade`、`Connection`。

解决的问题：

- 静态 Nginx 容器不再把 `/api` 请求当作前端路由处理。
- 浏览器侧避免跨域 Cookie / CORS 复杂度，统一走同源请求。

### 3.3 请求基础链路

修改文件：

- `src/services/common.ts`
- `src/utils/runtimeConfig.ts`

改造内容：

- 所有普通接口请求统一通过 `withBaseUrl(url)` 处理。
- `credentials` 根据运行时配置自动决定：
  - 同源代理模式默认使用 `same-origin`。
  - 跨域后端模式可通过 `WITH_CREDENTIALS=true` 启用 `include`。
- 请求头继续带上 `Accept-Language`。

### 3.4 登录重定向修复

修改文件：

- `src/utils/router.ts`
- `src/services/i18n.ts`

改造内容：

- `redirectToLogin` 增加幂等保护：当前已经在 `/login` 时不再重复跳转。
- 登录跳转使用 `replace`，避免历史记录和 redirect 参数持续膨胀。
- 登录页语言列表接口 `/api/i18n/lang/list` 增加 `skipErrorHandler`。

解决的问题：

- 登录页请求语言列表时，后端返回 `4010` 未登录，之前会触发全局错误处理并再次跳转登录页。
- 该跳转会把当前完整 `/login?...` 再次塞入 redirect 参数，形成无限嵌套。

### 3.5 i18n 默认中文与租户语言读取

修改文件：

- `src/constants/i18n.constants.ts`
- `src/services/i18nRuntime.ts`

改造内容：

- 默认语言从 `en-us` 改为 `zh-cn`。
- i18n 初始化语言优先级调整为：
  1. `localStorage.umi_locale`
  2. 租户缓存 `TENANT_CONFIG_INFO.templateConfig.language`
  3. 默认语言 `zh-cn`
- 语言接口失败或返回异常时，按当前语言使用本地字典兜底，不再强制回退英文。

当前验证结果：

- 租户接口返回 `siteName: 女娲智能体OS`。
- 当前租户 `templateConfig.language` 为 `null`。
- 首屏默认语言已为 `zh-CN`。

## 4. 当前运行结果

当前本地验证环境：

- 前端镜像：`nuwax-frontend:local`
- 最新镜像 ID：`sha256:a477ebf9bd6e5ea2bad72db35f6a624a53adfd7c17dfc0eae4da6967b981d3a6`
- 容器名：`docker-frontend-1`
- 端口：`0.0.0.0:80->80/tcp`
- 后端代理目标：`http://host.docker.internal:8080`

已验证：

- `GET /healthz` 返回 `ok`。
- `GET /runtime-config.js` 正常返回运行时配置。
- `GET /api/tenant/config` 通过前端 80 端口代理成功。
- 浏览器打开 `http://localhost/login` 不再重定向循环。
- 登录页正常显示中文租户内容。
- 使用管理员账号登录后进入 `http://localhost/home`。
- 首页中文内容和主要接口请求正常。

## 5. 验证命令

在真实仓库 `../nuwax` 执行：

```bash
pnpm build:prod
```

本地运行容器验证：

```bash
docker run -d \
  --name docker-frontend-1 \
  --add-host=host.docker.internal:host-gateway \
  -p 80:80 \
  nuwax-frontend:local
```

健康检查：

```bash
curl -i http://localhost/healthz
curl -i http://localhost/runtime-config.js
curl -i http://localhost/api/tenant/config
```

浏览器验证：

- 打开 `http://localhost/login`。
- 确认 URL 不再持续增长。
- 确认登录页显示中文和租户名称。
- 登录后确认进入 `/home`。

## 6. 注意事项

- 如果采用同源代理模式，推荐保持 `BASE_URL` 为空，让浏览器请求 `/api`。
- 如果必须跨域直连后端，需要同时配置：
  - `BASE_URL` 为后端地址。
  - `WITH_CREDENTIALS=true`。
  - 后端 CORS 精确允许前端域名，不能使用 `*`。
  - Cookie 满足跨站要求，例如 `SameSite=None; Secure`。
- 登录页语言接口在未登录时可能返回 `4010`，这类首屏公共接口不应触发全局登录跳转。

## 7. 版本记录

- 2026-06-22：新增 nuwax 单容器部署、运行时配置、同源代理、登录重定向与 i18n 默认中文改造记录。