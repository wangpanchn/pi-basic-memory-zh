# pi-basic-memory-zh

给 [Pi](https://www.npmjs.com/package/@earendil-works/pi-coding-agent) 用的 [Basic Memory](https://github.com/basicmachines-co/basic-memory) 扩展包 —— **中文适配版**。

装完之后，Pi 有了跨会话的长期记忆：开会话能接上上次的线索，工作到关键处把现场存成笔记，检索走本地知识图谱而不是靠模型想起来。

这个仓库是上游 `@basicmemory/pi-basic-memory` 的衍生版：**扩展代码保持上游原样**（便于跟随上游更新），改动集中在**中文技能包**和**实战中踩出来的配置**。

---

## 依赖

| 项 | 要求 | 说明 |
|:--|:--|:--|
| Pi | ≥ 0.85.1 | 扩展/技能加载依赖 |
| Node.js | ≥ 22.19 | Pi 的要求 |
| Basic Memory CLI | ≥ 0.24 | 必须能在 Pi 的 PATH 里调到 `bm` |

装 CLI（推荐 uv，别用 `pip` 装到系统环境）：

```bash
curl -LsSf -o /tmp/uv-install.sh https://astral.sh/uv/install.sh && bash /tmp/uv-install.sh
uv tool install basic-memory --with socksio
bm --version    # 应输出 0.24.x 或更高
```

> `--with socksio` 不是可选项。如果你的环境变量里有 `ALL_PROXY=socks5://...`，缺 socksio 会让向量索引**静默失败**——关键词搜索照常工作，所以很容易误判成"装好了"。判断标准是 `bm status` 里 embeddings 阶段有没有报错。

---

## 安装

```bash
# 从 GitHub（推荐，可跟版本标签）
pi install git:github.com/wangpanchn/pi-basic-memory-zh

# 或本地开发时直接从路径加载
pi install /path/to/pi-basic-memory-zh

# 只想试一次，不进 settings
pi -e git:github.com/wangpanchn/pi-basic-memory-zh
```

装完重启 Pi（或 `/reload`），`pi list` 里应能看到本包。

---

## 先卸掉上游包（重要）

本包与上游 `@basicmemory/pi-basic-memory`（以及 `bm install pi` 装出来的 `packages/basic-memory`）
**不能同时安装** —— 两者都注册 `bm_recall` / `bm_capture`，加载时会直接报
`Tool "bm_recall" conflicts with ...`。

```bash
pi remove npm:@basicmemory/pi-basic-memory   # 若装的是 npm 版
# 若之前用 bm install pi 装过，把 ~/.pi/agent/settings.json 里的 "packages/basic-memory" 删掉
pi list                                      # 确认只剩 pi-basic-memory-zh
```

## 三步接上

**1. 建一个知识库项目**（如果还没有）

```bash
bm project add main ~/my-memory
```

**2. 在工作区写路由文件**

```json
// <workspace>/.pi/basic-memory.json
{
  "transport": "cli",
  "project": "main",
  "captureFolder": "pi/sessions",
  "useHookFlow": true,
  "autoRecall": true,
  "autoCapture": true
}
```

放在 `/workspace/.pi/` 这类**更高一层**目录，其下所有子目录都能命中。

**3. 开工作区信任**（自动召回/捕获的前提）

```bash
BASIC_MEMORY_PI_TRUST_WORKSPACE=1 pi
```

systemd 服务用 drop-in：

```ini
# /etc/systemd/system/<unit>.service.d/basic-memory-trust.conf
[Service]
Environment=BASIC_MEMORY_PI_TRUST_WORKSPACE=1
```

`daemon-reload` + 重启服务。不开信任也能手动 `/bm-recall`、`/bm-capture`，只是自动化显示 `blocked`。

用 `/bm-status` 验证四行：`bm` 找到 → `project` 已配置 → `trust` on → `auto recall/capture` on。

---

## 两条通路，别搞混

| | CLI（默认） | MCP |
|:--|:--|:--|
| 需要什么 | 只要 `bm` 在 PATH | 额外 `pi install npm:pi-mcp-adapter` |
| 模型能调用 | `bm_recall` / `bm_capture` | 上面两个 + 27 个 MCP 工具（`search_notes` / `build_context` / `write_note` …） |
| 单次检索耗时 | **约 10 秒**（每次进程启动都要导入整个 MCP 工具面） | 会话内首次约 2 秒，之后 **约 0.14 秒** |

**想高频检索就上 MCP。** 从 CLI 换过来：

```bash
pi install npm:pi-mcp-adapter
# .pi/basic-memory.json 里把 "transport": "cli" 改成 "mcp"
```

MCP 模式下每个 Pi 会话会拉起一个 stdio MCP 服务，模型只加载一次。如果想跨会话共用、连启动都省掉，可以自己常驻一个：

```bash
bm mcp --transport streamable-http --host 127.0.0.1 --port 8765 --project main
```

---

## 中文知识库必须调的两个参数

默认的嵌入模型是英文小模型，中文语义检索基本无效（同一句改写后的问法，正确笔记根本不进 top-3）。换成中文模型：

```bash
bm config set semantic_embedding_model "BAAI/bge-small-zh-v1.5"
bm config set semantic_embedding_dimensions 512     # 必须同时改，否则嵌入全报维度错误
bm reindex --embeddings --project main              # 重算；48 篇约 1 分钟
```

换完再用同一句自然语言问法验证一次，正确笔记应该排第一。**如果忘了改 dimensions，会报 `model returned 512-dimensional vectors but provider was configured for 384`。**

---

## 已知边界（实测数据）

| 事实 | 影响 | 怎么办 |
|:--|:--|:--|
| autoRecall 写死了 `--type pi_session --after_date 7d` | **只返回最近的 Pi 会话检查点，不返回知识笔记** | 知识召回靠模型主动 `bm_recall` / `search_notes` |
| `autoCapture` 每会话写一篇检查点 | 检查点会累积 | 嫌吵设 `autoCapture: false` |
| 长会话笔记会挤占向量检索 | 把 136 个历史会话并入主库后，top-3 有 60% 位子被会话占掉，命中率 13/14 → 10/14 | 会话归档单独建一个 BM 项目，别混进知识库 |
| CLI 每次调用约 10 秒 | 高频检索体验差 | 用 MCP 模式 |
| 中文数字日期（"十月十七号"）检索不到 | 两边都失败 | 用 ISO 日期 `2026-10-17` |

---

## 包内有什么

```
extensions/                  # 与上游一致，未改动
├── index.ts                 # 扩展入口：会话钩子、斜杠命令、MCP 注册
├── config.ts                # .pi/basic-memory.json 解析与默认值
├── session.ts               # 会话状态
└── bm-cli.ts                # 调用 bm CLI
skills/
├── basic-memory-pi/         # 中文：什么时候召回、捕获写什么、写笔记硬规则
├── basic-memory-pi-setup/   # 中文：配置路由、工作区信任、MCP 切换
└── memory-onboarding/       # 中文：从零设计知识体系（schema/指令笔记/启动路由），已按 bm CLI 改写工具调用
```

`memory-onboarding` 原本是给 MCP 工具面写的，这里改写成 `bm tool …` / `bm schema …` 等 CLI 等价命令，所以在 `transport: cli` 下也能跑通全流程。

---

## 与上游的差异

只列改动，扩展逻辑没动：

| 项 | 上游 | 本仓库 |
|:--|:--|:--|
| 技能语言 | 英文 | 中文 |
| `memory-onboarding` | 不包含（在 Python 包的 wheel 里，且用 MCP 工具名） | 包含，已改写为 `bm` CLI 调用 |
| 文档 | 英文，只讲怎么装 | 中文，补充实测数据：CLI/MCP 延迟、中文嵌入模型配置、autoRecall 边界、会话归档掺入的代价 |
| 版本 | `@basicmemory/pi-basic-memory` | 跟随上游版本号 + `-zh.N` 后缀 |

追踪上游：`extensions/` 直接取自 `@basicmemory/pi-basic-memory@0.24.1`，可整目录替换后重发。

---

## 许可证

**AGPL-3.0-or-later**，与上游一致。详见 [LICENSE](./LICENSE) 与 [NOTICE.md](./NOTICE.md)。

由于是 AGPL：如果你把本包部署成对他人提供服务的网络应用，需要向使用者提供对应源码。
