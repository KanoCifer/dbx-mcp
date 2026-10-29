---
name: dbx-mcp-setup
description: 定位本机 DBX MCP Server 可执行文件并修正插件的 MCP 命令声明。当 dbx 服务启动失败、报 dbx-mcp-server 命令找不到、DBX 工具不可用，或用户要求安装/接入 DBX MCP 时使用。
---

# DBX MCP 接入

目标：让 `dbx` MCP Server 能在 MiniMax Code 里以 stdio 方式启动。

DBX 的 MCP Server 只有两种落位方式，可执行文件名不同：

| 来源 | 可执行文件 | 位置 |
| --- | --- | --- |
| 官方安装器，或在 DBX「设置 → MCP」里安装 MCP 服务 | `dbx-mcp` | `~/.dbx/bin/dbx-mcp` |
| npm 全局安装 `@dbx-app/mcp-server` | `dbx-mcp-server` | npm 全局 bin 目录 |

插件的 MCP 声明文件（本地安装路径）：

```
~/.minimax/plugins/dbx-mcp/dbx.mcp.json
```

若该路径不存在，先在 `~/.minimax/plugins/` 下找同名目录；插件由 Marketplace 安装时目录名可能不同。

## 1. 读当前声明

用 Read 工具打开上面的文件，记下 `mcpServers.dbx.command` 当前值。

## 2. 搜寻可执行文件

先查 PATH，再查常见落位目录：

```bash
command -v dbx-mcp-server dbx-mcp
ls -l ~/.dbx/bin/ 2>/dev/null
ls -l ~/.local/bin ~/.npm-global/bin /usr/local/bin /opt/homebrew/bin 2>/dev/null | grep dbx
npm bin -g 2>/dev/null
find ~ -maxdepth 5 -name 'dbx-mcp*' -not -path '*/Library/*' 2>/dev/null
```

Windows 落位：`%USERPROFILE%\.dbx\bin\dbx-mcp.exe`、`%APPDATA%\npm\dbx-mcp-server.cmd`。

## 3. 校验候选确实会说 MCP

对每个候选发一次 initialize，拿到 `serverInfo` 才算数：

```bash
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}' | <候选路径> 2>/dev/null
```

期望输出含 `"serverInfo"` 与 `"name":"dbx"`。不是 MCP 程序的候选会直接报错或没有输出。

## 4. 确认运行时 PATH 能解析到

GUI 启动的 MiniMax Code 不读 `.zshrc`，所以 `~/.dbx/bin` 里的文件通常不在进程 PATH 里：

```bash
echo "$PATH" | tr ':' '\n' | grep -qx "$HOME/.dbx/bin" && echo on-PATH || echo not-on-PATH
```

不在 PATH 时，在 PATH 内的目录建软链，让声明里的命令名可解析：

```bash
ln -sf "$HOME/.dbx/bin/dbx-mcp" "$HOME/.npm-global/bin/dbx-mcp-server"
```

软链目标目录先确认在 PATH 内（`~/.npm-global/bin`、`~/.local/bin` 是常见选择），也可换成 PATH 内其他目录。

## 5. 更新 MCP 声明

用 Edit 工具把 `mcpServers.dbx.command` 改成第 4 步确认过的可执行**名**，不能带 `/` 或 `\`。其余字段（`schemaVersion`、`type`、`description`、`timeout`）保持不变：

```json
{
  "schemaVersion": 1,
  "mcpServers": {
    "dbx": {
      "type": "stdio",
      "command": "dbx-mcp-server",
      "description": "通过本机 DBX 连接查询数据库：列出连接与表结构、执行 SQL、读取查询结果。",
      "timeout": 60000
    }
  }
}
```

改完校验 JSON 合法：`python3 -m json.tool ~/.minimax/plugins/dbx-mcp/dbx.mcp.json`

## 6. 验证

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
  | <命令名>
```

期望 `tools/list` 返回 `dbx_*` 工具列表（当前版本 16 个）。工具不会立刻出现在当前会话里，运行时重扫或新开会话后才可见。

## 7. 找不到可执行文件时

不要自己编造命令名。直接引导用户安装：

1. 打开 DBX（桌面端或 Web）。
2. 进入 **设置 → MCP**。
3. 在 MCP 服务区域安装 MCP 服务。装完后 DBX 会把 MCP Server 放到 `~/.dbx/bin`。
4. 装完回到第 2 步重跑搜寻，再更新声明。

用户说已经装过但仍搜不到时，把第 2 步的 `find` 范围扩到整个用户目录再找一次，并检查该路径是否在运行时 PATH 内（回到第 4 步）。
