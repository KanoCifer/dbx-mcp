# DBX 数据库助手（MiniMax Plugin）

把本机 [DBX](https://github.com/t8y2/dbx) 的数据库连接能力以 MCP 服务的形式接入 MiniMax Code：Agent 可以列出连接、查看表结构、执行 SQL 并读取结果。

## 包结构

```text
dbx-mcp/
  .minimax-plugin/plugin.json          # manifest 入口
  dbx.mcp.json                         # MCP 声明：stdio / dbx-mcp
  icon.png
  skills/
    dbx-mcp-setup/SKILL.md             # 定位 DBX MCP Server 并修正命令声明
  README.md
```

## 能力

- **MCP**：`dbx`，stdio 启动 `dbx-mcp`，暴露 16 个 `dbx_*` 工具（连接管理、表结构描述、SQL 执行、批量与事务、Redis 命令等）。
- **Skill**：`dbx-mcp-setup`。在 `dbx` 服务起不来时，用官方安装命令确认/安装 DBX MCP Server，定位环境内的 `dbx-mcp` / `dbx-mcp-server`，把 `~/.minimax/plugins/dbx-mcp/dbx.mcp.json` 的 `command` 改成可解析的命令名；环境里没有可执行文件时，引导用户到 DBX「设置 → MCP」安装 MCP 服务。

## 前置条件

插件不包含 DBX 本体，也不包含任何账号或连接配置。使用前需要：

1. 本机已安装 DBX Desktop 和  MCP Server，且可执行文件在 `PATH` 中；
2. DBX 中已配置好数据库连接（由 DBX Desktop / Web 完成，插件不接管这部分）。

官方安装命令：

```bash
curl -fsSL https://dbxio.com/install-mcp | sh
```

安装结果与替代方式：

| 安装方式 | 可执行文件 | 说明 |
| --- | --- | --- |
| 官方安装器 | `~/.dbx/bin/dbx-mcp` | 安装器会写入 `~/.zshrc`，图形界面启动的 App 不读该文件 |
| DBX Desktop「设置 → MCP」安装 | `~/.dbx/bin/dbx-mcp` | 官方安装器自己也推荐这条路径 |
| npm 全局安装 `@dbx-app/mcp-server` | `dbx-mcp-server` | npm 启动器名，官方安装器建议改用官方二进制 |

安装器给出的客户端配置用的是绝对路径（`~/.dbx/bin/dbx-mcp`），而 MiniMax Plugin 的 `stdio.command` 不允许带路径，所以还需要让 `dbx-mcp` 在运行时 PATH 里可解析：

```bash
ln -sf ~/.dbx/bin/dbx-mcp ~/.local/bin/dbx-mcp
```

macOS 上 DBX 的连接凭据存在系统钥匙串里，MCP 客户端还需要一次性授权才能读取加密密钥：

```bash
~/.dbx/bin/dbx-mcp --authorize-keychain
```

弹出系统提示时选「Always Allow」，然后重启 MCP 客户端。

## MCP 配置

```json
{
  "mcpServers": {
    "dbx": {
      "type": "stdio",
      "command": "dbx-mcp"
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
  | dbx-mcp
```

## 许可

MIT，见 [LICENSE](LICENSE)。本插件不包含 DBX 本体代码，也不分发任何凭据；DBX MCP Server 由用户自行安装，遵循其上游项目的许可。
