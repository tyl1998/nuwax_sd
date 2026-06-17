# nuwax-file-server：路径隔离与 Computer 工作空间

两个独立但相关的主题：普通项目的多租户路径隔离，以及 Computer Agent 专用的技能/Agent 工作空间管理。

## 1. 普通项目路径隔离（projectPathUtils.js）

所有涉及项目文件的接口都经过 `extractIsolationContext` + `resolveProjectPath` 两步构造实际路径。

```
请求 query 参数
  { tenantId, spaceId, isolationType, projectId }
            ↓ extractIsolationContext()
  isolationContext = { tenantId, spaceId, isolationType }
            ↓ resolveProjectPath(projectId, isolationContext)
            ↓
  三字段都非空？
    是 → PROJECT_SOURCE_DIR/{tenantId}/{spaceId}/{projectId}   ← 多租户隔离路径
    否 → PROJECT_SOURCE_DIR/{projectId}                        ← 默认路径
```

隔离路径的判断条件：`tenantId && spaceId && isolationType` 三者同时非空才启用，任一为空就退回默认的扁平结构。这让同一套代码既支持 SaaS 多租户部署（不同租户的项目文件物理隔离），也支持单租户/本地部署（直接 `projectId` 作目录名）。

`normalizeValue` 对所有字段做 `String().trim()`，防止 `undefined`/`null`/空白字符串绕过判断进入路径拼接。

## 2. Computer 工作空间目录结构

Computer Agent 的工作空间与普通项目完全独立，根目录由 `COMPUTER_WORKSPACE_DIR` 环境变量指定：

```
$COMPUTER_WORKSPACE_DIR/
└── {userId}/
    └── {cId}/              ← cId 是 session/conversation id
        ├── skills/          ← Agent 可调用的技能目录
        │   ├── skill-a/     ← 普通技能（可被 createWorkspace 覆盖）
        │   └── skill-b/
        │       └── .dynamic_add.lock  ← 有此文件 → 动态添加的，createWorkspace 时保留
        ├── agents/          ← Agent 定义目录
        └── .tmp/            ← 临时目录（操作完成后清理）
```

`userId` 对应 rcoder 的 Computer Agent 模式（每 user 一个容器），`cId` 对应具体的 conversation。

## 3. createWorkspace 流程

```
createWorkspace(userId, cId, file?, skillUrls?)
  │
  ├── ensureWorkspaceRoot()：确保 COMPUTER_WORKSPACE_DIR 存在
  ├── ensurePrimaryAgentDirs()：确保 skills/ agents/ 子目录存在
  │
  ├── 清理旧 skills/：
  │     含 .dynamic_add.lock 的子目录 → 先 mv 到 .tmp/preserved_*
  │     rm -rf targetSkillsDir
  │     mkdir targetSkillsDir
  │     把 preserved_* 中的目录 mv 回来    ← 动态添加的技能跨次保留
  │
  ├── rm -rf targetAgentsDir → mkdir targetAgentsDir
  │
  ├── 有 file（zip）？
  │     extractZip → findDir("skills") → 逐个 skill 目录 moveDirectory
  │     extractZip → findDir("agents") → moveDirectory
  │
  ├── 有 skillUrls[]？
  │     foreach url:
  │       downloadUrlToFile → extractZip
  │       识别 zip 结构：skills/<skillName>/ 或直接 <skillName>/
  │       逐个 skill moveDirectory → targetSkillsDir
  │
  ├── syncAgents(userWorkspaceRoot)   ← 同步 agents 配置
  │
  └── finally: rm -rf .tmp 下所有临时目录和下载的 zip
```

**`.dynamic_add.lock` 机制**：`push-skills-to-workspace` 接口动态推送技能时给技能目录写入此锁文件，标记"这个技能是运行时动态追加的"。`createWorkspace` 重建工作空间时只删没有锁文件的技能，有锁的先 mv 暂存再 mv 回来，保证动态追加的技能不被整体重建冲掉。

## 4. pushSkillsToWorkspace vs createWorkspace

| 维度 | createWorkspace | pushSkillsToWorkspace |
|------|-----------------|-----------------------|
| 语义 | **重建**工作空间 | **增量推送**技能 |
| skills 处理 | 先清空（保留锁文件技能），再写入 | 有同名 → 覆盖，没有 → 新增，不删已有 |
| agents 处理 | 清空 + 写入 | 不处理 |
| 适用时机 | 新建 conversation / 切换技能集 | 运行时动态添加单个技能 |

## 5. moveDirectory 跨设备兼容

```js
fs.promises.rename(src, dest)
  // 失败且 err.code === "EXDEV"（跨设备/挂载点）
  →  copyRecursive(src, dest) + rm -rf src
```

容器内 `.tmp` 和 `skills` 目录可能在不同挂载点（tmpfs vs 宿主机卷），`rename` 会报 EXDEV。自动回退到 copy+delete 保证跨挂载点移动正确。Windows 环境下 `rename` 到已存在目录会报 `EPERM`，调用方在移动前先 `removeDirIfExists` 规避。

## 6. 框架检测（frameworkDetectorUtils.js）

启动 dev server 前和生成 build 参数时用到，从 `package.json` 和配置文件推断框架类型：

```
detectFrontendFramework(projectPath)
  → 读 package.json dependencies + devDependencies
  → 有 react/react-dom       → "react"
  → 有 vue/vue-router        → "vue{major}"（解析 semver 主版本）
  → 否则                     → "other"

detectDevFramework(projectPath)
  → 存在 next.config.{js,ts,mjs,cjs}   → "nextjs"（优先）
  → 存在 vite.config.{js,ts,mjs,cjs}   → "vite"
  → 否则                                → "other"
```

Vue 版本解析支持多种 semver 变体（`^3.4.0`、`~2.7`、`npm:vue@^3`、`2.x`），用正则 `/(?:^|[^\d])v?(\d+)(?:\.|x|\b)/` 提取主版本号。结果用于 `extraArgsUtils` 生成不同框架的 `--base`、`--port` 等参数。

## 7. 目录隔离全景

```
PROJECT_SOURCE_DIR/                    ← 普通 Agent 项目（代码开发）
├── {projectId}/                       ← 默认模式
└── {tenantId}/{spaceId}/{projectId}/  ← 多租户隔离模式

UPLOAD_PROJECT_DIR/                    ← 上传文件 + 备份 zip 暂存
└── {projectId}/
    └── backup_temp_{ts}/

COMPUTER_WORKSPACE_DIR/                ← Computer Agent 工作空间（技能/Agent）
└── {userId}/{cId}/skills/agents/

DIST_TARGET_DIR/                       ← 生产构建产物
└── {projectId}/
```

四类目录通过环境变量配置，物理隔离，互不干扰。rcoder 把 `computer-project-workspace` 挂载到容器内 `COMPUTER_WORKSPACE_DIR` 对应路径，Agent 运行时直接读写此目录下的技能文件。

## 一句话总结

路径隔离通过 `tenantId+spaceId+isolationType` 三字段联合判断是否启用多租户子目录，Computer 工作空间通过 `userId/cId/skills+agents` 两级目录组织技能与 Agent，`.dynamic_add.lock` 文件标记运行时动态注入的技能使其在工作空间重建时得以保留。
