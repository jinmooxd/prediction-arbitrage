---
name: sync-docs
description: Update docs/design/DESIGN.md, docs/design/IMPLEMENTATION_PLAN.md, docs/runbooks/, docs/decisions/, and README.md after code changes so the next session inherits accurate context. Run at the end of any session that changed API surface, data schema, config/data files, milestone status, empirical findings, or a resolved decision.
---

# sync-docs

`docs/design/` and `docs/context/` are this repo's source of truth for continuing sessions — stale entries actively mislead the next agent. This is a staged POC (currently **Phase 0 — measurement**); keep the docs truthful about what is built, not aspirational. Do not let a synced doc imply execution-path code exists when it must not (CLAUDE.md invariant 1).

## What to update

1. **`docs/design/DESIGN.md`** (architecture as built):
   - §3 Architecture / component tree if the layout under `src/arb/` changed.
   - §4 Component Designs (§4.1–4.9) — bring the affected subsection in line with the code exactly: venue-client method surface (§4.1), matcher (§4.2), fee engine (§4.3), recorder (§4.4), spread detector (§4.5), paper trader (§4.6), analysis/reporting (§4.9). Do **not** add detail to §4.7/§4.8 (execution/risk) — those are phase-2 and must stay design-only.
   - §5 Data Schemas if parquet layout, `fee_params_history`, or the phase-2 SQLite ledger changed.
   - §7 Rate-Limit Budgets if measured WS/REST limits differ from the documented budget.
   - §9 (§9.1 hypotheses H1–H6, §9.3 gate rules) — **read-only here.** These are pre-registered (CLAUDE.md invariant 8); never edit thresholds in a sync. If code diverges from them, flag the conflict, don't reconcile it silently.
2. **`docs/design/IMPLEMENTATION_PLAN.md`** (milestone status):
   - Tick a milestone's `- [ ]` task boxes to `- [x]` only when its work is actually complete — "done" means the milestone's **Accept:** criteria pass (tests green, script runs), not "code written."
   - Keep task text matching reality: if scope shifted, edit the task line rather than leaving a stale description. Work only against the current milestone.
   - Keep milestone/section headers and numbering (M0–M6, phase headers) stable — sessions navigate by them.
3. **`README.md`:** quick-start commands (`uv sync`, `uv run pytest`, script names under `scripts/`) and the docs map — only if they changed.
4. **`docs/runbooks/`:** empirical findings discovered this session go here in the same PR (CLAUDE.md convention). Update `venues.md` (WS connection limits, instrument semantics, message rates, latencies), `recorder.md` (thresholds, restart procedure, systemd unit), `vps.md` (access/rebuild), or the relevant one-shot/scaffold runbook. Date-stamp new findings.
5. **`docs/decisions/`:** if a design/product decision was resolved this session (fee-schedule reconciliation, rounding direction, pair-approval mechanics, retry budget), record it as a numbered decision file (`NNNN-slug.md`) — these are the easiest thing to lose between sessions. Fee-schedule outcomes belong in `0003-fee-schedule-reconciliation.md`; the golden tests derive from it.
6. **`docs/context/MARKET_RESEARCH.md`:** only when a **new external fact was verified** (new venue API research, fee-schedule change, legal/geoblock finding — normally via the api-researcher agent). Date-stamp additions; never overwrite a prior fact silently. If code and MR conflict, surface the conflict per CLAUDE.md.
7. **`docs/REPO_LAYOUT.md`:** if directories were added/removed under `src/arb/`, `scripts/`, or `tests/`.

## Rules

- **Diff-driven:** derive every update from `git diff` / the actual session changes, not memory. Start by reading the diff.
- **Respect the invariants while syncing:** never document, imply, or scaffold execution-path code (invariant 1); never edit H1–H6 or gate thresholds (invariant 8); never edit a fee golden test to match new code (invariant 5) — a broken golden test is a signal, report it, don't paper over it in docs.
- If a decision resolved in conversation was tracked as an open item in `.claude/CLAUDE.md`, record it in DESIGN.md or `docs/decisions/` and remove it from the open list in CLAUDE.md.
- Fixed a bug or gap noted in a runbook/decision? Update that note — a stale "known issue" is as harmful as an unlisted one. For point-in-time review/decision docs, strike through resolved findings (with commit ref) rather than deleting them.
- Keep section numbering and anchors stable across all docs — sessions navigate by them.
