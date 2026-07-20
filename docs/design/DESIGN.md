# System Design — Prediction Market Cross-Platform Arbitrage

**Destination:** `docs/design/DESIGN.md`
**Grounding:** `docs/context/MARKET_RESEARCH.md` (cited below as MR §n)
**Scope:** Full system across all phases — measurement POC through live trading and scale — including the testing strategy that determines whether the POC validates or invalidates the project.

---

## 1. Thesis & Strategy

The same binary event trades on multiple order-book venues with fragmented liquidity. When `ask_YES(A) + ask_NO(B) + fees < $1.00`, buying both legs locks in profit regardless of outcome. Persistent post-fee spreads of 1.4–4.9% are documented academically between Kalshi and Polymarket Global (MR §6); whether they exist at executable depth on our legal pair — **Kalshi ↔ Polymarket US** — is unmeasured and is what the POC determines.

Target instruments: sports moneylines (binary, automated resolution, ~30–90 min settlement, same-day capital recycling). Adjacent strategy included for free: intra-venue bundle arb (YES+NO < $1 on one book).

## 2. Phase Model

The project is a staged POC. Each phase ends with an explicit gate; later phases are built **only** if the prior gate passes.

| Phase | Name | Capital at risk | Builds | Gate |
|---|---|---|---|---|
| 0 | Measurement | $0 | Venue clients, matcher, fee engine, recorder, spread detector, offline paper trader, analysis | G0: recorded spreads clear profitability hurdle (§9) |
| 1 | Live paper execution | $0 | Real-time execution simulator against live books incl. latency; order-state machine (simulated) | G1: simulated fills retain ≥60% of G0's theoretical edge |
| 2 | Live pilot | $2–5K | Execution engine, unwind policies, risk manager, position ledger, alerting | G2: 4+ weeks live, realized edge ≥70% of paper-predicted, no uncontrolled failure modes |
| 3 | Scale & extend | Sized by G2 results | Inventory rebalancer, Kalshi fee-tier optimization, maker-leg strategies, third venue (MIAXdx when live), category expansion | Ongoing review |

A **pivot path** exists at G0/G1: if sports fails but politics/econ pairs show edge (where academic evidence is strongest, MR §6), re-run the gate on those categories on the same recorder/detector/fee infrastructure. The matcher, however, is *not* category-agnostic (it keys on league/team/start-time): a pivot needs a title/rules-text matcher and a fresh resolution-risk review — politics is where contested resolutions concentrate — before re-running.

## 3. Architecture

```
                ┌────────────────────────────────────────────────┐
                │                   config (pydantic)            │
                │   fee params · rate limits · pairs · secrets   │
                └────────────────────────────────────────────────┘
┌─────────────┐   WS/REST   ┌──────────────┐        ┌───────────────┐
│ Kalshi       │───────────▶│              │ books  │ Storage        │
│ client       │            │  Collector    │──────▶│ (parquet,      │
├─────────────┤            │  (normalized  │        │  hourly        │
│ Polymarket   │───────────▶│  book state)  │        │  partitions)   │
│ US client    │   WS/REST  └──────┬───────┘        └───────┬───────┘
└─────────────┘                    │ live book state         │ logs
                                   ▼                         ▼
                            ┌──────────────┐        ┌───────────────┐
                            │ Spread        │ spread │ Analysis /     │
                            │ detector      │ events │ gate report    │
                            └──────┬───────┘        └───────────────┘
                          phase 0: │ log only
                          phase 1: │ → execution simulator
                          phase 2: ▼
                            ┌──────────────┐   ┌───────────────────┐
                            │ Execution     │──▶│ Risk manager       │
                            │ engine        │   │ (limits, kill      │
                            │ (two-leg,     │◀──│  switch, ledger,   │
                            │  unwind FSM)  │   │  inventory drift)  │
                            └──────────────┘   └───────────────────┘
```

Single async Python process per role (recorder, detector, executor) communicating through storage and, in phase 2, an in-process event bus — no microservices, no message brokers. Deployment: one small VPS in AWS us-east-1 (50–150ms venue RTT is sufficient; this is not HFT, MR §4.1/§7.1).

## 4. Component Designs

### 4.1 Venue clients (`src/arb/venues/`)

Common protocol; per-venue implementations:

```python
class VenueClient(Protocol):
    venue: str
    async def list_sports_markets(self) -> list[MarketMeta]
    async def subscribe_books(self, ids: list[str]) -> AsyncIterator[BookUpdate]
    async def fee_params(self, market: MarketMeta | None = None) -> FeeParams
        # fetched live, never hard-coded; market-level resolution because
        # Kalshi fee classes vary by series/event (MR §3.1)
```

Venue **write** methods (place/cancel/amend, open orders, balances) are deliberately absent from this protocol: a separate `TradingVenueClient` extension protocol is specified when it is first permitted (Kalshi sandbox-only harness at the end of phase 1, production in phase 2 — see IMPLEMENTATION_PLAN standing rule 3). No venue-write signatures — not even protocol stubs — exist in the repo before then, keeping this section consistent with CLAUDE.md invariant 1 and the security-review gate.

`MarketMeta`: venue, market_id, title, league, teams, event start, close time, tick_size, min_order_size, accepting_orders, seconds_delay, raw resolution rule text, fee_class (which venue fee schedule applies — Kalshi special events and maker-fee series carry different rates, MR §3.1).

**Kalshi:** RSA-PSS signing (path without query params; ms timestamps); client-side token bucket mirroring published tier budgets (429s have no Retry-After, MR §4.1); one WS connection, multiplexed `orderbook_delta` subscriptions; local book with sequence-gap detection → resubscribe on gap; sandbox (`demo-api.kalshi.com`) used for all order-path tests.

**Polymarket US:** Ed25519 auth with startup clock-drift check; REST budgeted ≤10 req/min against the 60/min cap — discovery/metadata only, all prices via WS; WS pool at ~10 instruments/connection with max-connections discovered empirically day 1 (MR §4.3). No sandbox exists → phase 2 order-path validation uses minimum-size live orders.

### 4.2 Market matcher (`src/arb/matching/`)

Stage 1 (automated): candidate pairs scored on league + normalized team names (alias table) + event start time tolerance. Stage 2 (human): CLI review showing both resolution rule texts side by side; approval writes the pair to a whitelist with a **hash of both rule texts**; any rule-text change auto-suspends the pair pending re-review. No pair is paper- or live-traded without approval — primary control for resolution risk (MR §7.2).

### 4.3 Fee engine (`src/arb/fees/`)

Pure functions per venue and side (taker/maker), including Kalshi per-contract round-up and the Polymarket US fee cap (MR §3). Rounding direction is a per-venue fact pinned by golden cases (Kalshi rounds up per fill; Polymarket US documents nearest-cent rounding). Params are resolved per market/series via `fee_class` — venue-global params are insufficient because Kalshi rates differ by series (MR §3.1). Params fetched live at startup + daily refresh; every change is versioned into `fee_params_history` so historical analysis uses fees in force at record time. Golden-case unit tests from MR §3 worked examples are permanent invariants.

### 4.4 Recorder (`src/arb/recorder/`)

Normalizes both WS streams into book state; appends to parquet on (a) any top-5-level change on a matched pair and (b) 1s heartbeat (gaps distinguishable from stasis); rows are buffered and flushed every `flush_interval_s` (config, default 15 — must stay ≤ 30 to meet M4's restart-loss criterion) via write-temp-then-rename. Hourly partitions: `data/books/date=/hour=/venue=/`. Health monitor: message rates, reconnect counts, per-market staleness; thresholds documented in `docs/runbooks/recorder.md`.

### 4.5 Spread detector (`src/arb/spread/`)

On every matched-pair book update, both directions:

```
edge(q) = 1.00 − vwap_ask_A(q) − vwap_ask_B(q) − fee_A(q) − fee_B(q)
q*      = argmax_q  edge(q) · q   s.t. edge(q) > 0
```

VWAP walks the book — depth-aware, never top-of-book only. **Staleness guard:** an edge is emitted only when both books are younger than `max_book_age_ms` (config, default 1500); a stale side suppresses detection rather than inflating it, since cross-venue timestamps are local receive times and a silently gapped feed would otherwise manufacture phantom spreads. Every spread event records both books' ages at detection so phantom-spread sensitivity is quantifiable in analysis. Each opportunity is tracked from first appearance to decay; **duration** is a first-class field. Bundle arb (single-venue YES+NO) computed on the same updates.

### 4.6 Paper trader — phase 0 (offline) and phase 1 (live simulator)

Phase 0: replays recorded streams, applies a configurable per-leg latency penalty (default 500ms; swept 100ms–2s) **plus each market's venue-imposed order-acceptance delay** (`seconds_delay` on Polymarket US; any Kalshi in-game delay) from market metadata — venue-imposed delay dominates network latency in-game — and asks whether both legs would have filled at q* given the books at detection + total latency. Logs fills, partials, one-leg failures, reported split pre-game vs in-game.
Phase 1: identical fill logic but running against the **live** stream in real time, driving a simulated order-state machine — validates that detection→decision→(would-be)submission fits inside real opportunity lifetimes, which offline replay can overstate.

### 4.7 Execution engine — phase 2 (`src/arb/execution/`)

Two-leg concurrent submission with an explicit finite-state machine per attempt:

```
DETECTED → SUBMITTING(both legs) → {BOTH_FILLED | ONE_LEG(a/b) | NONE}
ONE_LEG → policy: CHASE(re-price up to max_chase_ticks)
                  | HOLD(max_hold_seconds, only if |Δfair| small)
                  | UNWIND(market-out at loss ≤ max_unwind_cents)
```

Policy parameters are config; defaults set from phase 1 failure-mode data, not guesses. Orders are idempotent (client order IDs); crash recovery reconciles `open_orders()` + `balances()` on both venues before resuming. Fill sizing capped at min(q*, per-pair limit, per-venue exposure limit).

### 4.8 Risk manager — phase 2 (`src/arb/risk/`)

Hard limits enforced before any submission: per-pair max contracts, per-venue max exposure, daily max loss (realized + marked), max open unhedged contracts (from one-leg events), and a global kill switch (file flag + signal handler) that cancels all resting orders and halts. Inventory drift tracked continuously: projected days-to-depletion per venue at current fill skew triggers rebalancing alerts (ACH is 1–3 business days, MR §8 — rebalancing must be initiated *before* depletion). Position ledger in SQLite (transactional; parquet stays for market data).

### 4.9 Analysis & reporting (`src/arb/analysis/`)

Generates the gate reports (§9) from spread/fill logs; phase 2 adds a daily P&L report reconciling ledger vs. venue balances (any mismatch is a halt condition).

## 5. Data Schemas

**books**: ts_ms, venue, market_id, pair_id, side, level(0–4), price, size, is_heartbeat
**spreads**: first_seen_ms, last_seen_ms, pair_id, direction, max_edge_cents, q_star, gross_cents, fees_cents, duration_ms, book_ts_a, book_ts_b, book_age_a_ms, book_age_b_ms — the exact book timestamps used, so any emitted edge is auditable by joining back to **books**
**paper_fills / live_fills**: detected_ms, pair_id, direction, latency_ms, leg_a_result, leg_b_result, fill_q, realized_edge_cents, failure_mode, fsm_path
**fee_params_history**: effective_ts, venue, params_json
**pairs**: pair_id, kalshi_ticker, pmus_id, league, approved_ts, rule_hash_k, rule_hash_p, status
**ledger (SQLite, phase 2)**: fills, positions, transfers, balance snapshots

## 6. Tech Stack & Rationale

Chosen for optimality given: agent-written code with human review, async IO-bound workload, heavy tabular analysis, solo maintainer.

| Choice | Rationale |
|---|---|
| Python 3.12, fully async (`asyncio`) | Best ecosystem coverage for both venues (official/maintained SDK surface is Python-first); IO-bound workload where async is sufficient and threads/Go-style concurrency adds no edge at our latency budget; agents produce the most reliable code in Python; analysis layer shares the language. |
| `uv` + src-layout + `pyproject.toml` | Fast, reproducible, standard. |
| `httpx` + `websockets` | Async-native HTTP/WS without framework weight. |
| `pydantic` v2 (+ pydantic-settings) | Typed config and message models; validation at the venue boundary where malformed data enters. |
| `Decimal` for all money/prices | Cent-level arithmetic with per-contract rounding must be exact; floats are a correctness bug here. |
| `polars` + parquet for market data | Columnar append + fast scans over multi-GB book logs, zero DB ops. |
| SQLite for the phase-2 ledger | Transactional integrity for positions/P&L; still zero ops. |
| `pytest` + `ruff` + `mypy --strict` on `src/arb/fees` and `src/arb/spread` | The money-math modules are fully typed and exhaustively tested; strictness relaxed elsewhere to keep agent velocity. |
| Single VPS (AWS us-east-1), `systemd` units, no containers/k8s | Matches latency needs (MR §4.1); minimal ops for a solo project. Containerize only if/when a second deploy target exists. |

Explicitly rejected: TypeScript (weaker analysis ecosystem; two-language repo if analysis stays in Python), Rust/Go (latency budget doesn't require it; slower agent iteration), microservices/brokers (unjustified at this scale), Postgres (ops burden with no phase 0–2 payoff).

## 7. Rate-Limit Budgets

Worst case 30 live pairs — Kalshi: 1 WS conn / ~30 subs; REST ~1 req/min → within Basic tier (20 reads/s, 10 writes/s). Polymarket US: if a WS "instrument" is a market, 30 pairs fit in 3 conns × 10; if it is a token (YES/NO books separate, as on Polymarket Global), the same coverage needs 6 conns — against a max-connections cap that is unknown. Instrument granularity and the connection cap are day-1 M2 probes; the measurement run's pair count degrades to fit what the probe finds. REST <10/min vs 60/min cap. Phase 2 adds order writes: Kalshi Basic write bucket (≈10 orders/s equivalent) far exceeds our per-event needs; Polymarket US order-endpoint limits must be confirmed empirically before G2 (open item, MR §4.5).

## 8. Risk Register & Controls (all phases)

| Risk (MR §7–8) | Control |
|---|---|
| Resolution mismatch between venues | Human-approved pairs only; rule-text hashing with auto-suspend; sports moneylines only; per-league postponement policy review |
| One-leg fills | Execution FSM with pre-committed chase/hold/unwind policy; parameters from phase 1 data; unhedged-contract hard limit |
| Fee/API changes (4+ fee changes in 2026; V2 hard cutover) | Fees and limits as config/data; live param fetch + versioning; param-change alerts; pinned SDK versions with upgrade runbook |
| Inventory drift across venues | Continuous drift projection; rebalance alerts sized to ACH latency; capital split monitored as a first-class metric |
| Regulatory (state litigation, possible sports delistings) | Position sizing survivable under forced unwind; venue/state eligibility documented; halt-and-unwind runbook |
| Operational (crash, disconnect, clock drift) | Idempotent orders; startup reconciliation; NTP check; kill switch; ledger-vs-venue balance reconciliation as halt condition |
| Model bugs in money math | Golden-case tests as permanent invariants; `Decimal` everywhere; mypy-strict on fee/spread modules |

## 9. POC Testing & Validation Strategy

This section defines how we decide the project is validated or invalidated. Structured as falsifiable hypotheses with pre-registered thresholds — set **before** looking at results, to prevent motivated goalpost-moving.

### 9.1 Hypotheses

- **H1 (existence):** Net-of-fee spreads (edge > 0 after both venues' fees) occur on matched sports markets at ≥ 20 opportunities/week across all approved pairs (threshold pre-registered 2026-07-19). 
- **H2 (depth):** Median executable size q* at positive edge is ≥ 100 contracts (below this, fee rounding and minimums eat the edge, MR §8).
- **H3 (persistence):** ≥ 40% of opportunities survive longer than 1.5s (our detection + two-leg submission budget at VPS latency).
- **H4 (capturability):** Simulated two-leg execution at 500ms/leg latency retains ≥ 60% of theoretical edge (one-leg failure ≤ 20% of attempts).
- **H5 (economics):** Weekly executable paper profit per $1,000/venue deployed, after a 20% real-world slippage haircut, annualizes above the pre-registered hurdle rate of 10% (set 2026-07-19; meaningfully above T-bills for the operational risk and time).
- **H6 (sustainability):** Edge does not decay to zero within the measurement window (week-2 opportunity rate ≥ 50% of week-1's — a crude but honest competition check).

### 9.2 Measurement protocol

Two-week continuous run, ≥ 10 approved pairs across whatever leagues are in season, recorder uptime ≥ 95% (downtime logged and excluded), all six hypotheses computed by `scripts/report.py` from logs — no manual spreadsheet steps, so the run is reproducible and extendable. Opportunity rates are additionally reported normalized per covered game-hour: excluded downtime correlates with high-activity periods (crashes under load), so the raw weekly count and the coverage-weighted rate must both appear. Seasonality caveat: an August window is MLB/tennis-heavy and thin on Polymarket US; a marginal G0 in that window warrants a re-run after NFL season starts before a final verdict.

### 9.3 Decision rules (gate G0 → G1)

- **VALIDATED (build phase 1):** H1–H6 all pass.
- **WEAK SIGNAL (extend, don't build):** H1–H3 pass but H4/H5 marginal, **or** H1–H5 pass but H6 fails (edge decaying) → extend run 2 weeks and/or sweep latency assumptions; if still marginal, treat as pivot.
- **PIVOT:** Sports fails but the same infra run on politics/econ pairs passes H1–H6 → re-gate on that category.
- **INVALIDATED (stop building trading systems):** H1 or H2 fail on both categories. Deliverable becomes the dataset + writeup (`docs/design/PHASE0_RESULTS.md`) — itself a novel artifact, since Kalshi↔Polymarket US spread history exists nowhere publicly (MR §9).

### 9.4 Software testing strategy (all phases)

**Unit:** fee engine golden cases derived from the reconciled live fee schedules (`docs/decisions/0003-fee-schedule-reconciliation.md`, produced before M1 coding), each pinned with full inputs — prices, sizes, fee params, per-venue rounding direction (Kalshi rounds up per fill; PM-US nearest cent). Must include a small-spread case that is net-negative at 100 contracts and Kalshi maker round-up on low-priced small fills. (The MR §3.4 "$3.15" example is a Kalshi↔PM-Global literature anecdote, not derivable from our pair's schedules — never a golden expectation.) Matcher normalization cases; detector edge/q* math against hand-computed books.
**Property-based** (`hypothesis`): detector never emits edge > 0 whose recomputation from the same book is ≤ 0; fee functions monotone in size; VWAP(q) ≥ best ask.
**Replay/integration:** golden recorded-stream fixtures (committed, small) through collector→detector→paper trader with expected outputs pinned.
**Contract tests:** venue client parsers against captured real API responses (fixtures refreshed when APIs change; parse failures alert rather than silently drop).
**Pre-live checklist (gating G1 → G2):** built at the end of phase 1 — Kalshi *sandbox* full order lifecycle (the sole venue-write code permitted before phase 2; sandbox credentials, `demo-api.kalshi.com` only); kill-switch drill; crash-recovery drill (kill -9 mid-submission → reconcile cleanly); ledger reconciliation green for 3 consecutive paper days. Polymarket US minimum-size live order lifecycle (no sandbox exists) is the first phase-2 step, before any sized trading.

### 9.5 Gates G1 and G2 (defined now, evaluated later)

**G1:** live simulator over ≥ 2 weeks retains ≥ 60% of G0 edge with one-leg rate ≤ 20%; all pre-live checklist items green → fund accounts.
**G2:** ≥ 4 weeks live at pilot capital; realized edge ≥ 70% of paper-predicted; zero uncontrolled failure modes (every one-leg event resolved by policy, never by improvisation); drawdown within daily-loss limits → scale per §2 phase 3.

## 10. Phase 3 Directions (post-G2, design sketches only)

Maker-leg strategies (rest the Polymarket leg for zero fee + rebate; requires adverse-selection modeling); Kalshi fee-tier volume optimization; third venue integration when the Robinhood/Susquehanna MIAXdx exchange launches (venue protocol already abstracts this); category expansion (politics/econ); automated rebalancing via fastest available rails. None of this is specified further until G2 passes.
