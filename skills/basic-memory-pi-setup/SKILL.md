---
name: basic-memory-pi-setup
description: 为一个 Pi 工作区配置 Basic Memory 连续性：写 .pi/basic-memory.json、选项目、开工作区信任、验证 recall/capture 通路。用户说「配上记忆」「/bm-status 显示未配置」「新机器装好了怎么接」时使用。
---

# 配置 Pi 的 Basic Memory 连续性

目标：产出一个明确的、项目级的配置文件，让路由可预测，并且**不改动全局默认项目**。

## 步骤

1. 先看 `/bm-status`。它显示的四行是关键：`bm` 是否找到、`project` 是否配置、`workspace trust`、`auto recall/capture` 是 on / off / blocked。
2. 问清这个工作区归哪个 Basic Memory 项目；不确定就用 `bm tool list-projects` 列出来让用户选。
3. 在**工作区**（不是更深处）建 `.pi/basic-memory.json`：

```json
{
  "transport": "cli",
  "project": "<项目名>",
  "captureFolder": "pi/sessions",
  "useHookFlow": true,
  "autoRecall": true,
  "autoCapture": true
}
```

放在更高一层，`/workspace` 下所有子目录都能命中；放在具体项目目录则只对该项目生效。

4. **要自动召回就必须开工作区信任**：`BASIC_MEMORY_PI_TRUST_WORKSPACE=1` 必须在 Pi 的进程环境里。它没有配置文件开关，只能走环境变量：

   - 交互式：`BASIC_MEMORY_PI_TRUST_WORKSPACE=1 pi`
   - systemd 服务：加 drop-in

     ```ini
     # /etc/systemd/system/<unit>.service.d/basic-memory-trust.conf
     [Service]
     Environment=BASIC_MEMORY_PI_TRUST_WORKSPACE=1
     ```

     然后 `daemon-reload` + 重启服务。

   信任是**工作区级**的开关：不开，`/bm-recall` 仍可手动用，但自动召回/捕获会显示 `blocked`。

5. 想用 MCP 模式（27 个工具、检索更快）：`pi install npm:pi-mcp-adapter`，然后把 `transport` 改成 `mcp`。MCP 模式同样要求显式项目 + 工作区信任。

6. 验证：`/bm-status` 应显示项目已配置、自动化已开启；再跑 `/bm-recall <主题>` 或 `/bm-capture 验证检查点`，确认笔记真的落地（`bm tail` 能看到）。

## 选项速查

| 键 | 默认 | 作用 |
|:--|:--|:--|
| `transport` | `cli` | `cli` 走 `bm` 命令；`mcp` 走 `pi-mcp-adapter` |
| `project` / `projectId` | 未设 | 用哪个项目；两个都设时 `projectId` 优先 |
| `captureFolder` | `pi/sessions` | 检查点写到哪里 |
| `recallTimeframe` | `7d` | 自动召回的搜索时间窗 |
| `useHookFlow` | `true` | 用 `bm hook ... --harness pi` 生命周期钩子 |
| `autoRecall` | `true` | 每会话首次发言前召回一次 |
| `autoCapture` | `true` | 会话收敛时自动落检查点 |
| `captureMinChars` | `80` | 少于这个长度不捕获 |
| `bmPath` / `bmCommand` | `bm` | 覆盖 CLI 启动方式（需 `BASIC_MEMORY_PI_TRUST_BM_COMMAND=1`） |

## 边界与坑

- **autoRecall 只捞检查点**：扩展写死了 `--type pi_session --after_date <recallTimeframe>`，只返回最近的 Pi 会话检查点，不返回知识笔记。想要知识召回，靠模型主动 `bm_recall` / `search_notes`。
- **检查点会累积**：`autoCapture` 开着，每个会话都往 `captureFolder` 写一篇。嫌吵就设 `autoCapture: false`。
- **别把会话归档塞进主知识库**：长会话笔记会在向量检索里霸榜（实测 top-3 六成被占），要留就单独建 BM 项目。
- 这个包**不改**你的全局默认项目；`.pi/basic-memory.json` 之外什么都不会动。
