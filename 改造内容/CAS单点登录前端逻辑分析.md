# CAS 单点登录前端逻辑分析

## 1. 概述

前端对 CAS 的处理完全是**被动接收式**的——前端不主动发起 CAS 登录，也不解析 CAS 票据。CAS 的配置由后端管理，前端仅从租户配置接口获取 CAS 相关字段，并在 API 返回 `4011` 错误码时执行重定向。

---

## 2. 类型定义

**文件：** `types/interfaces/login.ts`

`TenantConfigInfo` 中与 CAS 相关的字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `casClientHostUrl` | `string` | CAS 客户端主机 URL |
| `casValidateUrl` | `string` | CAS ticket 验证 URL |
| `casLoginUrl` | `string` | CAS 登录 URL |
| `authType` | `number` | 认证类型（1=手机, 3=邮箱） |
| `casUserAttributesConfig?` | `string` | CAS 用户属性映射 JSON，例如 `{"uid":"cas:user","userName":"cas:edu_userName","nickName":"cas:edu_name","phone":"cas:edu_phone","email":"cas:edu_email"}` |

这些字段通过 `GET /api/tenant/config` 获取，但**前端代码中从未直接使用 `casClientHostUrl`、`casValidateUrl`、`casLoginUrl` 发起 CAS 请求**，仅使用 `authType` 区分登录表单使用邮箱还是手机号。

---

## 3. CAS 重定向的触发机制

### 3.1 错误码定义

**文件：** `constants/codes.constants.ts`

| 常量 | 值 | 含义 |
|------|-----|------|
| `USER_NO_LOGIN` | `4010` | 用户未登录 → 跳转内部登录页 `/login` |
| `REDIRECT_LOGIN` | `4011` | 重定向到外部登录（CAS） |

### 3.2 全局拦截器

**文件：** `services/common.ts:150-159`

每次 API 调用的响应拦截器中：

```typescript
case USER_NO_LOGIN:
  clearLoginStatusCache();
  redirectToLogin(-1);  // 跳转到内部登录页
  break;

case REDIRECT_LOGIN:
  clearLoginStatusCache();
  redirectToExternalLogin(errorMessage);  // 重定向到 CAS 服务器
  break;
```

`errorMessage` 是后端返回的错误消息，内容为**完整的 CAS 登录 URL**（如 `https://cas.example.com/login?service=...`）。

### 3.3 用户信息获取拦截

**文件：** `services/userService.ts:93-103`

`fetchUserInfoFromServer()` 调用 `/api/user/getLoginInfo` 时同样处理 `4010` / `4011`：

```typescript
case USER_NO_LOGIN:
  clearLoginStatusCache();
  redirectToLogin(-1);
  break;

case REDIRECT_LOGIN:
  clearLoginStatusCache();
  redirectToExternalLogin(errorMessage);
  break;
```

---

## 4. 重定向执行逻辑

**文件：** `utils/authRedirect.ts`

### 4.1 `redirectToExternalLogin(url)` — 核心重定向函数

```typescript
export const redirectToExternalLogin = (url: string) => {
  const resolvedTarget = resolveTargetUrl(url);
  if (!resolvedTarget) return;

  const record = readRedirectRecord();
  const now = Date.now();

  // 5 秒防抖：避免 CAS 回调后反复重定向
  if (record && now - record.time < 5000) return;

  writeRedirectRecord(resolvedTarget);
  window.location.href = resolvedTarget;
};
```

### 4.2 `resolveTargetUrl(url)` — URL 解析

- 如果 `url` 以 `http://` 或 `https://` 开头，视为完整 URL 直接使用
- 否则视为相对路径，拼接 `window.location.origin`
- 如果目标 URL 与当前页面相同，返回 `null`（防止自我重定向死循环）

### 4.3 防抖机制

使用 `sessionStorage` 存储上次重定向记录（URL + 时间戳）：

- `readRedirectRecord()` — 读取 `sessionStorage` 中的重定向记录
- `writeRedirectRecord(url)` — 写入 `sessionStorage`，5 秒内禁止重复重定向

这防止了 CAS 服务器回调到后端、后端返回 `4011`、前端再次重定向到 CAS 的无限循环。

---

## 5. CAS 完整数据流

```
用户未登录/会话过期
    ↓
前端 API 请求 → 后端返回 4011 (REDIRECT_LOGIN)
    ↓
services/common.ts 或 userService.ts 拦截错误码
    ↓
clearLoginStatusCache() — 清除本地登录缓存
    ↓
redirectToExternalLogin(errorMessage)
    ↓
authRedirect.ts:
  1. resolveTargetUrl(url) — 解析目标 URL
  2. readRedirectRecord() — 检查 5 秒内是否已重定向
  3. writeRedirectRecord(url) — 写入防抖记录
  4. window.location.href = CAS 服务器 URL
    ↓
浏览器跳转到 CAS 服务器登录页
    ↓
用户在 CAS 服务器完成身份认证
    ↓
CAS 服务器回调后端（服务器端处理，前端不参与）
    ↓
后端验证 ticket 并生成 token
    ↓
后端重定向回前端首页 → 前端恢复正常使用
```

**关键点：前端只在第 2-4 步参与 CAS 登录流程**，即接收到 `4011` 错误时被动跳转到 CAS 服务器。

---

## 6. 认证路由守卫

**文件：** `wrappers/authWithLoading.tsx`

- 所有受保护路由包裹了 `authWithLoading` 守卫
- 页面加载时调用 `UserService.fetchUserInfoFromServer(false)` 验证登录状态
- 登录失败（未登录）时调用 `redirectToLogin('-1')` 跳转内部登录页 `/login`
- 如果后端配置了 CAS，`/login` 页面加载后会返回 `4011`，触发上述重定向流程

**文件：** `utils/router.ts:106-113`

```typescript
export const redirectToLogin = (redirect: string | number) => {
  window.location.href = `/login?redirect=${encodeURIComponent(pathName)}`;
};
```

---

## 7. 退出登录

**文件：** `layouts/DynamicMenusLayout/User/index.tsx:46-65`

退出登录后调用 `redirectToLogin(currentPath)` 跳转到 `/login`。若后端配置了 CAS，此时 `/login` 页面初始化时会触发 `4011`，自动继续跳转到 CAS 登录页。

---

## 8. 与现有认证系统的关系

```
authType 字段（来自 /api/tenant/config）:
  ┌─ 1: 手机号认证（短信验证码）
  ├─ 3: 邮箱认证（邮件验证码）
  └─ 其他: 默认同 1（手机号）
```

登录页根据 `authType` 切换手机/邮箱输入框（`pages/Login/index.tsx:194`）。**CAS 不经过登录页 UI**，而是通过 `4011` 错误码绕过登录页直接跳转到外部 CAS 服务器。

---

## 9. 关键文件索引

| 文件 | 作用 |
|------|------|
| `types/interfaces/login.ts` | `TenantConfigInfo` 中 CAS 字段类型定义 |
| `constants/codes.constants.ts` | `4011` 错误码常量 `REDIRECT_LOGIN` |
| `services/common.ts` | API 响应全局拦截器，处理 `4011` 重定向 |
| `services/userService.ts` | 用户信息请求拦截器，处理 `4011` 重定向 |
| `utils/authRedirect.ts` | CAS 重定向执行逻辑（URL 解析 + 防抖） |
| `utils/router.ts` | `redirectToLogin()` / `redirectTo()` 工具函数 |
| `wrappers/authWithLoading.tsx` | 路由守卫，验证登录态 + 触发重定向 |
| `models/tenantConfigInfo.ts` | 获取租户配置（含 CAS 字段） |
| `services/account.ts` | `apiTenantConfig()` CAS 相关配置获取 |
| `pages/Login/index.tsx` | 登录页，仅使用 `authType` 区分邮箱/手机 |
| `constants/theme.constants.ts` | `AUTH_TYPE` localStorage 键名 |

---

## 10. 总结

- 前端**不处理 CAS 票据验证**，所有 ticket 验证由后端完成。
- 前端**不渲染 CAS 配置 UI**，CAS 配置字段仅存在于类型定义中，由后端管理。
- CAS 登录流程是 **4011 错误码驱动的重定向**，前端唯一的行为是：收到 `4011` → 清除缓存 → `window.location.href` 跳转到后端返回的 CAS 登录 URL。
- 5 秒防抖机制防止 CAS 回调后重定向死循环。
