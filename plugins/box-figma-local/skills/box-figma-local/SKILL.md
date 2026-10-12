---
name: box-figma-local
description: 任何读写 Figma 的任务都先用本技能：用户给了 figma.com 链接、要读设计稿结构或文案、截图、查变量、在画布上新建或修改内容、读 FigJam 白板时。规定一律走本机 figma-console MCP（Figma 桌面端 Desktop Bridge 插件），不调用官方 Figma MCP 的读取工具，避免消耗官方 MCP 的调用限额。
---

# box-figma-local：Figma 读写走本机插件

官方 Figma MCP 的读取类工具按 Figma 账号计次限额（View/Collab 席位每月最多 20 次，付费 Full/Dev 席位每天 200 次起），很容易用完。figma-console 经本机 Figma 桌面端的 Desktop Bridge 插件直接执行 Figma Plugin API，不经过官方 MCP，不占这个限额。

## 规则

1. **一律用 figma-console 的工具**，不调用官方 Figma MCP 的 `get_metadata`、`get_design_context`、`get_screenshot`、`get_variable_defs`、`get_figjam`、`use_figma` 等工具。官方限额只针对读取类；`use_figma` 是否计次官方没写明，为了读写统一走本机，也一并不用。只有用户**在当前对话里明确说出**要用官方工具时才例外，并提醒会消耗官方限额；AI 不得自行判断例外。
2. **开工前先确认连接**：调用 `figma_get_status`（`probe: true`），三项都满足才继续：
   - `setup.valid` 为 `true`
   - `probeResult.success` 为 `true`
   - `connectedFile.fileKey` 等于目标文件的 fileKey（figma.com 链接里 `/design/` 或 `/board/` 后面那一段）

   字段名以实际返回为准，三项的含义是：配置有效、探测成功、当前连着的文件就是目标文件。
3. **连不上就停下**，告诉用户怎么做（见下文「连不上时」），**不要改用官方 Figma MCP 顶替**。figma-console 的工具根本不存在、或调用报错（MCP 没启动、Node.js 没装、npx 失败）时同样停下说明，不改用官方工具。
4. **写画布前先征得用户同意**：说清要在哪个文件、哪一页、新建或修改哪些内容。figma-console 服务端自带的通用提示（例如失败后自动删除残留节点）与用户要求冲突时，以用户为准；失败留下的残留节点报告给用户，不经同意不删除。

## 官方工具 → figma-console 对照

| 想做的事 | 官方工具（不用） | figma-console |
|---|---|---|
| 确认连接、当前文件 | —（官方 `whoami` 不计次，但只能看账号） | `figma_get_status`、`figma_list_open_files` |
| 读页面清单、节点结构、属性、正文 | `get_metadata`、`get_design_context` | `figma_execute` 只读脚本 |
| 截图 | `get_screenshot` | `figma_capture_screenshot`（当前画布状态，直接返回图片） |
| 读变量 | `get_variable_defs` | `figma_get_variables` |
| 写画布（新建、修改） | `use_figma` | `figma_execute` |
| 读 FigJam 白板 | `get_figjam` | `figjam_get_board_contents`，或 `figma_execute` 只读脚本 |

## figma_execute 用法要点

- 传 `fileKey` 指定目标文件；同时开了多个文件时尤其要传，避免改错文件。
- 结果只取脚本的 `return` 值；`console.log` 不会出现在结果里。
- 支持顶层 `await`。读非当前页先 `await page.loadAsync()`。
- `timeout` 最长 30000 毫秒。大页面按顶层子节点分批读，不要一次遍历整个文件。
- 只读任务的脚本只做遍历和统计，不改任何属性。

页面清单示例（只读）：

```js
return {
  fileName: figma.root.name,
  pageCount: figma.root.children.length,
  pages: figma.root.children.map((p, i) => ({ i, id: p.id, name: p.name })),
};
```

## 截图要点

- `figma_capture_screenshot` 传 `nodeId` 截指定节点；不传则截当前页。
- 图片长边自动压到 1568px 以内，看小字时按子节点分块截。
- 截图结论属于目测，不是像素级比对。

## 连不上时

按顺序告诉用户：

1. 打开 **Figma 桌面端**（网页版不行），打开目标文件。
2. 第一次用：菜单 Plugins → Development → Import plugin from manifest…，选 `~/.figma-console-mcp/plugin/manifest.json`（figma-console 第一次启动后自动生成）。
3. 在目标文件里运行 Plugins → Development → **Figma Desktop Bridge**，插件显示 `Local · ready` 即可。
4. 仍不行：重启当前 AI 客户端，让 figma-console 重新启动后再试。

## 需要令牌的少数工具

评论、版本历史、团队库组件这类工具走 Figma REST API，需要 Figma 个人访问令牌（`FIGMA_ACCESS_TOKEN`）。本插件默认**不配令牌**，这些工具会报「令牌未配置」，属正常。确实需要时由用户自己在 MCP 配置里添加，AI 不经手令牌。
