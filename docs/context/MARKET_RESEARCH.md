# Prediction Market Cross-Platform Arbitrage — Market Research

**Project:** Cross-platform arbitrage system for binary-outcome prediction markets (initial focus: US sports)
**Prepared:** July 19, 2026
**Status:** Research phase — precedes design/implementation planning

---

## 1. Executive Summary

The core thesis — that the same binary event trades at different prices on different platforms, allowing a locked-in profit by buying YES on one venue and NO on another for a combined cost under $1.00 — is empirically validated. A 2026 SSRN study of Kalshi–Polymarket trade data found persistent post-fee arbitrage averaging 1.4–4.9% depending on event type, present on 64–89% of trading days. Independent research documented over $40M in arbitrage profits extracted from Polymarket in a single year.

However, three findings from this research materially narrow the opportunity as originally conceived:

1. **The venue universe is smaller than it looks.** Robinhood and Coinbase both route prediction-market order flow to Kalshi. They are distribution front-ends on the *same order book*, not independent venues. The only true US arbitrage pair today is **Kalshi vs. Polymarket US**.
2. **Prices are not set by "pricing algorithms."** All relevant venues are central limit order books (CLOBs). Spreads exist because of fragmented liquidity and capital friction between venues — the friction *is* the spread. This is good (the edge is structural, not a bug that gets patched) but it means the edge is largest across the frictions we cannot legally straddle (see §8).
3. **The documented persistent arbitrage is mostly Kalshi vs. Polymarket *Global*** — a pair separated by fiat/crypto friction and regulatory segmentation. Polymarket Global geoblocks US users. Our legal pair (Kalshi vs. Polymarket US) shares USD rails and CFTC regulation, so spreads are expected to be thinner than headline academic numbers. This is the single most important unknown, and it must be measured empirically (§11).

**Bottom line:** the strategy is real, competitive, and viable primarily for automated systems. The recommended first deliverable is a measurement/paper-trading system, not an execution engine.

---

## 2. Venue Landscape & Market Structure

### 2.1 Who is actually an independent venue

| Platform | Independent order book? | Notes |
|---|---|---|
| Kalshi | **Yes** | CFTC-regulated DCM/DCO. ~$500M daily volume (2026); $23.8B volume across 97M trades in 2025. ~93% of volume is sports. Banned in Nevada. |
| Polymarket Global | Yes | Crypto-native CLOB on Polygon, pUSD collateral (post-CLOB-V2). **Geoblocked for US users.** |
| Polymarket US | **Yes** | Separate CFTC-regulated exchange (via QCEX acquisition, $112M, July 2025). Own order book, own API (`api.polymarket.us`). Waitlist removed May 2026; iOS-only signup, KYC required. Unavailable in AZ, IL, MA, MD, MI, MT, NV, OH. |
| Robinhood | No | Routes to Kalshi (partner since March 2025) + ForecastEx. Adds ~$0.01/contract commission. |
| Coinbase | No | Routes to Kalshi (launched Jan 2026). |
| Crypto.com | Partially | Smaller volume; overtaken by Polymarket US in March 2026. |
| ForecastEx | Yes (small) | Interactive Brokers' exchange. Thin liquidity; economics-heavy. Worth monitoring, not launching with. |
| Robinhood/Susquehanna exchange (MIAXdx) | Future | Joint venture acquiring a DCM+DCO; launch expected 2026. **When live, this is a third independent US book** — architecture should make adding venues cheap. |

### 2.2 Market mechanics common to both target venues

Both Kalshi and Polymarket are binary contract CLOBs: contracts trade between $0.01 and $0.99 and settle at exactly $1.00 or $0.00. Prices emerge from user supply and demand. The arbitrage condition for a matched pair of markets is:

```
ask_YES(venue A) + ask_NO(venue B) + fees(A) + fees(B) < $1.00
```

Sports binaries (moneyline win/lose, no draw) are the cleanest instrument class: automated resolution from verified score feeds, fast settlement (capital recycles same-day), and unambiguous outcomes. Soccer draws are handled in practice via 3-way markets or draw-inclusive contract definitions — the real hazard is *mismatched resolution rules* between venues, not draws per se (§7.2).

### 2.3 Research still needed (empirical)

- Count of sports events listed on **both** Kalshi and Polymarket US per week, by league (NBA, NFL, MLB, NHL, NCAA, tennis, soccer).
- Whether Polymarket US and Polymarket Global list identical sports markets with linked or independent liquidity.
- ForecastEx sports coverage (likely near zero, confirm and deprioritize).

---

## 3. Fee Structures & Breakeven Economics

Fees are the primary determinant of whether a spread is tradable. Both platforms changed fees materially in 2026; any repo hard-coding pre-2026 assumptions (including most GitHub reference bots) is wrong.

### 3.1 Kalshi

- **Taker fee:** variable, charged on expected earnings; formula ≈ `0.07 × C × P × (1−P)` — peaks at ~$1.75 per 100 contracts at 50¢, near zero at price extremes. Special events (elections, major championships) can carry different rates.
- **Maker fees:** exist on *some* markets. On certain big events, a flat **0.25% per contract** maker fee applies, **rounded up per fill** — documented cases of effective 12–50% fees on low-priced contracts filled in small quantities. Fee model must simulate per-contract rounding, not percentages.
- **Fee tiers:** volume-based (30-day trailing). Taker from 12.0 bps (tier 0) down to 2.6 bps at the highest tiers on the perps/prediction schedule; a tier reset from a low-volume month can flip last month's profitable arb to unprofitable.
- **Deposits/withdrawals:** ACH free; debit card 2% deposit fee. No settlement fee. Kalshi pays APY on idle cash balances (reduces cost of parked capital).

### 3.2 Polymarket Global (reference — not our legal venue)

- Fee Structure V2 (effective March 30, 2026): category-based taker rates — crypto 0.07, sports 0.03 (raised to **0.05 in July 2026** with the smallest rebate share), finance/politics/tech/mentions 0.04, economics/culture/weather 0.05, **geopolitics free**. Makers pay nothing and earn rebates funded from taker fees.
- Same `price × (1−price)` fee curve shape as Kalshi (peak at 50¢, symmetric).
- Additional costs: Polygon gas (~$0.001–$0.01/trade), USDC/pUSD on/off-ramp friction.

### 3.3 Polymarket US (our second leg)

- US exchange fee schedule (effective April 3, 2026): **uniform taker rate 0.05, maker rebate −0.0125, capped at $1.25 per 100 contracts at 50¢**. Some sources quote this as ~0.30% flat taker with 0.20% maker rebate — reconcile against the live fee endpoint (`GET /fee-rate`) before coding.
- Collateral: USDC.e + POL for gas (EOA wallets); deposit-wallet signature types available.

### 3.4 Breakeven math (the number that gates everything)

At mid prices (both legs near 50¢), a two-taker-leg arb costs roughly:

| Cost component | Per 100 contracts |
|---|---|
| Kalshi taker (~50¢) | ~$1.75 |
| Polymarket US taker (~50¢, capped) | ~$1.25 |
| **Total fee load** | **~$3.00** |

→ **Minimum viable gross spread ≈ 3¢/contract at mid prices**, shrinking toward the price extremes where both fee curves approach zero. A worked example from the literature: a 2¢ gross spread on 100 contracts produced $2.00 gross against $3.15 in fees — unprofitable until scaled to ~1,000 contracts (where the same fixed spread clears the proportional fee load).

**Fee-reduction levers, in order of impact:**

1. **Make one leg a maker order** (zero fee + rebate on Polymarket) — converts risk-free arb into legged execution risk; requires an unwind policy.
2. Trade nearer the price extremes (favorites/longshots) where both fee curves collapse.
3. Climb Kalshi's volume tiers.
4. Exploit fee-category asymmetries (e.g., Polymarket geopolitics is free — irrelevant for sports but relevant if scope expands).

### 3.5 Research still needed

- Pull both live fee schedules programmatically and build a unit-tested fee calculator (including Kalshi per-contract rounding) — this is a research deliverable *and* the first repo module.
- Confirm which Kalshi sports series currently carry maker fees.
- Determine our realistic starting Kalshi fee tier and the volume needed to reach the next one.

---

## 4. API Capabilities, Rate Limits & Contract Mechanics

### 4.1 Kalshi

- REST + WebSocket + FIX; RSA-PSS request signing (sign path without query params; millisecond timestamps). Full sandbox at `demo-api.kalshi.com`.
- **Rate limits (token-bucket model, effective April 23, 2026):** five tiers, independent Read and Write buckets, default cost 10 tokens/request. Basic tier ≈ 20 reads/s and 10 writes/s; Premier Write refills 1,000 tokens/s (≈100 orders/s) with ~2s burst capacity. 429 responses carry **no Retry-After header** — client-side token accounting is mandatory.
- **WebSocket:** max 5 concurrent connections per user → multiplex all subscriptions on one connection; use `orderbook_delta` channel and maintain a local book instead of REST polling.
- Market discovery: `GET /markets` with `status=open`, cursor pagination (no total counts), up to 1,000/page. Historical: trades + OHLC candles via `/markets/{ticker}/history`; **no historical order book snapshots** from the exchange itself.
- Latency baseline: 50–150ms round trip from US East Coast; AWS us-east-1 VPS is sufficient (this is not equity HFT).

### 4.2 Polymarket Global (API reference; useful for research even if we don't trade it)

- Three APIs: CLOB (trading/order book), Gamma (market discovery/metadata), Data (positions/history), plus WebSocket channels (market/user/sports) and RTDS low-latency stream.
- **Breaking change:** CLOB **V2 hard cutover April 28, 2026** — new contracts, new order struct, new fee model, pUSD collateral, V1 SDKs (`py-clob-client`, `@polymarket/clob-client`) dead on production. Most public GitHub arb bots predate this and are broken. Build against V2 (`py-clob-client-v2`).
- Rate limits (Cloudflare throttling — degrades rather than hard-fails): general 15,000 req/10s; CLOB 9,000/10s; order placement 3,500/10s burst; batch endpoints accept up to 15 orders/call; book/price endpoints 1,500/10s. WebSocket: up to 500 subscriptions/connection.
- Auth: EIP-712 signed messages; four signature types (EOA, POLY_PROXY, GNOSIS_SAFE, POLY_1271).

### 4.3 Polymarket US (our actual trading API — the tight constraint)

- Separate stack at `api.polymarket.us`: 23 REST endpoints + 2 WebSocket endpoints. **Ed25519 auth** with strict timestamp checks.
- **Public REST limited to ~60 requests/minute** — the scanner must be WebSocket-first by necessity.
- **WebSocket subscriptions capped at ~10 instruments per connection** (`/v1/ws/markets`) — this caps concurrent game coverage per connection and is a hard architectural input. Determine max concurrent connections empirically.
- API keys require KYC through the iOS app + developer portal; private key shown once.

### 4.4 Contract-level mechanics to validate per order

Every order must respect: `tick_size` (valid price increments), `minimum_order_size`, `accepting_orders` flag, and `seconds_delay` (some markets delay order acceptance after creation). Kalshi equivalents: market open status, close vs. determination time distinction.

### 4.5 Research still needed

- Empirically measure Polymarket US: max concurrent WS connections, effective book depth message rates, order acknowledgment latency.
- Confirm whether Polymarket US exposes batch order endpoints (Global does; US docs are newer and thinner).
- Kalshi `GET /account/endpoint_costs` — enumerate non-default token costs for the endpoints we'll hit hardest.

---

## 5. Liquidity & Volume

- Kalshi: ~$500M/day total volume; 93% sports on peak days; deepest US sports books. Parlay volume is growing (>20% of volume on some days) — parlays are out of scope but drain retail flow from singles.
- Polymarket Global: $3.7–9B/month range across 2025–2026; deep on politics/crypto/international.
- Polymarket US: young and thin — ~$60M all-sports volume in early March Madness rounds, $90M+ during the 2026 Masters, while operating waitlisted. Post-waitlist depth is an open question.
- Thin books cut both ways: more frequent mispricings, but less executable size. Fill size is always capped by the thinner side — expect Polymarket US to be the binding constraint initially.

**Research still needed:** top-of-book depth distribution on matched NBA/NFL/MLB moneylines on both venues at game-day timescales; how depth evolves from market open → tipoff → in-game.

---

## 6. Evidence That the Edge Exists (and Its Size)

- **SSRN (Krause, 2026), Kalshi vs. Polymarket trade data:** legislative event (CLARITY Act): mean post-fee arb 4.87%, exploitable on 89.1% of trading days, max single-day 24.07%. July 2026 Fed decision: mean 1.38%, 63.9% of days, max 6.35%. Both p < .001. Attributed to capital lockup, fiat/crypto friction, and regulatory segmentation.
- **IMDEA Networks study:** >$40M arbitrage profit extracted from Polymarket alone, April 2024–April 2025, across 86M analyzed bets (includes intra-platform bundle arb, not only cross-venue).
- **Industry tracking (2026):** 2–5% spreads documented on major events; 5–8¢ gaps cited on some markets; World Cup outright winners trading 1.3pp apart across venues.
- **Federal Reserve Board / Northwestern / JHU (Feb 2026):** prediction market prices on FOMC outcomes beat fed funds futures forecasts — relevant as credibility evidence for the asset class.

**Caveats:** all headline persistence numbers involve Polymarket *Global*. The Kalshi ↔ Polymarket US spread is undocumented in the literature — measuring it is our project's first real contribution (§11). Also note within-platform arbitrage (bundle arb: YES+NO ≠ $1 on one venue; multi-outcome events summing ≠ 100%) is a documented, simpler adjacent strategy worth including in the scanner for free.

---

## 7. Risks

### 7.1 Competition / edge decay
Many of Polymarket's most profitable wallets are automated systems; latency is decisive once an inefficiency is widely known, and manual arbitrage is effectively impossible in 2026. Platforms actively fight latency arb (Polymarket's highest fees target 15-minute crypto markets for exactly this reason). Mitigating factors: prediction market liquidity is still too shallow to attract major HFT firms, latency requirements are VPS-grade (sub-second, not sub-millisecond), and the historical lesson from equity arb (Budish et al.) is that speed races raise entry costs without eliminating the prize. Treat "sports spreads are already gone" as a live hypothesis to test, not a reason to skip testing.

### 7.2 Resolution risk (the way a "risk-free" arb loses money)
The two venues can settle the same event differently in edge cases (postponements, data-source discrepancies, rule-wording differences). If both settle the same direction when the strategy needed opposite outcomes, the hedged leg becomes a loss. Kalshi uses regulated resolution sources with documented correction procedures; Polymarket Global uses UMA's optimistic oracle (history of contested resolutions); Polymarket US resolution mechanics under CFTC rules need verification. Mitigation: sports moneylines only, manual review of rule text before whitelisting any market pair, postponement policy comparison per league.

### 7.3 Execution risk
One leg fills, the other doesn't (spread closed, depth pulled, rate-limited mid-sequence). Requires a pre-committed unwind policy: chase / hold naked briefly / unwind at loss. Partial-fill handling and idempotent order state recovery after disconnects are core engineering requirements, not edge cases.

### 7.4 Regulatory tail risk
Sports event contracts are contested: state gaming commissions argue they are unlicensed sportsbooks. Minnesota enacted the first outright ban (effective Aug 1, 2026, felony exposure); CFTC immediately sued to block it and has separately sued CT, AZ, and IL on federal-preemption grounds. Kalshi is banned in Nevada; Polymarket US excludes 8 states. A court-ordered delisting mid-position is a tail risk — size the book so a forced unwind is survivable. California (our jurisdiction) is currently unaffected on both venues.

### 7.5 Platform risk
Fee schedules changed 4+ times across the two platforms in 2026 alone; Polymarket executed a no-backward-compatibility API cutover with ~2 months' notice. The system needs fee schedules and API versions as config/data, not constants, plus monitoring for schedule-change announcements.

---

## 8. Operational Constraints

**Capital velocity.** Kalshi sports markets resolve automatically ~30–90 minutes post-game; settlement credits balance within minutes to a few hours; funds are immediately re-tradable on-platform. But **cross-venue rebalancing runs through a bank**: ACH 1–3 business days (up to 5), wires 1–2 days with fees, debit/crypto rails faster where supported. Skewed fill outcomes drain one account and idle the system for days. Inventory drift across venues is a first-class risk parameter, not an afterthought.

**Capital requirements.** Practitioner consensus: $2,000–$5,000 split across platforms is the realistic minimum for fee-viable position sizes; under ~$1,000, per-contract rounding and minimum sizes eat the edge.

**Account setup (our jurisdiction — CA):** Kalshi: standard signup, API keys, sandbox available. Polymarket US: iOS app KYC → developer portal keys → fund with USDC.e + POL. Both legal in California.

**Taxes.** Kalshi issues 1099s (US-regulated). Polymarket US tax reporting under its FCM status needs confirmation. High-frequency arb generates large gross proceeds with thin net margins — understand reporting burden before scaling. *(Open research item; consult actual guidance, not blog posts.)*

---

## 9. Data for Backtesting

- Kalshi official API: trade history + OHLC candles per market; **no historical order book** — the exchange keeps no depth backfill.
- Polymarket: trade history via Data API and on-chain (The Graph subgraphs); ~1,050 unresolved/delisted markets exist — excluding them creates survivorship bias in any historical study.
- Third-party vendors: Lychee (36GB+ full Kalshi trade archive + Polymarket history, no-code), Oddpool (tick-level order book deltas + trades for both venues, cross-venue canonical IDs, Parquet — built exactly for this use case), KalshiBackTest (100ms book snapshots, crypto markets only).
- Known data trap: many Dune dashboards double-count Polymarket volume ~2×.
- **Implication:** because no exchange backfills order book depth, spread-at-depth history *only exists if someone was recording it*. Our own recorder, started now, becomes the dataset. Vendor data (Oddpool) is the shortcut if budget allows; otherwise self-recording is the first two weeks of the project anyway.

---

## 10. Strategic Assessment

| Factor | Assessment |
|---|---|
| Does the edge exist? | Yes — documented, peer-reviewed, persistent (Kalshi↔PM Global) |
| Does OUR edge exist? | Unknown — Kalshi↔PM US spread is unmeasured; likely thinner |
| Instrument choice (sports binaries) | Correct — clean resolution, fast capital recycling, deep Kalshi books |
| Main cost driver | Fees (~3¢/contract at mids, two taker legs) + capital lockup |
| Main technical constraint | Polymarket US API limits (60 req/min REST, 10 instruments/WS) |
| Main risk to "risk-free" | Resolution mismatch + one-legged fills |
| Main external risk | State-level sports-contract litigation |
| Competitive posture | Bots dominate; VPS-grade latency sufficient; shallow books deter institutions |
| Expansion path | Robinhood/Susquehanna MIAXdx exchange (2026) as third venue |

---

## 11. Open Empirical Questions (Phase-0 Research Plan)

These cannot be answered from published sources — they are the measurement work the repo does first:

1. **The core question:** what is the distribution of net-of-fee spreads between Kalshi and Polymarket US on matched sports moneylines — frequency, magnitude, duration, and *executable depth*? (Two weeks of live recording, both books, NBA finals / MLB / tennis in current season.)
2. How many markets are matched across both venues per week, and can matching be automated reliably from metadata alone (team names + date + league), with what error rate?
3. What is real order-ack latency and WS message latency on each venue from a us-east-1 VPS?
4. How often do spreads survive longer than our expected two-leg execution time (~1–2s)?
5. What is the realistic Polymarket US depth at top-of-book on game day vs. Kalshi's?
6. What fraction of theoretically profitable spreads occur at price extremes (cheap fees) vs. mid prices (expensive fees)?
7. Inventory drift simulation: given historical outcomes, how fast does capital skew to one venue, and what rebalancing cadence/cost does that imply?
8. Bundle-arb frequency within each venue (YES+NO < $1 on a single book) — a free byproduct of the same data feed.

**Decision gate:** if question 1 shows fewer than N net-positive, depth-executable opportunities per week (N set by capital and target return), the execution engine is not worth building for sports — pivot categories (politics/econ, where the SSRN evidence is strongest) or shelve.

---

## 12. Key Sources

Academic: Krause, *From Forecasting Tool to Financial Asset* (SSRN 6905683, 2026); IMDEA Networks Polymarket arbitrage study (2025); Diercks/Katz/Wright, FOMC prediction market accuracy (Feb 2026); Budish/Cramton/Shim (2015) on arb speed races.

Primary/official: Kalshi fee schedule PDF (July 7, 2026 update); Kalshi rate-limit docs (docs.kalshi.com); Kalshi Help Center (fees, market FAQs); Polymarket Help Center (trading fees); Polymarket docs (rate limits, CLOB V2, WebSocket channels); Polymarket US developer portal.

Industry/analysis: DropsTab Kalshi-vs-Polymarket comparison (June 2026); Sacra Polymarket profile; laikalabs/clawarbs/launchpoly/polycopy arbitrage guides (2026); AgentBets Polymarket API + rate-limit guides; QuantVPS Kalshi/Polymarket US API guides; Turbine historical-data survey; CoinDesk/Forbes/DL News coverage of venue partnerships and regulation; startpolymarket US legality timeline.
