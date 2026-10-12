# 其他 AI 接入（Codex、Cursor、VS Code、Claude Desktop 等）

这些客户端不认 Claude 插件，需要手动做两件事：**接入 figma-console MCP**，再**把使用规则告诉 AI**。前提同根目录 README：装好 Figma 桌面端和 Node.js 18+。

## 第一步：接入 figma-console

所有客户端填的都是同一组信息：命令 `npx`，参数 `-y`、`figma-console-mcp@latest`，**不需要填令牌**。

### Codex

- 设置界面（菜单名随版本可能不同）：Settings → MCP servers → Add MCP Server。名称 `figma-console`，Command 填 `npx`，Arguments 依次加 `-y` 和 `figma-console-mcp@latest`，环境变量留空，保存。
- 或编辑 `~/.codex/config.toml`，追加：

  ```toml
  [mcp_servers.figma-console]
  command = "npx"
  args = ["-y", "figma-console-mcp@latest"]
  ```

### Cursor / Windsurf / Claude Desktop / 其他支持 JSON 配置的客户端

在客户端的 MCP 配置文件里加上（已有其他服务就并到 `mcpServers` 里）：

```json
{
  "mcpServers": {
    "figma-console": {
      "command": "npx",
      "args": ["-y", "figma-console-mcp@latest"]
    }
  }
}
```

常见配置文件位置（Mac；Windows 请查各客户端文档）：

| 客户端 | 配置文件 |
|---|---|
| Cursor | `~/.cursor/mcp.json` |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` |
| Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` |

### VS Code（GitHub Copilot）

项目里建 `.vscode/mcp.json`：

```json
{
  "servers": {
    "figma-console": {
      "command": "npx",
      "args": ["-y", "figma-console-mcp@latest"]
    }
  }
}
```

改完配置都要**重启客户端**，然后按根目录 README「第一次连接 Figma」导入并运行 Desktop Bridge 插件。

## 第二步：把使用规则告诉 AI

把 [`plugins/box-figma-local/skills/box-figma-local/SKILL.md`](../plugins/box-figma-local/skills/box-figma-local/SKILL.md) 的正文放进 AI 的规则里。**只复制第二个 `---` 之后的内容**，开头那段 `---` 包起来的信息头不要贴。

| 客户端 | 放哪里 |
|---|---|
| Codex | 贴进 `~/.codex/AGENTS.md`（对所有项目生效）或项目根目录的 `AGENTS.md` |
| Cursor | 项目里 `.cursor/rules/` 新建一个规则文件贴进去 |
| VS Code（Copilot） | `.github/copilot-instructions.md` |
| 其他 | 客户端的「自定义指令 / Rules」设置 |

不放规则也能用，只是 AI 可能仍优先调用官方 Figma MCP；这时在对话里说一句「用 figma-console，不要用官方 Figma MCP」即可。

## 验证

对 AI 说：「用 figma-console 检查 Figma 连接状态」。返回 `Connected to Figma Desktop via WebSocket Bridge` 即成功。
