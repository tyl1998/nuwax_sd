# rcoder 容器内服务清单

生成时间：2026-06-25

## 1. rcoder 主容器（master-rcoder）内含的服务

rcoder 主容器是一个"重量级"容器，内含多个服务和工具链：

### 核心 Rust 二进制

| 二进制 | 路径 | 说明 |
|--------|------|------|
| `rcoder` | `/app/bin/rcoder` | 主服务（HTTP API + gRPC 客户端 + Docker 容器管理） |
| `agent_runner` | `/app/bin/agent_runner` | Agent Runner（gRPC 服务端，随子容器一起运行） |

### npm 全局工具

| 工具 | 安装方式 | 说明 |
|------|---------|------|
| `claude-code-acp-ts` | `npm install -g claude-code-acp-ts@latest` | Claude Code ACP TypeScript 适配器（默认 Agent 引擎） |
| `nuwaxcode` | `npm i -g nuwaxcode@latest` | 自研 AI 代码引擎 |
| `nuwax-file-server` | `npm i -g nuwax-file-server@latest` | 文件服务（管理网页应用项目的 dev server、构建、版本） |
| `@anthropic-ai/claude-code` | `npm install -g @anthropic-ai/claude-code@latest` | Anthropic 原版 Claude Code CLI |
| `@openai/codex` | `npm install -g @openai/codex@latest` | OpenAI Codex CLI |
| `@zed-industries/claude-code-acp` | `npm install -g @zed-industries/claude-code-acp@latest` | Zed 的 Claude Code ACP 适配 |
| `chrome-devtools-mcp` | `npm install -g chrome-devtools-mcp@latest` | Chrome DevTools MCP |
| `vite` | `npm install -g vite@latest` | 前端构建工具 |
| `pnpm` | `npm install -g pnpm` | 包管理器 |

### Python 工具

| 工具 | 说明 |
|------|------|
| `uv` | Python 包管理器（pip 安装） |

### Bun 运行时

| 工具 | 说明 |
|------|------|
| `bun` | JavaScript 运行时（npm 安装） |
| `@upstash/context7-mcp` | Context7 MCP 工具（bun 全局安装） |

### 系统服务

| 服务 | 说明 |
|------|------|
| `nginx` | 反向代理 + 静态文件托管（网页应用项目的预览和发布） |
| Chromium | 浏览器（Dockerfile.base 中安装） |

### MCP 工具（预装）

| 工具 | 安装方式 | 说明 |
|------|---------|------|
| `mcp-server-fetch` | `uv tool install` | HTTP 抓取 MCP 工具 |
| `@upstash/context7-mcp` | `bun add -g` | Context7 文档检索 MCP |

### Claude Code 配置

`/root/.claude/settings.json` 中预配置了：
- 禁用 `WebFetch`、`WebSearch` 权限
- 启用 `mcp-fetch`（uvx mcp-server-fetch）
- 启用 `chrome-devtools`（npx chrome-devtools-mcp）

### nuwaxcode 配置

`/root/.config/opencode/opencode.json` 中预配置了：
- 禁用 `websearch`、`webfetch`、`question` 权限

## 2. Agent 子容器（rcoder-agent-runner）内含的服务

### 核心二进制

| 二进制 | 路径 | 说明 |
|--------|------|------|
| `agent_runner` | `/usr/local/bin/agent_runner` | gRPC 服务端（接收 rcoder 下发的 AI 任务） |

### npm 全局工具

| 工具 | 说明 |
|------|------|
| `claude-code-acp-ts` | Claude Code ACP 适配器 |
| `nuwaxcode` | AI 代码引擎 |

### Rust 编译工具

| 工具 | 说明 |
|------|------|
| `mcp-stdio-proxy` | MCP 工具代理（cargo install） |
| `agent-browser` | 浏览器自动化 CLI（cargo install） |

### 桌面与远程访问

| 组件 | 端口 | 说明 |
|------|------|------|
| XFCE 桌面 | — | Linux 桌面环境（壁纸、任务栏） |
| noVNC | 6080 | 远程桌面 Web 端 |
| `audio_server.py` | 6089/6090 | 音频流服务 |
| `ime_server.py` | 6091 | 输入法代理服务 |
| Chromium | — | 浏览器（用于 AI 操控网页） |
| `fix-ime` | — | 中文输入法修复脚本 |

### 可选诊断工具（开发模式）

| 工具 | 说明 |
|------|------|
| `bpftrace` | eBPF 诊断工具 |
| `strace` | 系统调用追踪 |
| `sysstat` | 性能监控（iostat, mpstat） |
| `FlameGraph` | 火焰图工具 |
| `bpfcc-tools` | Off-CPU 阻塞分析 |
| Grafana Alloy | 持续性能数据采集 |

## 3. 端口分配

### rcoder 主容器

| 端口 | 服务 | 说明 |
|------|------|------|
| 8086 | rcoder HTTP API | Chat、Progress、Pod 管理 |
| 8088 | Pingora 反向代理 | VNC/音频/IME 透传 |
| 60000 | nuwax-file-server | 文件构建服务 |
| 80 | nginx | 网页应用预览/发布 |

### Agent 子容器

| 端口 | 服务 | 说明 |
|------|------|------|
| 8086 | agent_runner gRPC | AI 任务接收 |
| 6080 | noVNC | 远程桌面 |
| 6089 | audio_server | 音频流（WebSocket） |
| 6090 | audio_server | 音频流（HTTP） |
| 6091 | ime_server | 输入法代理 |

## 4. 一句话总结

rcoder 主容器 = **Rust 主服务 + AI Agent 引擎全家桶 + 文件服务 + nginx + 浏览器 + 包管理器**，是一个"什么都有的"重量级容器。Agent 子容器 = **轻量级 AI 桌面工作环境**（Agent Runner + VNC + 音频 + 输入法 + 浏览器 + 桌面）。
