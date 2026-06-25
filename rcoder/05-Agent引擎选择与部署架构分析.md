# rcoder Agent 引擎选择与部署架构分析

生成时间：2026-06-25

## 1. 前端两个创建入口的区别

### `/space/:spaceId/page-develop`（网页应用开发）

- 页面组件：`SpacePageDevelop`
- 创建方式：导入项目 / 在线开发 / 反向代理
- 发布类型：`Page`
- 后端接口：`POST /api/custom-page/create`
- 后端模块：`app-platform-custom-page`
- 运行方式：`nuwax-file-server` 管理 dev server，发布后由 nginx 托管静态文件
- **不需要 rcoder 沙箱**

### `/space/:spaceId/develop`（智能体开发）

- 页面组件：`SpaceDevelop`
- 创建方式：选择智能体类型（ChatBot 问答型 / TaskAgent 通用型）
- 发布类型：`Agent`
- 后端接口：智能体 CRUD 相关接口
- 后端模块：`app-platform-agent`

**两种智能体类型的区别**：

| 类型 | 枚举值 | 是否用 rcoder | 说明 |
|------|--------|--------------|------|
| ChatBot（问答型） | `AgentTypeEnum.ChatBot` | ❌ | 直接调用 LLM API，后端通过 AnthropicApi / OpenAiApi 等模型客户端直接对话 |
| TaskAgent（通用型） | `AgentTypeEnum.TaskAgent` | ✅ | 通过 `SandboxAgentClient` 调用 rcoder 的 `/computer/chat`，在 Docker 容器内执行代码/文件/浏览器操作 |

### 为什么网页应用不需要沙箱？

TaskAgent 需要沙箱是因为 AI 要**执行代码**（运行脚本、操作文件系统、操控浏览器）。

网页应用是**静态前端项目**（React/Vite/HTML），构建后直接由 nginx 托管，没有运行时代码执行需求。它的"沙箱"是 `nuwax-file-server`（Node.js 文件服务），只负责项目文件管理和 dev server 启动，不涉及 AI 代码执行。

## 2. Agent 引擎选择机制

### 当前状态

rcoder 的 `default_agents.json` 中**只注册了一个 Agent**：

```json
{
  "agent_servers": {
    "claude-code-acp-ts": {
      "agent_id": "claude-code-acp-ts",
      "agent_type": "claude",
      "command": "claude-code-acp-ts",
      "env": {
        "ANTHROPIC_API_KEY": "{MODEL_PROVIDER_API_KEY}",
        "ANTHROPIC_MODEL": "{MODEL_PROVIDER_DEFAULT_MODEL}",
        "ANTHROPIC_BASE_URL": "{MODEL_PROVIDER_BASE_URL}"
      }
    }
  }
}
```

虽然镜像中安装了多个 CLI，但**只有 `claude-code-acp-ts` 被注册使用**：

| CLI 工具 | 镜像中是否有 | 是否注册 | 说明 |
|----------|------------|---------|------|
| `claude-code-acp-ts` | ✅ | ✅ | 唯一注册的 Agent |
| `@anthropic-ai/claude-code` | ✅ | ❌ | 未在 default_agents.json 中注册 |
| `@openai/codex` | ✅ | ❌ | 未注册 |
| `nuwaxcode` | ✅ | ❌ | 未注册 |

### 选择优先级

```
1. API 请求参数：POST /chat 或 POST /computer/chat 中的 agent_server.agent_id
2. config.yml：default_agent_id 字段（默认 claude-code-acp-ts）
3. 编译时嵌入：default_agents.json 中的硬编码默认值
```

关键代码：[`PromptConfigAssembler.get_agent_id()`](../../vs_code_nuwax/rcoder/crates/agent_config/src/config/prompt_assembler.rs:204)

### 优化建议

当前 Agent 引擎选择存在以下问题：

1. **镜像中安装了多个 CLI 但只用一个**：`@anthropic-ai/claude-code`、`@openai/codex`、`nuwaxcode` 都装了但没注册，增加了镜像体积但没有实际用途。

2. **Agent 切换需要改配置文件**：如果要切换到 OpenAI Codex 或 nuwaxcode，需要手动修改 `default_agents.json` 并重新编译，不够灵活。

3. **缺少运行时动态选择**：无法通过 API 请求或环境变量动态切换 Agent 引擎。

**建议优化方向**：
- 在 `default_agents.json` 中注册所有可用的 Agent（claude-code-acp-ts、codex、nuwaxcode）
- 支持通过 `config.yml` 的 `default_agent_id` 或环境变量选择默认引擎
- 支持在 API 请求中指定 `agent_id` 动态切换（已有框架支持，只需补充配置）
- 考虑按需安装 CLI，减小镜像体积

## 3. rcoder 部署架构

### 两种部署方式

`nuwax_deploy` 和 `nuwax_computer_deploy` 中的 rcoder 是**同一个东西**，只是两种部署方式：

| 维度 | nuwax_deploy（单机模式） | nuwax_computer_deploy（分离模式） |
|------|------------------------|-------------------------------|
| 包含服务 | frontend + backend + 全套中间件 + rcoder | 只有 rcoder |
| rcoder 端口 | 8086/8088/8099/60000 | 9086/9088/9099/60001 |
| 网络名 | agent-network | agent-compouter-network |
| 适用场景 | 开发/测试/小规模生产 | 生产环境/大规模/需要 GPU |

### 分离部署的原因

1. **资源隔离**：rcoder 为每个用户创建 Docker 容器（2-4GB 内存），大量并发时会抢占主服务资源
2. **独立扩缩容**：沙箱机器可单独扩容（CPU/内存/GPU）
3. **安全隔离**：rcoder 挂载 Docker socket，独立机器可避免沙箱被攻破影响主服务
4. **GPU 支持**：AI Agent 需要 GPU 时可单独配备

### 架构图

```
模式一：单机部署
┌─────────────────────────────────┐
│  一台机器                         │
│  frontend + backend + rcoder     │
│  + MySQL + Redis + Milvus + ...  │
└─────────────────────────────────┘

模式二：分离部署
┌──────────────────────┐     ┌──────────────────────┐
│  主服务机器            │     │  沙箱专用机器          │
│  frontend + backend   │────▶│  rcoder              │
│  + MySQL + Redis      │ HTTP│  + Agent 容器         │
│  + Milvus + mcp-proxy │     │  (高配 CPU/内存/GPU)  │
└──────────────────────┘     └──────────────────────┘
```

## 4. rcoder 子容器内容

rcoder 启动的子容器有两种类型：

### 普通 Agent 子容器

- 镜像：`master-rcoder:latest`（与 rcoder 主容器相同）
- 入口：`/app/bin/agent_runner --port 8086`
- 资源限制：2GB 内存 / 2 CPU / 4GB swap

### Computer Agent 子容器

- 镜像：`rcoder-agent-runner:latest`（独立镜像）
- 入口：`/usr/local/bin/agent_runner --port 8086`
- 资源限制：4GB 内存 / 2 CPU / 6GB swap

**Computer Agent 子容器内含**：

| 组件 | 说明 |
|------|------|
| `agent_runner` | Rust gRPC 服务端 |
| `nuwaxcode` | AI 代码引擎 |
| `claude-code-acp-ts` | Claude Code ACP 适配器 |
| `mcp-stdio-proxy` | MCP 工具代理 |
| `agent-browser` | 浏览器自动化 |
| Chromium | 浏览器 |
| noVNC | 远程桌面（端口 6080） |
| `audio_server.py` | 音频流（端口 6089/6090） |
| `ime_server.py` | 输入法代理（端口 6091） |
| XFCE 桌面 | Linux 桌面环境 |
