# Extensibility Patterns

## The Core Insight
A production agent should be designed for extensibility at every layer. The key design philosophy: **extend through protocols and events, not through inheritance or code modification**. Users and developers can add capabilities without touching Claude Code's core.

## Pattern 1: Protocol-Based Extensibility (MCP)
A standard tool protocol (like MCP — Model Context Protocol) is the primary extensibility mechanism for tools. Instead of writing a plugin, you write a protocol-compliant server that:
1. Exposes tools with standard schemas
2. Connects via a standard transport (stdio, HTTP, WebSocket)
3. Gets automatically wrapped into agent tools

Why protocols over plugins? Protocol compliance is verifiable. Any MCP-compliant server works with any MCP-compliant client. The agent doesn't need to know the implementation details of the MCP server.

**For your agent**: Design a protocol layer for external integrations. Don't require contributors to understand your internals.

## Pattern 2: Event-Based Extensibility (Hooks)
Define hook points where users can inject custom behavior. Examples:
- `PreToolUse` — intercept before any tool runs
- `PostToolUse` — observe tool results
- `SessionStart` — run setup logic at conversation start
- `SessionEnd` — cleanup/notification
- `UserPromptSubmit` — transform user input before processing

Hooks can be shell commands configured in settings. The agent runs them and reads their output/exit codes to decide whether to proceed.

This is event-driven extensibility — instead of exposing internal APIs, expose **lifecycle events** that external processes can observe and influence.

**For your agent**: Identify the 5-10 most important lifecycle moments and expose them as hook points. This lets users customize behavior without modifying your agent.

## Pattern 3: Skill System (User-Defined Workflows)
Skills are markdown files that get injected into the system prompt when triggered. They let users define:
- When to trigger (via description matching)
- What to do (instructions in the skill body)
- What files to reference (bundled resources)

This gives users a **declarative** way to add expertise to Claude Code — no code required.

**For your agent**: Consider a skill/template system where users can define workflows as structured text files. The agent reads the relevant skill when the task matches.

## Pattern 4: Plugin System
A plugin system allows bundling:
- Tools (new capabilities)
- Skills (new workflows)
- MCP servers (new protocol servers)

...into a single distributable file. Plugins are installed and versioned independently.

Key design: plugins don't modify core code, they register into the tool/skill/MCP registries.

## Pattern 5: Configuration Layering
Extensibility isn't just about adding tools — it's also about allowing users to configure behavior. Layer configuration as follows:
1. Built-in defaults
2. Global user config (~/.agent/settings.json)
3. Project config (agent.md or similar)
4. CLI flags (override everything for this session)
5. Enterprise managed settings (override everything, always)

Each layer can add or override the previous. The enterprise layer is bypass-immune — even `bypassPermissions` can't override enterprise-managed settings.

## Extensibility Design Checklist
- Is there a standard protocol for adding tools/capabilities without code changes?
- Are lifecycle events exposed as hookable points?
- Can users define workflows/templates without modifying core logic?
- Is there a plugin/extension packaging mechanism?
- Does configuration support layering (global → project → session)?
- Are enterprise/admin settings protected from user override?

## Anti-Patterns
| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Core modification required to add tools | Tight coupling, hard to maintain | Protocol-based tool registration |
| No hook points | Can't customize without forking | Expose lifecycle events as hooks |
| Single flat config file | Can't override per-project | Layered config hierarchy |
| Plugin system requires code knowledge | Limits who can extend | Declarative skill/template system |
| No extension versioning | Breaking changes break all plugins | Versioned extension API |
