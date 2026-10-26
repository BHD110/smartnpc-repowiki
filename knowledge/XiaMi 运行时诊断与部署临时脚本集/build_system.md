## 1. 构建系统与工具链

仓库是一个混合语言项目，包含 C# SMAPI 模组（`smapi-mod/`）、Go MCP 服务（`smartnpc-mcp/`）以及大量 Shell/PowerShell 诊断脚本。统一入口是根级 `Taskfile.yml`（go-task v3），通过 `includes` 将子模块的 Taskfile 聚合为顶层任务。

- **C# / .NET**：使用 `dotnet` CLI 和 Stardew Valley ModBuildConfig 构建；目标框架为 `net6.0`，由 `smapi-mod/StardewMCPBridge.csproj` 声明。
- **Go**：使用 `go.work` 工作区管理 `smartnpc-mcp` 单个模块；依赖在 `smartnpc-mcp/go.mod`。
- **Docker**：Hermes 运行时镜像通过 `deploy/hermes/Dockerfile` 构建并推送到 GHCR。
- **CI/CD**：GitHub Actions（`.github/workflows/`）覆盖 CI、Release、Docker 镜像构建三条流水线。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `SmartNPC/Taskfile.yml` | 根聚合 Taskfile，定义 `build`/`test`/`lint`/`ci`/`profiles:verify`/`hooks:*` 等任务 |
| `SmartNPC/smartnpc-mcp/Taskfile.yml` | Go 模块任务：`build`/`test`/`run`/`health`/`stop` |
| `SmartNPC/smapi-mod/Taskfile.yml` | C# SMAPI 模组任务：`build`/`install`/`test` |
| `SmartNPC/global.json` | 指定 .NET SDK 版本 |
| `SmartNPC/go.work` | Go workspace 文件 |
| `SmartNPC/.github/workflows/ci.yml` | 主 CI（仅 Go，跳过 smapi-mod） |
| `SmartNPC/.github/workflows/release.yml` | 基于 semver tag 的 Release 流水线 |
| `SmartNPC/.github/workflows/docker-hermes.yml` | Hermes Docker 镜像构建与推送 |
| `SmartNPC/deploy/hermes/Dockerfile` | Hermes 运行时容器镜像定义 |

## 3. 架构与约定

### 3.1 根 Taskfile 聚合模式
根 `Taskfile.yml` 通过 `includes` 引入 `./smartnpc-mcp/Taskfile.yml` 和 `./smapi-mod/Taskfile.yml`，使开发者从仓库根执行 `task build`、`task test`、`task ci` 即可并行触发两个子模块的任务。每个子模块有自己的 Taskfile，职责清晰。

### 3.2 环境变量与 dotenv
所有 Taskfile 通过 `dotenv: ['.env']` 自动加载 `.env`（示例在文档中说明）。变量如 `SMARTNPC_GAME_PATH`、`DOTNET`、`PORT`、`WS_URL`、`ECHO` 等通过 Taskfile vars 或 `default(...)` 提供默认值，支持跨平台覆盖。

### 3.3 跨平台任务
几乎所有命令都使用 Taskfile 的 `platforms:` 条件分支，区分 Windows（PowerShell/cmd）与 Linux/macOS（bash）。例如 `smapi-mod install` 任务分别用 PowerShell 和 bash 实现进程终止、DLL 复制、`Unblock-File`（WDAC/SmartScreen 解除锁定）。

### 3.4 源码变更检测（sources/generates）
Go Taskfile 中 `build` 任务声明 `sources: ['**/*.go', 'go.mod', 'go.sum']` 和 `generates: ['bin/smartnpc-mcp(.exe)?']`，利用 Taskfile 内置缓存避免重复编译。

### 3.5 Git Hooks
`task hooks:enable` 将 `core.hooksPath` 指向 `.githooks/`，并在提交后自动安装 smapi-mod（当 `smapi-mod/` 有变更时）。

### 3.6 Profile 渲染验证
`profiles:render` 和 `profiles:verify` 调用 `scripts/render_profiles.sh` 和 `scripts/test_profile_render.sh`，对 `hermes/npcs.yaml` + `runtime-config.yaml` 生成 `hermes/profiles/<npc>/` 树并进行健全性检查。

## 4. CI 流水线约束

### 4.1 CI (`ci.yml`)
- 仅在 `main`/`master` 分支 push、PR 或手动触发。
- 使用 `dorny/paths-filter@v3` 检测 `smartnpc-mcp/**`、`go.work*`、`Taskfile.yml`、`.github/workflows/**` 变更。
- **明确跳过 C# 模组构建**：注释说明 GitHub-hosted runner 无法合法安装 Stardew Valley，因此 CI 只运行 Go 的 `go vet`、`go test -race`、`go build`。
- 最终 `ci-pass` job 作为分支保护门控，允许 `success` 或 `skipped`。

### 4.2 Release (`release.yml`)
- 触发条件：semver tag（`v[0-9]+.[0-9]+.[0-9]+-*`）或 `workflow_dispatch`。
- 矩阵交叉编译 Go 二进制（`windows-amd64`、`linux-amd64`），使用 `-trimpath`、`-ldflags="-s -w -X main.version=${VERSION}"` 注入版本号。
- 生成 `SHA256SUMS.txt`，通过 `softprops/action-gh-release@v2` 创建 GitHub Release。
- **SMAPI 模组不自动构建**：需本地 `task mod:build` 后手动上传到 Release。

### 4.3 Docker (`docker-hermes.yml`)
- 触发：push `main` 且路径匹配 `deploy/hermes/**`、`hermes/profiles/**`、`hermes/npcs.yaml`、`scripts/render_profiles.sh`。
- 先执行 `deploy/hermes/scripts/sync-profiles.sh` 将 profiles 同步进构建上下文，再 `docker/build-push-action@v6` 推送到 `ghcr.io/<owner>/smartnpc-hermes`，标签为 `latest` + `sha:<commit>`。

## 5. 观察到的规则与约束

1. **所有构建/测试入口必须通过 `task <name>`**：根 Taskfile 顶部注释明确要求 "All build / test / lint / release work goes through `task <name>` so that local invocations match CI exactly"，并有 `.codebuddy/rules/taskfile-usage.mdc` 作为规则引用。
2. **Go 版本固定为 1.22**：CI 与 release workflow 均通过 `GO_VERSION: '1.22'` 控制，`.github/workflows/ci.yml` 显式设置 `GO_VERSION: '1.22'`。
3. **C# 模组不在 CI 中构建**：CI 注释明确解释原因——需要本地 SDV 安装，GitHub Runner 无法合法安装付费游戏。
4. **Release 产物校验**：release 流水线强制生成 `SHA256SUMS.txt`，发布前通过 `find artifacts -type f -exec cp {} release/ \;` 扁平化收集所有二进制。
5. **Docker 镜像仅随特定路径变更触发**：`docker-hermes.yml` 使用 `paths` 过滤，只有 `deploy/hermes/**`、`hermes/profiles/**`、`hermes/npcs.yaml`、`scripts/render_profiles.sh` 变更才重新构建镜像。
6. **Windows 与 POSIX 双轨脚本**：所有跨平台任务使用 Taskfile `platforms:` 条件分支，禁止在单条 `cmd:` 中混用 shell 语法。
7. **Concurreny 分组取消旧运行**：CI、Release、Docker 三个 workflow 均配置 `concurrency.group` + `cancel-in-progress: true`，避免冗余运行。
8. **Go 测试目录隔离**：Go 测试输出临时目录统一设为 `GOTMPDIR={{ROOT_DIR}}/.gotmp`，避免污染工作区。
9. **SMAPI 模组部署前先终止游戏进程**：`smapi-mod install` 在 Windows 上 `Stop-Process` `Stardew Valley`/`StardewModdingAPI`，Linux/macOS 上 `pkill`，以释放 DLL 锁。
10. **版本号来源**：Release 流水线从 git tag 提取版本（`$GITHUB_REF#refs/tags/`），并通过 `-X main.version=${VERSION}` 注入 Go 二进制，用于 `--version` 输出。