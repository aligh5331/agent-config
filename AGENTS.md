# AGENTS.md — Go Agentic Coding Stack

## Project Identity

- **Language:** Go (primary), occasional shell/Python for tooling
- **Stack:** OpenCode v1.17.9, Go binaries, minimal containers
- **Hosts:** Devbox VM `192.168.0.11` (Ubuntu 22.04), home server `192.168.0.10`
- **Goal:** Production-grade Go services — spec-driven, minimal deps, well-factored

---

## Model Routing

| Mode | Model | When |
|------|-------|------|
| **Architect** | DeepSeek V4 Flash (OpenCode Zen) | Design docs, interface contracts, package structure decisions |
| **Author-Librarian** | DeepSeek V4 Flash (OpenCode Zen) | Write/update specs, AGENTS.md PRs, skill file edits, code review prose |
| **Builder** | Qwen3.6-28B (`localhost:7890`) | Implementation from spec — one feature per session |
| **Forensic Specialist** | Qwen3.6-28B (`localhost:7890`) | Debugging, tracing, goroutine/deadlock analysis |

### Routing Rules

- Always start on **DeepSeek** for the spec/plan phase. Only hand off to **Qwen3.6** when a `.feature` file exists and is approved.
- **Mode must be declared at session start.** Use: `MODE: [[ARCHITECT|BUILDER|FORENSIC|LIBRARIAN]]`
- If Qwen3.6 loops for ≥3 turns with no state-mutating output → force-switch to DeepSeek for re-scoping. Do not continue in Builder mode.
- **Approval fatigue rule:** Once a plan is approved, Builder executes without asking for per-step confirmation. Only stop for: file writes outside the declared scope, ambiguity that cannot be resolved from the spec, or a triggered termination heuristic.

---

## Hard Constraints

### Token Budget (Qwen3.6 — local)

- **Effective ceiling:** ~29K tokens (VRAM-bound, non-negotiable)
- **Skill file target:** ≤800 tokens each (7 core skills ≈ 5.6K)
- **AGENTS.md target:** ≤2K tokens (this file is an exception at ~3K — do not grow it further)
- **OpenCode output limit:** hard-capped at 4K tokens per response

### Session Input Budget

- Max prompt input: ~15K tokens (leaves ~14K for reasoning + output)
- If accumulated prompt + context would exceed 15K → reply `OVER-BUDGET: needs scoping` and stop
- Every 5th agent turn: emit `# CONTEXT REPORT: ~Xk used / ~Yk remaining`
- When <4K headroom remains → finish the current logical unit, commit, start a fresh session

### MCP Whitelist

- **Permitted:** `gopls` only (absolute path: `/home/ali/go/bin/gopls`)
- No filesystem MCP, no browser MCP, no shell MCP — OpenCode built-ins cover those
- Adding any new MCP requires a spec ticket and explicit human approval

---

## Termination Heuristics (Critical — Qwen3.6 Looping)

Qwen3.6 has a **confirmed looping failure mode** on underspecified tasks (root cause: missing termination heuristic, not capability gap). These rules are mandatory.

### Pre-task Validation
Before any constraint-satisfaction, logic, or multi-step reasoning task:
1. List every entity mentioned in the task.
2. Confirm each entity appears in at least one constraint or spec step.
3. If any entity is unconstrained → **STOP. Report:** `"Entity [[X]] has no constraint. Task is underspecified."`
4. Do not proceed until the spec is fixed.

### Loop Detection
1. ≥3 consecutive turns with no file writes or state-mutating tool calls → force-stop, re-scope
2. Same file rewritten ≥4 times in one session → halt, report `"OSCILLATING: [[filename]]"`, switch to DeepSeek
3. Case/sub-case nesting depth ≥3 → halt. Deeper nesting signals looping, not progress.
4. Output repetition (same error, same explanation) → emergency stop

### Named Failure Modes
- **Context hallucination:** Agent references a file, symbol, or fact not present in the current context window. When detected → stop, fetch the actual file, restart the reasoning step.
- **Approval fatigue:** Agent asks for human confirmation on every trivial step. Prevented by the routing rule above — once a plan is approved, execute it.

### Emergency Commands (in any file or comment)
- `// EMERGENCY STOP` → halt immediately, await human direction
- `// RE-SCOPE NEEDED` → current approach is over-budget or underspecified; do not continue

---

## JIT Downscoping (File-Tree Allowlist)

At the start of every Builder or Forensic session, declare the active scope:

```
SCOPE: [[comma-separated list of files or directories]]
```

Example:
```
SCOPE: internal/storage/postgres.go, internal/storage/postgres_test.go
```

- The agent must not read, write, or reference files outside the declared scope without explicit human approval.
- If a fix requires a file outside scope → stop and report `"OUT-OF-SCOPE: [[filename]] needed"`.
- Scope expands only via explicit human instruction in the same session.

---

## Workflow (Spec-Driven Development)

The spec is the primary artifact. Code is disposable. The spec is not.

### Session Template

Every session must begin with:

```
MODE: [[ARCHITECT|BUILDER|FORENSIC|LIBRARIAN]]
SPEC: [[path/to/feature.feature or "none — creating spec"]]
SCOPE: [[file-tree allowlist]]
BUDGET: [[estimated token cost: small <5K | medium 5–10K | large 10–15K]]
```

### The Five Stages

1. **Architect (DeepSeek):** Define interfaces, package boundaries, data flow. Output: design doc or ADR.
2. **Author-Librarian (DeepSeek):** Write BDD/Gherkin `.feature` file. Validate: every entity in every scenario appears in a Given/When/Then step. Declare token budget in spec header.
3. **Builder (Qwen3.6):** Implement from spec — one `.feature` file per session. No spec = no code.
4. **Forensic Specialist (Qwen3.6):** Debug failures by tracing against the spec steps, not intuition.
5. **Reviewer (DeepSeek):** Final pass — does the diff satisfy the spec? Runs Vibe Diff summary before merge.

### Required Before Any Code

- `.feature` file must exist and be committed
- Token budget declared in spec header
- Session template filled in
- Scope declared

---

## Placeholder Convention

Use `[[VARIABLE_NAME]]` for any context-specific value that must be filled in at session time. Never hardcode values that will change between sessions. Examples:

```
MODE: [[ARCHITECT]]
SCOPE: [[internal/bot/handler.go]]
SPEC: [[specs/telegram-routing.feature]]
```

---

## Eval Loop

Every session outcome is logged to `~/eval/sessions.db` (SQLite).

```sql
-- Schema reference
sessions(id, model, mode, task, corrections, outcome, ctx_tokens, ts)
-- outcome: success | partial | failure | looped
```

- **Trap/canary evals:** The eval suite includes underspecified prompts (entity with no constraint) to test whether looping behavior has regressed after config changes.
- **Session convergence metric:** correction count + outcome per task type. Used to tune skill files and AGENTS.md thresholds.

---

## Security Rules (READ ONLY — Git Versioned)

- **This file is a security artifact.** Never modify autonomously. Propose changes via PR only.
- **No autonomous `git push`.** Stage changes only. Human runs the final push.
- **No overwriting `.feature` files** without a spec review turn first.
- **No file operations outside the declared SCOPE** without explicit human approval.
- **Skill files are authoritative.** Do not bypass or contradict them.
- **Vibe Diff hook** runs on every commit: DeepSeek translates the staged diff to prose. Review the summary before confirming the commit.

---

## Skill Reference

Load before any Go code is written or reviewed. Each skill file: `.opencode/skills/[[skill-name]]/SKILL.md`

| Skill | Purpose |
|-------|---------|
| `golang-code-style` | Formatting, linting, idioms |
| `golang-naming` | Naming conventions |
| `golang-error-handling` | Error wrapping, sentinels, typed errors |
| `golang-concurrency` | Goroutines, channels, sync primitives |
| `golang-context` | context.Context propagation patterns |
| `golang-structs-interfaces` | Composition, interface design |
| `golang-project-layout` | Standard Go project structure |

Refresh from disk at session start. Do not rely on cached versions.

---

## Context Hygiene

- A session ends when the spec is satisfied — not when the agent feels "done"
- If context is tight and the task has >1 logical unit → split into separate sessions
- **Honcho memory layer:** cross-session continuity (configured, DeepSeek V4 Flash routing). AGENTS.md is the static anchor; Honcho handles dynamic context.
- Plan in, not plan out: carry the spec forward, not accumulated reasoning.