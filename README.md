# mcp-multiplexer

MCP 服务器多路复用器——将多个 MCP 服务器合并为一个，自动为工具名添加服务器前缀以避免命名冲突。

## 功能

- **工具名前缀**：自动为工具名添加服务器名前缀（如 `serverName__toolName`）
- **多服务器支持**：同时连接多个底层 MCP 服务器
- **透明代理**：将工具调用路由到正确的底层服务器
- **自定义分隔符**：可配置服务器名与工具名之间的分隔符
- **CLI 支持**：通过 `npx` 直接运行，无需编写脚本

## 安装

```bash
npm install mcp-multiplexer
```

## 使用方法

### CLI 用法（推荐，无需脚本文件）

```bash
# 使用配置文件
npx mcp-multiplexer --config ./config.json

# 使用内联 JSON
npx mcp-multiplexer --config '{"name":"mcp-multiplexer","version":"1.0.0","servers":[{"name":"github","command":"npx","args":["-y","@modelcontextprotocol/server-github"]}]}'

# 使用环境变量
export MCP_WRAPPER_CONFIG='{"name":"mcp-multiplexer","version":"1.0.0","servers":[...]}'
npx mcp-multiplexer
```

配置文件示例（`config.json`）：

```json
{
  "name": "mcp-multiplexer",
  "version": "1.0.0",
  "separator": "__",
  "servers": [
    {
      "name": "github1",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "token1" }
    },
    {
      "name": "github2",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "token2" }
    },
    {
      "name": "remote",
      "url": "http://localhost:3000/sse"
    }
  ]
}
```

### 编程用法

```typescript
import { McpWrapper } from "mcp-multiplexer";

const wrapper = new McpWrapper({
  name: "mcp-multiplexer",
  version: "1.0.0",
  servers: [
    {
      name: "github",
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-github"],
      env: { GITHUB_TOKEN: "your-token" },
    },
    {
      name: "filesystem",
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
    },
    {
      name: "remote",
      url: "http://localhost:3000/sse",
    },
  ],
  separator: "__", // 可选，默认为 "__"
});

// 连接所有底层服务器
await wrapper.connectToServers();

// 启动多路复用服务器
await wrapper.start();
```

### 配置说明

```typescript
interface WrapperConfig {
  name: string;                    // 多路复用服务器名称
  version?: string;                // 版本号（默认 "1.0.0"）
  servers: WrappedServerConfig[];  // 底层 MCP 服务器列表
  separator?: string;              // 分隔符（默认 "__"）
}

interface WrappedServerConfig {
  name: string;                    // 此服务器的唯一前缀名
  command?: string;                // 启动命令（stdio 传输）
  args?: string[];                 // 命令参数
  env?: Record<string, string>;    // 环境变量
  cwd?: string;                    // 工作目录
  url?: string;                    // MCP 服务器 URL（HTTP 传输）
}
```

> `command` 和 `url` 必须二选一，不可同时指定或同时缺省。

#### 服务器类型

**1. Stdio 服务器（使用 `command`）**

作为子进程启动，通过 stdin/stdout 通信：

```json
{
  "name": "local-server",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-github"],
  "env": { "GITHUB_TOKEN": "your_token" }
}
```

**2. HTTP 服务器（使用 `url`）**

通过 HTTP 访问。自动优先尝试 Streamable HTTP（现代协议），失败则回退到 SSE（旧协议）：

```json
{
  "name": "remote-server",
  "url": "http://localhost:3000/mcp"
}
```

### 在 Claude Desktop 中使用

**方法一：CLI + 配置文件（推荐）**

```json
{
  "mcpServers": {
    "mcp-multiplexer": {
      "command": "npx",
      "args": ["-y", "mcp-multiplexer", "--config", "/path/to/config.json"]
    }
  }
}
```

**方法二：内联 JSON（无需任何文件）**

```json
{
  "mcpServers": {
    "mcp-multiplexer": {
      "command": "npx",
      "args": ["-y", "mcp-multiplexer", "--config", "{\"name\":\"mcp-multiplexer\",\"version\":\"1.0.0\",\"servers\":[{\"name\":\"github1\",\"command\":\"npx\",\"args\":[\"-y\",\"@modelcontextprotocol/server-github\"],\"env\":{\"GITHUB_TOKEN\":\"token1\"}}]}"]
    }
  }
}
```

### 在 VS Code（GitHub Copilot）中使用

在工作区创建 `.vscode/mcp.json`：

```json
{
  "servers": {
    "mcp-multiplexer": {
      "command": "npx",
      "args": ["-y", "mcp-multiplexer", "--config", "/path/to/config.json"]
    }
  }
}
```

配置完成后，所有底层服务器的工具都会以前缀名形式出现：
- `github1__create_issue`
- `github1__list_pull_requests`
- `github2__create_issue`
- `github2__list_pull_requests`

## 错误处理与容错

### 优雅降级

当部分底层服务器连接失败时：
- 复用器会**继续启动**，使用已成功连接的服务器
- 失败的连接信息会输出到 stderr
- 已连接服务器的工具仍然可用
- 客户端（Claude Desktop、VS Code 等）可以正常初始化

### 连接超时

每个底层服务器有 **30 秒连接超时**，防止因服务器无响应导致复用器卡死。

### 示例场景

**场景一**：部分服务器失败
```
Failed to connect to server "unavailable": Error: spawn ENOENT
Skipping server "unavailable", will continue with remaining servers
Connected to server "github" with 15 tools
Connected to server "filesystem" with 8 tools
Successfully connected to 2 server(s)
Failed to connect to 1 server(s): unavailable
```

**场景二**：所有服务器失败
```
Failed to connect to server "server1": Error: spawn ENOENT
Failed to connect to server "server2": Connection timeout after 30000ms
Successfully connected to 0 server(s)
Failed to connect to 2 server(s): server1, server2
```

即使在场景二中，复用器也能正常启动（0 个工具），客户端可以完成初始化并收到清晰的错误信息。

## 故障排除

### HTTP 连接错误

**HTTP 405（Method Not Allowed）**：URL 不支持 MCP 协议，请检查是否指向了正确的 MCP 端点。

**HTTP 404（Not Found）**：端点不存在，请检查 URL 是否正确、服务器是否在运行。

## API

### McpWrapper

主类，用于创建多路复用服务器。

```typescript
new McpWrapper(config: WrapperConfig)
```

#### 方法

- `connectToServers(): Promise<void>` — 连接所有配置的底层 MCP 服务器
- `start(): Promise<void>` — 通过 stdio 传输启动多路复用服务器
- `close(): Promise<void>` — 关闭所有连接并停止服务器
- `getWrappedTools(): Array<WrappedToolInfo>` — 返回所有已包装工具的信息
- `reconnectServer(name: string): Promise<{success, message}>` — 尝试重连失败的服务器

### 工具函数

```typescript
// 创建带前缀的工具名
prefixToolName(serverName: string, toolName: string, separator?: string): string

// 解析带前缀的工具名
parseToolName(prefixedName: string, separator?: string): { serverName: string; originalName: string } | null

// 验证服务器名（不包含分隔符）
isValidServerName(serverName: string, separator?: string): boolean

// 验证工具名（不包含分隔符）
isValidToolName(toolName: string, separator?: string): boolean

// 获取 Windows 上需要补充的环境变量
getSupplementalEnv(): Record<string, string>
```

## 许可证

Apache-2.0
