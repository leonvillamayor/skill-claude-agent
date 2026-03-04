# Sessions — Deep Reference

## Session Lifecycle

1. `query()` starts → SDK creates session → emits `SystemMessage(subtype="init", session_id=...)`
2. Agent runs, tools execute, context accumulates
3. Result emitted → session persists in storage
4. Resume with `resume=session_id` → full context loaded → agent continues

## Extracting Session ID

```python
# Python
session_id = None
async for message in query(prompt="...", options=...):
    if hasattr(message, "subtype") and message.subtype == "init":
        session_id = message.data.get("session_id")
        print(f"Session: {session_id}")
    # ... handle other messages
```

```typescript
// TypeScript
let sessionId: string | undefined;
for await (const message of query({ prompt: "...", options })) {
  if (message.type === "system" && message.subtype === "init") {
    sessionId = message.session_id;
    console.log(`Session: ${sessionId}`);
  }
}
```

## Resuming Sessions

```python
# Python — resume keeps the SAME session (appends to history)
async for message in query(
    prompt="Continue the refactoring",
    options=ClaudeAgentOptions(
        resume=session_id,
        allowed_tools=["Read", "Edit", "Glob"]  # Can still specify tools
    ),
):
    if hasattr(message, "result"):
        print(message.result)
```

When resuming, Claude has full access to:
- All files it read previously
- Analysis and reasoning from prior turns
- Tool results from earlier in the session
- The full conversation history

## Forking Sessions

Fork creates a **new branch** from the resumed state — original session is unchanged.

```python
# Python
options = ClaudeAgentOptions(resume=session_id, fork_session=True)
```
```typescript
// TypeScript
const options = { resume: sessionId, forkSession: true };
```

### Fork vs Resume Comparison

| Aspect | Resume (default) | Fork |
|--------|-----------------|------|
| Session ID | Same | New ID generated |
| Original modified? | Yes | No (preserved) |
| Conversation history | Appends | New branch from resume point |
| Use case | Linear continuation | Parallel experiments |

### Exploring Multiple Approaches with Forks

```python
# Design stage: capture base session
session_id = None
async for msg in query(prompt="Analyze the auth module", options=ClaudeAgentOptions(allowed_tools=["Read", "Glob"])):
    if hasattr(msg, "subtype") and msg.subtype == "init":
        session_id = msg.data.get("session_id")

# Approach A: JWT refactor (fork, doesn't affect original)
async for msg in query(
    prompt="Refactor auth to use JWT tokens",
    options=ClaudeAgentOptions(resume=session_id, fork_session=True, allowed_tools=["Read", "Edit"])
):
    ...

# Approach B: OAuth refactor (fork, also from original base)
async for msg in query(
    prompt="Refactor auth to use OAuth 2.0",
    options=ClaudeAgentOptions(resume=session_id, fork_session=True, allowed_tools=["Read", "Edit"])
):
    ...
```

## Multi-Turn Sequential Workflow

```python
session_id = None

async def run_turn(prompt_text: str, extra_options: dict = {}):
    global session_id
    opts = ClaudeAgentOptions(
        allowed_tools=["Read", "Edit", "Write", "Glob", "Grep", "Bash"],
        resume=session_id,  # None for first turn
        **extra_options
    )
    async for message in query(prompt=prompt_text, options=opts):
        if hasattr(message, "subtype") and message.subtype == "init":
            session_id = message.data.get("session_id")
        if hasattr(message, "result") and message.subtype == "success":
            return message.result
    return None

# Multi-step pipeline sharing context
await run_turn("Read all Python files in src/ and understand the codebase structure")
await run_turn("Identify all functions that lack docstrings")
await run_turn("Add comprehensive docstrings to each function you found")
await run_turn("Run the test suite and fix any failures")
```

## Session Persistence

- Sessions persist to disk (location depends on `setting_sources` and OS)
- Default cleanup: 30 days (`cleanupPeriodDays` setting)
- Subagent transcripts persist independently from the main session
- Subagent transcripts survive main session compaction

## File Checkpointing

To track and revert file changes across sessions, see the file checkpointing docs:
`https://platform.claude.com/docs/en/agent-sdk/file-checkpointing`

File checkpointing lets you:
- Snapshot file state before agent runs
- Diff changes made during a session
- Revert to a previous snapshot

## Session Discovery (TypeScript)

TypeScript provides functions to list past sessions and read transcripts:

```typescript
import { listSessions, getSessionMessages } from "@anthropic-ai/claude-agent-sdk";

// List sessions (with optional filters)
const sessions = await listSessions({ dir: "/path/to/project", limit: 20 });
// SDKSessionInfo: sessionId, summary, lastModified, fileSize, customTitle?, firstPrompt?, gitBranch?, cwd?

// Read transcript with pagination
const messages = await getSessionMessages(sessions[0].sessionId, { limit: 50, offset: 0 });
```

## Additional Session Options

### continue_conversation
Continue the most recent conversation (instead of specifying a session ID):
```python
options = ClaudeAgentOptions(continue_conversation=True)
```

### TypeScript-only session options
```typescript
const options = {
  sessionId: "custom-uuid-here",       // Use a specific UUID
  persistSession: false,                // Don't save session to disk
  resumeSessionAt: "message-uuid",     // Resume at a specific message in the transcript
};
```

## V2 Preview Sessions (TypeScript — Unstable)

The V2 API provides a session-based interface:

```typescript
import { unstable_v2_createSession, unstable_v2_resumeSession } from "@anthropic-ai/claude-agent-sdk";

// Create
const session = await unstable_v2_createSession({ allowedTools: ["Read"] });
session.send("Hello");
for await (const msg of session.stream()) { ... }

// Resume
const resumed = await unstable_v2_resumeSession(session.sessionId, { allowedTools: ["Read"] });
resumed.send("Continue");
for await (const msg of resumed.stream()) { ... }

// Cleanup (also supports `await using`)
await session.close();
```

## Conversation Compaction

When conversations grow long, the SDK automatically compacts them (summarizes early turns).

**PreCompact hook** fires before compaction — useful for archiving full transcripts:

```python
async def archive_before_compact(input_data, tool_use_id, context):
    session_id = input_data.get("session_id")
    transcript_path = input_data.get("transcript_path")
    # Copy transcript to permanent storage before it gets summarized
    import shutil
    shutil.copy(transcript_path, f"archives/session-{session_id}.jsonl")
    return {}

options = ClaudeAgentOptions(
    hooks={"PreCompact": [HookMatcher(hooks=[archive_before_compact])]}
)
```
