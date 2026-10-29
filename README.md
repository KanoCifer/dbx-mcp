# DBX 数据库助手（MiniMax Plugin）

把本机 [DBX](https://github.com/t8y2/dbx) 的数据库连接能力以 MCP 服务的形式接入 MiniMax Code：Agent 可以列出连接、查看表结构、执行 SQL 并读取结果。

## 包结构

```text
dbx-mcp/
  .minimax-plugin/plugin.json          # manifest 入口
  dbx.mcp.json                         # MCP 声明：stdio / dbx-mcp-server
  icon.png
  skills/
    dbx-mcp-setup/SKILL.md             # 定位 DBX MCP Server 并修正命令声明
  README.md
```

## 能力

- **MCP**：`dbx`，stdio 启动 `dbx-mcp-server`，暴露 16 个 `dbx_*` 工具（连接管理、表结构描述、SQL 执行、批量与事务、Redis 命令等）。
- **Skill**：`dbx-mcp-setup`。在 `dbx` 服务起不来时，定位环境内的 `dbx-mcp` / `dbx-mcp-server`，把 `~/.minimax/plugins/dbx-mcp/dbx.mcp.json` 的 `command` 改成可解析的命令名；环境里没有可执行文件时，引导用户到 DBX「设置 → MCP」安装 MCP 服务。

## 前置条件

插件不包含 DBX 本体，也不包含任何账号或连接配置。使用前需要：

1. 本机已安装 DBX MCP Server，且可执行文件在 `PATH` 中；
2. DBX 中已配置好数据库连接（由 DBX Desktop / Web 完成，插件不接管这部分）。

两种安装方式对应的可执行文件名不同，插件声明的是 `dbx-mcp-server`：

| 安装方式 | 可执行文件 | 说明 |
| --- | --- | --- |
| 官方安装器 / DBX「设置 → MCP」安装 | `~/.dbx/bin/dbx-mcp` | 安装器会写入 `~/.zshrc`，图形界面启动的 App 不读该文件 |
| npm 全局安装 `@dbx-app/mcp-server` | `dbx-mcp-server` | `@dbx-app/mcp-server` 的 bin 名 |

若本机只有 `dbx-mcp`，把它软链到一个在 `PATH` 上的目录即可：

```bash
ln -sf ~/.dbx/bin/dbx-mcp ~/.npm-global/bin/dbx-mcp-server
```

## MCP 配置

```json
{
  "mcpServers": {
    "dbx": {
      "type": "stdio",
      "command": "dbx-mcp-server"
    }
  }
}
```

`dbx.mcp.json` 中的完整声明额外带 `description` 与 `timeout`（60000ms，数据库查询可能较慢）。

## 本地验证

```bash
# MCP 握手与工具列表
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
  | dbx-mcp-server
```

## 打包投稿

在包目录内打包，让 ZIP 根目录直接就是包内容：

```bash
cd ~/Code/dbx-mcp
zip -r ../dbx-mcp.zip . -x '*.DS_Store' './.git/*'
```

ZIP 根目录需直接包含 `.minimax-plugin/plugin.json`；若压成 `dbx-mcp/.minimax-plugin/plugin.json` 这种多套一层目录的形式会被直接驳回。

提交入口见 [插件开发与上架指南](https://agent.minimax.cn/docs/code/agents/plugin-submission)。包内改动后需提升 `plugin.json` 中的 `version`。
