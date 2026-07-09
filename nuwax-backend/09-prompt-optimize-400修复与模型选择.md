# backend 改动记录：prompt/optimize 400 修复 + 模型选择

- **改动仓库（真实源码）**：`../nuwax-backend`
- **改动日期**：2026-07-09
- **涉及模块**：`app-platform-agent`（core-ui、core-infra）
- **触发问题**：前端调用 `POST /api/assistant/prompt/optimize` 收到 HTTP 400

---

## 1. 问题根因

后端日志暴露两层原因，第一层是根因，第二层把它“放大”成了前端看到的 400。

### 1.1 根因：上游模型代理返回 400

```
org.springframework.web.reactive.function.client.WebClientResponseException
  400 BAD_REQUEST from POST https://openai-api.nuwax.com/v1/chat/completions [DefaultWebClient]
```

`/v1/chat/completions` 的 URL 链路（`AssistantController` → `ModelInvoker` → `ModelClientFactory`）：

```
ModelConfigDto.apiInfoList              // 数据库 nt_model_config.api_info 中的 [{url,key,weight}]
  → weightedRoundRobinStrategy.selectApi(model)   // 加权轮询选一个 endpoint
  → modelApiProxyRpcService.generateUserFrontendModelConfig(...)  // 调模型代理，生成前端 URL
  → frontendModelDto.getBaseUrl()       // 实际用到的 URL：openai-api.nuwax.com（模型代理，非厂商原始 API）
  → completeBaseUrl(url)                // 补 /v1
  → new OpenAiApi(url, ..., "/chat/completions")
```

> `openai-api.nuwax.com` 是**模型代理服务**，负责鉴权/限流/计费/路由转发，不等于模型厂商原始 API。代理层按当前租户生成 API Key 与 URL。

该接口走的是 `modelApplicationService.queryDefaultModelConfig()` 取得的**租户默认对话模型**。默认模型配置异常（API Key 失效、模型名不匹配、参数越界、prompt 超长等）会导致模型代理拒绝请求。

### 1.2 衍生问题：SSE 错误无法被序列化 → HttpMessageNotWritableException

模型调用失败经 `ModelInvoker.doOnError` 的 `sink.tryEmitError(throwable)` 传播到 Spring MVC，被全局 `AppExceptionHandler` 捕获并返回 `ReqResult`（一个 `LinkedHashMap`）。但该接口声明了 `produces = "text/event-stream"`，响应 Content-Type 已被锁为 `text/event-stream`，而 `ReqResult` 无法用 SSE 消息转换器序列化：

```
org.springframework.http.converter.HttpMessageNotWritableException:
  No converter for [class java.util.LinkedHashMap]
  with preset Content-Type 'text/event-stream;charset=utf-8'
        at ...AbstractMessageConverterMethodProcessor.writeWithMessageConverters
```

最终前端看到的 HTTP 400 实由此异常产生，而非模型代理的 400 直接透传。

### 1.3 prompt/optimize 与智能体对话的区别（同一 URL，不同模型上下文）

| 维度 | prompt/optimize（`buildModelContext`） | 智能体对话（`AgentExecutor`） |
|------|----------------------------------------|------------------------------|
| 模型来源 | `queryDefaultModelConfig()` 租户默认模型 | 智能体自身配置的 `modelId` |
| System Prompt | `OPTIMIZE_PROMPT` 固定模板 | 智能体配置的系统提示词 |
| 工具/插件 | 无（`functionCallbacks=[]`） | 有 Plugin/Workflow/Table/MCP 等回调 |
| Agent Context | 最小化（仅 userId/user） | 完整（agentConfig、componentConfigs、变量等） |
| 对话历史 | 无，每轮独立 | 有 `contextMessages` |
| Temperature/TopP | 硬编码 1.0 / 0.7 | 从 ModelConfig 读 |
| 对话标识 | `optimize:` + requestId | `agent:` + conversationId |

**关键差异**：prompt/optimize 用的是默认模型，智能体对话用的是智能体专属模型。若默认模型配置有问题，会出现“智能体对话正常、但 optimize 接口 400”的现象。

---

## 2. 代码改动

### 2.1 修复 1：SSE 接口错误信号吞掉（`AssistantController`）

三个 SSE 接口（`promptOptimize` / `codeOptimize` / `sqlOptimize`）在 `modelInvoker.invoke(...)` 后追加 `.onErrorResume(e -> Flux.empty())`。

`doOnError` 已经通过 `CallMessage`（`finishReason="ERROR"`）把错误文本发给前端，因此这里只需吞掉 error 信号让流正常结束，不再触发 SSE 序列化失败。

```java
// 改前
return modelInvoker.invoke(modelContext);
// 改后
return modelInvoker.invoke(modelContext).onErrorResume(e -> Flux.empty());
```

> 不影响 `AgentExecutor` 等其它调用方：它直接 `subscribe` 并通过 `doOnError`/error handler 处理异常，不经 SSE 序列化路径。

### 2.2 修复 2：`doOnError` 提取模型代理响应体（`ModelInvoker`）

从 `WebClientResponseException` 提取响应体，让前端/日志拿到模型代理返回的具体拒绝原因（便于定位 1.1 的真实错误）。

```java
// 新增 import
import org.springframework.web.reactive.function.client.WebClientResponseException;

// doOnError 内
String errorText = throwable.getMessage();
if (throwable instanceof WebClientResponseException wcre) {
    String body = wcre.getResponseBodyAsString();
    if (StringUtils.isNotBlank(body)) {
        errorText = errorText + "\n" + body;
    }
}
callMessage.setText(errorText);   // 原为 throwable.getMessage()
```

### 2.3 增强：支持模型选择（可选 `modelId`）

三个 DTO 新增可选字段 `modelId`；`buildModelContext` 在传入时使用 `queryModelConfigById`，否则回退默认模型；模型不存在抛 `agentModelNotFound`（9351）。

**DTO 改动**（均新增）：
```java
@Schema(description = "模型ID，可选，不传则使用租户默认对话模型")
private Long modelId;
```
涉及：`OptimizeDto`、`CodeOptimizeDto`、`SQLOptimizeDto`。

**`buildModelContext` 改动**：
```java
private ModelContext buildModelContext(String convId, String systemPrompt, String msg, Long modelId) {
    ModelConfigDto modelConfigDto = modelId != null
            ? modelApplicationService.queryModelConfigById(modelId)
            : modelApplicationService.queryDefaultModelConfig();
    if (modelConfigDto == null) {
        throw BizException.of(ErrorCodeEnum.INVALID_PARAM, BizExceptionCodeEnum.agentModelNotFound);
    }
    // ... 其余逻辑不变
}
```

**调用处**均改为传入 `dto.getModelId()`：
```java
buildModelContext(..., promptOptimizeDto.getModelId());
buildModelContext(..., codeOptimizeDto.getModelId());
buildModelContext(..., sqlOptimizeDto.getModelId());
```

### 2.4 前端调用示例

```json
{
  "requestId": "36dc7a66-fa4c-4f30-a5c1-44bcd70c5e04",
  "prompt": "你是S4系统的测试工程师……",
  "type": "AGENT",
  "id": 24,
  "modelId": 123
}
```

`modelId` 不传 → 走租户默认模型；传入 → 用指定模型。可作为默认模型配置异常时的临时规避手段。

---

## 3. 改动文件清单

| 文件 | 改动类型 |
|------|----------|
| `app-platform-modules/app-platform-agent/app-platform-agent-core-ui/.../controller/AssistantController.java` | SSE 错误吞掉 + `buildModelContext` 支持 modelId |
| `app-platform-modules/app-platform-agent/app-platform-agent-core-infra/.../component/model/ModelInvoker.java` | `doOnError` 提取模型代理响应体 |
| `.../controller/dto/OptimizeDto.java` | 新增 `modelId` |
| `.../controller/dto/CodeOptimizeDto.java` | 新增 `modelId` |
| `.../controller/dto/SQLOptimizeDto.java` | 新增 `modelId` |

---

## 4. 待办 / 注意事项

1. **默认模型配置问题需运维修复**：若 `nt_model_config` 中租户默认模型的 `api_info`（URL/Key）或 `model` 字段错误，仍会 400。新增 `modelId` 仅能规避，根因在数据库配置。
2. **跨租户 `modelId` 风险低**：`queryModelConfigById` 不做租户归属校验，但模型代理 `generateUserFrontendModelConfig` 按当前租户生成 API Key/URL（按租户计费），越权影响有限。如需严格，可在 `buildModelContext` 增加 `modelConfigDto.getTenantId()` 与当前租户一致性校验。

---

## 5. 前端改造：优化弹窗增加模型选择

后端 `modelId` 已就绪（见 2.3），前端在 **prompt 优化** 与 **code 优化** 两个弹窗的调用处接入模型选择，不传则继续使用租户默认模型，传入则走指定模型（与后端 `queryModelConfigById` 对齐）。

### 5.1 改动位置

| 文件 | 改动 |
|------|------|
| `nuwax/src/types/interfaces/assistant.ts` | `PromptOptimizeParams`、`CodeCreateParams` 新增可选 `modelId?: number` |
| `nuwax/src/components/OptimizeModelSelector/index.tsx` | **新增组件**：轻量模型下拉，数据源为当前用户有权限的模型（`/api/model/my`，System + Space 合并去重），首选项为「租户默认模型」(value=undefined)，仅选择、不含模型增删改 |
| `nuwax/src/components/PromptOptimizeModal/index.tsx` | 接入 `OptimizeModelSelector`，状态 `modelId`，拼入 `PromptOptimizeParams.modelId`（发送 / 回车两处） |
| `nuwax/src/components/CodeOptimizeModal/index.tsx` | 同上，拼入 `CodeCreateParams.modelId`（发送 / 回车两处） |
| `nuwax/src/components/PromptOptimizeModal/index.less` / `CodeOptimizeModal/index.less` | 新增 `.model-select` 布局样式 |
| `nuwax/src/locales/i18n/{zh-CN,zh-TW,zh-HK,en-US,ja-JP}.ts` | 新增 `PC.Components.OptimizeModelSelector.defaultModel` / `...selectModel` |

### 5.2 交互与数据流

```
OptimizeModelSelector（/api/model/my: System+Space 合并去重）
   ↓ 选中（首选项=租户默认模型 → modelId=undefined）
PromptOptimizeModal / CodeOptimizeModal
   ↓ params.modelId
assistantOptimize model → createSSEConnection(PROMPT_OPTIMIZE_URL/CODE_OPTIMIZE_URL, body: params)
   ↓
后端 buildModelContext(modelId) → queryModelConfigById 或 queryDefaultModelConfig
```

- 默认项 `value=undefined` 经 SSE body 透传：后端 `modelId == null` 走 `queryDefaultModelConfig()`，行为与改造前一致（向后兼容）。
- 选中具体模型：`modelId` 带入，后端 `queryModelConfigById(modelId)`。

### 5.3 注意事项

1. **数据源对齐**：复用 `/api/model/my`（系统模型 + 个人空间模型），只展示当前用户/租户可见模型，规避文档 4.2 的跨租户越权问题；与智能体对话 `ModelSelector`（`/api/agent/conversation/model/options/{agentId}`）走的是不同接口，二者模型集合可能不同，属预期。
2. **sql 优化弹窗未接入**：`SqlOptimizeModal` 本次未加选择器（用户仅要求 prompt/code），但后端 `SQLOptimizeDto` 同样支持 `modelId`，后续可按需补。
3. **默认值渲染**：antd `Select` 在 `value` 为 `undefined` 时不显示默认项，故组件内用哨兵值 `__default__` 表示「租户默认模型」，回传时再映射为 `undefined`。
4. **未改请求底座**：仍走 `src/services/common.ts` 统一封装与 `createSSEConnection`，仅 body 多了可选 `modelId` 字段。

