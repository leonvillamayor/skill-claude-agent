# Custom Tools — Deep Reference

Custom tools are in-process MCP servers that run directly within your application. Use them to give Claude access to your APIs, databases, or custom logic without running a separate server.

## Core Pattern

```python
# Python
from claude_agent_sdk import tool, create_sdk_mcp_server

@tool(
    "tool_name",
    "Human-readable description for Claude",
    {"param_name": type}  # Schema — Python types, dict, or JSON Schema
)
async def my_tool(args: dict) -> dict:
    # args contains the validated input
    result = do_something(args["param_name"])
    return {
        "content": [{"type": "text", "text": result}]
    }

server = create_sdk_mcp_server(name="my-tools", version="1.0.0", tools=[my_tool])
```

```typescript
// TypeScript
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const myTool = tool(
    "tool_name",
    "Human-readable description for Claude",
    { param_name: z.string().describe("What this parameter is for") },
    async (args) => {
        const result = doSomething(args.param_name);
        return { content: [{ type: "text", text: result }] };
    }
);

const server = createSdkMcpServer({ name: "my-tools", version: "1.0.0", tools: [myTool] });
```

## Schema Definition Options (Python)

### Simple type mapping
```python
@tool("my_tool", "Description", {
    "name": str,
    "age": int,
    "score": float,
    "active": bool,
    "tags": list,
    "config": dict,
})
```

### JSON Schema format (for constraints)
```python
@tool("my_tool", "Description", {
    "type": "object",
    "properties": {
        "name": {"type": "string", "minLength": 1, "maxLength": 100},
        "age": {"type": "integer", "minimum": 0, "maximum": 150},
        "status": {"type": "string", "enum": ["active", "inactive", "pending"]},
        "email": {"type": "string", "format": "email"},
        "tags": {"type": "array", "items": {"type": "string"}, "maxItems": 10},
    },
    "required": ["name", "age"]
})
```

## Schema Definition (TypeScript — Zod)

```typescript
import { z } from "zod";

// Basic types
{ name: z.string(), age: z.number(), active: z.boolean() }

// With constraints
{ name: z.string().min(1).max(100).describe("User's full name") }

// Enums
{ status: z.enum(["active", "inactive"]) }

// Optional with default
{ precision: z.number().optional().default(2) }

// Nested object
{ config: z.object({ timeout: z.number(), retries: z.number() }) }

// Arrays
{ ids: z.array(z.string()).describe("List of resource IDs") }
```

## Tool Output Format

```python
# Text response
return {"content": [{"type": "text", "text": "Result here"}]}

# Multiple content blocks
return {
    "content": [
        {"type": "text", "text": "Summary: 5 items found"},
        {"type": "text", "text": json.dumps(items, indent=2)},
    ]
}

# Error (Claude sees this and adjusts)
return {"content": [{"type": "text", "text": "Error: Connection refused to database"}]}
```

## Registration and Usage

```python
# Python — using ClaudeSDKClient (for custom tools)
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

options = ClaudeAgentOptions(
    mcp_servers={"my-tools": server},
    allowed_tools=["mcp__my-tools__tool_name"],
)

async with ClaudeSDKClient(options=options) as client:
    await client.query("Use the tool to do something")
    async for msg in client.receive_response():
        print(msg)
```

```typescript
// TypeScript — async generator required for streaming input
async function* messages() {
  yield {
    type: "user" as const,
    message: { role: "user" as const, content: "Use the tool to do something" }
  };
}

for await (const message of query({
  prompt: messages(),  // Must be async generator, not string
  options: {
    mcpServers: { "my-tools": server },
    allowedTools: ["mcp__my-tools__tool_name"]
  }
})) {
  if (message.type === "result") console.log(message.result);
}
```

## Complete Examples

### Database Query Tool

```python
import asyncpg, json

db_pool = None  # Set up your connection pool separately

@tool("query_db", "Run a read-only SQL query", {"sql": str, "params": list})
async def query_db(args: dict) -> dict:
    try:
        async with db_pool.acquire() as conn:
            rows = await conn.fetch(args["sql"], *args.get("params", []))
            data = [dict(row) for row in rows]
            return {
                "content": [{
                    "type": "text",
                    "text": f"Found {len(data)} rows:\n{json.dumps(data, indent=2, default=str)}"
                }]
            }
    except Exception as e:
        return {"content": [{"type": "text", "text": f"Query error: {str(e)}"}]}
```

### REST API Tool

```python
import aiohttp, os

@tool("call_api", "Make authenticated API calls", {
    "endpoint": str,
    "method": str,
    "body": dict
})
async def call_api(args: dict) -> dict:
    async with aiohttp.ClientSession() as session:
        method = args.get("method", "GET").upper()
        url = f"{os.environ['API_BASE_URL']}{args['endpoint']}"
        headers = {"Authorization": f"Bearer {os.environ['API_KEY']}"}

        async with session.request(method, url, headers=headers, json=args.get("body")) as resp:
            if resp.status >= 400:
                return {"content": [{"type": "text", "text": f"HTTP {resp.status}: {await resp.text()}"}]}
            data = await resp.json()
            return {"content": [{"type": "text", "text": json.dumps(data, indent=2)}]}
```

### Send Notification Tool

```python
import smtplib
from email.mime.text import MIMEText

@tool("send_email", "Send an email notification", {
    "to": str,
    "subject": str,
    "body": str
})
async def send_email(args: dict) -> dict:
    msg = MIMEText(args["body"])
    msg["Subject"] = args["subject"]
    msg["From"] = os.environ["SMTP_FROM"]
    msg["To"] = args["to"]
    try:
        with smtplib.SMTP(os.environ["SMTP_HOST"]) as s:
            s.send_message(msg)
        return {"content": [{"type": "text", "text": f"Email sent to {args['to']}"}]}
    except Exception as e:
        return {"content": [{"type": "text", "text": f"Failed to send: {e}"}]}
```

### Multiple Tools in One Server

```python
server = create_sdk_mcp_server(
    name="company-tools",
    version="1.0.0",
    tools=[query_db, call_api, send_email]
)

# Allow all tools
options = ClaudeAgentOptions(
    mcp_servers={"company-tools": server},
    allowed_tools=["mcp__company-tools__*"]
)

# Or allow selectively
options = ClaudeAgentOptions(
    mcp_servers={"company-tools": server},
    allowed_tools=[
        "mcp__company-tools__query_db",
        # send_email NOT in list — Claude can't use it
    ]
)
```

## TypeScript — Multiple Tools

```typescript
const server = createSdkMcpServer({
  name: "company-tools",
  version: "1.0.0",
  tools: [
    tool("query_db", "Run SQL queries",
      { sql: z.string(), params: z.array(z.any()).optional() },
      async (args) => { /* ... */ }
    ),
    tool("send_email", "Send emails",
      { to: z.string().email(), subject: z.string(), body: z.string() },
      async (args) => { /* ... */ }
    ),
  ]
});
```
