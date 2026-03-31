# Tool Design Patterns
---

## The Core Insight

In Claude Code, every tool is not just a function — it's a **specification object** that encodes safety metadata, permission rules, UI rendering instructions, and input/output contracts all in one place. This design means the rest of the system (permission engine, UI, token manager) can make decisions about tools without knowing their internals.

```typescript
// Claude Code's Tool<Input, Output> interface encodes:
// - What the tool does (call, prompt, description)
// - Whether it's safe (isReadOnly, isDestructive, isConcurrencySafe)
// - How to check permissions (checkPermissions)
// - How to display it (renderToolUseMessage, renderToolResultMessage)
// - How to manage its output (maxResultSizeChars)
// - When to load it (shouldDefer, alwaysLoad, searchHint)
```

---

## Pattern 1: Safety Metadata on Every Tool

**Problem**: How does the permission system know if a tool is safe to auto-approve?

**Claude Code Solution**: Each tool declares its own safety profile:

| Property | Type | Meaning |
|----------|------|---------|
| `isReadOnly()` | boolean | Tool never modifies state. Can be auto-approved in conservative modes. |
| `isDestructive()` | boolean | Tool can cause irreversible damage. Extra confirmation required. |
| `isConcurrencySafe()` | boolean | Safe to run in parallel with other tool calls. |

**Principle**: Safety metadata is self-declared by the tool, not inferred by the caller.

**Why this matters**: The permission engine, UI, and parallel execution system all use these properties without needing to understand each tool's internals. It scales to 40+ tools without a giant switch statement.

**Language-agnostic translation**:
```
# Every tool should expose:
tool.is_read_only()      → True/False
tool.is_destructive()    → True/False
tool.is_concurrent_safe() → True/False
```

**Anti-pattern**: Putting safety logic in the orchestrator:
```python
# ❌ BAD: orchestrator knows too much
if tool_name in ["delete_file", "run_migration"]:
    confirm_with_user()

# ✅ GOOD: tool declares its own safety profile
if tool.is_destructive():
    confirm_with_user()
```

---

## Pattern 2: Dual Descriptions (Model vs UI)

**Problem**: The description a model needs to use a tool correctly is different from the label a human needs to understand what just happened.

**Claude Code Solution**:
- `prompt()` — Long-form description **for the model**: includes usage examples, edge cases, when NOT to use, parameter explanations
- `description()` — Short label **for the UI**: human-readable summary shown in the tool call display

**Principle**: Optimize each description for its audience.

**What makes a good `prompt()`** (for the model):
1. Start with what the tool does in one sentence
2. Explain when to use it (and when NOT to use it — this prevents misuse)
3. Describe each parameter with examples
4. Document failure modes and how to handle them
5. Give the model hints about alternatives

**Example from Claude Code's BashTool**:
```
Execute a shell command. Use for system operations, running scripts, and
interacting with external tools.

Do NOT use this tool when a dedicated tool exists (FileRead, Glob, Grep).
Prefer dedicated tools — they provide better UX and are more auditable.

For long-running commands, add background execution hints...
```

**Anti-pattern**: Identical short descriptions for model and UI:
```python
# ❌ BAD: same terse description for model and user
description = "Runs a bash command"

# ✅ GOOD: separate descriptions
model_prompt = """Execute a shell command...
Use for: system operations, running scripts
Do NOT use when: reading files (use FileRead), searching (use Grep)
Parameters:
  command: The shell command to execute. Avoid interactive commands.
  ..."""

ui_label = "Bash"
```

---

## Pattern 3: Tool-Level Permission Check

**Problem**: Permission logic is complex and tool-specific. Who knows best whether a bash command is risky? The bash tool itself.

**Claude Code Solution**: Each tool implements `checkPermissions(input, context)` that returns one of:
- `allow` — proceed automatically
- `deny` — block immediately
- `ask` — show user confirmation dialog
- `passthrough` — delegate to the outer permission layer

The outer permission system calls `checkPermissions()` before executing any tool.

**Principle**: Tools own their permission logic. The outer system orchestrates, but doesn't duplicate tool knowledge.

**Language-agnostic translation**:
```python
class BaseTool:
    def check_permissions(self, input: dict, context: PermissionContext) -> PermissionResult:
        # Default: passthrough to outer permission system
        return PermissionResult.PASSTHROUGH

class DeleteFileTool(BaseTool):
    def check_permissions(self, input: dict, context: PermissionContext) -> PermissionResult:
        if context.mode == "bypass":
            return PermissionResult.ALLOW
        if self._is_in_safe_directory(input["path"]):
            return PermissionResult.ALLOW
        return PermissionResult.ASK  # Outside safe directories, always ask
```

---

## Pattern 4: Deferred Tool Loading

**Problem**: Including all tool schemas in every API request wastes tokens. A system with 40+ tools could add thousands of tokens per request, most of which are never needed.

**Claude Code Solution**: Tools have two loading states:
- **Always-loaded** (`alwaysLoad: true`): Core tools always in the API request
- **Deferred** (`shouldDefer: true`): Not included by default, but discoverable via `ToolSearchTool`

Deferred tools include a `searchHint` — a short string used to match against the model's current task description. When the model calls `ToolSearch`, it gets back only the tools relevant to its current task.

**Principle**: Load tools lazily. The model gets the tools it needs when it needs them.

**How to implement**:
```python
# Tool registry with two pools
core_tools = [BashTool, FileReadTool, FileEditTool, GlobTool]  # Always loaded
deferred_tools = [GitTool, DockerTool, DatabaseTool, SlackTool]  # Load on demand

# ToolSearch tool implementation
def tool_search(query: str) -> list[Tool]:
    """Returns relevant deferred tools based on query similarity"""
    return [t for t in deferred_tools if t.search_hint in query.lower()]
```

**Anti-pattern**: Including all tools in every request:
```python
# ❌ BAD: 40 tools × 500 tokens each = 20,000 tokens per request, every request
tools = get_all_tools()  # All 40 tools always loaded

# ✅ GOOD: progressive disclosure
tools = get_core_tools()  # Always-needed tools
# + ToolSearch to discover domain-specific tools on demand
```

---

## Pattern 5: Result Size Management

**Problem**: Some tools return massive outputs (full file contents, command output, search results). Putting all of this in the context window is wasteful and can exceed limits.

**Claude Code Solution**:
- `maxResultSizeChars` — each tool declares its output size limit
- If output exceeds limit: truncate OR persist to disk and return a reference
- BashTool uses `EndTruncatingAccumulator` — keeps the beginning (more relevant) and truncates the tail
- FileReadTool sets `Infinity` — it uses its own paging mechanism instead

**Language-agnostic principle**:
```python
class Tool:
    MAX_RESULT_CHARS = 50_000  # Default limit

    def call(self, input):
        result = self._execute(input)
        if len(result) > self.MAX_RESULT_CHARS:
            # Option A: Truncate
            return result[:self.MAX_RESULT_CHARS] + "\n[truncated]"
            # Option B: Save to disk, return path
            path = self._persist_result(result)
            return f"[Result too large. Saved to: {path}. Use ReadFile to inspect.]"
        return result
```

---

## Tool Design Checklist

Use this when designing or reviewing tools:

- [ ] Does each tool declare `is_read_only`, `is_destructive`, `is_concurrent_safe`?
- [ ] Does the model-facing description explain when NOT to use the tool?
- [ ] Does each tool implement `check_permissions()` rather than relying on the orchestrator?
- [ ] Are rarely-used tools deferred rather than always-loaded?
- [ ] Does each tool have a result size limit with a truncation/persistence strategy?
- [ ] Are the tool names distinct, non-overlapping, and unambiguous? (Model needs to choose between them)
- [ ] Does the tool's input schema use strict validation (no loose `any` types)?

---

## Anti-Patterns Summary

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Safety logic in orchestrator | Doesn't scale, knowledge duplication | Move `check_permissions()` to tool |
| Single description for model + UI | Model gets too little context, UI too verbose | Separate `prompt()` (model) and `description()` (UI) |
| All tools always loaded | Token waste on every request | Use deferred loading + ToolSearch |
| No output size limits | Context overflow on large outputs | Add `MAX_RESULT_CHARS` with truncation |
| Overlapping tool capabilities | Model picks the wrong tool | Ensure each tool has a unique, non-overlapping purpose |
