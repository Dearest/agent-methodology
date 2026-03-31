# Multi-Agent Orchestration Patterns

## The Core Insight
Sub-agents should be treated as a first-class primitive, not an afterthought. The AgentTool is just another tool that happens to spawn a new agent with its own context, tools, and lifecycle. This means recursion is natural: agents can spawn agents can spawn agents.

## Pattern 1: Agent Definition Schema
Every agent (built-in or custom) is defined by a schema, not subclassing:
```
AgentDefinition:
  agentType: string          # unique identifier
  whenToUse: string          # description for orchestrator model
  tools: string[]            # allowed tools (allowlist)
  disallowedTools: string[]  # blocked tools (denylist)
  permissionMode: PermissionMode  # trust level for this agent
  isolation: 'none' | 'worktree'  # filesystem isolation
  model: string              # can differ from orchestrator
  maxTurns: number           # prevents runaway agents
  systemPrompt: string | fn  # agent-specific system prompt
  background: boolean        # can run without user watching
```

Example agent types: general-purpose, explore/research, planning, verification, domain-specialist

The "whenToUse" string is what the orchestrator model reads to decide which agent to spawn — it's essentially the tool's description, but for agents.

## Pattern 2: Fork vs Fresh Agent

This is the most important architectural decision in multi-agent design.

**Fork (context-inheriting) agent:**
- Inherits the parent's system prompt bytes verbatim
- Shares the parent's prompt cache (dramatic API cost savings)
- Gets the parent's tool set and context
- Fast to spawn, cheap to run
- Use when: parallel research tasks, tasks that need parent's context

**Fresh (isolated) agent:**
- Starts with its own system prompt
- No shared cache with parent
- Custom tool set, can be more restricted
- Isolated filesystem state (optionally)
- Use when: risky operations, distinct task domains, worktree isolation needed

Decision rule:
```
If task needs parent context AND is research/exploration → Fork
If task involves writes AND parent can't trust the output → Fresh
If task needs filesystem isolation → Fresh + worktree
If task is fire-and-forget → Fresh + background: true
```

## Pattern 3: Worktree Isolation
For agents that will make filesystem changes (code edits, file creation), one effective approach is to put them in a git worktree — a separate checkout of the same repository on a new branch.

Why: The parent agent can continue working while the sub-agent works in isolation. The sub-agent's changes are auditable (diff the branch) and reversible (just delete the branch). No merge conflicts with parent.

When to use: Any sub-agent that will make writes that the user hasn't yet approved. Any "speculative" code generation.

## Pattern 4: Background Execution with Task Tracking
Long-running agents should auto-migrate to background after a configurable timeout (e.g., ~120 seconds). The system:
1. Creates a Task record with: agent_id, status (running/done/failed), output_path
2. The task runs independently
3. User can query task status, read partial output, cancel
4. On completion, output is persisted to disk (not held in parent context)

Key insight: **background execution breaks the request-response model**. The parent doesn't wait. It continues chatting with the user and checks in on tasks asynchronously.

```python
# Anti-pattern: blocking
result = agent.run(task)  # blocks for 10 minutes

# Claude Code pattern: non-blocking
task_id = agent.spawn_background(task)
# Continue other work...
result = await agent.get_result(task_id)  # check when needed
```

## Pattern 5: Agent Communication
Use a message-passing tool for inter-agent communication. An orchestrator can send messages to running sub-agents, and sub-agents can push status updates back to the orchestrator.

For team-based coordination (multiple agents working on different parts of a problem), a team management system can create named agent groups with a shared message channel.

## Multi-Agent Design Checklist
- Does each agent type have a clear, non-overlapping responsibility?
- Is isolation level (fork vs fresh vs worktree) appropriate for the risk level?
- Do agents have maxTurns limits to prevent infinite loops?
- Are long-running agents designed for background execution?
- Is there a way to audit what sub-agents did (logs, task records)?
- Can the orchestrator recover if a sub-agent fails?

## Anti-Patterns
| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Monolithic single agent | Single point of failure, context bloat | Decompose into specialized agents |
| Agents without maxTurns | Infinite loops possible | Always set maxTurns |
| Blocking on sub-agent results | Poor UX for long tasks | Background execution + task tracking |
| Agents sharing mutable state | Race conditions | Isolated worktrees or message-passing only |
| No audit trail for sub-agent actions | Can't debug or review | Task records + action logs |
