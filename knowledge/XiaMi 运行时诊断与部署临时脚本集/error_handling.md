## 1. 总体方案

仓库包含两个运行侧，各自采用本语言生态的惯用错误模型：

- Go (`smartnpc-mcp`)：使用 `errors`、`fmt.Errorf` + `%w` 进行错误传播，统一通过 `log/slog`（JSON 结构化日志，输出到 stderr）上报；不存在自定义 error 类型或 sentinel 变量。
- C# SMAPI Mod (`smapi-mod`)：在工具处理器中用 `try { ... } catch (Exception ex)` 捕获异常，并返回 `Response.Failure(id, "invalid_params", ex.Message)` 这类 JSON-RPC 失败帧；进程级崩溃由 SMAPI 托管，代码中未见 `throw new` 的主动抛出点，也没有 `try/finally` 作为通用清理手段。
- Shell 脚本 (`scripts/`)：未使用统一的错误检查函数，直接依赖命令退出码和逐条 `if err != nil` 风格（实际 grep 未发现 shell 级别的集中式错误处理逻辑）。

没有跨语言的错误码枚举、全局错误中间件或 panic/recover 机制。

## 2. 关键文件

- `SmartNPC/smartnpc-mcp/internal/log/logger.go` — 唯一日志入口，强制所有日志写 stderr，避免污染 MCP stdio 协议流。
- `SmartNPC/smartnpc-mcp/cmd/smartnpc-mcp/main.go` — 启动期配置解析错误通过 `logger.Error` + `os.Exit(1)` 终止进程。
- `SmartNPC/smartnpc-mcp/pkg/relay/hermes/relay.go` — HTTP 请求错误、4xx/5xx 响应、转译失败均以 `logger.Warn` / `logger.Info` 记录，不向上抛错；`New` 中对必填字段校验用 `fmt.Errorf` 返回错误给调用方。
- `SmartNPC/smartnpc-mcp/adapters/stardew/bridge/ws_client.go` — WebSocket 重连与回退延迟，连接失败仍启动后台 `readLoopForever` 自动重连。
- `SmartNPC/smartnpc-mcp/pkg/agentbridge/config.go` — YAML 配置解析错误用带前缀的 `fmt.Errorf("...: %w", err)` 包装。
- `SmartNPC/smapi-mod/Query/GameQueryHandler.cs`、`MailHandler.cs`、`UnattendedController.cs`、`ChatInputCapture.cs`、`FarmEconomyHandler.cs` — C# 侧统一的 `catch (Exception ex) { return Response.Failure(...) / _log.Log(...) }` 模式。

## 3. 架构与约定

### Go 侧：值错误 + 结构化日志

- 构造期错误（如 `hermesrelay.New` 缺 URL/Conversation/Model、`LoadConfigFile` 读不到文件）通过 `return nil, fmt.Errorf(...)` 上抛，由 `main()` 层 `logger.Error` + `os.Exit(1)` 终止。
- 运行时错误（HTTP POST 失败、非 2xx 状态码、marshal 失败）不在调用栈上传播，而是记录 `logger.Warn`，并把遥测数据写入 `LogRelayError` / `emitPayload`，然后 `return` 静默结束 goroutine。
- 对远端错误的特殊恢复：当 Hermes 返回包含 `empty array` + `tool_calls` 的 4xx 响应时，`post()` 主动调用 `RotateSession` 切换到新会话名，仅影响当前 NPC 的对话，不影响其他 NPC。
- 日志格式固定为 JSON（`slog.NewJSONHandler(os.Stderr, ...)`），且通过 `AddSource: true` 附加 source 字段（file/line/function）。
- 没有定义任何自定义 error 类型、sentinel 变量或 `errors.Wrap`（也未引入第三方 wrap 包）。

### C# 侧：try/catch → Response.Failure

- 工具参数反序列化统一用 `try { JsonSerializer.Deserialize<T>(...) } catch (Exception ex) { return Task.FromResult(Response.Failure(id, "invalid_params", ex.Message)); }`。
- 游戏 API 调用异常（如存档读取、农场经济查询）用 `_monitor.Log(..., LogLevel.Error/Warn)` 记录后继续执行，把异常限制在单条工具调用内。
- 没有自定义异常类型，全部使用 `System.Exception`。

### 进程边界约定

- `internal/log/logger.go` 文档明确声明“所有日志必须写到 stderr”，因为 MCP stdio 传输占用 stdout 做 JSON-RPC 流。
- `main.go` 注释同样强调：stdio 模式下不得向 stdout 写日志，否则会破坏协议。

## 4. 约定与约束

- **Go 构造期错误必须返回 error**：`hermesrelay.New`、`LoadConfigFile`、`agentbridge.LoadConfig` 等构造函数都遵循此约定，调用方统一以 `logger.Error` + `os.Exit(1)` 处理。
- **运行时 I/O 错误不上抛**：`hermesrelay.post` 对 HTTP 失败、非 2xx 状态码只记录 warn 并返回，不中断事件转发 goroutine；这是有意设计（`HandleEvent` 在新 goroutine 中调用 `post`，与 bridge 读循环解耦）。
- **Hermes 会话损坏自愈**：检测到特定错误响应体时自动 `RotateSession`，属于硬编码的错误恢复策略，仅限该 relay。
- **C# 工具处理器禁止让异常逃逸**：所有工具方法包裹 `try/catch (Exception)`，将异常转为 `Response.Failure` 或 `_log.Log(..., LogLevel.Warn/Error)`，保证 SMAPI 主循环不被打断。
- **日志通道不可变**：stderr 是唯一合法日志输出目标；stdout 专用于 MCP JSON-RPC 流。这一约束由 `internal/log` 包的实现和 `main.go` 注释共同保证。
- **无全局错误码/中间件**：仓库未定义跨模块共享的错误码枚举，也没有类似 middleware 的统一错误拦截器；每个组件自行决定是上抛、记录还是降级处理。