# nuwax_sd

## 工作准则

### 代码修改落点标准

- `sd/` 目录用于文档沉淀、链路梳理、方案设计和阅读索引。
- 如果任务目标是“修改真实项目代码 / 配置 / 脚本 / Dockerfile / 部署文件”，默认应在 `sd/` 上层目录的对应项目仓库中完成，而不是在 `sd/` 镜像文档目录中修改。
- 对应关系示例：
	- `sd/nuwax/` 对应上层真实项目 `../nuwax`
	- `sd/nuwax-backend/` 对应上层真实项目 `../nuwax-backend`
	- `sd/mcp-proxy/` 对应上层真实项目 `../mcp-proxy`
	- `sd/rcoder/` 对应上层真实项目 `../rcoder`
- 只有在目标本身就是补充说明、沉淀方案、更新阅读文档时，才在 `sd/` 内修改 Markdown 文档。

### 执行约定

- 先在 `sd/` 中定位链路、方案和仓库映射，再切换到上层真实项目目录实施修改。
- 回答中如果提到“已修改项目”，默认指上层对应真实仓库，不指 `sd/` 下的文档副本。
- 若需把实现结果同步沉淀为文档，再回写到 `sd/` 对应目录。

## 当前进度补充（2026-06-22）

### 问题定位结论

- “单容器启动后的 nuwax 连接后端后持续重定向 URL” 的核心方向，已定位为跨域 cookie / SSO 场景支持不完整，不是单纯 `/login` 路由本身有问题。
- 当前 localhost 直测结果里，`/login` 页面本身可正常返回前端页面；`/api/user/getLoginInfo` 返回的是 `4010`，不是 `4011`。
- `4011` 代表后端要求前端跳转到外部认证地址；如果前端跨域请求未携带凭证，或后端 SSO 回调 / Cookie / CORS 配置不匹配，就可能出现持续重定向。

### 已完成的真实代码改造范围

- 真实改动仓库为上层项目 `../nuwax`，不是 `sd/nuwax/` 文档目录。
- 已在 `../nuwax` 中补入运行时配置开关 `WITH_CREDENTIALS`，用于控制前端是否以 `credentials=include` 方式访问后端。
- 已在普通请求链路中补入凭证支持与 `4011` 跳转去重逻辑。
- 已在 SSE / 流式请求链路中补入凭证支持。
- 已补充运行时配置模板、容器启动注入逻辑、TypeScript 全局声明等配套内容。

### 已修改文件清单（真实仓库 `../nuwax`）

- `public/runtime-config.template.js`
- `docker/entrypoint.sh`
- `Dockerfile`
- `typings.d.ts`
- `src/utils/authRedirect.ts`（新增）
- `src/services/common.ts`
- `src/services/userService.ts`
- `src/utils/fetchEventSource.ts`
- `src/utils/fetchEventSourceConversationInfo.ts`

### 当前剩余阻塞

- `../nuwax/src/utils/runtimeConfig.ts` 在一次失败写入后被污染，文件中混入了重复定义与脏字符，导致 `pnpm build:prod` 失败。
- 当前最优先动作是在真实仓库中手工清理或整体重写该文件，然后重新执行生产构建验证。

### `runtimeConfig.ts` 应保留的最终能力

- 导出接口 `NuwaxRuntimeConfig`，至少包含以下字段：
	- `BASE_URL`
	- `WS_BASE_URL`
	- `ROUTER_BASENAME`
	- `PUBLIC_PATH`
	- `ENABLE_MOBILE_REDIRECT`
	- `WITH_CREDENTIALS`
	- `APP_ENV`
	- `APP_VERSION`
- 导出方法：
	- `getRuntimeConfig()`
	- `getBaseUrl()`
	- `shouldUseCredentials()`
	- `isAbsoluteUrl()`
	- `withBaseUrl()`
- `shouldUseCredentials()` 需要支持字符串布尔值解析：
	- 真值：`1` / `true` / `yes` / `on`
	- 假值：`0` / `false` / `no` / `off`
	- 默认值：`false`

### 验证与联调要求

- 修复 `../nuwax/src/utils/runtimeConfig.ts` 后，在真实仓库目录执行：`pnpm build:prod`。
- 若以前端单容器跨域访问后端为目标，运行时至少需要关注：
	- `BASE_URL=https://实际后端域名`
	- `WITH_CREDENTIALS=true`
- 后端还需要同时满足：
	- CORS 允许前端精确域名，不能使用 `*`
	- 允许携带凭证（credentials）
	- 跨站 Cookie 需满足 `SameSite=None; Secure`（通常要求 HTTPS）
	- SSO / CAS 回调地址、站点域名与前端实际访问域保持一致
