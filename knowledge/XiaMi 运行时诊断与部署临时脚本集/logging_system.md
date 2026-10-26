## 概述

本仓库包含两个运行子系统的日志方案：

1. **Go 侧 (`smartnpc-mcp`)**：使用 Go 标准库 `log/slog`，集中通过 `internal/log` 初始化，输出 JSON 到 stderr。
2. **C# SMAPI Mod 侧 (`smapi-mod`)**：通过 SMAPI 注入的 `Log.Log(..., LogLevel.X)` API 输出带前缀的人类可读消息。

两者完全独立，没有跨语言共享的日志框架。

---

## Go 侧 (`smartnpc-mcp/internal/log`)

### 核心文件

- `internal/log/logger.go` — 统一入口 `New(level string) *slog.Logger`
- `internal/log/console_windows.go` / `console_other.go` — Windows UTF-8 控制台切换（`EnableUTF8Console`）
- `adapters/stardew/tools/action_logger.go` — 按业务分文件的专用日志（`npc_actions.log`、`schedule.log`、`mcp/schedule_triggers.log`、`logs/mcp/events.log`）
- `pkg/relay/hermes/config.go` — 可选 payload 日志（环境变量 `SMARTNPC_RELAY_PAYLOAD_LOG`）

### 架构与约定

- **唯一入口**：所有进程级 logger 都经 `log.New(level)` 创建，禁止直接调用 `slog.Default()` 作为主入口。测试代码例外，直接使用 `slog.New(slog.NewTextHandler(io.Discard, nil))` 丢弃日志。
- **输出目标强制为 stderr**：`logger.go` 文档注释明确声明「All logs MUST go to stderr」，因为 MCP stdio 传输协议把 stdout 留给 JSON-RPC 流，写到 stdout 会破坏协议。
- **JSON 格式 + source 信息**：`slog.NewJSONHandler(os.Stderr, &slog.HandlerOptions{Level: lvl, AddSource: true})`，每条记录附带 `source.group`（`file`、`line`、`function`），代价是每次调用一次 `runtime.Callers`，注释中评估为 negligible。
- **级别映射**：`debug → LevelDebug`、`warn|warning → LevelWarn`、`error → LevelError`、默认 `LevelInfo`。CLI 参数名为 `--log-level`（`cmd/smartnpc-mcp/main.go`、`cmd/agent-bridge/main.go`）。
- **Windows 控制台编码**：`EnableUTF8Console()` 在 Windows 上调用 `windows.SetConsoleOutputCP(65001)` 以正确显示 CJK 等非 ASCII 字符；非 Windows 平台为 no-op。

### 按业务切分的文件日志

`action_logger.go` 暴露 `LogRoot()` 和以下固定落盘位置（优先级：`$SMARTNPC_LOG_DIR` > `<exe>/../logs/` > CWD）：

| 文件 | 内容 | 格式 |
|---|---|---|
| `npc_actions.log` | NPC 工具调用及原始入参 | `slog.JSONHandler(LevelInfo)` |
| `schedule.log` | 每日 NPC 计划摘要 | 人类可读文本（含 ANSI 绿色 console 镜像到 stderr） |
| `mcp/schedule_triggers.log` | 每个 game_time_tick 触发的调度条目 | 自定义 tab-separated 行（header + rows） |
| `logs/mcp/events.log` | 事件记录（由同文件后续代码维护） | 结构体序列化 |
| `logs/mcp/workflow_runs/*.jsonl` | 工作流执行轨迹（见目录树） | JSON Lines |
| `logs/exp/*.jsonl` | 实验追踪（`explog.New`，`cmd/smartnpc-mcp/main.go` 第 193 行） | JSON Lines |

这些文件均通过 `sync.Once` + 单例 file handle 懒加载打开，写失败时回退到 `slog.Default().Warn(...)` 而非 panic。

### 可插拔 payload 日志

`pkg/relay/hermes/config.go` 根据环境变量 `SMARTNPC_RELAY_PAYLOAD_LOG` 动态开启一个独立的 debug 级 JSON handler，用于记录 Hermes relay 的 payload，便于隔离网络层问题。

---

## C# SMAPI Mod 侧 (`smapi-mod`)

SMAPI Mod 不使用 `log/slog`，而是通过 SMAPI 注入的 `IModHelper.Log`（表现为 `Log.Log(message, LogLevel)`）写入日志。

- **级别**：`LogLevel.Debug`、`LogLevel.Info`、`LogLevel.Warn`（未见 `Error` 用法）。
- **消息风格**：字符串拼接 + 方括号前缀 `[npc_xxx]`，例如 `[npc_wander]`、`[npc_water_crops]`、`[npc_deposit_items]`、`[npc_plant_seeds]`、`[npc_ship_items]` 等，集中在 `Behavior/WorldActionHandlers.cs` 中大量出现。
- **无结构化字段**：所有上下文信息（NPC 名、坐标、数量等）全部嵌入字符串模板，不是键值对。
- **无自定义 sink**：日志交给 SMAPI 宿主处理，本仓库不配置文件、stdout/stderr 分流。

---

## 关键约束（有源码或注释依据）

1. **Go 侧主日志必须输出到 stderr**：`internal/log/logger.go` 的包注释强制规定「All logs MUST go to stderr. The MCP stdio transport reserves stdout for the JSON-RPC stream; writing logs there will corrupt the protocol」。
2. **进程级 logger 必须经 `log.New` 创建**：`main.go` 中 `logger := log.New(*logLevel)` 是唯一入口；业务代码接收该 logger 实例而非调用 `slog.Default()`。
3. **Windows 控制台启用 UTF-8**：`EnableUTF8Console()` 将输出 codepage 设为 65001，避免 GBK/cp936 下的乱码。
4. **`SMARTNPC_LOG_DIR` 环境变量覆盖日志根路径**：`action_logger.go::logDir()` 优先使用该变量决定 `npc_actions.log`、`schedule.log` 等的落盘位置。
5. **payload 日志开关**：`SMARTNPC_RELAY_PAYLOAD_LOG` 环境变量控制是否额外写出 Hermes relay 的 JSON payload。
6. **C# 侧使用 SMAPI Log API**：Mod 代码通过 `Log.Log(msg, LogLevel.X)` 输出，无自定义日志框架。

---

## 关键文件清单

- `SmartNPC/smartnpc-mcp/internal/log/logger.go`
- `SmartNPC/smartnpc-mcp/internal/log/console_windows.go`
- `SmartNPC/smartnpc-mcp/internal/log/console_other.go`
- `SmartNPC/smartnpc-mcp/adapters/stardew/tools/action_logger.go`
- `SmartNPC/smartnpc-mcp/pkg/relay/hermes/config.go`
- `SmartNPC/smartnpc-mcp/cmd/smartnpc-mcp/main.go`
- `SmartNPC/smartnpc-mcp/cmd/agent-bridge/main.go`
- `SmartNPC/smapi-mod/Behavior/WorldActionHandlers.cs`
