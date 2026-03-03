# Streaming & Input Modes — Deep Reference

## Two Input Modes

### 1. String Input (simple)
```python
# Python — single prompt, agent runs to completion
async for message in query(
    prompt="Fix the bug in auth.py",
    options=ClaudeAgentOptions(allowed_tools=["Read", "Edit"])
):
    ...
```

### 2. Async Generator Input (streaming/interactive)
```python
# Python — stream user messages for multi-turn or canUseTool
async def message_stream():
    yield {
        "type": "user",
        "message": {"role": "user", "content": "Fix the bug in auth.py"}
    }
    # Can yield more messages while agent is running

async for message in query(prompt=message_stream(), options=...):
    ...
```

**When you MUST use streaming input:**
- Using `canUseTool` callback in Python
- Using custom SDK MCP servers in TypeScript
- Sending mid-session follow-up messages
- Building chat-like interactive interfaces
- Sending cancel signals while agent is working

## Streaming Input in TypeScript

```typescript
// TypeScript — async generator
async function* messages() {
  yield {
    type: "user" as const,
    message: { role: "user" as const, content: "What are the system dependencies?" }
  };
}

for await (const message of query({
  prompt: messages(),  // Async generator, not string
  options: { allowedTools: ["Bash"] }
})) {
  ...
}
```

## Multi-Turn Streaming (Chat Interface)

```python
from asyncio import Queue

message_queue: Queue = Queue()

async def user_messages():
    while True:
        msg = await message_queue.get()
        if msg is None:  # Sentinel to stop
            break
        yield {
            "type": "user",
            "message": {"role": "user", "content": msg}
        }

# Agent consumes messages as they arrive
agent_task = asyncio.create_task(
    run_agent_loop(message_queue)
)

# Elsewhere in your app, push messages to the queue
await message_queue.put("Please analyze auth.py")
# ... agent responds ...
await message_queue.put("Now also check for SQL injection")
# ... agent continues with full context ...
await message_queue.put(None)  # Stop
```

## Sending Control Messages

```python
# Cancel signal — stop the agent mid-task
async def message_stream():
    yield {"type": "user", "message": {"role": "user", "content": "Long task..."}}
    await asyncio.sleep(2)  # Let it start
    yield {"type": "cancel"}  # Stop execution

# Inject context mid-stream
async def message_stream():
    yield {"type": "user", "message": {"role": "user", "content": "Process these files..."}}
    await asyncio.sleep(1)  # After agent starts
    # Inject additional context without waiting for Claude to ask
    yield {
        "type": "user",
        "message": {"role": "user", "content": "Also exclude test files from the analysis"}
    }
```

## Collecting Results Without Streaming

For background jobs or CI/CD where you don't need live output:

```python
# Python — collect all results at once
results = []
async for message in query(prompt="Fix bugs", options=...):
    if isinstance(message, ResultMessage) and message.subtype == "success":
        results.append(message.result)

# Process results after agent finishes
final_result = results[-1] if results else None
```

## Processing Progress in Real-Time

```python
# Show progress as the agent works
async for message in query(prompt="Large codebase analysis", options=...):
    if isinstance(message, SystemMessage) and message.subtype == "init":
        print(f"Session {message.data['session_id']} started")
    elif isinstance(message, AssistantMessage):
        for block in message.content:
            if hasattr(block, "text"):
                print(f"Claude: {block.text}")
            elif hasattr(block, "name"):
                print(f"→ {block.name}: {getattr(block, 'input', {})}")
    elif isinstance(message, ToolResultMessage):
        print(f"✓ Tool completed")
    elif isinstance(message, ResultMessage):
        status = "✅" if message.subtype == "success" else "❌"
        print(f"{status} Done: {message.subtype}")
```

## Streaming with ClaudeSDKClient (Python)

The `ClaudeSDKClient` is used for custom tools and more control over the session:

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

options = ClaudeAgentOptions(
    mcp_servers={"my-tools": custom_server},
    allowed_tools=["mcp__my-tools__*"]
)

async with ClaudeSDKClient(options=options) as client:
    # Send the initial message
    await client.query("Use the tools to analyze the database schema")

    # Stream responses
    async for msg in client.receive_response():
        if isinstance(msg, AssistantMessage):
            print(msg)
        elif isinstance(msg, ResultMessage):
            print(f"Result: {msg.result}")
            break

    # Continue the conversation
    await client.query("Now create an ER diagram of the schema")
    async for msg in client.receive_response():
        ...
```

## TypeScript — Streaming vs Single Mode

```typescript
// Live streaming (default) — messages arrive as Claude works
for await (const message of query({ prompt: "...", options })) {
  if (message.type === "assistant") {
    // Process in real-time
    console.log(message);
  }
}

// Collect all at once (not built-in, but easy to do)
const messages = [];
for await (const message of query({ prompt: "...", options })) {
  messages.push(message);
}
const result = messages.find(m => m.type === "result");
```
