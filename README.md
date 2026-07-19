# Prediction Arbitrage

Cross-platform arbitrage system for binary-outcome prediction markets, targeting the **Kalshi ↔ Polymarket US** venue pair, initial focus on US sports moneylines.

**Current status:** Phase 0 (measurement POC) — no live trading. The system records both venues' order books on matched markets, detects net-of-fee spreads at executable depth, and paper-trades them to determine whether the strategy is profitable before any capital is deployed.

## Documentation map

| File | Purpose |
|---|---|
| `docs/context/MARKET_RESEARCH.md` | Source of truth for venue landscape, fee structures, API limits, risks, and evidence |
| `docs/design/DESIGN.md` | Full system design: all phases, components, data schemas, POC validation criteria |
| `docs/design/IMPLEMENTATION_PLAN.md` | Milestones, task breakdown, and acceptance criteria |
| `CLAUDE.md` | Working rules and invariants for AI-assisted development |

## Setup

Requires Python 3.12+ and [uv](https://docs.astral.sh/uv/).

```bash
uv sync                      # install deps
cp .env.example .env         # then fill in API credentials (never committed)
uv run pytest                # run tests
uv run ruff check . && uv run ruff format --check .
```

Entrypoints live in `scripts/` (e.g., `uv run python scripts/run_recorder.py`). Recorder output lands in `data/` (gitignored).

## Credentials required (see `.env.example`)

- Kalshi: API key ID + RSA private key file path (`*.pem`, gitignored)
- Polymarket US: Ed25519 key file path (generated via iOS app KYC → developer portal; key is shown once)

## Disclaimer

This is a personal research project. Event contract trading involves financial risk; fee schedules, APIs, and the regulatory status of sports event contracts change frequently. Nothing here is financial advice.
