---
name: plan-and-ship
description: Plan and orchestrate implementation work for this repo by decomposing the current milestone into tasks, dispatching them to the right subagents (in parallel only when file-disjoint), and verifying completion against the milestone's acceptance criteria. Use this whenever the user asks to implement a milestone, "build M2", "ship the next task", "work on the plan", "continue implementation", or requests any multi-task coding work — even if they don't say "plan" or "orchestrate".
---

# Plan and Ship

Orchestration procedure for implementing milestones from `docs/design/IMPLEMENTATION_PLAN.md`. The orchestrator (you) plans, dispatches, and verifies. Subagents write code. Acceptance is decided by running the milestone's acceptance scripts/tests — never by self-assessment.

## Procedure

### 1. Load state
Read, in order: `CLAUDE.md`, `docs/design/IMPLEMENTATION_PLAN.md` (find the current milestone — first one with unchecked tasks), and the relevant sections of `docs/design/DESIGN.md` for the components involved. If a task's spec conflicts with `docs/context/MARKET_RESEARCH.md`, stop and flag it to the user — do not silently pick one.

### 2. Plan
Decompose the milestone's unchecked tasks into work units. For each unit, record: target files, which subagent, which model, dependencies on other units, and the acceptance signal (test name or script) that proves it done. Present the plan briefly to the user before dispatching if the milestone spans more than ~3 work units or any task is ambiguous.

### 3. Route to agents

| Work type | Agent | Model |
|---|---|---|
| Anything in `src/arb/fees/` or `src/arb/spread/` (money math, edge/VWAP, fee formulas) | money-math | Opus |
| Concurrency, WS reconnect/sequence-gap logic, auth signing, order-state machines | implementer | Opus (escalate) |
| Standard modules, parsers, config, scripts, boilerplate | implementer | Sonnet |
| Tests for any component | test-writer | Sonnet (money-math tests: Opus) |
| Verifying live API behavior, capturing response fixtures, checking docs | api-researcher | Sonnet |
| Reviewing completed diffs | code-reviewer | Opus |

### 4. Parallelism rules (hard)
- Dispatch units in parallel **only when their file sets are disjoint** (e.g., `venues/kalshi.py` and `venues/polymarket_us.py` in parallel is fine; two units touching `fees/engine.py` is not).
- Serialize any unit that edits a file another in-flight unit reads (interface churn causes silent breakage).
- test-writer runs AFTER the implementer's unit completes, and writes tests from the DESIGN spec — pass it the DESIGN section, not the implementation diff.
- Maximum 3 concurrent subagents; beyond that, review quality drops faster than throughput rises.

### 5. Verify (non-negotiable)
For every completed unit and at milestone end:
1. `uv run pytest` — full suite, not just new tests.
2. `uv run ruff check . && uv run mypy src/arb/fees src/arb/spread`
3. Run the milestone's acceptance script(s) named in IMPLEMENTATION_PLAN.md.
4. Dispatch code-reviewer on the combined diff; address its findings.
5. Run the `security-review` skill on the diff before declaring the milestone shippable.

A unit is DONE only when its acceptance signal passes. If it fails, iterate with the same agent (pass the failure output); after 2 failed iterations, escalate model or surface to the user.

### 6. Close out
Check off completed tasks in IMPLEMENTATION_PLAN.md, write any empirical discoveries to `docs/runbooks/` or `docs/decisions/` (same PR), and summarize for the user: what shipped, what's verified, what's next.

## Standing constraints (inherited from CLAUDE.md — enforce on every unit)
No execution-path/venue-write code in phase 0. `Decimal` for all money math. Fees/limits/thresholds as config, never constants. Golden fee tests are invariants — a failing golden test means the code (or a real-world fee change) is wrong, never the test. Never modify H1–H6 validation thresholds in implementation work.
