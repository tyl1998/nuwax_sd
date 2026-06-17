# nuwax-cli `init` 与 `status` 源码解读

这篇专门拆 `nuwax-cli` 最适合入门源码的两个入口：

- `init`
- `status`

原因很简单：

- `init` 代表“系统从 0 到 1 怎么落地”
- `status` 代表“系统出问题时怎么自检”

这两条线搞清楚之后，再看 `upgrade`、`docker-service`、`auto-upgrade-deploy` 会轻松很多。

## 1. 为什么这两个命令值得单独看

从 [main.rs](../../nuwax-cli/nuwax-cli/src/main.rs) 可以看到，作者没有把所有命令都统一丢进 `CliApp::run_command()`。

相反：

- `init` 被提前特判
- `status` 被提前特判
- `diff-sql` 也被提前特判

这说明作者明确知道：

- 有些命令在“配置和数据库都还没就绪时”也必须能用
- 有些命令在“系统出故障时”仍然要尽量给用户反馈

`init` 和 `status` 正好就是这两类命令的代表。

## 2. `init` 的代码路径

### 2.1 入口在哪里

第一入口在 [main.rs](../../nuwax-cli/nuwax-cli/src/main.rs)：

```rust
if let Commands::Init { force } = cli.command {
    if let Err(e) = run_init(force).await {
        ...
    }
    return;
}
```

这段逻辑的重点不是“调用了 `run_init`”，而是：

- 它根本不尝试初始化 `CliApp`
- 做完后直接 `return`

也就是说，`init` 是一条完全独立于常规应用装配流程的支线。

### 2.2 真正执行在哪里

实现文件：

- [init.rs](../../nuwax-cli/nuwax-cli/src/init.rs)

函数：

- [run_init](../../nuwax-cli/nuwax-cli/src/init.rs)

## 3. `init` 实际做了哪三步

### 第一步：创建配置和目录

关键代码：

- [init.rs:23-47](../../nuwax-cli/nuwax-cli/src/init.rs)

它做的事情：

1. `AppConfig::default()`
2. `config.save_to_file("config.toml")`
3. 创建 `docker/`
4. 创建备份目录
5. 调用 `config.ensure_cache_dirs()`

这一步的意义：

- 先把本地工作目录搭出来
- 后面数据库、下载缓存、服务包路径才能有稳定落点

### 第二步：初始化数据库

关键代码：

- [init.rs:49-65](../../nuwax-cli/nuwax-cli/src/init.rs)

它做的事情：

1. 用 `config::get_database_path()` 取 DuckDB 路径
2. `Database::connect(&db_path).await?`
3. `database.init_database().await?`
4. `database.get_or_create_client_uuid().await?`

这一步的意义：

- 不只是“连上一个 DB”
- 而是显式初始化表结构，并生成客户端身份

这里和普通命令很不一样的一点在于：

- 数据库初始化只在 `init` 中显式执行
- 常规 `CliApp::new_with_config` 只会检查数据库是否已经初始化

这点可以在 [app.rs:53-63](../../nuwax-cli/nuwax-cli/src/app.rs) 对照看出来。

### 第三步：向服务端注册客户端

关键代码：

- [init.rs:67-95](../../nuwax-cli/nuwax-cli/src/init.rs)

它做的事情：

1. 收集本机 `os` 和 `arch`
2. 创建不带 `client_id` 的 `ApiClient::new(None, None)`
3. 调用 `register_client`
4. 如果成功，用服务端返回的 `client_id` 覆盖本地 UUID
5. 如果失败，只发 warning，不阻断本地初始化完成

这一点非常值得记住：

`init` 的设计目标不是“远端必须成功”，而是“本地可先工作起来”。

也就是说，作者把它设计成：

- 远端注册成功更好
- 远端注册失败也不能毁掉本地初始化

这是很典型的“本地优先、远端增强”的 CLI 初始化思路。

## 4. `init` 的保护和兜底

### 已存在配置或数据库时默认不重建

关键代码：

- [init.rs:12-21](../../nuwax-cli/nuwax-cli/src/init.rs)

逻辑：

- 如果已有配置或数据库
- 且没有 `--force`
- 就直接提示并退出

这能避免误覆盖已有环境。

### 初始化后给出“下一步操作建议”

关键代码：

- [init.rs:97-119](../../nuwax-cli/nuwax-cli/src/init.rs)

虽然里面有少量提示文案还没完全跟当前 CLI 对齐，但设计意图很清楚：

- 初始化不是终点
- 初始化成功后，用户通常会马上进入“下载服务包 / 启动服务 / 自动升级部署”

## 5. `status` 的代码路径

### 5.1 第一入口在哪

在 [main.rs:66-100](../../nuwax-cli/nuwax-cli/src/main.rs)。

整体逻辑可以概括成：

```text
先显示基础版本
再尝试初始化应用
成功就显示完整状态
失败也给用户可操作的排障提示
```

这个流程和 `init` 一样，都是“特判之后直接 return”，不会进入通用命令分发。

### 5.2 为什么 `status` 不走统一流程

如果 `status` 也要求先完整初始化 `CliApp`，那一旦配置损坏、数据库异常、目录不对，用户连“现在坏在哪”都看不到。

作者显然不接受这一点，所以先做：

- [show_client_version](../../nuwax-cli/nuwax-cli/src/commands/status.rs)

然后才尝试：

- [CliApp::new_with_auto_config](../../nuwax-cli/nuwax-cli/src/app.rs)

## 6. `status` 成功时会看哪些东西

真正的详细状态输出在：

- [run_status_details](../../nuwax-cli/nuwax-cli/src/commands/status.rs)

它主要检查 4 类信息。

### 6.1 基础信息

关键代码：

- [status.rs:27-36](../../nuwax-cli/nuwax-cli/src/commands/status.rs)

输出：

- Docker service version
- 配置文件名
- Client UUID

### 6.2 文件状态

关键代码：

- [status.rs:38-73](../../nuwax-cli/nuwax-cli/src/commands/status.rs)

会检查：

- `docker-compose.yml` 是否存在
- 当前版本对应的服务包文件是否存在

这里有一个很实用的细节：

- 服务包不是死写文件名
- 而是调用 `app.config.get_version_download_file_path(...)` 自动找当前版本和格式对应的归档文件

### 6.3 Docker 服务状态

关键代码：

- [status.rs:75-98](../../nuwax-cli/nuwax-cli/src/commands/status.rs)
- [status.rs:139-165](../../nuwax-cli/nuwax-cli/src/commands/status.rs)

流程：

1. 根据配置构建 `DockerManager`
2. 创建 `HealthChecker`
3. 执行 `health_check`
4. 输出 running / failed 状态

这说明 `status` 并不是只看文件，而是真的会探测容器健康状态。

### 6.4 基于状态给出建议

关键代码：

- [status.rs:100-127](../../nuwax-cli/nuwax-cli/src/commands/status.rs)

逻辑上分了三种场景：

1. 没 compose，没服务包：像首次使用
2. 没 compose，但有服务包：像“包已下载但还没部署”
3. compose 和系统文件齐：像“系统已基本可用”

这部分的本质不是“打印提示”，而是在做一种轻量状态机判断。

## 7. `status` 失败时为什么仍然有价值

如果 `CliApp::new_with_auto_config().await` 失败，`main.rs` 不会直接 panic 或报一大坨堆栈，而是给用户这类信息：

- 当前目录可能不对
- 配置或数据库文件可能不在当前目录
- 数据库可能被锁住

对应代码：

- [main.rs:82-98](../../nuwax-cli/nuwax-cli/src/main.rs)

这说明 `status` 被作者设计成一个“对用户友好的失败入口”。

从产品思路上说，它承担的是：

- 检查器
- 导航器
- 排障入口

而不是单纯的“查版本号”。

## 8. `CliApp` 在这两条流程中的角色

理解 `init` / `status` 的关键，还在于理解 `CliApp` 什么时候会被绕开、什么时候会被用到。

### `init`

- 完全绕开 `CliApp`
- 因为此时配置和数据库还不存在或尚未完成初始化

### `status`

- 先绕开 `CliApp` 输出基础版本
- 再尝试用 `CliApp::new_with_auto_config()` 进入完整状态检查

### 常规命令

- 都要走 [app.rs:48-106](../../nuwax-cli/nuwax-cli/src/app.rs)

在那里会统一装配：

- `AppConfig`
- `Database`
- `AuthenticatedClient`
- `ApiClient`
- `DockerManager`
- `BackupManager`
- `UpgradeManager`

换句话说：

- `init` 是“建世界”
- `status` 是“探世界”
- `CliApp` 是“进入世界后的标准运行环境”

## 9. 看完这两条线，你应该得到的认知

### 认知 1

这个 CLI 不是简单脚本拼起来的，它有明确的“冷启动路径”和“故障路径”。

### 认知 2

作者非常在意：

- 初始化不依赖完整环境
- 排障不依赖完整环境

这对部署类 CLI 非常重要。

### 认知 3

常规业务命令和“系统自救命令”被刻意分流了。

这也是为什么后面你看 `upgrade download`、`auto-upgrade-deploy offline-deploy` 这些命令时，会继续看到类似的特判设计。

## 10. 下一步建议

如果你想继续顺着这条思路往下读，最自然的下一站是：

- [03-关键流程梳理.md](./03-关键流程梳理.md)

如果你想继续做这种“按入口深拆”的源码解读，建议下一个主题看：

1. `upgrade` 与 `upgrade download`
2. `auto-upgrade-deploy run`
3. `backup`
