# SmartNPC RepoWiki

星露谷物语（Stardew Valley）AI NPC 项目 **SmartNPC / Hermes** 的架构文档站，由 Qoder RepoWiki 自动生成，快照日期 2026-10-26，增量更新 2026-10-27（商店工具 / 日程冻结 / chat_say 白名单）。

内容对应的代码仓库是 [OmniStormX/SmartNPC](https://github.com/OmniStormX/SmartNPC)。本仓库只放文档，不含源码。

## 目录结构

```
wiki/        面向读者的 Wiki 正文（124 个页面，含 Mermaid 架构图）
knowledge/   按模块拆分的知识卡（概述 / 架构设计 / 技术栈 / 编码规范 / 构建系统 等）
```

## 从哪读起

先读 [项目概览与快速开始](<wiki/项目概览与快速开始.md>)，再按需要进具体章节：

| 章节 | 页面数 | 入口 |
| --- | --- | --- |
| 核心架构设计 | 33 | [核心架构设计](<wiki/核心架构设计/核心架构设计.md>) |
| SMAPI 模组开发（C# Mod） | 32 | [SMAPI 模组开发](<wiki/SMAPI 模组开发/SMAPI 模组开发.md>) |
| MCP 服务开发（Go 服务） | 24 | [MCP 服务开发](<wiki/MCP 服务开发/MCP 服务开发.md>) |
| API 接口参考 | 12 | [API 接口参考](<wiki/API 接口参考/API 接口参考.md>) |
| Hermes Agent Profiles | 9 | [Hermes Agent Profiles](<wiki/Hermes Agent Profiles/Hermes Agent Profiles.md>) |
| 扩展开发指南 | 6 | [扩展开发指南](<wiki/扩展开发指南/扩展开发指南.md>) |
| 高级特性 | 5 | [高级特性](<wiki/高级特性/高级特性.md>) |
| 开发与部署 | 1 | [开发与部署](<wiki/开发与部署.md>) |
| 故障排查与维护 | 1 | [故障排查与维护](<wiki/故障排查与维护.md>) |

## 阅读建议

GitHub 会直接渲染 Markdown 与 Mermaid 图，在线读即可。想把整套文档当知识库检索、看关系图，用 Obsidian 打开 `wiki/` 目录效果更好。

## 已知限制

这些文档是工具生成的，不是人工维护的，请注意几点：

每个页面开头的 `<cite>` 引用块列出它依据的源码文件，链接使用 Qoder 内部的 `file://` 方案，在 GitHub 上**不可点击**；路径以仓库根为基准，需要自己拼到 [SmartNPC](https://github.com/OmniStormX/SmartNPC) 里查看。

生成时的代码快照早于最近的目录重构，部分被引用路径已过期（例如 `smapi-mod` 曾被写作 `smap-mod`），若发现文档与实际代码不一致，以代码仓库为准。

文中出现的示例密钥（如 `smartnpc-test-key`、`your-api-key`）均为占位符；真实密钥走环境变量注入，未写入文档。

## 许可与来源

文档内容随项目源码的许可约定；如发现涉及不应公开的信息，请开 issue 指出，会立即处理。
