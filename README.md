# agentsessions-manifests

官方清单仓库（PR 贡献，在线更新适配器）

适配器清单集中维护在本官方仓库，所有使用者共享同一份在线清单，PR 合并即全量生效，无需重新发布客户端：

- **使用者**：设置页「适配器清单（远端仓库）」填入官方链接即可（或配置 `manifestRepo` 字段）：
  `https://raw.githubusercontent.com/odayou/agentsessions-manifests/main/agent-manifests.json`
  刷新间隔由客户端的 `manifestRefreshMinutes` 控制（默认 60 分钟，拉取失败自动回退内置/缓存），可在设置中更改
- **贡献者**：fork 官方清单仓库 → 在 `agent-manifests.json` 的 `manifests` 数组中新增/修正清单条目 → 提交 PR；合并后所有用户在下一个刷新周期自动获得
- **仓库维护者**：审阅并合并 PR；仓库种子文件即 [docs/manifest-repo/agent-manifests.json](docs/manifest-repo/agent-manifests.json)（顶层 `{"manifests": []}`）
- 合并优先级：用户目录（`~/.agentsessions/adapters/`，本地试验/私有清单用）> 远端 > 内置，同名 id 高优先级覆盖

**条目通用字段**：

| 字段      | 必填 | 说明                                                                                     |
| --------- | ---- | ---------------------------------------------------------------------------------------- |
| `manifestVersion` | ✅  | 固定为 `1`（条目级字段；顶层文件无版本字段要求）                                            |
| `id`      | ✅   | 小写 slug（`a-z0-9-`），勿与内置适配器重名（重名时硬编码优先）                             |
| `label`   | ✅   | 展示名（agent 列表 / 会话来源显示）                                                        |
| `engine`  | ✅   | `jsonl` / `json` / `sqlite` / `markdown` / `external`（**远端清单拒绝 `external`**）       |
| `detect`  | —    | 安装检测：`dirs[]`（目录，支持 `~` 前缀，任一存在即视为已装）、`bins[]`（PATH 中任一可执行） |
| `sourceFormat` | — | 格式描述串，仅展示用途                                                                  |

**jsonl / json 引擎**（逐文件记录流，参考内置 `qwen-code.json` / `continue-dev.json`）：

| 字段                       | 说明                                                                  |
| -------------------------- | --------------------------------------------------------------------- |
| `source.root`              | 会话目录（支持 `~`）                                                  |
| `source.ext`               | 会话文件扩展名，默认 `.jsonl`                                         |
| `source.cwdFromPath`       | `true` 时从文件路径反推项目目录（记录内无 cwd 字段时用）              |
| `session.id` / `cwd` / `model` / `ts` | 记录内的字段路径（会话 ID / 项目 / 模型 / 时间戳）           |
| `session.messagesPath`     | `json` 引擎必填：消息数组的字段路径                                   |
| `rules[]`                  | 字段→角色映射规则，至少一条，语法见下                                 |

**rules 语法**（数组，逐条匹配）：

| 字段              | 说明                                                                                          |
| ----------------- | --------------------------------------------------------------------------------------------- |
| `when`            | 可选 `{字段: 值}` 条件，记录命中才应用本条（如 `{"type": "user"}`）                            |
| `role`            | 目标角色：`user` / `assistant` / `thinking` / `title` / `tool`                                 |
| `text`            | `role` 为 user/assistant/thinking/title 时必填：内容字段路径（字符串或候选数组，取首个存在者） |
| `name` / `input`  | `role=tool` 时必填：工具名与入参的字段路径                                                    |
| `ts`              | 可选：该记录的时间字段路径                                                                    |

**sqlite 引擎**（单库多会话，参考内置 `mimo-code.json`）：

| 字段                    | 说明                                                        |
| ----------------------- | ----------------------------------------------------------- |
| `sqlite.candidates[]`   | 数据库路径数组（支持 `~`），逐库扫描                        |
| `sqlite.session`        | `{table, id, cwd}`：会话表与列                              |
| `sqlite.message`        | `{table, sessionId, data}`：消息表、会话外键与内容列        |
| `sqlite.part`           | 可选 `{table, messageId, data}`：分段内容表                 |
| `sqlite.mirrorTables[]` | 可选 `[{table, column}]`：镜像会话过滤                      |

**markdown 引擎**（多次运行累积的单文件日志，参考内置 `aider.json`）：

| 字段                        | 说明                                              |
| --------------------------- | ------------------------------------------------- |
| `source.fileName`           | 精确文件名，在 `source.root`（默认主目录）下递归匹配 |
| `markdown.runHeading`       | 分轮标题正则（每次命中开新会话）                  |
| `markdown.headingLevel`     | 标题级别（正整数）                                |
| `markdown.roleHeadings`     | `{user, assistant}`：轮内角色标题正则             |
| `markdown.ignoreHeadings[]` | 忽略的标题正则                                    |
| `markdown.filters[]`        | 行级过滤正则                                      |

> 安全与校验：远端清单不允许 `engine: "external"`（防止仓库被劫持后借扫描执行任意命令，声明即拒绝并记日志）；sqlite 的表/列名仅允许字母数字下划线（拼接 SQL 前白名单校验）；校验失败的条目跳过并记日志，不影响其他条目。

最小示例（jsonl 引擎）：

```json
{
  "manifestVersion": 1,
  "id": "example-tool",
  "label": "示例工具",
  "engine": "jsonl",
  "detect": { "dirs": ["~/.example-tool/sessions"], "bins": ["example-tool"] },
  "source": { "root": "~/.example-tool/sessions", "ext": ".jsonl", "cwdFromPath": true },
  "session": { "id": "sessionId", "cwd": "cwd", "model": "model", "ts": "timestamp" },
  "rules": [
    { "when": { "type": "user" }, "role": "user", "text": ["content", "text"] },
    { "when": { "type": "assistant" }, "role": "assistant", "text": ["content", "text"] },
    { "when": { "type": "tool_use" }, "role": "tool", "name": "name", "input": "arguments" },
    { "when": { "type": "ai-title" }, "role": "title", "text": "aiTitle" }
  ]
}
```
