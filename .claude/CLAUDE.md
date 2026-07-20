# CLAUDE.md

## What this project is

Cross-platform arbitrage system for binary prediction markets on the **Kalshi ↔ Polymarket US** pair (US sports moneylines first). Structured as a staged POC: we are proving the edge exists before building trading systems. **Current phase: Phase 0 (measurement)** — record both order books on matched markets, detect net-of-fee spreads at executable depth, paper-trade offline, and produce a go/no-go report. No live orders are placed in this phase.

## Read these before non-trivial work

- `docs/context/MARKET_RESEARCH.md` — source of truth for fees, API limits, venue facts, risks. If code contradicts it, flag the conflict; don't silently pick one.
- `docs/design/DESIGN.md` — component specs, data schemas, phase gates, validation hypotheses (H1–H6).
- `docs/design/IMPLEMENTATION_PLAN.md` — current milestone, task list, acceptance criteria. Work against the current milestone only.

## Hard invariants (violations fail review)

1. **No execution-path code in phase 0.** Nothing that places, cancels, or amends orders on any venue. Do not scaffold it "for later."
2. **All money math uses `Decimal`.** Floats never touch prices, fees, or edges. Fee calculations include Kalshi's per-contract round-up and Polymarket US's fee cap.
3. **Fees, rate limits, thresholds, and pair whitelists are config/data, never code constants.** Both venues changed fee schedules multiple times in 2026; the system fetches live params and versions them.
4. **No market pair is used without human approval** in the pair registry (resolution-rule hashes; hash change auto-suspends). Never bypass or auto-approve.
5. **The fee-engine golden tests are permanent invariants.** If a change breaks them, the change is wrong or the fee schedule changed — investigate, don't adjust the test to pass.
6. **Rate-limit budgets are hard:** Polymarket US REST ≤ 10 req/min of the 60/min cap (WS for all price data, ~10 instruments/connection); Kalshi client-side token bucket per docs (429s have no Retry-After).
7. **Secrets:** credentials come from `.env` / key files only; never read, print, or commit them. `data/` and `*.pem` are off-limits and gitignored. Key files (Kalshi RSA, Polymarket US Ed25519) live outside the repo tree entirely (e.g. `~/.keys/`) and are referenced from `.env` by absolute path — this repo directory never contains key material, so there's nothing inside it for a tool to leak. `.claude/settings.json` permission denials on `.env`/`*.pem`/`data/` are a second layer, not the primary control (they don't stop `Bash` from reading file contents another way).
8. **Validation thresholds (DESIGN §9) are pre-registered.** Never modify H1–H6 thresholds in the same change that computes results.

## Conventions

- Python 3.12, async throughout; `uv` for everything: `uv run pytest`, `uv run ruff check .`, `uv run ruff format .`, `uv run mypy src/arb/fees src/arb/spread` (strict on those two packages only).
- src-layout: code in `src/arb/`, entrypoints in `scripts/` as thin wrappers, tests in `tests/`.
- Market data → parquet in `data/` (hourly partitions); phase-2 ledger → SQLite. No other storage.
- Empirical findings (WS connection limits, latencies, fee discrepancies) get written to `docs/runbooks/` or `docs/decisions/` in the same PR that discovers them.
- Prefer small PRs scoped to one milestone task; each milestone's acceptance criteria in IMPLEMENTATION_PLAN.md define done.

## Reporting failures

When an agent stops instead of finishing (failing acceptance criteria, a spec conflict, an unreproducible golden case, an ambiguous scope boundary), report in this form so the orchestrator can act on it without follow-up questions:
- **Attempted:** what you tried.
- **Failure:** the exact command and error/output (not a paraphrase).
- **Hypothesis:** one line on the likely cause, if you have one.

Iteration budget: an agent may fix a failure it can directly diagnose within its current dispatch (one verify-fail-fix-verify cycle), but must not loop beyond that — report using the format above instead. The orchestrator (`plan-and-ship`) owns the only iteration counter across re-dispatches; it does not compound with any agent-internal retry.

## Things that look like good ideas but aren't

- Adding Robinhood/Coinbase as venues (they route to Kalshi's book — same liquidity, MR §2.1).
- Using Polymarket *Global* APIs or V1 SDK examples from GitHub (US users are geoblocked from Global; V1 is dead post-April 2026 cutover — build against Polymarket US / V2 docs only).
- Polling REST for prices (blows the PM-US 60/min cap; WS-first by design).
- Top-of-book-only spread math (edge must be VWAP-at-depth with fees; see DESIGN §4.5).
- Building the execution engine, risk manager, or rebalancer now (phase 2; gated on G0/G1 results).
