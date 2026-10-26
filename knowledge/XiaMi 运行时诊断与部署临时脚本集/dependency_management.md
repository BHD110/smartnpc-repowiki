## 1. 使用的系统与工具

本仓库是一个混合语言项目，依赖管理按语言划分：

- **Go 子模块** `SmartNPC/smartnpc-mcp`：使用 Go modules，通过 `go.mod` + `go.sum` 声明与锁定依赖；根目录存在 `go.work` 将多个 Go 模块聚合为 workspace。
- **.NET / C# 子项目** `SmartNPC/smapi-mod`、`SmartNPC/tests/animations`：使用 `.csproj` + NuGet；SDK 版本通过根级 `global.json` 固定。
- **Node.js**：仓库中未发现任何 `package.json` 或 `node_modules` 被纳入版本控制（`.gitignore` 忽略 `node_modules`），说明 Node 不是构建期依赖。
- **打包策略**：未使用 vendor 目录；Go 依赖由 `go mod tidy` 拉取并写入 `go.sum`，.NET 依赖由 NuGet 还原。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `SmartNPC/smartnpc-mcp/go.mod` | Go module 定义，Go 版本 `1.25.0`，直接依赖仅 5 个第三方包（websocket、uuid、modelcontextprotocol/go-sdk、x/sys、yaml.v3） |
| `SmartNPC/smartnpc-mcp/go.sum` | Go 依赖的哈希校验和锁文件 |
| `SmartNPC/global.json` | 固定 .NET SDK 为 `8.0.0`，`rollForward: latestMajor`，`allowPrerelease: false` |
| `SmartNPC/smartnpc-mcp/StardewMCPBridge.csproj` | C# 项目文件（目标框架 `net6.0`） |
| `SmartNPC/smartnpc-mcp/Taskfile.yml` | Go 侧 build/test/lint 任务 |
| `SmartNPC/Taskfile.yml` | 根 Taskfile，统一编排 mcp + mod 两个子项目的构建、测试、lint、tidy |
| `SmartNPC/.gitignore` | 显式忽略 `bin/`、`obj/`、`go.work.sum`、`node_modules` 等生成产物 |
| `SV.v1.6.15/.../StardewModdingAPI.deps.json` / `StardewValley.GameData.xml` | SMAPI 运行时已解析的依赖清单（只读，非源码依赖声明） |

## 3. 架构与约定

### Go 依赖
- 唯一 module path：`github.com/OmniStormX/SmartNPC`。
- 直接依赖极少且均为稳定库：WebSocket、UUID、MCP SDK、系统调用、YAML 解析。
- 间接依赖（`// indirect`）包括 jsonschema-go、segmentio/asm+encoding、uritemplate、oauth2，均由 go-sdk 引入。
- 没有 GOPRIVATE / GONOSUMCHECK / GONOPROXY 配置；依赖全部来自公共 Go proxy（默认 golang.org 代理）。
- 根 `Taskfile.yml` 提供 `task tidy`，执行 `cd smartnpc-mcp && go mod tidy` 后再 `go work sync`，确保 workspace 同步。
- `go.sum` 被提交到版本库，作为可复现依赖快照。

### .NET 依赖
- `global.json` 把构建用 .NET SDK 钉死在 `8.0.0`，`rollForward: latestMajor` 允许升级到 8.x，但禁止预发行版（`allowPrerelease: false`）。注释明确解释原因：C# Dev Kit 需要 >= .NET 8 才能启动项目服务器，而 mod 本身仍 target `net6.0`。
- C# 项目通过 NuGet 恢复依赖（未见 `NuGet.config`，走默认 nuget.org）。
- `smapi-mod/bin/`、`smapi-mod/obj/` 被 `.gitignore` 排除，不提交编译产物。

### 工作流整合
- 根 `Taskfile.yml` 通过 `includes:` 聚合 `smartnpc-mcp/Taskfile.yml` 与 `smapi-mod/Taskfile.yml`，所有跨子项目的操作（build、test、lint、clean、ci）都经由 `task <name>` 调用，保证本地与 CI 一致。
- `task lint` 会先跑 `go work sync` 再 `go vet`，把 Go workspace 一致性纳入静态检查流程。
- `task ci-fast` / `task ci` 是本地等效的 GitHub Actions 入口，lint 与 test 前不单独触发依赖还原——依赖变更应在 PR 中更新 `go.sum` 后合并。

## 4. 约定与约束

- **Go 依赖版本由 `go.mod` 声明、`go.sum` 锁定**，两者均提交至版本库；修改依赖需运行 `task tidy`（即 `go mod tidy` + `go work sync`）以保持一致。
- **Go 版本固定为 `1.25.0`**（写在 `go.mod` 顶部），开发者应使用相同版本的 Go toolchain。
- **.NET SDK 版本固定为 `8.0.0`**（写在 `global.json`），不允许回退到 7.x 或升级至 9.x；预发行版被明确禁止。
- **C# 项目目标框架保持 `net6.0`**（见 StardewMCPBridge.csproj），与构建 SDK 分离——这是 SMAPI mod 的兼容性要求。
- **不使用 vendor 目录**，也不配置私有 Go proxy 或 NuGet 源；所有第三方包从公共源下载。
- **构建产物不被追踪**：`.gitignore` 显式忽略 `*.exe`、`*.dll`、`*.so`、`*.dylib`、`bin/`、`obj/`、`go.work.sum`、`node_modules`，避免二进制依赖污染仓库。
- **CI 一致性**：根 `Taskfile.yml` 注释写明“所有 build/test/lint/release 工作通过 `task <name>` 进行，以便本地调用与 CI 完全一致”，因此新增依赖管理步骤时应优先通过 Taskfile 暴露，而非直接写 shell 命令。