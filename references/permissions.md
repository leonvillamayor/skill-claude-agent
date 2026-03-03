# Permissions & User Input — Deep Reference

## Permission Evaluation Order

```
1. Hooks (PreToolUse) → can allow/deny/modify
2. Declarative rules from settings.json → deny → allow → ask
3. Permission mode (default/acceptEdits/bypassPermissions/plan)
4. canUseTool callback → runtime user decision
```

## Permission Modes

### default
- No auto-approvals
- Unmatched tools call `canUseTool` callback
- Best for interactive applications where users approve actions

### acceptEdits
Auto-approves:
- `Edit` tool (file edits)
- `Write` tool (create files)
- `Bash` filesystem commands: `mkdir`, `touch`, `rm`, `mv`, `cp`

Everything else still needs approval or uses `canUseTool`.

```python
options = ClaudeAgentOptions(permission_mode="acceptEdits")
```

### bypassPermissions
- All tools run without prompts
- Hooks still run and can block
- ⚠️ Subagents inherit `bypassPermissions` — they get full access too
- Use only in controlled/trusted environments (CI/CD, sandboxed containers)

```python
options = ClaudeAgentOptions(
    permission_mode="bypassPermissions",
    system_prompt="CI/CD automation agent. Work autonomously.",
)
```

### plan
- No tool execution at all
- Claude can analyze files and create plans
- Claude may call `AskUserQuestion` to gather requirements
- Good for "dry run" / pre-approval workflows

```python
# Show Claude's plan without executing
options = ClaudeAgentOptions(
    permission_mode="plan",
    allowed_tools=["Read", "Glob", "Grep", "AskUserQuestion"]
)
async for message in query(prompt="Plan a refactor of the auth module", options=options):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if hasattr(block, "text"):
                print(block.text)  # Claude's plan, no files changed
```

## Dynamic Permission Mode Changes

Change permission mode mid-session:

```python
# Start strict, loosen after user reviews Claude's approach
q = query(prompt="Refactor the database layer", options=ClaudeAgentOptions(permission_mode="plan"))

# After a few turns reviewing Claude's plan...
await q.set_permission_mode("acceptEdits")  # Let it execute now

async for message in q:
    if hasattr(message, "result"):
        print(message.result)
```

```typescript
const q = query({ prompt: "...", options: { permissionMode: "plan" } });
await q.setPermissionMode("acceptEdits");
for await (const message of q) { ... }
```

## canUseTool Callback

### Signatures

```python
# Python
async def can_use_tool(
    tool_name: str,
    input_data: dict,       # Tool-specific arguments
    context: ToolPermissionContext
) -> PermissionResultAllow | PermissionResultDeny:
    ...
```

```typescript
// TypeScript
type CanUseTool = (
  toolName: string,
  input: Record<string, unknown>
) => Promise<{ behavior: "allow"; updatedInput: unknown } | { behavior: "deny"; message: string }>;
```

### Return Values

```python
# Python
from claude_agent_sdk.types import PermissionResultAllow, PermissionResultDeny

# Allow (optionally modify input)
return PermissionResultAllow(updated_input=input_data)

# Deny (Claude sees the message and may adjust)
return PermissionResultDeny(message="Cannot delete production database")
```

```typescript
// TypeScript
return { behavior: "allow", updatedInput: input };
return { behavior: "deny", message: "Cannot delete production database" };
```

### Python Workaround (Required)

In Python, `can_use_tool` requires streaming input AND a dummy `PreToolUse` hook:

```python
from claude_agent_sdk import HookMatcher

async def dummy_hook(input_data, tool_use_id, context):
    return {"continue_": True}  # Keep stream open

async def prompt_generator():
    yield {
        "type": "user",
        "message": {"role": "user", "content": "Your prompt here"}
    }

options = ClaudeAgentOptions(
    can_use_tool=my_can_use_tool,
    hooks={"PreToolUse": [HookMatcher(matcher=None, hooks=[dummy_hook])]},
)

async for message in query(prompt=prompt_generator(), options=options):
    ...
```

### AskUserQuestion via canUseTool

```python
async def can_use_tool(tool_name, input_data, context):
    if tool_name == "AskUserQuestion":
        answers = {}
        for q in input_data.get("questions", []):
            print(f"\n{'=' * 40}")
            print(f"Q: {q['question']}")
            for i, opt in enumerate(q["options"]):
                print(f"  {i+1}. [{opt['label']}] {opt['description']}")

            if q.get("multiSelect"):
                raw = input("Select (comma-separated numbers or text): ")
                try:
                    indices = [int(x.strip()) - 1 for x in raw.split(",")]
                    selected = [q["options"][i]["label"] for i in indices if 0 <= i < len(q["options"])]
                    answers[q["question"]] = ", ".join(selected) if selected else raw
                except ValueError:
                    answers[q["question"]] = raw
            else:
                raw = input("Select (number or text): ").strip()
                try:
                    idx = int(raw) - 1
                    answers[q["question"]] = q["options"][idx]["label"]
                except (ValueError, IndexError):
                    answers[q["question"]] = raw

        return PermissionResultAllow(updated_input={
            "questions": input_data["questions"],
            "answers": answers
        })

    # Handle tool approvals
    print(f"\nTool: {tool_name}")
    print(f"Input: {json.dumps(input_data, indent=2)}")
    approved = input("Allow? (y/n/modify): ").strip().lower()

    if approved == "y":
        return PermissionResultAllow(updated_input=input_data)
    elif approved == "modify":
        # Let user modify the input
        new_input = json.loads(input("New input JSON: "))
        return PermissionResultAllow(updated_input=new_input)
    else:
        reason = input("Reason for denial: ")
        return PermissionResultDeny(message=reason or "User denied")
```

## Declarative Rules (settings.json)

For file-based permission rules (loaded when `setting_sources=["project"]`):

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Read(*)",
      "Glob(*)",
      "Grep(*)",
      "Bash(git status)",
      "Bash(git log*)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Write(/etc/*)",
      "Edit(/etc/*)"
    ],
    "ask": [
      "Bash(*)",
      "Write(*)"
    ]
  }
}
```

Load with:
```python
options = ClaudeAgentOptions(setting_sources=["project"])
```

## allowedTools vs disallowedTools

```python
# allowedTools: whitelist (only these tools allowed)
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep"]  # ONLY these tools
)

# disallowedTools: blacklist (everything EXCEPT these)
options = ClaudeAgentOptions(
    disallowed_tools=["Bash", "Write", "Edit"]  # Everything else allowed
)

# Both together: allowedTools takes priority as the initial whitelist,
# disallowedTools removes from that set
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Write", "Bash"],
    disallowed_tools=["Bash"]  # Net result: Read, Edit, Write only
)
```

## Typical Configurations

```python
# Read-only / safe analysis
ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep"],
    permission_mode="bypassPermissions"
)

# Trusted development (auto-approve file ops)
ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Write", "Glob", "Grep", "Bash"],
    permission_mode="acceptEdits"
)

# Full automation (CI/CD)
ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Write", "Bash", "Glob", "Grep"],
    permission_mode="bypassPermissions"
)

# Interactive with user approval
ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Write", "Bash", "Glob", "Grep"],
    permission_mode="default",
    can_use_tool=my_approval_callback,
)

# Planning mode (no changes)
ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "AskUserQuestion"],
    permission_mode="plan"
)
```
