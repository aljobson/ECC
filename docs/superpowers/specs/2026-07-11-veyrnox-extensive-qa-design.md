# Veyrnox Extensive QA & Testing — Design Spec

**Date:** 2026-07-11  
**Branch:** `claude/ecc-qa-skills-2dd7c3`  
**Goal:** Find bugs AND produce a coverage/health report, with inline fixes for P0/P1 issues and a prioritised backlog for the rest.

---

## Scope

Two targets tested in parallel:

1. **ECC plugin** — `<REDACTED_PATH>\Users\<username>\Downloads\VEYRNOX-CLONE-ECC\.claude\worktrees\upbeat-bartik-2317a9`
2. **Veyrnox wallet app** — `<REDACTED_PATH>\Users\<username>\Downloads\VEYRNOX-CLONE-ECC`

---

## Approach

**Parallel fan-out (Approach B).** Five specialist agents run simultaneously, each owning one test dimension. A synthesis agent collects all findings into one ranked report. Wall clock target: ~45–60 min.

---

## Test Dimensions & Agent Assignments

| Agent | Dimension | Method |
|---|---|---|
| `ecc-unit` | ECC plugin integrity | `node tests/run-all.js`, markdown lint (`npx markdownlint-cli`), CI validator scripts |
| `veyrnox-unit` | Veyrnox unit + integration | `npm test` (Vitest + pretest guards: CSPRNG, deniability, finding-ID, typecheck:core), `--coverage` for baseline |
| `veyrnox-e2e-web` | Playwright E2E — existing specs + new flows | Existing `test:e2e`, plus new spec: seed import (throwaway seed), balance fetch (testnet RPC), send flow (testnet), demo mode smoke (`?demo=1`) |
| `veyrnox-wallet-core` | wallet-core security audit — static analysis | Read vault, KEK, mnemonic, derivation, duress, panic, stealth, send — check Argon2id params, AES-GCM nonce reuse, deniability string leaks, ring-boundary violations |
| `veyrnox-ui` | UI/UX, visual, accessibility | All routes at 375/768/1440px; light mode + dark mode + font rendering per page; axe-core a11y (WCAG 2.2 AA); Core Web Vitals |

---

## Environment & Tooling

| Concern | Detail |
|---|---|
| ECC plugin root | `…/upbeat-bartik-2317a9` |
| Veyrnox app root | `<REDACTED_PATH>\Users\<username>\Downloads\VEYRNOX-CLONE-ECC` |
| Demo mode | `?demo=1` query param (no real keys needed for UI agent) |
| Testnet seed | `bamboo lyrics harvest potato seat carry equip nation slam begin admit pet` (throwaway — E2E agent only) |
| Derived addresses | EVM `0x90f9f1F9F5a1938B21ef0C20352C7b792E68a729`, BTC testnet `tb1qztdfvzkdup458v6nk555ztzsgduh7lhggekx54`, SOL devnet `Cp5MYrCMbUe7wra4ziGsVN672ZjpeLi5CFNj4Je7yFWK` |
| RPC | Testnet defaults from `src/wallet-core/rpcConfig.js` — no overrides needed |
| Coverage | Vitest `--coverage` — establishes baseline; no thresholds enforced today |
| Mobile E2E | Out of scope — WebdriverIO/Appium requires connected device; flagged P2 in backlog |
| Secrets hygiene | Throwaway seed used as test fixture only — never committed, redacted from screenshots |

---

## Report Format

Output: `docs/qa/2026-07-11-qa-report.md` committed to this branch.

```
## Executive Summary
- Overall verdict: SHIP / SHIP WITH FIXES / DO NOT SHIP
- Counts: X bugs found, Y fixed inline, Z queued to backlog

## ECC Plugin
- Test results, markdown lint output, coverage gaps

## Veyrnox Unit / Integration
- Pass/fail, coverage baseline (lines / functions / branches), untested surface map

## Wallet-Core Security Audit
- Findings with severity: CRITICAL / HIGH / MEDIUM / LOW
- Each finding: description, file:line, recommendation

## E2E Web Flows
- Seed import, balance display, send flow, demo mode results
- Screenshots attached for failures

## UI / UX / Accessibility
- Per-page results at 375 / 768 / 1440px
- Light mode + dark mode + font rendering per page
- a11y violations (WCAG 2.2 AA)

## Backlog (Prioritised)
- P0 — block ship
- P1 — fix next sprint
- P2 — nice to have
```

---

## Fix Policy

- **Inline fixes:** P0 and P1 bugs that are self-contained (wrong crypto param, missing ARIA label, broken selector, clear missing test) are fixed immediately by the relevant specialist agent and committed on this branch.
- **Backlog:** Everything else is documented in the report as a prioritised item — not touched in this pass.
- **No fixes to wallet-core signing/key/seed logic** without a failing test written first (TDD gate).

---

## Out of Scope

- Mobile E2E (WebdriverIO/Appium) — requires connected device/emulator
- Mainnet transactions — testnet only
- Ledger/Trezor hardware wallet flows — requires physical device
- Performance load testing
