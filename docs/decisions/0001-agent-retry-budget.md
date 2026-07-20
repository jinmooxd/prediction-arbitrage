# 0001: Single authoritative retry/escalation budget for agent dispatch

Date: 2026-07-19

## Decision

The orchestrator (`plan-and-ship` skill) owns the only iteration counter for a work unit: it may re-dispatch a failing unit up to 2 times before escalating (to the next model tier, or to the user if already at Opus).

Within a single dispatch, a coding agent (`implementer`, `money-math`) may perform at most one verify-fail-fix-verify self-correction cycle for a failure it can directly diagnose, then must stop and report using the shared format in `CLAUDE.md` ("Reporting failures") rather than continuing to loop.

## Why

Earlier drafts had two independent, unsynchronized counters: `implementer.md` said "stop after two attempts," and `plan-and-ship` separately said "after 2 failed iterations, escalate." It was ambiguous whether these compounded (up to 4 attempts before anyone noticed) or described the same budget, and there was no defined behavior for a unit already routed to Opus that failed twice.

Removing the agent-level counter entirely would waste an orchestrator round-trip on trivial failures (e.g. a missing import) that an agent can fix immediately after seeing its own test output. Keeping two independent counters risks silent over-retrying. The chosen middle ground keeps exactly one authoritative budget (the orchestrator's) while still letting the workhorse agents fix what they can see in front of them without a round-trip.

## Where this is encoded

- `.claude/CLAUDE.md` — "Reporting failures" section (shared format + the one-cycle rule).
- `.claude/skills/plan-and-ship/SKILL.md` — step 5 (the 2-iteration orchestrator budget, escalation path).
- `.claude/agents/implementer.md`, `.claude/agents/money-math.md` — definition-of-done sections reference the one-cycle rule instead of restating a separate counter.
