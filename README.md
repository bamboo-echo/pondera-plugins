# pondera-plugins

Pondera 桌面应用（Settings → 能力 → 插件 → 插件市场）读取的插件市场索引仓库。索引文件是 Claude 兼容的 `.claude-plugin/marketplace.json`，Pondera 用 `internal/pluginmarket` 解析，用 `internal/installsource` + `internal/pluginpkg` 安装插件包。

当前收录 53 个插件，覆盖六个分类与 13 个领域，每个分类都同时包含纯 skill、slash command、MCP server 声明三种形态；其中 6 条 MCP 条目只声明 server，不含实现（`description` 已注明「示例，需自行提供 server」）。

领域归属写在 `description` 的 `【领域】` 前缀里（索引没有 `tags` 字段，写了也不会被解析），面板分类仍只用六类取值。

## 领域覆盖

| 领域 | 条数 | 插件 |
| --- | --- | --- |
| 学术写作 | 3 | academic-paper-outline、citation-format-check、peer-review-response |
| 数学建模 | 3 | modeling-paper-workflow、model-selection-matrix、sensitivity-analysis-checklist |
| Blender | 3 | blender-python-geometry、blender-topology-audit、gltf-export-tuneup |
| CAD | 3 | ezdxf-drawing-script、freecad-parametric-macro、drawing-annotation-standard |
| 嵌入式 | 3 | stm32-init-review、kicad-netlist-check、serial-log-analysis |
| 数据分析 | 2 | pandas-cleaning-template、sql-plan-review |
| 影音 | 2 | ffmpeg-edit-recipes、subtitle-proofread |
| 教学 | 2 | quiz-generator、mistake-attribution |
| 法律合规 | 2 | contract-clause-diff、privacy-policy-audit |
| 本地化 | 2 | i18n-copy-review、long-text-segmentation |
| 运维 | 3 | systemd-unit-audit、nginx-config-check、log-triage-runbook |
| 测试 | 2 | test-case-gap-analysis、repro-steps-standardizer |
| 移动端/游戏 | 2 | flutter-layout-audit、unity-asset-naming |

涉及 Blender / FreeCAD / AutoCAD / KiCad / Unity 的条目**不自带建模或制图能力**：它们产出的是脚本、宏或检查清单，实际执行依赖用户已安装对应软件（`ezdxf`、`ffmpeg` 之类同理）。

## 目录结构

```
.claude-plugin/marketplace.json          # 索引（唯一入口）
plugins/<插件名>/
  pondera-plugin.json                    # 原生插件清单（v2）
  skills/<技能名>/SKILL.md               # 纯 skill 型
  commands/<命令名>.md                   # slash command 型
```

## 收录清单

| 分类 | skill | command | MCP 声明 |
| --- | --- | --- | --- |
| 开发工具 | go-concurrency-review、sql-migration-audit、stm32-init-review、systemd-unit-audit、log-triage-runbook、repro-steps-standardizer、flutter-layout-audit | api-contract-diff、dockerfile-slim、kicad-netlist-check、serial-log-analysis、nginx-config-check、test-case-gap-analysis、unity-asset-naming | godoc-index-mcp |
| 效率提升 | meeting-notes-to-tasks、long-doc-summary | weekly-report | calendar-digest-mcp |
| 界面设计 | a11y-audit、blender-python-geometry、ezdxf-drawing-script | design-token-check、blender-topology-audit、gltf-export-tuneup、freecad-parametric-macro、drawing-annotation-standard | color-contrast-mcp |
| 调研分析 | paper-fast-read、modeling-paper-workflow、pandas-cleaning-template | competitor-brief、model-selection-matrix、sensitivity-analysis-checklist、sql-plan-review | docs-lookup-mcp |
| 内容创作 | blog-outline、academic-paper-outline、subtitle-proofread、mistake-attribution、long-text-segmentation | changelog-writer、citation-format-check、peer-review-response、ffmpeg-edit-recipes、quiz-generator、i18n-copy-review | glossary-lint-mcp |
| 其他 | subscription-audit、privacy-policy-audit | rebalance-checklist、contract-clause-diff | ledger-import-mcp |

精选（`featured: true`）6 条：`go-concurrency-review`、`sql-migration-audit`、`meeting-notes-to-tasks`、`a11y-audit`、`paper-fast-read`、`blog-outline`。

## 索引格式

Pondera 只认这几个字段（多余字段会被忽略，但下面的字段名不能拼错）：

| 字段 | 说明 |
| --- | --- |
| `name`（必填） | 市场名，不能为空 |
| `owner` | 市场所有者，`{ "name", "url" }` |
| `metadata.pluginRoot` | 插件目录前缀，相对路径（如 `./plugins`），下面的 `source` 会拼在它后面 |
| `metadata.description` / `metadata.version` | 可选展示信息 |
| `plugins`（必填） | 1–64 条，超过 64 条整份索引报错 |

每条 `plugins[]` 条目：

| 字段 | 说明 |
| --- | --- |
| `name`（必填） | 插件名。必须是合法包名：字母或数字开头，仅含 `A-Za-z0-9._-`，最长 64 字符；重复名会让整份索引报错 |
| `source`（必填） | 相对路径字符串（如 `./go-concurrency-review`）；或固定 URL 对象 `{ "source": "url", "url": "https://github.com/owner/repo", "sha": "<40 位 commit>" }` |
| `description` | 一句话说明，显示在卡片上 |
| `version` | 版本号，显示为 `v1.2.0` |
| `author` | 字符串，或 `{ "name", "url" }`（也接受 `email`） |
| `category` | 中文分类，见下表 |
| `icon` / `logo` | 图标 URL 或本地相对路径；加载失败时 UI 回退到首字母 |
| `featured` | `true` 时进入「推荐」筛选与精选区 |
| `permissions` | 字符串数组，只作展示（如 `读取工作区源码`），不代表真实权限 |
| `sha` | 仅配合字符串 `source` 时的展示字段，必须是完整 40 位 commit；不参与相对路径解析 |

`category` 可用值（Pondera 会做归一化，未识别一律落到 `other`）：

| 写入索引 | 面板 chips |
| --- | --- |
| `开发工具` | 开发工具 |
| `效率提升` | 效率提升 |
| `界面设计` | 界面设计 |
| `调研分析` | 调研分析 |
| `内容创作` | 内容创作 |
| `其他` | 其他 |

`source` 解析规则：

- 本地源（目录 / `file://` / 索引文件路径）：相对 `source` 相对市场根目录（或 `metadata.pluginRoot`）解析，解析结果必须留在市场根目录内，越界会被跳过。
- GitHub 源（`https://raw.githubusercontent.com/<owner>/<repo>/<ref>/.claude-plugin/marketplace.json`）：相对 `source` 会被解析成 `https://github.com/<owner>/<repo>/tree/<ref>/<pluginRoot>/<source>`，Pondera 随后 `git clone` 该地址安装。
- 非 GitHub 的 HTTP 索引无法解析相对路径，这些条目会被跳过；这种情况请改用固定 URL 对象。

## 插件清单

每个插件目录必须有下列三种清单之一（按优先级）：

1. `pondera-plugin.json` —— Pondera 原生清单，必须声明 `"apiVersion": "pondera.io/plugin/v2"`；这是**严格模式**，出现未知字段会解析失败。
2. `.codex-plugin/plugin.json`
3. `.claude-plugin/plugin.json` —— Claude 兼容清单；`skills/`、`commands/`、`agents/`、`hooks/hooks.json`、`.mcp.json` 会按约定自动发现。

本仓库统一使用原生 v2 清单。合法字段：

```json
{
  "apiVersion": "pondera.io/plugin/v2",
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "一句话说明",
  "homepage": "https://...",
  "repository": "https://...",
  "contributes": {
    "skills": ["skills"],
    "commands": ["commands"],
    "agents": ["agents"],
    "prompts": ["prompts"],
    "themes": ["themes"],
    "hooks": { "SessionStart": [{ "command": "scripts/hello.ps1", "shell": "powershell" }] },
    "mcpServers": {
      "my-server": { "type": "stdio", "command": "my-server-mcp", "args": ["--root", "."] }
    }
  }
}
```

约束：

- `name` 必须与索引条目 `name` 完全一致，否则安装时会被拒绝。
- `skills` / `commands` / `agents` / `prompts` / `themes` 都是**插件目录内的相对路径**，不能是绝对路径或 `..`。
- `skills` 指向的目录里，技能是 `<目录>/<技能名>/SKILL.md`，frontmatter 需要 `name` 与 `description`。
- `commands` 指向的目录里，命令是扁平的 `<命令名>.md`，frontmatter 需要 `description`，可带 `argument-hint`，正文用 `$ARGUMENTS` / `$1..$N`。
- `mcpServers` 的键必须是合法名字；`auto_start: false` 表示安装后不自动拉起。
- 以 `skills` + `commands` + `hooks` + `mcpServers` + `agents` 全为空的原生包会被判定为「无可用能力」而拒绝安装。

## 新增插件

1. 在 `plugins/<插件名>/` 放 `pondera-plugin.json` 与真实内容（`skills/…/SKILL.md` 或 `commands/<名>.md`）。
2. 在 `.claude-plugin/marketplace.json` 的 `plugins` 数组里加一条，`name` 与清单里的 `name` 保持一致，`source` 写 `./<插件名>`。
3. 本地自检（不写仓库、不装依赖）：
   ```bash
   cd <pondera 源码仓库>
   go test -overlay=<overlay.json> ./internal/pluginmarket ./internal/installsource -run TestPonderaPlugins -v
   ```
   `overlay.json` 把临时测试文件映射到目标包路径即可，不会在仓库里留下任何文件。
4. 提交前确认索引条目数 ≤ 64、索引体积 ≤ 8 MiB。

## 切换本地 / 远程索引源

Pondera 把索引源存在 `%APPDATA%\pondera\plugin-marketplace-source.json`，格式：

```json
{ "source": "..." }
```

- 本地源（离线可用，改插件后立刻生效，无需重启）：
  `"source": "C:\\Users\\<你>\\Desktop\\dp1\\reasonix魔改\\pondera-plugins"`
  也接受 `file:///C:/.../pondera-plugins/.claude-plugin/marketplace.json`。
- 远程源（需要联网，走 ETag/Last-Modified 缓存，断网时回退到上次成功索引并标注「缓存」）：
  `"source": "https://raw.githubusercontent.com/bamboo-echo/pondera-plugins/main/.claude-plugin/marketplace.json"`

也可以在插件市场面板点「索引源」直接改，保存后会自动重新加载。改文件前先备份原文件。

## 已知限制

- 6 条 `*-mcp` 条目只声明 MCP server，不含 server 实现（示例，需自行提供 server）：`docs-lookup-mcp`、`godoc-index-mcp`、`calendar-digest-mcp`、`color-contrast-mcp`、`glossary-lint-mcp`、`ledger-import-mcp`。
- `rebalance-checklist` 与 `subscription-audit` 只做计算与流程建议，不构成投资建议。
- 市场只做索引与安装，不校验插件内部代码质量；安装第三方插件前请自行审查。
