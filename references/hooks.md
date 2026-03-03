# Hooks — Deep Reference

## How Hooks Execute

1. **Event fires** (e.g., PreToolUse before Bash runs)
2. **Matchers filter** — regex against tool name (or event-specific field)
3. **Callbacks run in order** — array position = execution order
4. **deny > ask > allow** — if any hook denies, operation is blocked

## Matcher Patterns

Matchers use regex against tool name for tool-based hooks:
- `"Write|Edit"` — matches Write or Edit tools
- `"^mcp__"` — matches all MCP tools
- `"Bash"` — exact match for Bash
- `None` / omitted — matches every event of that type

## Complete Hook Input Fields

All hook inputs share these common fields:
```json
{
  "hook_event_name": "PreToolUse",
  "session_id": "abc123",
  "cwd": "/path/to/working/dir",
  "tool_name": "Edit",             // PreToolUse / PostToolUse only
  "tool_input": { ... },           // PreToolUse / PostToolUse only
  "tool_response": { ... }         // PostToolUse only
}
```

### PreToolUseHookInput
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test",
    "description": "Clean up temp files",
    "timeout": 30000
  }
}
```

### PostToolUseHookInput
```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Read",
  "tool_input": { "file_path": "/src/auth.py" },
  "tool_response": { "content": "..." }
}
```

### NotificationHookInput
```json
{
  "hook_event_name": "Notification",
  "message": "Claude is waiting for approval",
  "title": "Permission Required",
  "notification_type": "permission_prompt"
}
```
Notification types: `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`

### SubagentStopHookInput
```json
{
  "hook_event_name": "SubagentStop",
  "agent_id": "agent-xyz",
  "agent_transcript_path": "/path/to/transcript.jsonl",
  "stop_hook_active": false
}
```

## Complete Hook Output Schema

```python
return {
    # Top-level: affects conversation
    "systemMessage": "Note: file was redirected to sandbox",  # Injected into context
    "continue_": True,  # True = agent keeps running (Python uses continue_)

    # hookSpecificOutput: affects current operation
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",  # Required — identifies hook type

        # For PreToolUse:
        "permissionDecision": "deny",   # "allow" | "deny" | "ask"
        "permissionDecisionReason": "Cannot write to .env files",
        "updatedInput": { ... },        # Modified tool input (requires allow)

        # For PostToolUse:
        "additionalContext": "This file was last modified 2 days ago.",
    },

    # Async mode (fire-and-forget)
    "async_": True,          # Python (avoid keyword conflict)
    "asyncTimeout": 30000,   # ms
}
```

## Pattern: Block Dangerous Commands

```python
import re

DANGEROUS_PATTERNS = [
    r"rm\s+-rf\s+/",      # rm -rf /
    r"dd\s+if=",           # disk destroyer
    r":\s*\(\)\s*\{",      # fork bomb
    r">\s*/dev/sd",        # overwrite disk
]

async def block_dangerous_bash(input_data, tool_use_id, context):
    if input_data["tool_name"] != "Bash":
        return {}
    command = input_data["tool_input"].get("command", "")
    for pattern in DANGEROUS_PATTERNS:
        if re.search(pattern, command):
            return {
                "systemMessage": "Dangerous command blocked. Please use safer alternatives.",
                "hookSpecificOutput": {
                    "hookEventName": "PreToolUse",
                    "permissionDecision": "deny",
                    "permissionDecisionReason": f"Matched dangerous pattern: {pattern}",
                },
            }
    return {}
```

## Pattern: Redirect All Writes to Sandbox

```python
async def sandbox_writes(input_data, tool_use_id, context):
    if input_data["tool_name"] not in ("Write", "Edit"):
        return {}
    tool_input = input_data["tool_input"]
    original_path = tool_input.get("file_path", "")
    if not original_path.startswith("/sandbox"):
        sandboxed = {**tool_input, "file_path": f"/sandbox{original_path}"}
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "allow",
                "updatedInput": sandboxed,
            }
        }
    return {}
```

## Pattern: Comprehensive Audit Trail

```python
import json
from datetime import datetime

async def audit_all_tools(input_data, tool_use_id, context):
    entry = {
        "timestamp": datetime.now().isoformat(),
        "event": input_data["hook_event_name"],
        "tool": input_data.get("tool_name"),
        "tool_use_id": tool_use_id,
        "input": input_data.get("tool_input"),
        "session": input_data.get("session_id"),
        "cwd": input_data.get("cwd"),
    }
    with open("audit.jsonl", "a") as f:
        f.write(json.dumps(entry) + "\n")
    return {}

# Register for both Pre and Post to track before/after
options = ClaudeAgentOptions(hooks={
    "PreToolUse": [HookMatcher(hooks=[audit_all_tools])],
    "PostToolUse": [HookMatcher(hooks=[audit_all_tools])],
})
```

## Pattern: Rate Limiting

```python
from collections import defaultdict
from time import time

call_counts = defaultdict(list)
MAX_CALLS_PER_MINUTE = 10

async def rate_limiter(input_data, tool_use_id, context):
    tool = input_data.get("tool_name", "unknown")
    now = time()
    # Clean entries older than 60s
    call_counts[tool] = [t for t in call_counts[tool] if now - t < 60]
    if len(call_counts[tool]) >= MAX_CALLS_PER_MINUTE:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": f"Rate limit: {tool} called too frequently",
            }
        }
    call_counts[tool].append(now)
    return {}
```

## Pattern: Slack Notification on Stop

```python
import asyncio, json, urllib.request

async def notify_slack_on_stop(input_data, tool_use_id, context):
    session_id = input_data.get("session_id", "unknown")

    async def send():
        data = json.dumps({"text": f"Agent session {session_id} completed"}).encode()
        req = urllib.request.Request(
            "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
            data=data, headers={"Content-Type": "application/json"}, method="POST"
        )
        try:
            urllib.request.urlopen(req)
        except Exception as e:
            print(f"Slack notification failed: {e}")

    asyncio.create_task(send())
    return {"async_": True, "asyncTimeout": 10000}

options = ClaudeAgentOptions(hooks={"Stop": [HookMatcher(hooks=[notify_slack_on_stop])]})
```

## TypeScript Hook Types

```typescript
import {
  HookCallback,
  PreToolUseHookInput,
  PostToolUseHookInput,
  PostToolUseFailureHookInput,
  NotificationHookInput,
  SubagentStartHookInput,
  SubagentStopHookInput,
  StopHookInput,
  UserPromptSubmitHookInput,
  PreCompactHookInput,
  SessionStartHookInput,   // TypeScript only
  SessionEndHookInput,     // TypeScript only
  TaskCompletedHookInput,  // TypeScript only
  TeammateIdleHookInput,   // TypeScript only
  WorktreeCreateHookInput, // TypeScript only
  WorktreeRemoveHookInput, // TypeScript only
  ConfigChangeHookInput,   // TypeScript only
} from "@anthropic-ai/claude-agent-sdk";
```

## Troubleshooting Hooks

**Hook not firing:**
- Event name is case-sensitive: `PreToolUse` not `preToolUse`
- Add `HookMatcher(hooks=[logger])` (no matcher) to log all events and verify
- Hooks don't fire after `max_turns` is reached

**Matcher not working:**
- Matchers only filter by tool NAME, not by file path or arguments
- Check tool input inside the callback: `input_data["tool_input"]["file_path"]`

**updatedInput not applied:**
- Must include `permissionDecision: "allow"` alongside `updatedInput`
- `updatedInput` must be inside `hookSpecificOutput`, not top-level

**Python sessionStart/sessionEnd not available:**
- These hooks are TypeScript-only for SDK callbacks
- In Python: use `setting_sources=["project"]` to load shell command hooks from `.claude/settings.json`
