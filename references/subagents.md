# Subagents — Deep Reference

## Why Use Subagents

| Problem | Subagent solution |
|---------|-------------------|
| Main context getting cluttered | Subagents have isolated context |
| Long sequential tasks | Run multiple subagents in parallel |
| Needing specialized expertise | Each subagent has its own system prompt |
| Preventing accidental modifications | Restrict subagent to read-only tools |

## Three Ways to Define Subagents

1. **Programmatic** (recommended for SDK) — `agents` parameter in `query()`
2. **Filesystem** — Markdown files in `.claude/agents/` directories
3. **Built-in** — `general-purpose` agent available automatically when `Task` is in allowedTools

## AgentDefinition — All Fields

```python
# Python
from claude_agent_sdk import AgentDefinition

AgentDefinition(
    description="When to use me (Claude reads this to decide auto-invocation)",
    prompt="My system prompt — expertise, constraints, output format",
    tools=["Read", "Glob", "Grep"],  # None = inherit all parent tools
    model="sonnet",  # "sonnet" | "opus" | "haiku" | "inherit" | model ID
)
```

```typescript
// TypeScript
interface AgentDefinition {
  description: string;  // Required
  prompt: string;       // Required
  tools?: string[];     // Optional — omit to inherit all
  model?: "sonnet" | "opus" | "haiku" | "inherit" | string;  // Optional
}
```

**Key notes:**
- `description` drives auto-invocation — write it to match when you want the subagent used
- Subagents CANNOT spawn their own subagents (no `Task` in subagent `tools`)
- Subagent messages have `parent_tool_use_id` set to link them to their Task call

## Invoking Subagents

### Auto-invocation
Claude reads descriptions and decides. Better descriptions = better delegation.

**Good description:** `"Security vulnerability scanner. Use for OWASP analysis, CVE detection, and injection vulnerability checks in Python/TypeScript code."`

**Bad description:** `"Checks code"` (too vague — Claude won't know when to use it)

### Explicit invocation
Force Claude to use a specific agent:
```
"Use the security-scanner agent to check auth.py for SQL injection"
```

## Parallel Multi-Agent Patterns

```python
# Main agent delegates to multiple specialists simultaneously
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Bash", "Task"],
    system_prompt="""You coordinate multiple specialist agents. When reviewing code:
1. SIMULTANEOUSLY dispatch security-scanner, style-checker, and test-runner
2. Collect their results
3. Synthesize into a unified report""",
    agents={
        "security-scanner": AgentDefinition(
            description="Security vulnerability scanner. Use to find OWASP issues, injection vulnerabilities, auth flaws.",
            prompt="Identify all security vulnerabilities. Report each with: severity (critical/high/medium/low), location, description, and fix recommendation.",
            tools=["Read", "Grep", "Glob"],
        ),
        "style-checker": AgentDefinition(
            description="Code style and quality analyst. Use to check linting, formatting, and best practices.",
            prompt="Analyze code style. Check: naming conventions, function length, complexity, dead code, missing types.",
            tools=["Read", "Glob", "Grep", "Bash"],
            model="haiku",  # Cheaper model for simpler task
        ),
        "test-runner": AgentDefinition(
            description="Test execution specialist. Use to run tests and analyze failures.",
            prompt="Run all tests. Report: pass/fail counts, failing test names, error messages, potential fixes.",
            tools=["Bash", "Read", "Grep"],
        ),
    },
)
```

## Dynamic Agent Creation

Create agent definitions at runtime based on conditions:

```python
def make_reviewer(language: str, strictness: str = "normal") -> AgentDefinition:
    models = {"strict": "opus", "normal": "sonnet", "fast": "haiku"}
    return AgentDefinition(
        description=f"{language} code reviewer",
        prompt=f"""You are a {'strict' if strictness == 'strict' else 'practical'}
{language} expert. Review code for correctness, performance, and {language} idioms.""",
        tools=["Read", "Glob", "Grep"],
        model=models.get(strictness, "sonnet"),
    )

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Task"],
    agents={
        "python-reviewer": make_reviewer("Python", "strict"),
        "typescript-reviewer": make_reviewer("TypeScript", "normal"),
    }
)
```

## Detecting Subagent Messages

```python
# Python — messages from subagents have parent_tool_use_id set
async for message in query(prompt="...", options=options):
    if hasattr(message, "parent_tool_use_id") and message.parent_tool_use_id:
        print(f"  [SUBAGENT] {message}")
    else:
        print(f"[MAIN] {message}")

    # Detect when a subagent IS invoked (Task tool call in main agent)
    if hasattr(message, "content"):
        for block in message.content:
            if getattr(block, "type", None) == "tool_use" and block.name == "Task":
                print(f"Dispatching subagent: {block.input.get('subagent_type')}")
```

## Resuming Subagents

Subagents can be resumed with their full transcript intact:

```python
import re, json

agent_id = None
session_id = None

# First run
async for message in query(
    prompt="Use the Explore agent to map all API endpoints",
    options=ClaudeAgentOptions(allowed_tools=["Read", "Grep", "Glob", "Task"]),
):
    if hasattr(message, "session_id"):
        session_id = message.session_id
    if hasattr(message, "content"):
        content_str = json.dumps(message.content, default=str)
        match = re.search(r"agentId:\s*([a-f0-9-]+)", content_str)
        if match:
            agent_id = match.group(1)
    if hasattr(message, "result"):
        print(message.result)

# Resume subagent in same session
if agent_id and session_id:
    async for message in query(
        prompt=f"Resume agent {agent_id} and list the 5 most complex endpoints",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Grep", "Glob", "Task"],
            resume=session_id,  # Must resume SAME session
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)
```

**Important:** Must use same `session_id` to access the subagent transcript.

## Tool Restriction Patterns

```python
# Read-only analyst
AgentDefinition(
    description="Architecture reviewer",
    tools=["Read", "Grep", "Glob"],
    # Cannot Edit, Write, or run Bash
)

# Test-only executor
AgentDefinition(
    description="Test runner",
    tools=["Bash", "Read", "Grep"],
    # Can run commands but not modify source files
)

# Web researcher
AgentDefinition(
    description="Web research specialist",
    tools=["WebSearch", "WebFetch"],
    # No filesystem access
)
```

## Filesystem-Based Subagents

Place Markdown files in `.claude/agents/`:

```markdown
---
name: my-agent
description: When to use this agent
tools: Read, Glob, Grep
model: sonnet
---

# Agent Name

Your system prompt here.
```

Load with `setting_sources=["project"]` in your options.
Programmatic definitions take priority over filesystem-based ones with the same name.

## Troubleshooting

**Claude not using subagents:**
1. Verify `"Task"` is in `allowedTools`
2. Use explicit naming: `"Use the X agent to..."`
3. Make description more specific and action-oriented

**Windows subagent failures:**
Long prompts may fail due to 8191-char command line limit. Use filesystem-based agents or keep prompts under 1000 chars.

**Infinite loops with UserPromptSubmit + subagents:**
A `UserPromptSubmit` hook that spawns subagents can recurse. Check `parent_tool_use_id` in hook input to detect if you're already in a subagent context.
