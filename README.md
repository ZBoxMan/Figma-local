# box-figma-local：让 AI 读写 Figma 不再撞官方 MCP 限额

用 Claude、Codex、Cursor 等 AI 操作 Figma 时，官方 Figma MCP 很容易提示「超出调用限额」。这个插件让 AI 改走你电脑上 **Figma 桌面端里的一个本地插件** 来读写 Figma，不经过官方 MCP，也就不占那份限额。

- 不需要 Figma 令牌，不占官方 MCP 的配额；只读要能打开文件，让 AI 修改画布要有编辑权限
- 读结构、读文案、截图、读变量、在画布上新建和修改都能做
- 每个人用自己的 AI 账号和自己的 Figma 账号，互不影响

---

## 为什么会撞限额

官方 Figma MCP 的「读取类」工具（读结构、读设计上下文、截图、读变量）按 **Figma 账号** 计次，换 AI 客户端也不会重置。Figma 官方公布的额度（2026-10-12 核对，Figma 会调整，以[官方文档](https://developers.figma.com/docs/figma-mcp-server/rate-limits-access/)为准）：

| 席位 | Starter（免费） | Professional | Organization | Enterprise |
|---|---|---|---|---|
| View / Collab | 每月 20 次 | 每月 20 次 | 每月 20 次 | 每月 20 次 |
| Dev / Full | 官方表未单列 | 每天 200 次、每分钟 10 次 | 每天 200 次、每分钟 15 次 | 每天 600 次、每分钟 20 次 |

教育版按 Professional 的 Dev / Full 计。限额只针对读取类工具，部分写入类工具豁免。

AI 做一次稍复杂的设计稿检查，读几十次很正常，免费账号一下就用完了。

## 原理

```
AI 客户端 ──► figma-console（本机运行的 MCP 服务）──► Figma 桌面端里的 Desktop Bridge 插件 ──► 你的设计文件
```

- [figma-console-mcp](https://github.com/southleft/figma-console-mcp) 是第三方开源项目（MIT 协议，持续维护中）。
- Desktop Bridge 插件在 Figma 桌面端里执行 Figma 官方的 Plugin API，和你手动运行任何 Figma 插件一样，用的是你自己的 Figma 登录与文件权限。
- 本插件做两件事：**自动注册 figma-console**；附带一个技能 **box-figma-local**，规定 AI 读写 Figma 一律走 figma-console，不调用官方 MCP 的读取工具。

## 安装前准备

1. **Figma 桌面端**（网页版不行）：<https://www.figma.com/downloads/>
2. **Node.js 18 或更高**：终端输入 `node --version` 检查，没有就到 <https://nodejs.org> 装 LTS 版。
3. 对要操作的 Figma 文件有权限：只读需要能打开；要让 AI 修改画布，需要编辑权限。

## 安装

### Claude Cowork（Claude 桌面端）

1. 下载本仓库的 [`dist/box-figma-local.plugin`](dist/box-figma-local.plugin)。
2. 在 Cowork 里把这个文件发给 Claude（拖进对话框即可），出现插件卡片后点「安装」。如果你的版本没有出现安装卡片，就让 Claude「帮我安装这个插件」。
3. 重启 Claude 桌面端。

### Claude Code

```
/plugin marketplace add ZBoxMan/Figma-local
/plugin install box-figma-local@box-figma-local
```

装完重启 Claude Code。

### Codex、Cursor、VS Code 等其他 AI

这些客户端不认 Claude 插件，需要手动配两样东西，见 [docs/其他AI接入.md](docs/其他AI接入.md)。

## 第一次连接 Figma（每台电脑只做一次）

1. 先按上面装好插件并**重启 AI 客户端**。figma-console 第一次启动时会在 `~/.figma-console-mcp/plugin/` 生成桥接插件文件。检查一下 `~/.figma-console-mcp/plugin/manifest.json` 是否存在：不存在说明 figma-console 没启动起来，先回去检查 Node.js 和安装步骤。
2. 打开 Figma 桌面端和你的设计文件。
3. 菜单 **Plugins → Development → Import plugin from manifest…**，选择 `~/.figma-console-mcp/plugin/manifest.json`。
   - Mac 上 `~` 是你的用户目录；选择文件时按 `Cmd + Shift + .` 可以显示隐藏文件夹。
4. 运行 **Plugins → Development → Figma Desktop Bridge**，插件状态显示 `Local · ready`（据 figma-console 文档）即连接成功。

以后每次用：打开 Figma 文件，运行一下 Figma Desktop Bridge 插件就行。

## 试一下

对 AI 说：

> 检查 Figma 连接状态，然后列出这个文件的所有页面：<你的 figma.com 文件链接>

正常会返回「Connected to Figma Desktop via WebSocket Bridge」和页面列表。再试截图：

> 截一张这个画板的图：<画板链接>

## 常见问题

**提示连不上 Figma？**
确认 Figma 桌面端开着目标文件，并且 Desktop Bridge 插件在运行；还不行就重启 AI 客户端。

**我已经装了官方 Figma MCP，要删吗？**
不用删。技能规定 AI 读写 Figma 一律走 figma-console，不调用官方 MCP 的工具（除非你在当前对话里明确要求）。如果发现 AI 还是去调官方工具，可以在对话里提醒「用 box-figma-local」，或者把官方 Figma MCP 停用。

**同时开了好几个 AI 客户端会冲突吗？**
不会。每个客户端各自启动一个 figma-console，自动占用 9223–9232 中的空闲端口，Desktop Bridge 插件会扫描这个端口范围去连接。

**需要 Figma 令牌吗？**
基本功能不需要。评论、版本历史、团队库组件这几类工具走 Figma REST API，需要个人访问令牌；不配时它们会报「令牌未配置」，属正常。确实要用的话，在 MCP 配置的 `env` 里加 `FIGMA_ACCESS_TOKEN`，**令牌自己保管，不要提交到任何仓库，也不要发给 AI**。

**AI 会不会乱改我的设计稿？**
技能要求 AI 改画布前先说清要改哪里、征得你同意。重要文件建议先复制一份页面再让 AI 动手。

**安全吗？**
figma-console 通过 `npx -y figma-console-mcp@latest` 运行，每次启动会取 npm 上的最新版。介意自动更新的话，可以把插件里 `.mcp.json` 的 `@latest` 换成固定版本号（先到 <https://www.npmjs.com/package/figma-console-mcp> 查当前版本，写成 `@版本号`）；之后想升级时再手动改这个版本号。

## 仓库结构

```
.claude-plugin/marketplace.json        Claude Code 插件源清单
plugins/box-figma-local/
  .claude-plugin/plugin.json           插件信息
  .mcp.json                            自动注册 figma-console（不含令牌）
  skills/box-figma-local/SKILL.md      给 AI 的使用规则
dist/box-figma-local.plugin            Cowork 安装包（上面目录的 zip 打包）
docs/其他AI接入.md                      Codex、Cursor 等手动接入说明
LICENSE                                MIT 许可证
```

## 许可证

MIT，见 [LICENSE](LICENSE)。figma-console-mcp 是独立的第三方项目，遵循它自己的 MIT 协议。

## 更新记录

- **1.0.0**（2026-10-12）：首版。自动注册 figma-console；box-figma-local 技能规定读写 Figma 走本机插件、连接预检、写前确认、官方工具对照表。
