# Permission & Safety System Patterns

---

## The Core Insight

A production agent should treat permission as a **multi-mode spectrum** with a three-phase check pipeline and AI-assisted classification for ambiguous cases. The fundamental design philosophy: **fail-closed**. When in doubt, deny.

---

## Pattern 1: Permission Modes as a Trust Spectrum

4 permission modes, ordered from most cautious to most autonomous:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `default` | Ask user for anything write/destructive | Normal interactive use |
| `plan` | Model can read freely, all writes require approval | Review-heavy workflows |
| `bypassPermissions` | Explicit user opt-in to auto-approve everything | Trusted automation, CI |
| `auto` | AI classifier decides; only available when explicitly enabled | Monitored autonomous runs |

This is not a switch (safe vs unsafe) but a **dial** that users can adjust based on their trust level and task context. The progression is deliberate: each step up the spectrum requires explicit user action to unlock.

Key design consequence: the mode is a **user-controlled parameter**, not a programmer-controlled constant. The agent code does not decide which mode is appropriate for a task — the user or operator does.

---

## Pattern 2: Three-Phase Check Pipeline

Every tool call goes through three gates in strict sequence:

```
Phase 1: validateInput()
   ↓ (invalid input → reject immediately, no permission prompt)
Phase 2: tool.checkPermissions(input, context)
   ↓ (tool's own domain-specific check)
   → allow:       proceed to tool execution
   → deny:        block with explanation
   → ask:         show user confirmation dialog
   → passthrough: continue to Phase 3
Phase 3: canUseTool(tool, input, context)
   ↓ (outer permission engine: rules + hooks + classifiers)
   → final allow / deny / ask decision
```

**Why three phases?** Each phase has different information and different authority:

- **Phase 1** — The tool knows what syntactically valid input looks like. Rejecting malformed input early avoids misleading permission prompts ("do you want to allow this?" for something that would fail anyway).
- **Phase 2** — The tool knows what's risky within its own domain. A shell tool understands dangerous command patterns. A file-write tool understands path traversal. The outer system cannot know these domain specifics.
- **Phase 3** — The outer system knows user preferences, session rules, and can invoke expensive AI classifiers. Individual tools should not call the AI classifier; that's the outer engine's responsibility.

The `passthrough` result from Phase 2 is important: it means "I have no opinion on this, defer to the outer system." Tools should not overreach into decisions that belong to the outer permission engine.

---

## Pattern 3: Fail-Closed Principle

Claude Code's permission system defaults to `ask` when:

- A tool is new or unknown to the permission engine
- Input doesn't match any configured allow/deny rules
- Classifier confidence is below the acceptance threshold
- The classifier itself throws an exception

This is the **fail-closed** principle: the safe failure mode is always `ask` or `deny`, never silent allow.

```python
try:
    decision = await classifier.classify(tool, input, history)
    if decision.confidence < THRESHOLD:
        return PermissionResult.ASK  # Uncertain → ask user
    return decision.result
except Exception:
    return PermissionResult.DENY  # Classifier failure → deny
```

The contrast with fail-open: a fail-open system that auto-approves when uncertain will eventually be exploited. A fail-closed system is annoying in edge cases but never catastrophic. For autonomous agents, the cost of an extra confirmation is always lower than the cost of an unintended destructive action.

---

## Pattern 4: Rule-Based Pre-Classification

Before reaching the AI classifier, the system checks a set of user-configured rules. Rules have three sources:

| Source | Who sets it | Examples |
|--------|-------------|---------|
| `settings` | User config file | `alwaysAllowRules: ["read /tmp/**"]` |
| `cli` | Command line flags | `--allow-write=/workspace` |
| `managed` | Enterprise policy | Org-wide deny rules pushed via MDM |

Rules are evaluated in order:
1. `alwaysDenyRules` — auto-block matching patterns
2. `alwaysAskRules` — always trigger confirmation even in `auto` mode
3. `alwaysAllowRules` — auto-approve matching patterns
4. Fall through to AI classifier if no rule matches

**Why rules before AI?** Rules are cheap (O(1) pattern match) and deterministic. AI classification is expensive (latency + tokens) and probabilistic. Running the AI classifier on `cat /tmp/foo.txt` when the user has already said "always allow reads in /tmp" wastes resources and adds latency with no safety benefit.

The rule system also lets users and operators encode known-safe and known-dangerous patterns without relying on the classifier to rediscover them every time.

---

## Pattern 5: Two-Phase AI Classification

For ambiguous cases in `auto` mode, A recommended two-phase approach:

**Phase A — Fast classifier** (heuristic):
- Keyword matching, regex patterns, structural analysis
- Runs in microseconds, in-process
- Handles 80%+ of cases (clear allows and clear denies)
- Returns `uncertain` for genuinely ambiguous cases

**Phase B — Slow classifier** (full AI inference):
- Invokes a language model with full conversation history
- Evaluates whether the request makes sense in context
- Example: "is this `rm -rf build/` command part of a legitimate build cleanup flow, given the preceding messages?"
- Only invoked when Phase A returns `uncertain`

The two-phase design reflects a practical principle: **sort by cost, cheapest first**. Most tool calls are obviously fine or obviously problematic. Paying AI inference costs for every `ls` command would be wasteful and would add hundreds of milliseconds of latency per turn.

The slow classifier's access to conversation history is its key differentiating capability. A command that looks dangerous in isolation may be clearly appropriate given context. The fast classifier cannot evaluate context; the slow classifier can.

---

## Pattern 6: Bypass-Immune Operations

Even in `bypassPermissions` mode, some operations are never auto-approved:

- Actions that affect other users' data (cross-user operations)
- Network requests to sensitive endpoints (configured at the operator level)
- Operations explicitly marked as requiring human confirmation by the tool author

The principle: **bypass is user-controlled, not programmer-controlled**. Users can escalate their own permissions within the bounds of what the system allows, but they cannot escalate beyond those bounds. The ceiling is set by the operator (org policy, managed rules), not by individual users.

This creates a two-level trust hierarchy:
1. **Operator level**: Sets the absolute ceiling (what can never be bypassed)
2. **User level**: Controls their own position within that ceiling (which level of trust they grant the agent)

Tool authors mark bypass-immune operations at the tool definition level, not at the call site. This ensures the restriction cannot be accidentally omitted when the tool is called from a new code path.

---

## Permission System Design Checklist

- [ ] Is your default mode fail-closed (deny/ask) rather than fail-open?
- [ ] Does each tool implement its own domain-specific permission check (Phase 2)?
- [ ] Does Phase 2 return `passthrough` rather than making outer-system decisions?
- [ ] Do you have a rule-based pre-filter before expensive AI classification?
- [ ] Is there an explicit audit trail of all permission decisions with reasons?
- [ ] Can users escalate permissions without code changes (via config/flags)?
- [ ] Are there operations that bypass mode can't override (operator-level ceiling)?
- [ ] Does classifier exception produce `deny`, not `allow`?
- [ ] Does permission failure produce useful error messages explaining what to do?
- [ ] Are permission modes a spectrum, not a binary?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Single global `is_safe()` function | Doesn't scale; misses domain-specific context; single point of failure | Per-tool `checkPermissions()` in Phase 2 |
| Default allow with deny list | Fail-open; new tools automatically get auto-approved until explicitly denied | Default deny with allow list |
| Binary safe/unsafe | Loses context (plan mode, trust level, task type) | Permission spectrum with modes |
| No audit trail | Can't debug or review agent actions after the fact | Log every permission decision with reason, tool, input, and outcome |
| Classifier exception = allow | Silent safety hole; exception handling becomes a bypass vector | Classifier exception = deny, always |
| AI classifier for every call | Adds 100-500ms latency to routine operations | Rule-based pre-filter + two-phase classification |
| Bypass overrides everything | Users can escalate beyond operator-set ceilings | Bypass-immune operations for cross-cutting concerns |
| Permission check in tool execution path | Permission and execution logic entangled; hard to test independently | Separate permission pipeline from execution pipeline |
