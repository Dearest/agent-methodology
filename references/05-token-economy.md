# Context & Token Economy Patterns

## The Core Insight
At scale, token usage is a first-class engineering concern. Treat tokens like memory in systems programming — you think carefully about allocation, reuse, and garbage collection. Bad token economics don't matter at prototype scale; they become critical at production scale.

## Pattern 1: The Four Token Pools
Every agent's context has four types of content, each with different management needs:

| Pool | Content | Management Strategy |
|------|---------|-------------------|
| **System Prompt** | Instructions, personality, constraints | Static/dynamic split for caching |
| **Tool Schemas** | Tool names, parameters, descriptions | Deferred loading — load only what's needed |
| **Conversation History** | User/assistant/tool messages | Compress on threshold, summarize old turns |
| **Tool Results** | Output from tool calls | Truncate large results, persist to disk |

Each pool needs a different management strategy.

## Pattern 2: Prompt Caching Strategy
Claude's API supports prompt caching: if the first N tokens of the system prompt match a previous request, they're reused (billed at a fraction of the cost).

Recommended approach:
1. Put all stable instructions BEFORE the cache boundary
2. Put all volatile data (session-specific) AFTER the boundary
3. Memoize section computation — recompute only when /clear is called

Token savings observed in practice: 60-80% reduction in system prompt token cost for sessions with multiple turns.

**Implementation principle**: Ask "does this change between turns?" — if no, it belongs in the cached prefix.

## Pattern 3: Deferred Tool Loading
Tool schemas are tokens. An agent with 40+ tools that includes all schemas every request would add ~20,000 tokens per request.

Solution: Two-tier tool loading
- **Core tools** (~8-10 tools): Always included. These are the tools used in 90%+ of tasks.
- **Deferred tools**: Not included by default. Model can discover them via ToolSearch.

ToolSearch takes a query and returns only the tools relevant to the current task. This means the model loads exactly the tools it needs, when it needs them.

For your agent: identify which tools are used in most tasks (always-load) vs domain-specific tools (deferred).

## Pattern 4: Tool Result Budget
The system tracks total token usage of tool results in a conversation. When results accumulate past a threshold, older results are replaced with summaries like `[Result replaced to save context]`.

Why: Tool results are the fastest-growing part of context in an active coding session. Without management, a session with 20+ tool calls would exceed context limits.

Implementation:
```python
class ToolResultBudget:
    MAX_TOTAL_CHARS = 200_000  # example

    def add_result(self, result: str) -> str:
        if self.total + len(result) > self.MAX_TOTAL_CHARS:
            # Evict oldest results first
            self._evict_oldest_results()
        self.total += len(result)
        return result
```

## Pattern 5: Context Compression
When total context approaches the model's limit, trigger automatic summarization:
1. Takes the oldest N turns of conversation
2. Spawns a summarization sub-agent to condense them
3. Replaces those turns with a compact summary
4. Continues the conversation

Key design: summarization happens **proactively** (before hitting the limit) not reactively (after hitting it). Running out of context mid-task is much worse than a brief pause for compression.

## Pattern 6: Large Output Offloading
For tools that can produce very large outputs (file contents, search results, command output), offload to disk when output exceeds the limit:

```typescript
if (output.length > MAX_RESULT_CHARS) {
  const path = persistToDisk(output)
  return `[Output too large (${output.length} chars). Saved to: ${path}. Use ReadFile to inspect sections.]`
}
```

The model can then selectively read parts of the output using FileRead with line ranges, rather than loading all of it into context.

## Token Economy Checklist
- Do you have a caching strategy? What's in the static prefix vs volatile suffix?
- Are rarely-used tools deferred to reduce per-request token cost?
- Is there a tool result budget/eviction strategy?
- What triggers context compression, and does it happen proactively?
- For large outputs: truncate OR offload to disk with a reference?
- Do you track token usage per session to spot problems early?

## Anti-Patterns
| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| All tools in every request | ~20K wasted tokens per request | Deferred tool loading |
| Volatile data in system prompt prefix | Busts prompt cache every request | Static/dynamic split |
| Tool results grow unbounded | Context overflow after many tool calls | Result budget + eviction |
| Reactive compression (at limit) | Mid-task context loss | Proactive compression (before limit) |
| Large output in context | Single large output can fill context | Disk offload with file reference |
