---
name: dbx-mcp-setup
description: 定位或安装 DBX MCP Server，并把插件的 MCP 命令声明改成可解析的可执行名。当 dbx 服务启动失败、报 dbx-mcp 或 dbx-mcp-server 命令找不到、DBX 工具不可用，或用户要求安装/接入 DBX MCP 时使用。
---

# DBX MCP 接入

目标：让 `dbx` MCP Server 能在 MiniMax Code 里以 stdio 方式启动。

DBX 官方 MCP Server 由一条命令安装：

```bash
curl -fsSL https://dbxio.com/install-mcp | sh
```

安装结果：

- 二进制落在 `~/.dbx/bin/dbx-mcp`，**可执行名是 `dbx-mcp`**；
- 安装器会把 `export PATH="$HOME/.dbx/bin:$PATH"` 追加到 `~/.zshrc`；
- Windows 用 `install-mcp.ps1`。

安装器给出的官方客户端配置用的是**绝对路径**：

```json
{"mcpServers":{"dbx":{"command":"/Users/<user>/.dbx/bin/dbx-mcp"}}}
```

MiniMax Plugin 的 `stdio.command` 只允许 PATH 中的可执行名、不能带路径，所以必须让 `dbx-mcp` 成为一个 PATH 可解析的名字。这是本技能存在的原因。

插件的 MCP 声明文件（本地安装路径）：

```
~/.minimax/plugins/dbx-mcp/dbx.mcp.json
```

若该路径不存在，先在 `~/.minimax/plugins/` 下找同名目录；插件由 Marketplace 安装时目录名可能不同。

## 1. 读当前声明

用 Read 工具打开上面的文件，记下 `mcpServers.dbx.command` 当前值。

## 2. 确认已安装

```bash
ls -l ~/.dbx/bin/ 2>/dev/null
~/.dbx/bin/dbx-mcp --version 2>/dev/null
```

没有输出就执行官方安装命令（会自行校验 npm 包 sha512 与 macOS 签名，已安装时输出 `already up to date`）：

```bash
curl -fsSL https://dbxio.com/install-mcp | sh
```

用户也可以不装命令行版本，改用 DBX 桌面端 **设置 → MCP** 安装 MCP 服务；安装器自己也是这么推荐的。二选一即可，不要同时装两套。

## 3. 搜寻可执行文件

先查 PATH，再查常见落位目录：

```bash
command -v dbx-mcp dbx-mcp-server
ls -l ~/.dbx/bin/ ~/.local/bin ~/.npm-global/bin /usr/local/bin /opt/homebrew/bin 2>/dev/null | grep dbx
npm bin -g 2>/dev/null
find ~ -maxdepth 5 -name 'dbx-mcp*' -not -path '*/Library/*' 2>/dev/null
```

Windows 落位：`%USERPROFILE%\.dbx\bin\dbx-mcp.exe`、`%APPDATA%\npm\dbx-mcp-server.cmd`。

## 4. 校验候选确实会说 MCP

对每个候选发一次 initialize，拿到 `serverInfo` 才算数：

```bash
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}' | <候选路径> 2>/dev/null
```

期望输出含 `"serverInfo"` 与 `"name":"dbx"`。不是 MCP 程序的候选会直接报错或没有输出。

## 5. 让命令名在运行时 PATH 里可解析

GUI 启动的 MiniMax Code 不读 `.zshrc`，所以 `~/.dbx/bin` 里的文件通常不在进程 PATH 里：

```bash
echo "$PATH" | tr ':' '\n' | grep -qx "$HOME/.dbx/bin" && echo on-PATH || echo not-on-PATH
```

不在 PATH 时，在 PATH 内的目录建软链，让声明里的命令名可解析：

```bash
ln -sf "$HOME/.dbx/bin/dbx-mcp" "$HOME/.local/bin/dbx-mcp"
```

`~/.local/bin` 与 `~/.npm-global/bin` 都在 PATH 内，任选其一。若 PATH 里只有 npm 启动器 `dbx-mcp-server`（`npm i -g @dbx-app/mcp-server` 装的），同样建软链，或改用官方安装器换掉 npm 版本——官方安装器明确建议：`npm rm -g @dbx-app/mcp-server`。

## 6. 更新 MCP 声明

用 Edit 工具把 `mcpServers.dbx.command` 改成第 5 步确认过的可执行**名**，不能带 `/` 或 `\`。其余字段（`schemaVersion`、`type`、`description`、`timeout`）保持不变：

```json
{
  "schemaVersion": 1,
  "mcpServers": {
    "dbx": {
      "type": "stdio",
      "command": "dbx-mcp",
      "description": "通过本机 DBX 连接查询数据库：列出连接与表结构、执行 SQL、读取查询结果。",
      "timeout": 60000
    }
  }
}
```

改完校验 JSON 合法：`python3 -m json.tool ~/.minimax/plugins/dbx-mcp/dbx.mcp.json`

## 7. 验证

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
  | dbx-mcp
```

期望 `tools/list` 返回 `dbx_*` 工具列表（当前版本 16 个）。改完 `command` 需要 MCP 客户端重新拉起服务才生效，工具不会立刻出现在当前会话里。

## 8. macOS 钥匙串授权

DBX 的连接凭据存在系统钥匙串里。MCP 客户端读不到 DBX 加密密钥时，连接会报密钥不可用，需要一次性授权：

```bash
~/.dbx/bin/dbx-mcp --authorize-keychain
```

弹出系统提示时选「Always Allow」（不是「Allow」），然后重启 MCP 客户端。DBX 桌面端在 **设置 → MCP** 里也能代做这一步。

## 9. 仍然找不到可执行文件时

不要自己编造命令名。直接引导用户：

1. 打开 DBX（桌面端或 Web）。
2. 进入 **设置 → MCP**。
3. 在 MCP 服务区域安装 MCP 服务。装完后 DBX 会把 MCP Server 放到 `~/.dbx/bin`。
4. 装完回到第 3 步重跑搜寻，再更新声明。

用户说已经装过但仍搜不到时，把第 3 步的 `find` 范围扩到整个用户目录再找一次，并回到第 5 步确认路径与 PATH 关系。
