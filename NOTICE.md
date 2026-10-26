# NOTICE

本仓库是 [Basic Memory](https://github.com/basicmachines-co/basic-memory) 官方 Pi 扩展包的衍生作品。

## 上游

| 项 | 值 |
|:--|:--|
| 包名 | `@basicmemory/pi-basic-memory` |
| 版本 | `0.24.1` |
| 源码 | https://github.com/basicmachines-co/basic-memory （`integrations/pi`） |
| 许可证 | AGPL-3.0-or-later |
| 版权 | Copyright (C) Basic Machines 及贡献者 |

## 本衍生版的改动

1. **`extensions/` 未改动**，逐字节取自上游 `0.24.1`。升级路径：用上游新版覆盖该目录。
2. **技能中文化**：`skills/basic-memory-pi`、`skills/basic-memory-pi-setup` 按中文重写，工具面与上游一致。
3. **新增 `skills/memory-onboarding`**：该技能在上游的 Python 包 `data/skills/` 里（不在 Pi 扩展包内），
   且正文用 MCP 工具名（`write_note` / `search_notes` / `schema_infer` …）。本仓库版本已改写为
   `bm tool write-note` / `bm tool search-notes` / `bm schema infer` 等 CLI 等价调用，
   并补充 Pi 专属的配置落点（`AGENTS.md` 与 `.pi/basic-memory.json` 的分工）。
4. **README 补充实测数据**：CLI 与 MCP 通路的延迟差异、中文嵌入模型与维度的配置要求、
   `autoRecall` 只检索检查点的边界、把会话归档并入主知识库的检索代价。
5. **`package.json`** 改为本仓库的 name/version/repository，并声明三个技能。

除上述改动外，功能行为与上游一致。

## 许可证继承

上游为 AGPL-3.0-or-later，本衍生版同样以 AGPL-3.0-or-later 发布，完整许可证文本见 `LICENSE`。
分发本包时请保留本 NOTICE 与上游版权声明。
