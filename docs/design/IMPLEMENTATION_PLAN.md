# Implementation Plan

**Destination:** `docs/design/IMPLEMENTATION_PLAN.md`
**Companion to:** `docs/design/DESIGN.md` (component specs), `docs/context/MARKET_RESEARCH.md` (facts)
**Working model:** agents implement, owner reviews. Every milestone has acceptance criteria checkable by tests or by running a script — review happens against those criteria, not vibes.

---

## Phase 0 — Measurement POC

### M0: Repo & accounts (owner tasks, blocking)
- [ ] Complete Polymarket US KYC (iOS app) and generate API keys via developer portal; store Ed25519 key outside repo. **Do this first — it is the long pole and a hard blocker if it fails.**
- [ ] Create Kalshi account + API key + RSA keypair; verify sandbox (`demo-api.kalshi.com`) login.
- [ ] `uv init` src-layout; commit scaffold: `pyproject.toml`, `.gitignore` (`.env`, `data/`, `*.pem`, `.venv/`), `.env.example`, pre-commit (`ruff`, `ruff format`, `gitleaks`).
- [ ] `.claude/settings.json`: deny read/write on `.env`, `*.pem`, `data/`; allow `uv run pytest`, `uv run ruff check`.
- [ ] Provision the us-east-1 VPS needed from M4 onward: create instance, harden access, install Python 3.12 + `uv`, create `~/.keys/` (key files live there, outside any repo checkout); document access + rebuild steps in `docs/runbooks/vps.md`.

**Accept:** fresh clone → `uv sync && uv run pytest` passes (empty suite ok); `gitleaks` hook blocks a planted fake key.

### M1: Config + fee engine
- [ ] **Fee-schedule reconciliation first (api-researcher, blocks the rest of M1):** verify both venues' current schedules against MR §3 — Polymarket US published an updated schedule effective July 1, 2026 (after MR was compiled), and Kalshi's maker fee is characterized both as "0.25% per contract" (MR §3.1) and as "25% of the taker fee" in current sources. Record resolved schedules, per-venue rounding directions, and fully-worked golden examples (all inputs pinned) in `docs/decisions/0003-fee-schedule-reconciliation.md`. Golden tests derive from that document.
- [ ] `config.py` (pydantic-settings): venue credentials from env; fee params, rate limits, pair whitelist path as data files.
- [ ] Fee engine: `kalshi_taker_fee`, `kalshi_maker_fee` (per-contract round-up), `pmus_taker_fee` (cap), `pmus_maker_rebate` — all `Decimal`, mypy-strict; params resolved per market/series via `fee_class` (DESIGN §4.3).
- [ ] Golden tests from `docs/decisions/0003-fee-schedule-reconciliation.md` with full inputs pinned (prices, sizes, params, per-venue rounding direction — Kalshi rounds up per fill, PM-US nearest cent). Must include: a small-spread case net-negative at 100 contracts (the shape of MR §3.4's literature anecdote — do NOT encode its $3.15 figure, which comes from the Kalshi↔PM-Global pair) and Kalshi maker round-up on low-priced small fills.
- [ ] Live fee-param fetchers for both venues + `fee_params_history` writer; discrepancy between fetched and documented values logged as WARNING and recorded in `docs/decisions/`.

**Accept:** golden tests green; `scripts/show_fees.py` prints both venues' live params and the breakeven spread curve by price.

### M2: Venue clients (read-only)
- [ ] Kalshi client: RSA-PSS auth, `list_sports_markets` (status=open, cursor pagination), WS `orderbook_delta` with local book + sequence-gap resubscribe, client-side token bucket.
- [ ] Polymarket US client: Ed25519 auth + clock-drift check, market discovery within 10 req/min budget, WS pool (~10 instruments/conn).
- [ ] **Day-1 empirical probes (record results in `docs/runbooks/venues.md`):** PM-US max concurrent WS connections; whether a WS "instrument" is a market or a token (YES/NO on separate books would halve per-connection coverage — DESIGN §7); observed message rates; order-ack path NOT tested (phase 2).
- [ ] Contract tests: parsers against committed fixtures captured from real responses.

**Accept:** `scripts/probe_venues.py` streams live books for one matched game on both venues for 10 minutes with zero unhandled disconnects (the game is an explicitly designated probe pair under CLAUDE.md invariant 4's discovery carve-out — the registry doesn't exist until M3, and probe output stays out of `data/books/`); fixtures + contract tests green.

### M3: Matcher + pair registry
- [ ] Candidate generation: league + team-alias normalization + start-time tolerance; scored output.
- [ ] `scripts/refresh_matches.py --review`: side-by-side resolution rule texts → approve/reject; approval writes whitelist entry with rule-text hashes; hash change auto-suspends pair.
- [ ] Owner approves ≥ 10 pairs in current-season leagues.

**Accept:** matcher unit tests green (incl. near-miss cases that must NOT auto-match); ≥ 10 approved pairs in registry; suspending-on-rule-change covered by test.

### M4: Recorder
- [ ] Collector: both WS streams → normalized book state; parquet writer (top-5 change + 1s heartbeat; hourly partitions).
- [ ] Health monitor: message rates, reconnects, per-market staleness; thresholds + restart procedure in `docs/runbooks/recorder.md`; systemd unit file.
- [ ] 48-hour unattended soak on VPS (us-east-1).

**Accept:** 48h soak with ≥ 95% per-market coverage (heartbeat gaps ≤ 5%); restart mid-game loses < 30s of data; runbook written.

### M5: Spread detector + offline paper trader + report
- [ ] Detector: both directions, VWAP-at-depth, fee-inclusive edge(q), q*, opportunity lifetime tracking; bundle-arb detection on same updates.
- [ ] Property tests (`hypothesis`): recomputation consistency; VWAP(q) ≥ best ask; fee monotonicity.
- [ ] Offline paper trader over recorded streams with latency sweep (100ms–2s).
- [ ] `scripts/report.py`: computes H1–H6 (DESIGN §9.1) end-to-end from logs; golden replay fixture with pinned expected output.

**Accept:** replay fixture green; report runs on soak data and renders all six hypotheses with pass/fail against thresholds.

### M6: The measurement run
- [ ] Two-week continuous run, ≥ 10 pairs, uptime ≥ 95%.
- [ ] Mid-run weekly checkpoint: report on week 1 (look, don't change thresholds).
- [ ] Final report → `docs/design/PHASE0_RESULTS.md` with G0 decision per DESIGN §9.3 (validated / weak-signal / pivot / invalidated) and, if pivot, the politics/econ re-run plan.

**Accept:** PHASE0_RESULTS.md committed with an explicit decision and the data to back it.

---

## Phase 1 — Live paper execution (build only if G0 = validated or pivot-validated)

- [ ] Real-time execution simulator on the live stream: detection → simulated two-leg submission → fill adjudication against subsequent book states; simulated order-state machine (FSM per DESIGN §4.7, no venue writes).
- [ ] Failure-mode telemetry: one-leg rate, edge retention vs. theoretical, latency distribution detection→decision.
- [ ] ≥ 2-week run; G1 evaluation (retain ≥ 60% edge, one-leg ≤ 20%).
- [ ] Pre-live checklist build-out: Kalshi sandbox full order lifecycle test; kill-switch mechanism; crash-recovery reconciliation (`open_orders` + `balances` on startup).

**Accept:** G1 report committed; pre-live checklist items each have a passing drill logged.

## Phase 2 — Live pilot (build only if G1 passes)

- [ ] Execution engine with unwind FSM; policy params initialized from phase 1 failure data.
- [ ] Risk manager: per-pair/per-venue/daily-loss/unhedged limits; kill switch; SQLite ledger; daily ledger-vs-venue reconciliation (mismatch = halt).
- [ ] Polymarket US order-path validation with minimum-size live orders (no sandbox exists); confirm order-endpoint rate limits empirically.
- [ ] Fund accounts $2–5K split; 4-week pilot; G2 evaluation (realized ≥ 70% of paper-predicted; zero uncontrolled failures).

## Phase 3 — Scale (post-G2; plan, don't build)

Maker-leg strategy on the Polymarket leg; Kalshi fee-tier volume planning; inventory rebalancer with drift-triggered ACH initiation; MIAXdx venue integration when live; category expansion. Each gets its own design addendum before implementation.

---

## Standing rules for all milestones

1. Money math is `Decimal`; a float in a price/fee path fails review.
2. Fees, rate limits, thresholds, and policies live in config/data files — never constants in code.
3. No execution-path code (venue writes) exists in the repo during phase 0. Sole later exception: the Kalshi *sandbox* order-lifecycle harness built as the final phase-1 pre-live checklist item (sandbox credentials, `demo-api.kalshi.com` only). Production venue-write code arrives only in phase 2; the Polymarket US minimum-size live-order validation is the first phase-2 step. Agents proposing venue-write code outside those windows are out of scope by definition.
4. Every milestone ends with its acceptance script/tests runnable by `uv run` — review is against those.
5. Anything empirical discovered (WS limits, latencies, fee discrepancies) is written to `docs/runbooks/` or `docs/decisions/` in the same PR that discovered it.
