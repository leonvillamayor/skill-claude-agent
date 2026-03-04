# skill-claude-agent

A Claude Code skill providing complete coverage of the **Claude Agent SDK** — build AI agents that autonomously read files, run commands, search the web, edit code, and more.

## Installation

```bash
npx skills add <your-github-username>/skill-claude-agent
```

Or install via `.skill` file:
```bash
npx skills install skill-claude-agent.skill
```

## What This Skill Covers

The Claude Agent SDK (Python: `claude-agent-sdk`, TypeScript: `@anthropic-ai/claude-agent-sdk`) lets you build autonomous AI agents with:

| Feature | What it enables |
|---------|----------------|
| **query() / Built-in tools** | Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, TodoWrite, NotebookEdit, ListMcpResources, ReadMcpResource |
| **ClaudeSDKClient** | Multi-turn client with dynamic model/permission changes, interrupt, MCP status, server info |
| **Hooks** | PreToolUse, PostToolUse, Stop, Notification, SubagentStart/Stop, and more |
| **Sessions** | Resume multi-turn conversations, fork for parallel exploration |
| **Subagents** | Spawn specialized agents with isolated context and tool restrictions |
| **MCP Servers** | Connect to databases, browsers, APIs via Model Context Protocol |
| **Custom Tools** | In-process tool servers using `createSdkMcpServer` / `create_sdk_mcp_server` |
| **Skills & Plugins** | Load skills from filesystem, invoke via Skill tool, programmatic plugins |
| **Permissions** | `default`, `acceptEdits`, `bypassPermissions`, `plan`, `dontAsk` modes + `canUseTool` callback |
| **System Prompts** | String, preset objects, CLAUDE.md integration, output styles |
| **Cost Tracking** | Per-message and cumulative token/cost tracking |
| **File Checkpointing** | Track file changes and rewind to previous state |
| **Interrupt & Stop** | Cancel running tasks, handle stop reasons |
| **Sandbox** | Sandboxed execution with network/capability restrictions |
| **Authentication** | Anthropic API, Amazon Bedrock, Google Vertex AI, Azure AI Foundry |

## Skill Structure

```
skill-claude-agent/
├── SKILL.md                    # Main skill (quick reference for all features)
└── references/
    ├── hooks.md                # All hook events, input/output schemas, examples
    ├── sessions.md             # Session lifecycle, forking, multi-turn patterns
    ├── subagents.md            # Subagent patterns, parallel workflows, resumption
    ├── mcp.md                  # MCP transports, auth, .mcp.json, error handling
    ├── custom-tools.md         # In-process tools, Zod schemas, type safety
    ├── permissions.md          # Permission modes, declarative rules, canUseTool
    └── streaming.md            # Streaming vs single-turn, input streaming
```

## Trigger Phrases

This skill activates when you:
- Build agents with `claude_agent_sdk` or `@anthropic-ai/claude-agent-sdk`
- Use `query()`, `ClaudeSDKClient`, `ClaudeAgentOptions`, `AgentDefinition`, hooks, sessions, or subagents
- Connect MCP servers or custom tools to Claude agents
- Implement permission controls, `canUseTool` callbacks, or tool restrictions
- Work with skills, plugins, system prompts, cost tracking, or file checkpointing
- Ask about "Claude agent", "agent SDK", "agentic loop", or autonomous Claude workflows

## SDK Quick Start

```python
# Python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Find and fix the bug in auth.py",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Glob"],
            permission_mode="acceptEdits",
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

```typescript
// TypeScript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and fix the bug in auth.py",
  options: {
    allowedTools: ["Read", "Edit", "Glob"],
    permissionMode: "acceptEdits"
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

## Official Documentation

- [Agent SDK Overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Quickstart](https://platform.claude.com/docs/en/agent-sdk/quickstart)
- [TypeScript SDK](https://github.com/anthropics/claude-agent-sdk-typescript)
- [Python SDK](https://github.com/anthropics/claude-agent-sdk-python)
- [Example Agents](https://github.com/anthropics/claude-agent-sdk-demos)
