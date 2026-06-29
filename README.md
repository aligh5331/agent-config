# Agent Config — 5-Agentic Coding Pipeline

This repo defines the agent prompts and workflow configuration for a **spec-driven, 5-agent Go development pipeline** running on OpenCode v1.17.9.

## The Five Agents

### 1. Architect — DeepSeek V4 Flash (temperature 0.7)
**Role:** Design, don't implement.
- Defines package boundaries, interfaces, and data flow
- Writes ADRs with alternatives considered
- Identifies risks before any code is written
- Hands off to: Librarian when design is settled
[View prompt →](agents/architect.md)

### 2. Librarian — DeepSeek V4 Flash (temperature 0.5)
**Role:** Write and verify specs.
- Translates design to BDD/Gherkin .feature files
- Validates every entity in every scenario
- Declares token budget in the spec header
- Hands off to: Builder when spec is approved
[View prompt →](agents/librarian.md)

### 3. Builder — Qwen3.6-28B local (temperature 0.2)
**Role:** Implement from spec. No spec = no code.
- Pre-flight: confirm .feature exists and approved, declare SCOPE, estimate budget
- Loads skill files, follows termination heuristics
- One .feature per session, strict scope enforcement
- Hands off to: Tester when implementation is done
[View prompt →](agents/builder.md)

### 4. Tester — Qwen3.6-28B local (temperature 0.2)
**Role:** Verify the project actually runs.
- Detects language/toolchain/deployment from the repo (project-agnostic)
- Authors E2E suite if missing, compiles before spin-up
- Spins stack, runs tests, tears down after each run
- Does not fix code — hands failures to Forensic
[View prompt →](agents/tester.md)

### 5. Forensic Specialist — Qwen3.6-28B local (temperature 0.1)
**Role:** Diagnose failures against spec, not intuition.
- Compares actual to expected spec steps
- Traces leaks, deadlocks, race conditions
- Produces structured diagnosis: Symptom → Spec → Root cause → Fix
- Temperature 0.1 — deterministic reasoning only
[View prompt →](agents/forensic.md)

## Pipeline Flow

Architect → Librarian → Builder → Tester → Forensic (loops to Builder on fail)

A feature is done when Tester reports PASS.

## Model Routing

| Agent | Model | Host |
|-------|-------|------|
| Architect | DeepSeek V4 Flash | OpenCode Zen |
| Librarian | DeepSeek V4 Flash | OpenCode Zen |
| Builder | Qwen3.6-28B | localhost:7890 |
| Tester | Qwen3.6-28B | localhost:7890 |
| Forensic | Qwen3.6-28B | localhost:7890 |

See [AGENTS.md](AGENTS.md) for full config, termination heuristics, and security rules.
