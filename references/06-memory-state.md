# Memory & State Management Patterns

## The Core Insight
A production agent should distinguish between four types of state with different lifetimes and access patterns. Conflating them leads to either data loss (too volatile) or stale data (too persistent).

## The Four State Types

| Type | Lifetime | Storage | Access |
|------|----------|---------|--------|
| **Turn state** | Single API call | In-context | Immediate |
| **Session state** | Conversation duration | In-memory (AppState) | Fast |
| **Persistent memory** | Across sessions | Disk (memdir) | Loaded at session start |
| **Project config** | Project lifetime | Config files (e.g., agent.md) | Read at init + on demand |

## Pattern 1: Hierarchical Configuration
A production agent should read configuration from a hierarchy of config files:

```
~/.agent/config.md            ← global user preferences
  /project-root/agent.md      ← project-wide config
    /src/agent.md             ← subdirectory overrides
      /src/api/agent.md       ← even more specific
```

Each level can add instructions, and lower levels override higher levels. This allows:
- Global: always use TypeScript strict mode
- Project: this project uses PEP8
- Subdirectory: this API module requires extra security care

**For your agent**: Design a config hierarchy that lets users set preferences at multiple scopes without requiring code changes.

## Pattern 2: Memory Extraction and Injection
A robust agent should have an automatic memory system:
1. **Extraction**: After conversations, an extraction agent identifies facts worth remembering (user preferences, project constraints, recurring patterns)
2. **Storage**: Facts stored in structured files on disk
3. **Injection**: At session start, relevant memories are loaded into the `memory` dynamic prompt section

Categories to extract: user preferences, project context, feedback (what not to do), reference pointers

Key design: memory is **extracted by a dedicated agent**, not manually curated. The agent decides what's worth remembering.

## Pattern 3: AppState as Session-Scoped Store
A session-scoped store (AppState) holds everything that changes during a session:
- Current conversation messages
- In-progress tool calls
- File state cache (what's been read/written)
- Permission decisions made this session
- Background task registry
- Cost/token tracking

It follows a React-like immutable update pattern:
```typescript
setAppState(prev => ({
  ...prev,
  messages: [...prev.messages, newMessage]
}))
```

Why immutable? Makes debugging easier (you can replay state changes), prevents accidental mutations across async operations.

## Pattern 4: File State Cache
Track files that have been read or modified during the session. This serves multiple purposes:
- Detect external modifications (another process changed a file we're about to edit)
- Avoid re-reading files that haven't changed
- Build an audit trail of file accesses

```python
file_state_cache = {
  "/path/to/file.ts": {
    "last_read": timestamp,
    "content_hash": hash,
    "last_modified_by_us": True
  }
}
```

## Pattern 5: Tool Decision Tracking
Record every permission decision made during a session:
```typescript
toolDecisions: Map<toolUseId, {
  source: string,    // what made the decision (classifier, user, rule)
  decision: 'accept' | 'reject',
  timestamp: number
}>
```

This enables: debugging unexpected denials, auditing what the agent did, supporting "undo" workflows.

## Memory & State Design Checklist
- Have you identified all four state types in your agent and assigned appropriate storage?
- Is there a hierarchical config system for user/project/subdirectory preferences?
- Does your agent extract useful facts for future sessions?
- Is session state immutably updated?
- Do you cache expensive lookups (file reads, API calls) within a session?
- Is there an audit trail of significant decisions?

## Anti-Patterns
| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Global mutable state | Hard to debug, async race conditions | Immutable update pattern |
| Re-reading files on every access | Unnecessary I/O, misses external changes | File state cache |
| Monolithic config file | Can't override per-project/directory | Hierarchical config (CLAUDE.md pattern) |
| Manual memory management | Users forget to update, stale data | Automatic extraction agent |
| No audit trail | Can't debug agent decisions | Decision tracking per session |
