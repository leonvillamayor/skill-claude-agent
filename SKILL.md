---
name: claude-agent-sdk
description: |
  Expert guide for building AI agents with the Claude Agent SDK (Python & TypeScript).
  Use this skill PROACTIVELY whenever:
  - Building or configuring AI agents with `claude_agent_sdk` or `@anthropic-ai/claude-agent-sdk`
  - Working with `query()`, `ClaudeAgentOptions`, `AgentDefinition`, hooks, sessions, subagents, MCP, or custom tools
  - User asks how to automate tasks, build pipelines, or orchestrate multi-agent workflows with Claude
  - Implementing permission controls, `canUseTool` callbacks, or tool restrictions
  - Connecting external services (databases, APIs, browsers) to a Claude agent via MCP
  - Debugging or optimizing agent behavior, streaming, sessions, or hooks
  - Any mention of "Claude agent", "agent SDK", "agentic loop", or autonomous Claude workflows
  Covers the full SDK: query API, built-in tools, hooks lifecycle, session management, subagents,
  MCP servers, custom in-process tools, permission modes, user input flows, and authentication.
---

# Claude Agent SDK — Complete Guide

The Claude Agent SDK lets you build AI agents that autonomously read files, run commands, search the web, edit code, and more — using the same tools and agent loop that power Claude Code.

**Languages:** Python (`claude-agent-sdk`) and TypeScript (`@anthropic-ai/claude-agent-sdk`)

## Quick Reference

| Goal | Go to |
|------|-------|
| First agent | [Setup & Installation](#setup--installation) |
| Configure tools/permissions | [Core Options (ClaudeAgentOptions)](#core-options-claudeagentoptions) |
| React to tool calls | [Hooks](#hooks) |
| Multi-turn memory | [Sessions](#sessions) |
| Delegate to specialists | [Subagents](#subagents) |
| Connect external APIs/DBs | [MCP Servers](#mcp-servers) |
| Build in-process tools | [Custom Tools](#custom-tools) |
| Load skills & plugins | [Skills & Plugins](#skills--plugins) |
| Control tool approvals | [Permissions & User Input](#permissions--user-input) |
| Deploy to production | [Authentication & Hosting](#authentication--hosting) |
| Advanced reference | `references/` directory |

---

## Setup & Installation

```bash
# TypeScript
npm install @anthropic-ai/claude-agent-sdk

# Python (pip)
pip install claude-agent-sdk

# Python (uv — recommended)
uv add claude-agent-sdk
```

Set your API key:
```bash
export ANTHROPIC_API_KEY=your-api-key
```

**Minimal working agent:**

```python
# Python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="What files are in this directory?",
        options=ClaudeAgentOptions(allowed_tools=["Bash", "Glob"]),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

```typescript
// TypeScript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "What files are in this directory?",
  options: { allowedTools: ["Bash", "Glob"] }
})) {
  if ("result" in message) console.log(message.result);
}
```

---

## Core Options (ClaudeAgentOptions)

All options for `query()` — Python uses `snake_case`, TypeScript uses `camelCase`.

| Option (Python / TS) | Type | Description |
|---|---|---|
| `allowed_tools` / `allowedTools` | `string[]` | Whitelist of tools Claude can use |
| `disallowed_tools` / `disallowedTools` | `string[]` | Blacklist of tools |
| `permission_mode` / `permissionMode` | `string` | `"default"`, `"acceptEdits"`, `"bypassPermissions"`, `"plan"` |
| `system_prompt` / `systemPrompt` | `string` | Custom system prompt (prepended) |
| `model` | `string` | Model ID or shorthand (`"sonnet"`, `"opus"`, `"haiku"`) |
| `max_turns` / `maxTurns` | `number` | Maximum agentic turns |
| `resume` | `string` | Session ID to resume |
| `fork_session` / `forkSession` | `boolean` | Fork session instead of continuing |
| `mcp_servers` / `mcpServers` | `object` | MCP server configurations |
| `agents` | `object` | Subagent definitions (keyed by name) |
| `hooks` | `object` | Hook event callbacks |
| `can_use_tool` / `canUseTool` | `function` | Callback for runtime tool approval |
| `setting_sources` / `settingSources` | `string[]` | Load from filesystem (`["project"]`) |
| `cwd` | `string` | Working directory for the agent |
| `env` | `object` | Environment variables to pass |
| `plugins` | `array` | Programmatic plugins (slash commands, agents, MCP) |

### Built-in Tools Reference

| Tool | Purpose |
|------|---------|
| `Read` | Read any file |
| `Write` | Create new files |
| `Edit` | Precise edits to existing files |
| `Bash` | Run terminal commands, scripts, git |
| `Glob` | Find files by pattern (`**/*.ts`) |
| `Grep` | Search file contents with regex |
| `WebSearch` | Search the web |
| `WebFetch` | Fetch and parse web pages |
| `AskUserQuestion` | Ask user clarifying questions (multiple choice) |
| `Task` | Invoke subagents (required when using `agents`) |
| `Skill` | Invoke installed skills (requires `setting_sources`) |

**Common tool combinations:**
- Read-only: `["Read", "Glob", "Grep"]`
- Code editing: `["Read", "Edit", "Write", "Glob", "Grep"]`
- Full automation: `["Read", "Edit", "Write", "Bash", "Glob", "Grep"]`
- Web research: `["WebSearch", "WebFetch", "Read"]`
- With subagents: add `"Task"` to any combination
- With skills: add `"Skill"` and configure `setting_sources`

---

## Message Types

The `query()` async iterator yields typed messages:

```python
# Python — check message type
from claude_agent_sdk import AssistantMessage, ResultMessage, SystemMessage

async for message in query(prompt="...", options=...):
    if isinstance(message, SystemMessage) and message.subtype == "init":
        session_id = message.data.get("session_id")
    elif isinstance(message, AssistantMessage):
        for block in message.content:
            if hasattr(block, "text"):
                print(block.text)          # Claude's reasoning
            elif hasattr(block, "name"):
                print(f"Tool: {block.name}")  # Tool call
    elif isinstance(message, ResultMessage):
        print(f"Done ({message.subtype}): {message.result}")
```

```typescript
// TypeScript — check message type
for await (const message of query({ prompt: "..." })) {
  if (message.type === "system" && message.subtype === "init") {
    const sessionId = message.session_id;
  } else if (message.type === "assistant") {
    for (const block of message.message.content) {
      if ("text" in block) console.log(block.text);
      else if ("name" in block) console.log(`Tool: ${block.name}`);
    }
  } else if (message.type === "result") {
    console.log(`Done (${message.subtype}): ${message.result}`);
  }
}
```

**Result subtypes:** `"success"` | `"error_during_execution"` | `"max_turns_reached"`

---

## Hooks

Hooks are callback functions that intercept agent events. Use them to block/allow tools, log operations, inject context, or notify external systems.

### Hook Events

| Event | Trigger | Python | TypeScript |
|-------|---------|--------|------------|
| `PreToolUse` | Before tool call (can block/modify) | ✅ | ✅ |
| `PostToolUse` | After tool result | ✅ | ✅ |
| `PostToolUseFailure` | On tool error | ✅ | ✅ |
| `UserPromptSubmit` | On user prompt | ✅ | ✅ |
| `Stop` | Agent stops | ✅ | ✅ |
| `SubagentStart` | Subagent spawned | ✅ | ✅ |
| `SubagentStop` | Subagent finishes | ✅ | ✅ |
| `Notification` | Agent status message | ✅ | ✅ |
| `PreCompact` | Before conversation compaction | ✅ | ✅ |
| `PermissionRequest` | Permission dialog | ✅ | ✅ |
| `SessionStart` / `SessionEnd` | Session lifecycle | ❌ | ✅ |
| `Setup`, `TeammateIdle`, `TaskCompleted` | Advanced lifecycle | ❌ | ✅ |
| `WorktreeCreate` / `WorktreeRemove` | Git worktree events | ❌ | ✅ |
| `ConfigChange` | Config file changes | ❌ | ✅ |

### Hook Configuration

```python
# Python
from claude_agent_sdk import query, ClaudeAgentOptions, HookMatcher

async def my_hook(input_data, tool_use_id, context):
    # input_data has: hook_event_name, tool_name, tool_input, session_id, cwd
    file_path = input_data.get("tool_input", {}).get("file_path", "")
    if ".env" in file_path:
        return {
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "deny",
                "permissionDecisionReason": "Cannot modify .env files",
            }
        }
    return {}  # Empty = allow

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="Write|Edit", hooks=[my_hook])
        ]
    }
)
```

```typescript
// TypeScript
import { query, HookCallback, PreToolUseHookInput } from "@anthropic-ai/claude-agent-sdk";

const myHook: HookCallback = async (input, toolUseID, { signal }) => {
  const preInput = input as PreToolUseHookInput;
  const toolInput = preInput.tool_input as Record<string, unknown>;
  const filePath = toolInput?.file_path as string;

  if (filePath?.includes(".env")) {
    return {
      hookSpecificOutput: {
        hookEventName: preInput.hook_event_name,
        permissionDecision: "deny",
        permissionDecisionReason: "Cannot modify .env files"
      }
    };
  }
  return {}; // Empty = allow
};

// Usage
const options = {
  hooks: {
    PreToolUse: [{ matcher: "Write|Edit", hooks: [myHook] }]
  }
};
```

### Hook Output Fields

**Top-level fields** (affect conversation):
- `systemMessage: string` — inject context Claude can see
- `continue` (`continue_` in Python): `boolean` — whether agent keeps running

**`hookSpecificOutput`** fields (affect the current operation):
- `permissionDecision`: `"allow"` | `"deny"` | `"ask"`
- `permissionDecisionReason`: `string` — shown to Claude on deny
- `updatedInput`: `object` — modified tool input (requires `permissionDecision: "allow"`)
- `additionalContext`: `string` — append to tool result (PostToolUse only)

**Async output** (fire-and-forget, for logging/webhooks):
```python
return {"async_": True, "asyncTimeout": 30000}   # Python
```
```typescript
return { async: true, asyncTimeout: 30000 };      // TypeScript
```

For detailed examples (audit logging, sandbox redirection, Slack notifications): see [references/hooks.md](references/hooks.md)

---

## Sessions

Sessions maintain full context (files read, analysis, conversation) across multiple queries.

```python
# Python — capture and resume session
session_id = None

# First query
async for message in query(prompt="Read the auth module", options=ClaudeAgentOptions(allowed_tools=["Read", "Glob"])):
    if hasattr(message, "subtype") and message.subtype == "init":
        session_id = message.data.get("session_id")

# Resume with full context
async for message in query(
    prompt="Now find all callers of it",  # "it" = auth module from previous query
    options=ClaudeAgentOptions(resume=session_id),
):
    if hasattr(message, "result"):
        print(message.result)
```

```typescript
// TypeScript
let sessionId: string | undefined;

// First query
for await (const message of query({ prompt: "Read the auth module", options: { allowedTools: ["Read", "Glob"] } })) {
  if (message.type === "system" && message.subtype === "init") {
    sessionId = message.session_id;
  }
}

// Resume
for await (const message of query({ prompt: "Now find all callers", options: { resume: sessionId } })) {
  if ("result" in message) console.log(message.result);
}
```

### Fork a Session

Create a new branch from a session without modifying the original:

```python
# Python
options = ClaudeAgentOptions(resume=session_id, fork_session=True)
```
```typescript
// TypeScript
const options = { resume: sessionId, forkSession: true };
```

| | `forkSession: false` (default) | `forkSession: true` |
|--|--|--|
| Session ID | Same as original | New ID generated |
| Original | Modified | Preserved |
| Use case | Continue conversation | Explore alternatives |

For multi-turn patterns and session persistence details: see [references/sessions.md](references/sessions.md)

---

## Subagents

Subagents are specialized agent instances the main agent can delegate to. Benefits: isolated context, parallel execution, specialized tools/prompts.

**Key requirement:** Include `"Task"` in `allowedTools` — subagents are invoked via the Task tool.

```python
# Python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async for message in query(
    prompt="Use the code-reviewer agent to review auth.py",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Glob", "Grep", "Task"],
        agents={
            "code-reviewer": AgentDefinition(
                description="Expert code reviewer. Use for quality, security, and maintainability reviews.",
                prompt="You are a security-focused code reviewer. Identify vulnerabilities and suggest fixes.",
                tools=["Read", "Glob", "Grep"],  # Read-only
                model="sonnet",  # Override model for this subagent
            )
        },
    ),
):
    if hasattr(message, "result"):
        print(message.result)
```

```typescript
// TypeScript
for await (const message of query({
  prompt: "Use the code-reviewer agent to review auth.py",
  options: {
    allowedTools: ["Read", "Glob", "Grep", "Task"],
    agents: {
      "code-reviewer": {
        description: "Expert code reviewer. Use for quality, security, and maintainability reviews.",
        prompt: "You are a security-focused code reviewer. Identify vulnerabilities and suggest fixes.",
        tools: ["Read", "Glob", "Grep"],
        model: "sonnet"
      }
    }
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

### AgentDefinition Fields

| Field | Required | Description |
|-------|----------|-------------|
| `description` | ✅ | When to use this agent (determines auto-invocation) |
| `prompt` | ✅ | Subagent's system prompt / expertise |
| `tools` | ❌ | Allowed tools (omit to inherit all) |
| `model` | ❌ | `"sonnet"`, `"opus"`, `"haiku"`, or `"inherit"` |

**Notes:**
- Subagents cannot spawn their own subagents (don't add `"Task"` to subagent tools)
- Use explicit naming in prompt to guarantee a specific subagent: `"Use the X agent to..."`
- Messages from subagent context have `parent_tool_use_id` set

For subagent resumption and parallel patterns: see [references/subagents.md](references/subagents.md)

---

## MCP Servers

Connect your agent to databases, browsers, APIs, and hundreds of external tools via the Model Context Protocol.

### MCP Tool Naming
MCP tools follow: `mcp__<server-name>__<tool-name>`
Example: server `"github"` + tool `list_issues` → `mcp__github__list_issues`

### Add MCP Servers

```python
# Python — connect to GitHub MCP server
from claude_agent_sdk import query, ClaudeAgentOptions
import os

options = ClaudeAgentOptions(
    mcp_servers={
        "github": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": os.environ["GITHUB_TOKEN"]},
        }
    },
    allowed_tools=["mcp__github__list_issues", "mcp__github__search_issues"],
)
```

```typescript
// TypeScript
const options = {
  mcpServers: {
    github: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-github"],
      env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN }
    }
  },
  allowedTools: ["mcp__github__list_issues", "mcp__github__search_issues"]
};
```

### Transport Types

| Type | When to use | Config |
|------|-------------|--------|
| **stdio** | Local CLI tools (`npx`, `uvx`) | `command`, `args`, `env` |
| **HTTP** | Remote REST APIs | `type: "http"`, `url`, `headers` |
| **SSE** | Remote streaming APIs | `type: "sse"`, `url`, `headers` |
| **SDK MCP** | In-process custom tools | See [Custom Tools](#custom-tools) |

```python
# Remote SSE server with auth
mcp_servers = {
    "remote-api": {
        "type": "sse",
        "url": "https://api.example.com/mcp/sse",
        "headers": {"Authorization": f"Bearer {os.environ['API_TOKEN']}"},
    }
}
```

### Wildcard Tool Access
```python
allowed_tools=["mcp__github__*"]   # All tools from github server
```

### MCP Tool Search (large tool sets)
When tools exceed 10% of context window, auto-activates to load tools on-demand:
```python
env={"ENABLE_TOOL_SEARCH": "auto"}    # default
env={"ENABLE_TOOL_SEARCH": "auto:5"}  # activate at 5% threshold
env={"ENABLE_TOOL_SEARCH": "true"}    # always on
env={"ENABLE_TOOL_SEARCH": "false"}   # always load all upfront
```
*Requires: claude-sonnet-4 or claude-opus-4+. Not available on Haiku.*

For .mcp.json config files and error handling patterns: see [references/mcp.md](references/mcp.md)

---

## Custom Tools

Define in-process MCP servers to give Claude custom capabilities without running a separate process.

```python
# Python
from claude_agent_sdk import tool, create_sdk_mcp_server, ClaudeSDKClient, ClaudeAgentOptions
import aiohttp, asyncio
from typing import Any

@tool("get_weather", "Get temperature for coordinates", {"latitude": float, "longitude": float})
async def get_weather(args: dict[str, Any]) -> dict[str, Any]:
    async with aiohttp.ClientSession() as session:
        async with session.get(
            f"https://api.open-meteo.com/v1/forecast?latitude={args['latitude']}&longitude={args['longitude']}&current=temperature_2m"
        ) as response:
            data = await response.json()
    return {"content": [{"type": "text", "text": f"Temp: {data['current']['temperature_2m']}°C"}]}

custom_server = create_sdk_mcp_server(name="weather-tools", version="1.0.0", tools=[get_weather])

async def main():
    options = ClaudeAgentOptions(
        mcp_servers={"weather-tools": custom_server},
        allowed_tools=["mcp__weather-tools__get_weather"],
    )
    async with ClaudeSDKClient(options=options) as client:
        await client.query("What's the weather in Paris (48.85, 2.35)?")
        async for msg in client.receive_response():
            print(msg)

asyncio.run(main())
```

```typescript
// TypeScript
import { query, tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const customServer = createSdkMcpServer({
  name: "weather-tools",
  version: "1.0.0",
  tools: [
    tool("get_weather", "Get temperature for coordinates",
      { latitude: z.number(), longitude: z.number() },
      async (args) => {
        const res = await fetch(`https://api.open-meteo.com/v1/forecast?latitude=${args.latitude}&longitude=${args.longitude}&current=temperature_2m`);
        const data = await res.json();
        return { content: [{ type: "text", text: `Temp: ${data.current.temperature_2m}°C` }] };
      }
    )
  ]
});

// Important: use async generator for prompt when using SDK MCP servers
async function* messages() {
  yield { type: "user" as const, message: { role: "user" as const, content: "Weather in Paris (48.85, 2.35)?" } };
}

for await (const message of query({
  prompt: messages(),
  options: {
    mcpServers: { "weather-tools": customServer },
    allowedTools: ["mcp__weather-tools__get_weather"]
  }
})) {
  if (message.type === "result") console.log(message.result);
}
```

**Important:** Custom MCP tools require streaming/async generator input in TypeScript.

For complex schema patterns, error handling, and database/API gateway examples: see [references/custom-tools.md](references/custom-tools.md)

---

## Skills & Plugins

Skills are reusable capability packages (`SKILL.md` files) that Claude loads on-demand based on context. The SDK can load skills from the filesystem and invoke them via the `Skill` built-in tool.

### Loading Skills from Filesystem

By default, the SDK does **not** load any filesystem settings (including skills). You must explicitly configure `setting_sources` to enable skill discovery.

```python
# Python — enable skills from user and project directories
from claude_agent_sdk import query, ClaudeAgentOptions

options = ClaudeAgentOptions(
    cwd="/path/to/project",                    # Project with .claude/skills/
    setting_sources=["user", "project"],       # Load skills from filesystem
    allowed_tools=["Skill", "Read", "Write", "Bash"],  # Enable Skill tool
)

async for message in query(prompt="Help me process this PDF document", options=options):
    print(message)
```

```typescript
// TypeScript — enable skills from user and project directories
for await (const message of query({
  prompt: "Help me process this PDF document",
  options: {
    cwd: "/path/to/project",                   // Project with .claude/skills/
    settingSources: ["user", "project"],        // Load skills from filesystem
    allowedTools: ["Skill", "Read", "Write", "Bash"]  // Enable Skill tool
  }
})) {
  console.log(message);
}
```

### How Skills Work in the SDK

1. **Filesystem artifacts**: Skills are `SKILL.md` files in `.claude/skills/` directories
2. **Auto-discovered**: Skill metadata (name + description) is loaded at startup
3. **Model-invoked**: Claude autonomously decides when to activate a skill based on context
4. **On-demand loading**: Full skill content is loaded only when triggered (saves context)

### Skill Discovery Locations

| `setting_sources` value | Skills loaded from |
|-------------------------|-------------------|
| `"user"` | `~/.claude/skills/` |
| `"project"` | `.claude/skills/` in `cwd` |
| `"local"` | `.claude/settings.local.json` |

### Discovering Available Skills

```python
# List all skills available to the agent
options = ClaudeAgentOptions(
    setting_sources=["user", "project"],
    allowed_tools=["Skill"],
)

async for message in query(prompt="What Skills are available?", options=options):
    print(message)
```

### Loading Plugins Programmatically

Plugins provide skills, slash commands, and agents from local directories — without requiring them to be in `.claude/skills/`.

```python
# Python — load plugins from local paths
async for message in query(
    prompt="Hello",
    options={
        "plugins": [
            {"type": "local", "path": "./my-plugin"},
            {"type": "local", "path": "/absolute/path/to/another-plugin"},
        ]
    },
):
    print(message)
```

```typescript
// TypeScript — load plugins from local paths
for await (const message of query({
  prompt: "Hello",
  options: {
    plugins: [
      { type: "local", path: "./my-plugin" },
      { type: "local", path: "/absolute/path/to/another-plugin" }
    ]
  }
})) {
  console.log(message);
}
```

**Key differences between Skills and Plugins:**
- Skills must be filesystem artifacts (`SKILL.md` files) — no programmatic registration API
- Plugins load from any local path via the `plugins` array
- Both require `"Skill"` in `allowed_tools` to be invokable
- Programmatic options (`agents`, `allowed_tools`) always override filesystem settings

---

## Permissions & User Input

### Permission Modes

| Mode | Behavior | Best for |
|------|----------|---------|
| `"default"` | No auto-approvals; uses `canUseTool` callback | Interactive apps |
| `"acceptEdits"` | Auto-approves file edits + mkdir/rm/mv/cp | Trusted dev workflows |
| `"bypassPermissions"` | All tools run without prompts | CI/CD, automation |
| `"plan"` | No tool execution; Claude plans only | Pre-approval review |

```python
# Dynamic permission mode change mid-session
q = query(prompt="...", options=ClaudeAgentOptions(permission_mode="default"))
await q.set_permission_mode("acceptEdits")  # Switch after review
async for message in q: ...
```

### canUseTool Callback

Handle interactive tool approvals and `AskUserQuestion` responses:

```python
# Python
from claude_agent_sdk.types import PermissionResultAllow, PermissionResultDeny

async def can_use_tool(tool_name: str, input_data: dict, context):
    if tool_name == "AskUserQuestion":
        # Claude is asking the user a question
        answers = {}
        for q in input_data.get("questions", []):
            print(f"\n{q['question']}")
            for i, opt in enumerate(q["options"]):
                print(f"  {i+1}. {opt['label']} — {opt['description']}")
            choice = input("Your choice: ")
            answers[q["question"]] = q["options"][int(choice)-1]["label"]
        return PermissionResultAllow(updated_input={"questions": input_data["questions"], "answers": answers})

    # For other tools
    print(f"Claude wants to use {tool_name}: {input_data}")
    response = input("Allow? (y/n): ")
    if response.lower() == "y":
        return PermissionResultAllow(updated_input=input_data)
    return PermissionResultDeny(message="User denied")

# NOTE: Python requires streaming input + dummy PreToolUse hook for canUseTool to work
async def dummy_hook(input_data, tool_use_id, context):
    return {"continue_": True}

async def prompt_stream():
    yield {"type": "user", "message": {"role": "user", "content": "Your task here"}}

options = ClaudeAgentOptions(
    can_use_tool=can_use_tool,
    hooks={"PreToolUse": [HookMatcher(matcher=None, hooks=[dummy_hook])]},
)
```

```typescript
// TypeScript
const options = {
  canUseTool: async (toolName: string, input: any) => {
    if (toolName === "AskUserQuestion") {
      const answers: Record<string, string> = {};
      for (const q of input.questions) {
        console.log(`\n${q.question}`);
        q.options.forEach((opt: any, i: number) => console.log(`  ${i+1}. ${opt.label}`));
        const choice = parseInt(await prompt("Choice: ")) - 1;
        answers[q.question] = q.options[choice].label;
      }
      return { behavior: "allow", updatedInput: { questions: input.questions, answers } };
    }
    console.log(`Tool: ${toolName}`, input);
    const approved = (await prompt("Allow? (y/n): ")).toLowerCase() === "y";
    return approved
      ? { behavior: "allow", updatedInput: input }
      : { behavior: "deny", message: "User denied" };
  }
};
```

For declarative permission rules (settings.json), plan mode patterns, and input modification: see [references/permissions.md](references/permissions.md)

---

## Authentication & Hosting

### API Providers

```bash
# Anthropic (default)
export ANTHROPIC_API_KEY=your-key

# Amazon Bedrock
export CLAUDE_CODE_USE_BEDROCK=1
# Configure AWS credentials (aws configure or env vars)

# Google Vertex AI
export CLAUDE_CODE_USE_VERTEX=1
# Configure Google Cloud credentials

# Microsoft Azure AI Foundry
export CLAUDE_CODE_USE_FOUNDRY=1
# Configure Azure credentials
```

### Docker Deployment

```dockerfile
FROM node:18-slim
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
ENV ANTHROPIC_API_KEY=""
CMD ["node", "agent.js"]
```

### Filesystem Configuration

Load Claude Code settings (CLAUDE.md, slash commands, skills, agents from `.claude/agents/`):
```python
options = ClaudeAgentOptions(setting_sources=["project"])
```

Available setting sources: `"project"` (`.claude/`), `"user"` (~/.claude/), `"mcpServers"`

---

## Common Patterns

### Read-Only Analysis Agent
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep"],
    permission_mode="bypassPermissions",
    system_prompt="You are a code analyst. Never modify files.",
)
```

### CI/CD Automation Agent
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Write", "Bash", "Glob", "Grep"],
    permission_mode="bypassPermissions",
    max_turns=20,
)
```

### Multi-Agent Parallel Workflow
```python
# Main agent delegates to specialists
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Task"],
    agents={
        "security-scanner": AgentDefinition(
            description="Security vulnerability scanner",
            prompt="Find security issues. Report CVEs and OWASP violations.",
            tools=["Read", "Grep", "Glob"],
        ),
        "test-runner": AgentDefinition(
            description="Test execution and analysis specialist",
            prompt="Run tests and analyze failures.",
            tools=["Bash", "Read", "Grep"],
        ),
    },
)
```

### Audit Logging Hook
```python
async def audit_logger(input_data, tool_use_id, context):
    with open("audit.log", "a") as f:
        f.write(f"{datetime.now()}: {input_data['tool_name']} — {input_data.get('tool_input', {})}\n")
    return {}

options = ClaudeAgentOptions(
    hooks={"PostToolUse": [HookMatcher(hooks=[audit_logger])]}
)
```

### Skills-Enabled Agent
```python
# Agent that can use installed skills (PDF, XLSX, etc.)
options = ClaudeAgentOptions(
    cwd="/path/to/project",
    setting_sources=["user", "project"],  # Load skills from filesystem
    allowed_tools=["Skill", "Read", "Write", "Edit", "Bash", "Glob"],
)
```

### Session-Based Multi-Turn Workflow
```python
session_id = None
prompts = ["Read auth.py", "Find security issues", "Fix the top 3 issues"]

for prompt_text in prompts:
    opts = ClaudeAgentOptions(
        allowed_tools=["Read", "Edit", "Glob"],
        resume=session_id,
    )
    async for message in query(prompt=prompt_text, options=opts):
        if hasattr(message, "subtype") and message.subtype == "init":
            session_id = message.data.get("session_id")
        if hasattr(message, "result"):
            print(message.result)
```

---

## Reference Files

For detailed coverage of each topic, read the appropriate reference:

- [references/hooks.md](references/hooks.md) — All hook events, input/output schemas, full examples
- [references/sessions.md](references/sessions.md) — Session lifecycle, forking, file checkpointing
- [references/subagents.md](references/subagents.md) — Subagent patterns, resumption, parallel workflows
- [references/mcp.md](references/mcp.md) — MCP transports, auth, .mcp.json, error handling
- [references/custom-tools.md](references/custom-tools.md) — In-process tools, Zod schemas, type safety
- [references/permissions.md](references/permissions.md) — Permission modes, declarative rules, canUseTool
- [references/streaming.md](references/streaming.md) — Streaming vs single-turn, input streaming, interrupts
