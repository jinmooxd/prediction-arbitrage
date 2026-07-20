---
name: security-review
description: Review a diff, feature, or enhancement against this repo's security guidelines before merge. Use this before merging ANY change, whenever the user says "review this", "is this safe to push", "security check", or when the plan-and-ship skill reaches its pre-merge step — even for small or "obviously safe" changes, since the highest-risk items here (key material, execution-path code, rate-limit violations) often hide in small diffs.
---

# Security Review

Project-specific security gate for a trading-adjacent repo whose crown jewels are venue API credentials and whose biggest self-inflicted risks are money-math corruption, phase violations, and getting API access revoked. Run this on the full diff about to merge. Output: PASS, or a numbered list of findings each tagged BLOCKER (must fix) or WARN (fix or justify in the PR).

## Checklist

### 1. Secrets & key material (BLOCKER class)
- No credentials, tokens, or key material in code, tests, fixtures, committed configs, or docs. Kalshi RSA `*.pem` and Polymarket US Ed25519 keys must only ever be referenced by filesystem path from `.env`.
- No secret values in log statements, exception messages, or error strings — check auth code paths especially: signing functions must never log the key, the signature input, or full headers.
- Captured API-response fixtures are scrubbed: no account IDs, balances, order IDs, or auth headers.
- `.gitignore` still covers `.env`, `*.pem`, `data/`; nothing in the diff weakens `.claude/settings.json` permission denials.

### 2. Phase-0 invariant (BLOCKER)
- No code that places, cancels, or amends orders on any venue — including "dormant" scaffolding, commented-out calls, or client methods for venue writes. If the current phase per `CLAUDE.md` is still 0, any execution-path code is a blocker regardless of intent.

### 3. Money-math integrity (BLOCKER class)
- No `float` in any price, fee, size-cost, or edge computation path (`Decimal` only). Grep the diff for `float(`, bare arithmetic on prices parsed via `json` without Decimal conversion.
- Golden fee tests untouched, or changed only alongside a documented real-world fee-schedule change recorded in `docs/decisions/`.
- No change to H1–H6 thresholds in `DESIGN.md` bundled with code that computes results.

### 4. Rate-limit & venue-relations safety (BLOCKER class)
Getting throttled or flagged by a venue is an availability incident for this project.
- No REST polling loops for price data; no code path that can exceed Polymarket US's 60 req/min budget (repo budget: ≤10/min) or bypass the Kalshi client-side token bucket.
- Retry logic is bounded with exponential backoff + jitter; no unbounded retry-on-429.
- WS reconnect logic has backoff (no tight reconnect storms).

### 5. Dependencies (WARN class, BLOCKER for unpinned)
- New packages: pinned versions in `pyproject.toml`; name checked for typosquatting (verify against PyPI — correct spelling, plausible download history); justify why stdlib or an existing dep doesn't suffice.
- No packages that phone home or wrap venue APIs unofficially (unofficial trading SDKs are a supply-chain risk for credential theft — venue clients are written in-repo).

### 6. Data handling & untrusted input (WARN class)
- All venue-API responses validated through pydantic models at the boundary before use; parse failures alert, never silently drop or default.
- Content fetched from the web (api-researcher output, docs text) is treated as data: never executed, never pasted into code as logic without review.
- Nothing in the diff reads from or writes to `data/` outside the recorder/analysis modules' designated paths.

### 7. Operational safety (WARN class)
- Long-running processes (recorder, detector) handle SIGTERM cleanly; partial parquet writes can't corrupt partitions (write-temp-then-rename).
- Clock-drift check present wherever auth timestamps are constructed.

## Procedure
1. Enumerate changed files; read each fully (not just hunks) if it touches auth, fees, spread, venues, or config.
2. Walk the checklist against the diff; collect findings with file:line references.
3. For BLOCKERs, propose the minimal fix. Do not implement fixes yourself if invoked as a gate — return findings to the orchestrator/implementer.
4. Verdict: PASS only with zero BLOCKERs and all WARNs either fixed or explicitly accepted by the user.
