# pondera-plugins

Pondera 桌面应用（Settings → 能力 → 插件 → 插件市场）读取的插件市场索引仓库。索引文件是 Claude 兼容的 `.claude-plugin/marketplace.json`，Pondera 用 `internal/pluginmarket` 解析，用 `internal/installsource` + `internal/pluginpkg` 安装插件包。

当前收录 3 个可直接安装的示例插件：`go-concurrency-review`（纯 skill）、`changelog-writer`（纯 slash command）、`docs-lookup-mcp`（MCP server 声明示例）。

## 目录结构

```
.claude-plugin/marketplace.json          # 索引（唯一入口）
plugins/
  go-concurrency-review/
    pondera-plugin.json                  # 原生插件清单（v2）
    skills/go-concurrency-review/SKILL.md
  changelog-writer/
    pondera-plugin.json
    commands/changelog.md
  docs-lookup-mcp/
    pondera-plugin.json                  # 仅声明 mcpServers，不含 server 实现
```

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

- `docs-lookup-mcp` 只声明 MCP server，不含 server 实现（示例，需自行提供 server）。
- 市场只做索引与安装，不校验插件内部代码质量；安装第三方插件前请自行审查。
