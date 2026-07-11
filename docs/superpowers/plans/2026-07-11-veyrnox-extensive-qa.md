# Veyrnox Extensive QA & Testing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Run a comprehensive, parallel QA pass across the ECC plugin and the Veyrnox wallet app, produce a ranked bug report, fix all P0/P1 self-contained issues inline, and queue the rest as a prioritised backlog.

**Architecture:** Five specialist agents run in parallel (Tasks 1–5), each owning one test dimension. Task 6 synthesises all findings into a single report, applies inline fixes, and commits everything to the branch. Tasks 1–5 have no dependency on each other and may be dispatched simultaneously.

**Tech Stack:** Node.js 18+ (ECC plugin), React 18 + Vite 6 + Capacitor 8 (Veyrnox), Vitest + Playwright (testing), @noble/curves + @scure/bip32/39 + ethers v6 (crypto), axe-core (a11y).

## Global Constraints

- ECC plugin root: `C:\Users\aljob\Downloads\VEYRNOX-CLONE-ECC\.claude\worktrees\upbeat-bartik-2317a9`
- Veyrnox app root: `C:\Users\aljob\Downloads\VEYRNOX-CLONE-ECC`
- Throwaway testnet seed (E2E agent only, never commit): `bamboo lyrics harvest potato seat carry equip nation slam begin admit pet`
- Derived addresses: EVM `0x90f9f1F9F5a1938B21ef0C20352C7b792E68a729`, BTC testnet `tb1qztdfvzkdup458v6nk555ztzsgduh7lhggekx54`, SOL devnet `Cp5MYrCMbUe7wra4ziGsVN672ZjpeLi5CFNj4Je7yFWK`
- No mainnet transactions — testnet/devnet only
- No wallet-core signing/key/seed fixes without a failing test written first (TDD gate)
- All findings written to `docs/qa/2026-07-11-qa-report.md` using the schema in Task 6
- Inline fixes committed on branch `claude/ecc-qa-skills-2dd7c3`
- Mobile E2E (WebdriverIO/Appium) is out of scope
- Report severity scale: CRITICAL / HIGH / MEDIUM / LOW; priority scale: P0 / P1 / P2

---

## Tasks 1–5 run in parallel. Dispatch them simultaneously.

---

### Task 1: ECC Plugin Integrity

**Agent type:** `veyrnox-recon` for discovery, `act` for running commands  
**Working directory:** ECC plugin root

**Files:**
- Read: `tests/run-all.js`, `package.json`, `scripts/ci/validate-*.js`
- Produce findings to: `docs/qa/findings/ecc-plugin.md` (create)

**Interfaces:**
- Produces: `docs/qa/findings/ecc-plugin.md` — consumed by Task 6

- [ ] **Step 1: Run the full ECC test suite**

```bash
cd "C:\Users\aljob\Downloads\VEYRNOX-CLONE-ECC\.claude\worktrees\upbeat-bartik-2317a9"
node tests/run-all.js 2>&1
```

Expected: summary line showing pass/fail counts. Capture full output.

- [ ] **Step 2: Run markdown lint across all skill/agent/command/rule files**

```bash
npx markdownlint-cli "**/*.md" --ignore node_modules --ignore docs 2>&1
```

Capture all violations. Each violation is a finding.

- [ ] **Step 3: Run each CI validator script individually**

```bash
node scripts/ci/validate-agents.js 2>&1
node scripts/ci/validate-commands.js 2>&1
node scripts/ci/validate-skills.js 2>&1
node scripts/ci/validate-hooks.js 2>&1
node scripts/ci/validate-rules.js 2>&1
node scripts/ci/validate-install-manifests.js 2>&1
node scripts/ci/check-unicode-safety.js 2>&1
```

Record any failures — each is a finding.

- [ ] **Step 4: Check for skill files missing required frontmatter fields**

```bash
# Skills must have: name, description in YAML frontmatter
grep -rL "^name:" skills/*/SKILL.md 2>/dev/null
grep -rL "^description:" skills/*/SKILL.md 2>/dev/null
```

Any file listed is a finding (MEDIUM).

- [ ] **Step 5: Write findings to docs/qa/findings/ecc-plugin.md**

Create the file with this structure:

```markdown
# ECC Plugin — QA Findings

## Test Suite
- Result: PASS / FAIL
- Output: [paste summary]
- Failures: [list each failing test with error]

## Markdown Lint
- Violations: [count]
- [file:line — rule — description] per violation

## CI Validators
- [validator name]: PASS / FAIL [error if fail]

## Frontmatter Gaps
- [file path] — missing: [field name]

## Summary
- Total findings: N
- CRITICAL: N | HIGH: N | MEDIUM: N | LOW: N
```

- [ ] **Step 6: Fix any P0/P1 issues inline**

P0 = CI validator hard failures (broken install manifests, unicode safety violations).
P1 = Missing required frontmatter on any skill/agent/command.

Fix inline, then:

```bash
git add -A
git commit -m "fix(ecc): inline QA fixes from Task 1"
```

---

### Task 2: Veyrnox Unit & Integration Tests + Coverage Baseline

**Agent type:** `act`  
**Working directory:** Veyrnox app root

**Files:**
- Read: `package.json`, `vitest.config.js`, `src/wallet-core/**/__tests__/`
- Produce findings to: `docs/qa/findings/veyrnox-unit.md` (create in ECC plugin root)

**Interfaces:**
- Produces: `docs/qa/findings/veyrnox-unit.md` — consumed by Task 6

- [ ] **Step 1: Run the full test suite with pretest guards**

```bash
cd "C:\Users\aljob\Downloads\VEYRNOX-CLONE-ECC"
npm test 2>&1
```

Expected: pretest guards run first (CSPRNG check, deniability scan, finding-ID check, typecheck:core), then Vitest. Capture full output including any pretest failures.

- [ ] **Step 2: Run with coverage enabled to establish baseline**

```bash
npm test -- --coverage 2>&1
```

Capture the coverage table output (lines / functions / branches / statements per file).

- [ ] **Step 3: Identify untested wallet-core surface**

```bash
# List all wallet-core source files
find src/wallet-core -name "*.js" -not -path "*/__tests__/*" -not -name "*.test.js" | sort

# List all wallet-core test files
find src/wallet-core -name "*.test.js" -o -name "*.spec.js" | sort
```

For each source file, check whether a corresponding test file exists. Flag any source file with no test coverage as a finding (severity based on criticality — vault/KEK/mnemonic/signing = HIGH, utilities = LOW).

- [ ] **Step 4: Check for failing pretest guards specifically**

```bash
node scripts/check-crypto-rng.mjs 2>&1
node scripts/check-deniability-strings.mjs 2>&1
node scripts/check-finding-id-consistency.mjs 2>&1
npx tsc --project tsconfig.wallet-core.json --noEmit 2>&1
```

Each failure is a finding (CRITICAL for CSPRNG, HIGH for deniability leak).

- [ ] **Step 5: Write findings to docs/qa/findings/veyrnox-unit.md**

```markdown
# Veyrnox Unit & Integration — QA Findings

## Pretest Guards
- CSPRNG check: PASS / FAIL
- Deniability scan: PASS / FAIL
- Finding-ID consistency: PASS / FAIL
- typecheck:core: PASS / FAIL

## Test Suite
- Result: PASS / FAIL
- Total tests: N passed, N failed, N skipped
- Failed tests: [test name — error] per failure

## Coverage Baseline
| File | Lines | Functions | Branches | Statements |
|---|---|---|---|---|
| src/wallet-core/vault.js | % | % | % | % |
| [... all files] | | | | |

## Untested Surface
- [file path] — severity — reason

## Summary
- Total findings: N
- CRITICAL: N | HIGH: N | MEDIUM: N | LOW: N
```

- [ ] **Step 6: Fix any P0/P1 issues inline**

P0 = Any pretest guard failure (CSPRNG, deniability leak).
P1 = Any Vitest test failure in wallet-core signing, vault, or KEK.

For wallet-core fixes: write a failing test first, verify it fails, implement fix, verify test passes, then commit. For non-wallet-core fixes: fix directly and commit.

```bash
git add -A
git commit -m "fix(veyrnox-unit): inline QA fixes from Task 2"
```

---

### Task 3: Veyrnox E2E Web Flows (Playwright)

**Agent type:** `act`  
**Working directory:** Veyrnox app root

**Files:**
- Read: `tests/web/specs/web-deniability-e2e.spec.ts`, `playwright.config.ts` (or `playwright.config.js`)
- Create: `tests/web/specs/qa-seed-import-e2e.spec.ts`
- Produce findings to: `docs/qa/findings/veyrnox-e2e.md` (create in ECC plugin root)

**Interfaces:**
- Produces: `docs/qa/findings/veyrnox-e2e.md` — consumed by Task 6

- [ ] **Step 1: Run existing Playwright E2E specs**

```bash
cd "C:\Users\aljob\Downloads\VEYRNOX-CLONE-ECC"
npm run test:e2e 2>&1
```

Capture pass/fail per spec. Each failure is a finding.

- [ ] **Step 2: Run demo mode smoke test**

Start dev server if not running, then navigate to `http://localhost:5173?demo=1`.

Write an inline Playwright script (do not commit — throw-away):

```typescript
// throwaway smoke - run inline
import { chromium } from 'playwright';
const browser = await chromium.launch({ headless: true });
const page = await browser.newPage();
await page.goto('http://localhost:5173?demo=1');
await page.waitForLoadState('networkidle');
// Verify demo banner is visible
const demoBanner = await page.locator('[data-testid="demo-banner"], text=Demo, text=DEMO').first().isVisible().catch(() => false);
// Verify dashboard loads
const hasBalance = await page.locator('text=/\\$[0-9]|balance|Balance/i').first().isVisible().catch(() => false);
console.log('Demo banner:', demoBanner, '| Balance shown:', hasBalance);
const errors = [];
page.on('console', m => { if (m.type() === 'error') errors.push(m.text()); });
await page.waitForTimeout(2000);
console.log('Console errors:', errors);
await browser.close();
```

Record: whether demo mode loaded, whether fake balances appear, any console errors.

- [ ] **Step 3: Create and run seed-import E2E spec**

Create `tests/web/specs/qa-seed-import-e2e.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';

const THROWAWAY_SEED = 'bamboo lyrics harvest potato seat carry equip nation slam begin admit pet';
const EXPECTED_EVM = '0x90f9f1F9F5a1938B21ef0C20352C7b792E68a729';

test.describe('Seed Import — Throwaway Testnet Wallet', () => {
  test('imports seed and shows correct EVM address', async ({ page }) => {
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    // Navigate to seed import / restore flow
    // Try common entry points
    const importBtn = page.getByRole('button', { name: /import|restore|existing wallet/i });
    await importBtn.click();
    await page.waitForLoadState('networkidle');

    // Fill seed phrase — 12 words
    const seedInput = page.getByPlaceholder(/seed|mnemonic|recovery phrase/i)
      .or(page.locator('textarea').first());
    await seedInput.fill(THROWAWAY_SEED);

    // Submit
    await page.getByRole('button', { name: /continue|next|import|confirm/i }).click();
    await page.waitForLoadState('networkidle');

    // Verify address derived correctly
    await expect(page.getByText(EXPECTED_EVM, { exact: false })).toBeVisible({ timeout: 10000 });
  });

  test('shows testnet balance for derived EVM address', async ({ page }) => {
    // After import, navigate to dashboard
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    // Balance area should be visible (may be 0 on testnet — just check it renders)
    const balanceArea = page.locator('[data-testid="balance"], text=/ETH|balance|\\$0/i').first();
    await expect(balanceArea).toBeVisible({ timeout: 15000 });
  });

  test('demo mode does not use real seed data', async ({ page }) => {
    await page.goto('/?demo=1');
    await page.waitForLoadState('networkidle');

    // Demo mode must never show the throwaway address
    const pageContent = await page.content();
    expect(pageContent).not.toContain(EXPECTED_EVM);
  });
});
```

Run it:

```bash
npx playwright test tests/web/specs/qa-seed-import-e2e.spec.ts --reporter=list 2>&1
```

Capture pass/fail and any error output. Each failure is a finding.

- [ ] **Step 4: Test send flow (read-only verification — UI only, no broadcast)**

```typescript
// Add to qa-seed-import-e2e.spec.ts
test('send form validates address and amount', async ({ page }) => {
  await page.goto('/send');
  await page.waitForLoadState('networkidle');

  // Invalid address
  const addrInput = page.getByLabel(/address|recipient/i).or(page.locator('input[placeholder*="0x"]').first());
  await addrInput.fill('not-an-address');
  await page.getByRole('button', { name: /next|continue|send/i }).click();
  const errVisible = await page.getByRole('alert').or(page.locator('text=/invalid|error/i')).first().isVisible().catch(() => false);
  expect(errVisible).toBe(true);

  // Valid address, zero amount
  await addrInput.fill('0x90f9f1F9F5a1938B21ef0C20352C7b792E68a729');
  const amtInput = page.getByLabel(/amount/i).or(page.locator('input[type="number"]').first());
  await amtInput.fill('0');
  await page.getByRole('button', { name: /next|continue|send/i }).click();
  const amtErr = await page.getByRole('alert').or(page.locator('text=/invalid|error|zero/i')).first().isVisible().catch(() => false);
  expect(amtErr).toBe(true);
});
```

Run the full updated spec again and record results.

- [ ] **Step 5: Write findings to docs/qa/findings/veyrnox-e2e.md**

```markdown
# Veyrnox E2E Web — QA Findings

## Existing Playwright Specs
- web-deniability-e2e: PASS / FAIL [details]

## Demo Mode Smoke
- Loaded: yes/no
- Fake balances visible: yes/no
- Console errors: [list]

## Seed Import Flow
- Correct EVM address derived: yes/no
- Balance area renders: yes/no
- Demo mode isolation (no real address): yes/no

## Send Form Validation
- Invalid address rejected: yes/no
- Zero amount rejected: yes/no

## Findings
- [description] — severity — file:line or URL

## Summary
- Total findings: N
- CRITICAL: N | HIGH: N | MEDIUM: N | LOW: N
```

- [ ] **Step 6: Commit the new E2E spec**

```bash
git add tests/web/specs/qa-seed-import-e2e.spec.ts
git commit -m "test(e2e): add seed-import and send-form QA specs"
```

---

### Task 4: Wallet-Core Security Audit (Static)

**Agent type:** `veyrnox-honest-reviewer`  
**Working directory:** Veyrnox app root

**Files:**
- Read (do not modify in this task):
  - `src/wallet-core/vault.js`
  - `src/wallet-core/keystore/kek.js`, `keyStore.js`, `native.js`, `web.js`
  - `src/wallet-core/mnemonic.js`
  - `src/wallet-core/derivation.js`
  - `src/wallet-core/duress.js`, `deniabilityUnlock.js`, `deniabilitySession.js`
  - `src/wallet-core/panic.js`
  - `src/wallet-core/stealth.js`
  - `src/wallet-core/evm/send.js`, `btc/send.js`, `sol/send.js`
  - `src/wallet-core/coldkey/` (all files)
- Produce findings to: `docs/qa/findings/wallet-core-security.md` (create in ECC plugin root)

**Interfaces:**
- Produces: `docs/qa/findings/wallet-core-security.md` — consumed by Task 6

- [ ] **Step 1: Audit Argon2id KDF parameters in kek.js**

Read `src/wallet-core/keystore/kek.js`. Verify:
- `m` (memory) ≥ 65536 (64 MiB minimum per OWASP)
- `t` (iterations) ≥ 3
- `p` (parallelism) ≥ 1
- Output length ≥ 32 bytes

Any parameter below threshold = CRITICAL finding.

- [ ] **Step 2: Audit AES-GCM nonce handling in vault.js**

Read `src/wallet-core/vault.js`. Verify:
- Nonce (IV) is generated fresh via `crypto.getRandomValues` on every encrypt call — never reused, never hardcoded, never derived deterministically from the plaintext
- Nonce length is 12 bytes (96 bits)
- Authentication tag is checked on decrypt (GCM provides this automatically via SubtleCrypto — verify the code does not strip or ignore it)
- Nonce is stored alongside ciphertext (not separate)

Nonce reuse or hardcoded nonce = CRITICAL.

- [ ] **Step 3: Audit mnemonic.js for entropy and validation**

Read `src/wallet-core/mnemonic.js`. Verify:
- Entropy source is `crypto.getRandomValues` (not `Math.random`)
- Entropy size ≥ 128 bits for 12-word, ≥ 256 bits for 24-word
- BIP-39 checksum is verified on import
- Mnemonic is zeroed from memory after use where possible (look for explicit buffer clearing)

`Math.random` usage = CRITICAL. No checksum validation = HIGH.

- [ ] **Step 4: Audit deniability / duress for information leakage**

Read `src/wallet-core/duress.js`, `deniabilityUnlock.js`, `deniabilitySession.js`. Verify:
- Duress PIN and real PIN unlock flows are timing-equivalent (no early return that distinguishes them)
- Error messages are identical for wrong PIN vs wrong duress PIN
- No console.log, debug output, or storage key that reveals which mode is active
- The duress wallet is indistinguishable from the real wallet at the API layer

Timing side-channel or distinguishable error = HIGH.

- [ ] **Step 5: Audit send flows for pre-broadcast validation**

Read `src/wallet-core/evm/send.js`, `btc/send.js`, `sol/send.js`. Verify:
- Recipient address is validated before signing (checksum for EVM, bech32 for BTC, base58 for SOL)
- Amount is checked > 0 before signing
- Gas / fee estimation failure is handled (does not silently send with zero gas)
- Signed transaction is not logged to console or any analytics

Missing address validation before signing = HIGH. Console-logged signed tx = CRITICAL.

- [ ] **Step 6: Audit ring-boundary compliance**

```bash
cd "C:\Users\aljob\Downloads\VEYRNOX-CLONE-ECC"
npm run lint:rings 2>&1
```

Any ring boundary violation = HIGH finding.

- [ ] **Step 7: Check for stray console.log in wallet-core**

```bash
grep -rn "console\.log" src/wallet-core/ --include="*.js" | grep -v "__tests__"
```

Any hit outside of dev-gated blocks = MEDIUM (potential key/data leak).

- [ ] **Step 8: Write findings to docs/qa/findings/wallet-core-security.md**

```markdown
# Wallet-Core Security Audit — QA Findings

## Argon2id KDF Parameters
- m (memory): [value] — [PASS/FAIL — threshold: 65536]
- t (iterations): [value] — [PASS/FAIL — threshold: 3]
- p (parallelism): [value] — [PASS/FAIL]
- Output length: [value] bytes — [PASS/FAIL]

## AES-GCM Nonce Handling
- Fresh nonce per encrypt: yes/no
- Nonce source: crypto.getRandomValues / other
- Nonce length: [N] bytes
- Auth tag checked on decrypt: yes/no

## Mnemonic Entropy
- Entropy source: crypto.getRandomValues / Math.random / other
- Entropy size: [N] bits
- BIP-39 checksum validated on import: yes/no

## Deniability / Duress
- Timing-equivalent unlock paths: yes/no
- Error messages identical: yes/no
- No distinguishing storage keys: yes/no

## Send Flow Validation
- EVM address validated pre-sign: yes/no
- BTC address validated pre-sign: yes/no
- SOL address validated pre-sign: yes/no
- Amount > 0 checked pre-sign: yes/no
- Signed tx not logged: yes/no

## Ring Boundary
- lint:rings result: PASS / FAIL [violations if any]

## Stray console.log
- Count: N [file:line per hit]

## Findings
| Severity | Description | File:line | Recommendation |
|---|---|---|---|

## Summary
- Total findings: N
- CRITICAL: N | HIGH: N | MEDIUM: N | LOW: N
```

- [ ] **Step 9: Fix P0/P1 issues inline (with TDD gate for wallet-core)**

For every CRITICAL or HIGH finding in wallet-core:
1. Write a failing test in the appropriate `__tests__` file
2. Run: `npm test -- --testPathPattern=<file> 2>&1` — verify it FAILS
3. Implement the fix
4. Run tests again — verify PASS
5. Commit:

```bash
git add src/wallet-core/<changed-files> src/wallet-core/**/__tests__/<test-file>
git commit -m "fix(wallet-core): <description of fix>"
```

---

### Task 5: UI / UX / Accessibility — All Pages, Light & Dark Mode, Fonts

**Agent type:** `veyrnox-ui`  
**Working directory:** Veyrnox app root

**Files:**
- Read: `src/App.jsx` (route list), `src/index.css` or equivalent (font config), `tailwind.config.js`
- Produce findings to: `docs/qa/findings/veyrnox-ui.md` (create in ECC plugin root)

**Interfaces:**
- Produces: `docs/qa/findings/veyrnox-ui.md` — consumed by Task 6

- [ ] **Step 1: Start the dev server**

```bash
cd "C:\Users\aljob\Downloads\VEYRNOX-CLONE-ECC"
npm run dev 2>&1 &
# Wait for server ready signal
```

Verify app is reachable at `http://localhost:5173?demo=1`.

- [ ] **Step 2: Enumerate all routes to test**

From `src/App.jsx`, extract the full route list. Priority order for this pass:

**Tier 1 — always test (core wallet UX):**
`/`, `/send`, `/receive`, `/tx-history`, `/receipt`, `/security`, `/security-dashboard`, `/settings`, `/landing`

**Tier 2 — test if time permits:**
`/analytics`, `/network-manager`, `/hardware-wallet`, `/walletconnect`, `/address-book`, `/audit-log`, `/hd-wallet`

**Tier 3 — spot check only:**
`/docs`, `/features`, `/plans`, `/safety-plus`, `/terms-legal`

- [ ] **Step 3: For each Tier 1 route — light mode audit**

Navigate to `http://localhost:5173<route>?demo=1` with viewport 1440×900.

Check per page:
- Page renders without blank/white screen
- No console errors (filter: ignore analytics/third-party)
- Primary heading visible
- CTA / primary action visible and clickable
- Font renders correctly (no system fallback, no invisible text, no FOUT)
- No horizontal overflow / scrollbar at 1440px

Record: PASS / FAIL per check per route.

- [ ] **Step 4: For each Tier 1 route — dark mode audit**

Toggle dark mode (look for theme toggle button, or add `?theme=dark` if supported, or use `prefers-color-scheme: dark` via devtools emulation).

Check per page:
- Background is dark (not white bleed-through)
- All text passes WCAG AA contrast (4.5:1 for normal text, 3:1 for large)
- Icons / illustrations visible (not invisible on dark background)
- Input fields have visible borders/outlines
- Font renders correctly in dark mode
- No unstyled (white) elements bleeding through

Record: PASS / FAIL per check per route.

- [ ] **Step 5: For each Tier 1 route — responsive audit**

Test at 375px (mobile) and 768px (tablet) widths.

Check per breakpoint per route:
- No horizontal overflow
- Navigation accessible (hamburger menu or bottom nav)
- Primary CTA visible above the fold
- Forms usable (inputs not too small to tap)
- Font sizes readable (≥ 14px body, ≥ 16px for inputs to prevent iOS zoom)

Record: PASS / FAIL per check per route per breakpoint.

- [ ] **Step 6: Font audit**

Read font configuration from `tailwind.config.js` and `src/index.css`.

Check:
- All declared font families actually load (check Network tab for 404s on font files)
- Font weight variants used in CSS are actually loaded (e.g. if `font-bold` = weight 700, weight 700 must be in the loaded font)
- No FOUT (flash of unstyled text) on initial load
- Fallback stack is sensible (system-ui or serif fallback, not Comic Sans)
- Variable font used where possible (reduces requests)

Record findings with specifics (which weight missing, which file 404s).

- [ ] **Step 7: Accessibility audit — axe-core on Tier 1 routes**

Run axe-core on each Tier 1 route. Use a Playwright script:

```typescript
// throwaway axe script
import { chromium } from 'playwright';
import AxeBuilder from '@axe-core/playwright';

const routes = ['/', '/send', '/receive', '/tx-history', '/security', '/settings'];
const browser = await chromium.launch({ headless: true });
const violations: any[] = [];

for (const route of routes) {
  const page = await browser.newPage();
  await page.goto(`http://localhost:5173${route}?demo=1`);
  await page.waitForLoadState('networkidle');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag22aa'])
    .analyze();
  for (const v of results.violations) {
    violations.push({ route, id: v.id, impact: v.impact, description: v.description, nodes: v.nodes.length });
  }
  await page.close();
}

await browser.close();
console.log(JSON.stringify(violations, null, 2));
```

Install if needed: `npm install --save-dev @axe-core/playwright`

Record all violations with route, rule ID, impact, and node count.

- [ ] **Step 8: Write findings to docs/qa/findings/veyrnox-ui.md**

```markdown
# Veyrnox UI / UX / Accessibility — QA Findings

## Route Audit — Light Mode (1440px)
| Route | Renders | No Errors | Heading | CTA | Fonts | No Overflow |
|---|---|---|---|---|---|---|
| / | ✓/✗ | ✓/✗ | ✓/✗ | ✓/✗ | ✓/✗ | ✓/✗ |

## Route Audit — Dark Mode (1440px)
| Route | Dark BG | Contrast | Icons | Inputs | Fonts | No Bleed |
|---|---|---|---|---|---|---|

## Responsive Audit
| Route | 375px overflow | 375px nav | 375px CTA | 768px overflow |
|---|---|---|---|---|

## Font Audit
- Declared fonts: [list]
- 404s: [list]
- Missing weights: [list]
- FOUT observed: yes/no
- Fallback stack: [value]

## Accessibility Violations (axe-core, WCAG 2.2 AA)
| Route | Rule | Impact | Nodes | Description |
|---|---|---|---|---|

## Findings
| Severity | Description | Route | Recommendation |
|---|---|---|---|

## Summary
- Total findings: N
- CRITICAL: N | HIGH: N | MEDIUM: N | LOW: N
```

- [ ] **Step 9: Fix P0/P1 UI issues inline**

P0 = Page fails to render (blank screen) or CRITICAL a11y violation (missing form labels, no keyboard access to primary CTA).
P1 = Dark mode text invisible (contrast < 3:1), horizontal overflow on mobile, font 404.

Fix in the relevant component/CSS file, verify in browser, then:

```bash
git add src/
git commit -m "fix(ui): inline QA fixes from Task 5"
```

---

### Task 6: Synthesis — Final Report, Inline Fixes, Backlog

**Agent type:** `act`  
**Depends on:** Tasks 1–5 all complete (all findings files present)  
**Working directory:** ECC plugin root

**Files:**
- Read: `docs/qa/findings/ecc-plugin.md`, `docs/qa/findings/veyrnox-unit.md`, `docs/qa/findings/veyrnox-e2e.md`, `docs/qa/findings/wallet-core-security.md`, `docs/qa/findings/veyrnox-ui.md`
- Create: `docs/qa/2026-07-11-qa-report.md`

**Interfaces:**
- Consumes: all five findings files from Tasks 1–5
- Produces: `docs/qa/2026-07-11-qa-report.md`

- [ ] **Step 1: Collect and deduplicate all findings**

Read all five findings files. Build a unified list. Deduplicate any finding that appears in more than one dimension (e.g., a deniability string leak caught by both unit tests and security audit = one finding).

- [ ] **Step 2: Apply severity → priority mapping**

```
CRITICAL → P0 (block ship)
HIGH     → P0 or P1 (P0 if in crypto surface, P1 otherwise)
MEDIUM   → P1 or P2 (P1 if user-facing, P2 if internal)
LOW      → P2
```

- [ ] **Step 3: Determine overall verdict**

```
Any P0 open (not fixed inline) → DO NOT SHIP
All P0 fixed, P1s remain      → SHIP WITH FIXES
All P0 and P1 fixed           → SHIP
No baseline (no visual comparison) → append INCONCLUSIVE to visual section
```

- [ ] **Step 4: Write the final report**

Create `docs/qa/2026-07-11-qa-report.md`:

```markdown
# Veyrnox Extensive QA Report — 2026-07-11

## Executive Summary

**Verdict:** [SHIP / SHIP WITH FIXES / DO NOT SHIP]

| Dimension | Findings | Fixed inline | Backlog |
|---|---|---|---|
| ECC Plugin | N | N | N |
| Veyrnox Unit | N | N | N |
| Wallet-Core Security | N | N | N |
| E2E Web Flows | N | N | N |
| UI / UX / A11y | N | N | N |
| **Total** | **N** | **N** | **N** |

---

## ECC Plugin

[Paste summary section from docs/qa/findings/ecc-plugin.md]

---

## Veyrnox Unit & Integration

[Paste summary + coverage baseline table from docs/qa/findings/veyrnox-unit.md]

---

## Wallet-Core Security Audit

[Paste full findings table from docs/qa/findings/wallet-core-security.md]

---

## E2E Web Flows

[Paste summary from docs/qa/findings/veyrnox-e2e.md]

---

## UI / UX / Accessibility

[Paste route audit tables + a11y violations from docs/qa/findings/veyrnox-ui.md]

---

## Backlog (Prioritised)

### P0 — Block Ship
[Any P0 not fixed inline]

### P1 — Fix Next Sprint
[All P1 findings]

### P2 — Nice to Have
[All P2 findings, including mobile E2E setup, Ledger/Trezor flows]
```

- [ ] **Step 5: Commit the final report**

```bash
git add docs/qa/
git commit -m "docs(qa): add 2026-07-11 extensive QA report"
```

- [ ] **Step 6: Final summary to user**

Print to terminal:

```
═══════════════════════════════════════
  VEYRNOX QA COMPLETE — 2026-07-11
═══════════════════════════════════════
Verdict:        [SHIP / SHIP WITH FIXES / DO NOT SHIP]
Total findings: N
  Fixed inline: N
  Backlog P0:   N
  Backlog P1:   N
  Backlog P2:   N

Report: docs/qa/2026-07-11-qa-report.md
═══════════════════════════════════════
```

---

## Execution Order

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Task 1    │  │   Task 2    │  │   Task 3    │  │   Task 4    │  │   Task 5    │
│ ECC Plugin  │  │Veyrnox Unit │  │  E2E Web    │  │Wallet-Core  │  │  UI / A11y  │
│  Integrity  │  │  Coverage   │  │   Flows     │  │  Security   │  │  Fonts DM   │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │                │
       └────────────────┴────────────────┴────────────────┴────────────────┘
                                         │
                                         ▼
                                  ┌─────────────┐
                                  │   Task 6    │
                                  │  Synthesis  │
                                  │   Report    │
                                  └─────────────┘
```

Tasks 1–5 dispatch simultaneously. Task 6 starts only after all five are complete.
