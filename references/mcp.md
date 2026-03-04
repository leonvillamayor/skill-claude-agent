# MCP Servers — Deep Reference

## MCP Tool Naming Convention
`mcp__<server-name>__<tool-name>`

Examples:
- Server `"github"` + tool `create_issue` → `mcp__github__create_issue`
- Server `"playwright"` + tool `browser_click` → `mcp__playwright__browser_click`
- Wildcard: `"mcp__github__*"` allows all github tools

## Configuration Methods

### 1. Inline (in code)

```python
# Python
options = ClaudeAgentOptions(
    mcp_servers={
        "playwright": {
            "command": "npx",
            "args": ["@playwright/mcp@latest"]
        }
    },
    allowed_tools=["mcp__playwright__*"]
)
```

### 2. .mcp.json file (auto-loaded)

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"],
      "env": { "NODE_ENV": "production" }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"]
    },
    "remote-api": {
      "type": "sse",
      "url": "https://api.example.com/mcp/sse",
      "headers": {
        "Authorization": "Bearer ${API_TOKEN}"
      }
    }
  }
}
```
The `${VAR}` syntax expands environment variables at runtime.

Load from filesystem with `setting_sources=["project"]`.

## Transport Types

### stdio (local process)
```python
mcp_servers = {
    "my-server": {
        "command": "npx",         # or "uvx", "python", executable path
        "args": ["-y", "@org/mcp-server", "--flag", "value"],
        "env": { "API_KEY": os.environ["API_KEY"] }
    }
}
```

### HTTP (remote, non-streaming)
```python
mcp_servers = {
    "remote": {
        "type": "http",
        "url": "https://api.example.com/mcp",
        "headers": { "Authorization": f"Bearer {token}" }
    }
}
```

### SSE (remote, streaming)
```python
mcp_servers = {
    "stream-api": {
        "type": "sse",
        "url": "https://api.example.com/mcp/sse",
        "headers": { "X-API-Key": os.environ["API_KEY"] }
    }
}
```

### SDK MCP Server (in-process)
```python
from claude_agent_sdk import create_sdk_mcp_server, tool
custom_server = create_sdk_mcp_server(name="my-tools", version="1.0.0", tools=[my_tool])
mcp_servers = {"my-tools": custom_server}
```

### claudeai-proxy (TypeScript only)
```typescript
mcpServers: {
  "proxy-server": {
    type: "claudeai-proxy",
    url: "https://proxy.example.com/mcp",
    id: "server-id"
  }
}
```

## Dynamic MCP Management (TypeScript)

The TypeScript `Query` object supports runtime MCP server control:

```typescript
const q = query({ prompt: "...", options });

// Replace all MCP servers
const result = await q.setMcpServers({
  "new-server": { command: "npx", args: ["-y", "some-server"] }
});
// result: McpSetServersResult { added: string[], removed: string[], errors: string[] }

// Reconnect a failed server
await q.reconnectMcpServer("github");

// Enable/disable a server
await q.toggleMcpServer("github", false);
await q.toggleMcpServer("github", true);

// Get status of all servers
const statuses = await q.mcpServerStatus();
// McpServerStatus: name, status ("connected"|"failed"|"needs-auth"|"pending"|"disabled"), tools[], error?
```

## Permission Patterns

```python
# Allow specific tools
allowed_tools=["mcp__github__list_issues", "mcp__github__create_issue"]

# Allow all tools from a server
allowed_tools=["mcp__github__*"]

# Allow multiple servers
allowed_tools=["mcp__github__*", "mcp__postgres__query", "mcp__playwright__*"]

# Alternative: use permission mode (no allowedTools needed)
permission_mode="acceptEdits"   # Auto-approves most operations
permission_mode="bypassPermissions"  # Skip all approval prompts
```

## Error Handling

```python
# Python — check connection status in init message
from claude_agent_sdk import SystemMessage, ResultMessage

async for message in query(prompt="...", options=options):
    if isinstance(message, SystemMessage) and message.subtype == "init":
        mcp_servers_status = message.data.get("mcp_servers", [])
        for server in mcp_servers_status:
            if server.get("status") != "connected":
                print(f"WARNING: {server['name']} failed to connect: {server.get('error')}")
    elif isinstance(message, ResultMessage) and message.subtype == "error_during_execution":
        print("Agent execution failed")
```

```typescript
// TypeScript
for await (const message of query({ prompt: "...", options })) {
  if (message.type === "system" && message.subtype === "init") {
    const failed = message.mcp_servers?.filter(s => s.status !== "connected") ?? [];
    if (failed.length > 0) console.warn("Failed servers:", failed);
  }
}
```

## Popular MCP Servers

| Server | Package | Use case |
|--------|---------|---------|
| Filesystem | `@modelcontextprotocol/server-filesystem` | File operations across directories |
| GitHub | `@modelcontextprotocol/server-github` | Issues, PRs, repos, code search |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | SQL queries on Postgres DB |
| Playwright | `@playwright/mcp` | Browser automation |
| Slack | `@modelcontextprotocol/server-slack` | Messages, channels |
| Google Drive | `@modelcontextprotocol/server-gdrive` | Docs, sheets, files |
| Claude Code Docs | `https://code.claude.com/docs/mcp` (HTTP) | Claude Code documentation |

Full list: https://github.com/modelcontextprotocol/servers

## MCP Tool Search Configuration

When you have large tool sets (many MCP servers), tool descriptions can fill context. Enable dynamic tool loading:

```python
options = ClaudeAgentOptions(
    mcp_servers={ ... },
    env={"ENABLE_TOOL_SEARCH": "auto:5"}  # Load tools on-demand when >5% of context
)
```

**ENABLE_TOOL_SEARCH values:**
- `"auto"` (default) — activates at 10% context threshold
- `"auto:N"` — activates at N% threshold
- `"true"` — always active, all MCP tools deferred
- `"false"` — disabled, all tools loaded upfront

**Requirements:** claude-sonnet-4+ or claude-opus-4+. Haiku does not support tool search.

## Connecting to Claude Code Docs MCP

```python
# Useful for agents that need to look up Claude documentation
options = ClaudeAgentOptions(
    mcp_servers={
        "claude-code-docs": {
            "type": "http",
            "url": "https://code.claude.com/docs/mcp"
        }
    },
    allowed_tools=["mcp__claude-code-docs__*"]
)
```

## OAuth2 with MCP

The SDK doesn't handle OAuth flows automatically. Complete OAuth in your app first, then pass the access token:

```python
access_token = await your_oauth_flow()

options = ClaudeAgentOptions(
    mcp_servers={
        "oauth-api": {
            "type": "http",
            "url": "https://api.example.com/mcp",
            "headers": {"Authorization": f"Bearer {access_token}"}
        }
    }
)
```
