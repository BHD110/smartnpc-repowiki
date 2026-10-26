## 1. 总体方案

本仓库没有使用统一的配置框架（如 .NET `IConfiguration`、Go `viper`），而是采用 **Bash + YAML + Shell 环境变量** 的自研多层配置体系，核心思路是：

- **单一事实源**：`SmartNPC/hermes/npcs.yaml` 是唯一权威 NPC/Profile 注册表。
- **生成阶段**：`scripts/render_profiles.sh` 从 `_master/` 模板和 `npcs.yaml` 渲染出每 NPC 的 profile 目录（`hermes/profiles/<npc>/`）以及 `hermes/runtime-config.yaml`。
- **安装阶段**：`hermes/install.sh`（外部工具）把 `config-overlay.yaml` 追加到 Hermes 生成的 `config.yaml` 上，并注入模型/provider/compression 等运行时参数。
- **调优阶段**：`scripts/apply_hermes_tuning.sh` 用 PyYAML 对已安装的 `~/.hermes/profiles/<npc>/config.yaml` 做键级合并（不破坏既有字段）。
- **运行阶段**：`smartnpc-mcp` 读取 `runtime-config.yaml`，通过 `${SMARTNPC_HERMES_GATEWAY_HOST}` / `$WSL_IP` / `127.0.0.1` 三级回退解析 gateway URL；密钥走 `api_key_env: SMARTNPC_HERMES_KEY` 指向的环境变量。

## 2. 关键文件

| 路径 | 角色 |
|---|---|
| `SmartNPC/hermes/npcs.yaml` | 唯一 NPC 元数据注册表（id / game_name / display_name / gateway_port / enabled / kind / peer_a_* / peer_b_*） |
| `SmartNPC/scripts/lib/npc_registry.sh` | Bash 侧轻量 YAML 解析器，提供 `smartnpc_load_registry` |
| `SmartNPC/scripts/render_profiles.sh` | 模板渲染器：复制 `_master/` 模板 → sed 替换占位符 → 写出 per-NPC profile 与 `runtime-config.yaml` |
| `SmartNPC/hermes/runtime-config.yaml` | 由 render_profiles.sh 生成的多 profile fan-out 配置，供 `smartnpc-mcp --hermes-config` 消费 |
| `SmartNPC/hermes/profiles/_master/config-overlay.yaml` | 每 NPC 最终注入 Hermes `config.yaml` 的覆盖片段（被 install.sh 追加） |
| `SmartNPC/scripts/apply_hermes_tuning.sh` | 用 PyYAML 精确写入 model/provider/compression/agent.tool_use_enforcement 等键 |
| `SmartNPC/scripts/reset_hermes_overlay.sh` | 按 marker `# ── injected by hermes/install.sh (smartnpc profile) ──` 剥离 overlay，支持 `--wipe-state` |
| `SmartNPC/smapi-mod/assets/agent_npcs.json` | render_profiles.sh 同时生成的 C# mod 端启用列表（`{"GameName": true|false}`） |
| `SmartNPC/hermes/profiles/<npc>/.env` | 每 NPC 私有的 Langfuse / LLM 密钥（仅首次 seed，之后不被重渲染覆盖） |

## 3. 架构与约定

### 3.1 三层配置叠加

1. **Hermes bootstrap**：`hermes/install.sh` 在 `~/.hermes/profiles/<npc>/config.yaml` 中写入默认值。
2. **Overlay 覆盖**：`config-overlay.yaml` 被 append 到 bootstrap config 末尾——由于 Hermes 的 YAML 加载会去重顶层 key，所以 overlay 只能追加新块（如 `mcp_servers`、`API_SERVER_*`），不能重写已有 block（如 `model`、`compression`）。这就是 `apply_hermes_tuning.sh` 存在的根本原因。
3. **Tuning 补丁**：`apply_hermes_tuning.sh` 以 Python + PyYAML 直接 load/write 整个 YAML，只 `update()` 指定键，从而安全地设置 `model.*`、`providers.smartnpc-agent.*`、`compression.*`、`agent.tool_use_enforcement=false`。

### 3.2 模板与占位符

`render_profiles.sh` 用 GNU `sed` 在以下文件中执行替换：
- `{{NPC_NAME}}` — PascalCase 内部名（XiaMi、Abigail…）
- `{{NPC_DISPLAY}}` — 中文显示名
- `{{NPC_DIR}}` — 小写目录名（xiami、abigail…）
- `{{NPC_PORT}}` — gateway 端口
- `{{PEER_A_NAME}}` / `{{PEER_A_DISPLAY}}` / `{{PEER_B_NAME}}` / `{{PEER_B_DISPLAY}}` — 配对 NPC

脚本明确禁止直接编辑 rendered 文件（skills/、`config-overlay.yaml`、`cron-recipes.md`、`critical-policy.md`），必须改 `_master/` 后重新运行。

### 3.3 运行时环境变量解析

`runtime-config.yaml` 中的 `gateway_url` 使用 shell 风格变量：`http://${SMARTNPC_HERMES_GATEWAY_HOST}:8642`。注释说明 `smartnpc-mcp` 启动时会展开该变量，依次回退到 `$WSL_IP` 和 `127.0.0.1`。LLM 密钥则通过 `api_key_env: SMARTNPC_HERMES_KEY` 间接引用环境变量，避免明文泄露。

### 3.4 跨语言桥接

`npcs.yaml` 是唯一的“单点真相”：
- Bash 通过 `lib/npc_registry.sh` 解析它来渲染 profile。
- C# mod 通过 `smapi-mod/assets/agent_npcs.json`（同由 render_profiles.sh 生成）获知哪些 NPC 受管理。

## 4. 约定与约束

- **`npcs.yaml` 是唯一权威注册表**：所有 per-NPC 元数据必须在此声明，脚本据此防止 profile/render/runtime 配置漂移（见 `npcs.yaml` 顶部注释与 `render_profiles.sh` 的 sanity check）。
- **Port 唯一性校验**：`smartnpc_load_registry` 在解析时检查 `gateway_port` 非空、纯数字且全局唯一，重复端口会报错退出（`npc_registry.sh` 第 69–87 行）。
- **必填字段校验**：每个 NPC 条目必须提供 `game_name`、`display_name`、`gateway_port`、`kind`，否则解析失败（`npc_registry.sh` 第 58–68 行）。
- **Rendered 文件不可手改**：`render_profiles.sh` 文档明确声明 rendered per-NPC files 由脚本再生成，手动修改会在下次渲染时被覆盖。
- **`.env` 只 seed 一次**：若目标 profile 下已存在 `.env`，不会从 `_master/.env` 覆盖，保证每 NPC 可拥有独立 Langfuse/LLM 密钥。
- **Overlay 注入边界**：`reset_hermes_overlay.sh` 严格按字符串 marker `# ── injected by hermes/install.sh (smartnpc profile) ──` 截断 `config.yaml`，marker 之后的内容全部丢弃（保留之前备份 `.bak`）。
- **状态清理范围**：`reset_hermes_overlay.sh --wipe-state` 显式列出可删除的 transient 工件（`sessions/`、`state.db*`、`response_store.db*`、`compression.db*`、`conversations/`、`logs/`、`gateway_state.json` 等），未列出的长时记忆（`memories/`、`plans/`）被刻意保留。
- **`agent.tool_use_enforcement = false` 是强制调优项**：`apply_hermes_tuning.sh` 的注释解释这是为了避免 Hermes 向 system prompt 注入 "keep calling tools until task complete" 指令，从而与 SmartNPC “每次唤醒仅调用一次 `chat_say` 即结束回合” 的规则冲突。
- **Windows/WSL 兼容性**：`render_profiles.sh` 通过 WSL 命令提示如何运行，并使用兼容 Windows bash 的 `cp -r` 而非 `rsync`。