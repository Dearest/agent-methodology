# System Prompt Architecture Patterns

---

## The Core Insight

A production agent should split its system prompt into two categories separated by a `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker:

- **Static sections** (before boundary): Same across all users/sessions. Cacheable at the API level = significant token savings.
- **Dynamic sections** (after boundary): Session/user-specific. Volatile, not cached.

This is not just an optimization — it's an architectural discipline.

---

## Pattern 1: Static vs Dynamic Separation

**Static sections** (cacheable, identical across all sessions):

| Section | Content |
|---------|---------|
| Introduction / role definition | Who the model is, security posture declaration |
| System rules | Rules about tool use, formatting, permissions, etc. |
| Task guidelines | Coding standards, over-engineering warnings |
| Actions / safety guidelines | Risky action handling, reversibility |
| Tool usage instructions | Prefer dedicated tools, tool selection guidance |
| Tone and style | Communication norms |
| Output efficiency | Token-conscious output rules |

**Dynamic sections** (volatile, re-computed per session):

| Section | Key | Why it's dynamic |
|---------|-----|-----------------|
| Session guidance | `session_guidance` | Which tools are enabled this session |
| Memory | `memory` | Extracted memory from past conversations |
| Environment info | `env_info` | CWD, git status, OS, model name, current date |
| Language preference | `language` | User's language setting |
| Output style | `output_style` | Custom output style config |
| MCP instructions | `mcp_instructions` | Instructions from connected MCP servers (uses `DANGEROUS_uncachedSystemPromptSection` because MCP servers connect/disconnect dynamically) |
| Scratchpad | `scratchpad` | Temp directory info |

---

## Pattern 2: The Cache Boundary

```typescript
// Everything before this is cacheable (scope: 'global')
const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'

// Static sections assembled first
const prompt = [
  getSimpleIntroSection(),       // role + security declaration
  getSimpleSystemSection(),       // 6 system rules
  getSimpleDoingTasksSection(),   // task guidelines
  getActionsSection(),            // dangerous action handling
  getUsingYourToolsSection(),     // tool usage guidance
  getSimpleToneAndStyleSection(),
  getOutputEfficiencySection(),
  // === BOUNDARY ===
  SYSTEM_PROMPT_DYNAMIC_BOUNDARY,
  // Dynamic sections below (not cached)
  ...resolvedDynamicSections,
]
```

**Why this matters**: The Claude API supports prompt caching. If the first N tokens of the system prompt are identical across requests, they are cached and not re-billed. Mixing stable content with volatile content (e.g., including the current date in a static section) would bust the cache on every single request, eliminating all savings.

The boundary is a hard contract: anything placed before it must be truly invariant across all users and sessions.

---

## Pattern 3: Memoized Section Computation

The framework defines two types of prompt section functions:

- `systemPromptSection(name, compute)` — computed once, memoized until `/clear` or `/compact`
- `DANGEROUS_uncachedSystemPromptSection(name, compute, reason)` — recomputed every turn (the `reason` parameter is mandatory to force the author to justify the decision)

The memoization prevents expensive recomputation (e.g., loading memory files from disk, scanning the filesystem for project context) on every single API call within a session.

The `DANGEROUS_` prefix is a deliberate naming choice: it makes the performance cost visible at the call site. Reviewers can grep for it to audit all hot-path computations.

---

## Pattern 4: Modular Section Functions

Each section is its own function with a clear single responsibility:

- `introSection()` — role definition + security constraints
- `systemRulesSection()` — operating rules
- `taskGuidelinesSection()` — task execution standards
- `actionsSection()` — safety/reversibility guidelines
- `toolUsageSection()` — tool preference hierarchy

This is NOT just organizational tidiness. Each section can be:

1. **Independently tested** — unit test each section's output without rendering the full prompt
2. **Conditionally included** — different content for internal vs external users, or different agent modes
3. **Independently cached** — smaller granularity of what changes means better cache utilization
4. **Replaced for specific agent types** — autonomous agents can get different task guidelines by swapping a single section function

The alternative — a single large template string — makes all of these impossible without string surgery.

---

## Pattern 5: Audience Variants

A production agent may need different prompt content for different user types:

- **External users**: Conservative, concise guidelines optimized for general use
- **Internal (Anthropic) users**: More detailed communication norms, additional debugging instructions, stricter code review standards

The mechanism: an environment variable or configuration flag gates additional instruction blocks. These blocks are injected into the relevant section functions, not appended at the end.

The key insight is that different audiences need different **instruction sets**, not just different data. An internal user debugging a model issue needs different operating norms than an external user writing a web app. Trying to serve both with the same instructions results in prompts that are either too verbose for external users or too sparse for internal ones.

---

## System Prompt Design Checklist

- [ ] Can you draw a clear line between "things that are the same for every user" and "things specific to this session"?
- [ ] Are stable instructions placed in cacheable sections (before the boundary)?
- [ ] Is each section a single responsibility (one function, one concern)?
- [ ] Does the model-facing prompt tell the model WHAT to do AND WHY?
- [ ] Are there audience variants for different user trust levels?
- [ ] Is the prompt modular enough to swap sections for different agent modes?
- [ ] Are expensive section computations memoized?
- [ ] Is uncached computation explicitly marked and justified?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Monolithic single string | Unmaintainable, uncacheable, untestable | Modular section functions |
| Date/time mixed into static instructions | Busts cache on every request | Put volatile data in dynamic sections |
| Same prompt for all user types | Suboptimal for different trust levels | Audience variant sections |
| Prompt grows linearly with features | Token bloat, slower inference | Static/dynamic separation + memoization |
| No explanation of WHY in instructions | Model compliance without understanding; brittle under edge cases | Explain reasoning behind each constraint |
| Uncached sections with no justification | Silent performance regression | `DANGEROUS_uncachedSystemPromptSection` with mandatory reason |
| Dynamic data before the boundary | Cache poisoning — every session looks different | Strict section discipline; boundary enforced by convention |
