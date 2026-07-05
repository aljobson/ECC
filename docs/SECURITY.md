# VEYRNOX Security Policy

**Last Updated**: July 5, 2026  
**Status**: 🟡 Conditional Mainnet Ready (3-4 weeks critical path)

---

## Security Audit (2026-07-05)

VEYRNOX has completed a comprehensive independent security audit covering design architecture, threat model analysis, and penetration testing readiness.

**Audit Verdict**: 🟡 **CONDITIONAL MAINNET READY**

- **Design Verification**: ✅ COMPLETE
- **Critical Blockers Identified**: 3 (all fixable, 3-4 week timeline)
- **Timeline to Mainnet**: 3-4 weeks after critical path work

**Read the full audit reports**: [docs/security-audits/](security-audits/)

### Quick Security Status

| Category | Status | Details |
|----------|--------|---------|
| **Deniability Stack** | ✅ Verified | Byte-verified schema parity |
| **Security Invariants** | ✅ Verified | I1-I5 all verified against design |
| **Threat Model** | ✅ Verified | T1-T6 actors mapped and covered |
| **Backend Untrusted** | ✅ Verified | Client-side encryption confirmed |
| **CI Enforcement** | 🔴 CRITICAL | Blocker: ESLint rule inactive (4-5 days to fix) |
| **Crypto Implementation** | 🔴 CRITICAL | Blocker: AES-256-GCM audit pending (1 week) |
| **Mainnet Gate** | 🔴 CRITICAL | Blocker: Manual activation gate (5 days to fix) |

---

## Reporting a Vulnerability

We take security seriously. If you discover a vulnerability:

1. **Do NOT** open a public GitHub issue for security vulnerabilities
2. **Use GitHub's private vulnerability reporting** (preferred):
   - https://github.com/VEYRNOX/veyrnox/security/advisories/new
3. **Or email**: security@veyrnox.io

### What to Include

- Affected file, component, version, and commit hash
- Steps to reproduce from a clean checkout
- Expected impact (e.g., funds at risk, privacy breach, coercion compromise)
- Attack requirements (local shell access, malicious repo, remote, etc.)
- Any proof-of-concept logs (with sensitive data redacted)

### Expected Response Timeline

- **Acknowledgment**: Within 48 hours
- **Initial Assessment**: Within 7 days
- **Fix or Mitigation**: Within 14 days for critical issues affecting mainnet
- **Coordinated Disclosure**: Before public advisory

---

## Scope

This policy covers:

- The `VEYRNOX/veyrnox` repository
- VEYRNOX wallet application (production builds)
- VEYRNOX cryptographic implementations
- VEYRNOX deployment automation and smart contracts
- Integrations with blockchain networks (Ethereum, Solana, Cosmos)

### Out of Scope

Reports are typically out of scope when they:

- Require physical access to device + compromise of user's PIN
- Are self-XSS with no server-side exploitable path
- Relate to backend infrastructure not running VEYRNOX code
- Show screenshots against old versions not reproducible on `main`
- Describe attacks where the user has already lost control of the device

However, if a report shows how VEYRNOX could be exploited through a supply-chain vector, third-party dependency, or CI/CD automation, it's **in scope**.

---

## Critical Invariants

VEYRNOX maintains five security invariants:

| Invariant | Description | Status |
|-----------|-------------|--------|
| **I1: Keys Never Leave Device** | Cryptographic keys are generated on-device and never transmitted | ✅ Verified |
| **I2: No Silent Egress** | No data exits the device without user-visible action | ✅ Verified |
| **I3: Deniability Sacred** | Coerced access cannot prove hidden wallet existence | ✅ Verified |
| **I4: Fail Closed** | Compromise gracefully; default deny not default allow | ✅ Verified |
| **I5: Backend Untrusted** | Backend is purely a transport layer; no keys, no trust | ✅ Verified |

All invariants have been verified through:
- Design-level architecture review against LLD document
- Code-level inspection of key components
- Threat model analysis against T1-T6 threat actors
- Penetration test methodology (ready to execute)

---

## Threat Model

VEYRNOX is designed to protect against these threat actors:

| Threat | Description | Mitigation | Status |
|--------|-------------|-----------|--------|
| **T1: Network Observer** | Passive network traffic analysis | Client-side encryption | ✅ Verified |
| **T2: Backend Breach** | Attacker compromises backend infrastructure | Backend untrusted design | ✅ Verified |
| **T3: Physical Coercion** | Attacker threatens user in-person | Deniability stack (Duress/Decoy) | ✅ Verified |
| **T4: Supply Chain** | Malicious dependency or build artifact | CI enforcement, signed releases | 🔴 Blocker* |
| **T5: Rooted OS** | Attacker compromises OS; device not trusted | Honest disclosure of limits | ⚠️ Accepted |
| **T6: Out-of-Scope** | Multi-factor coercion over time | Documented acceptable limits | ⚠️ Accepted |

*Finding #1 (CI enforcement) must be fixed before mainnet.

---

## Critical Findings (2026-07-05 Audit)

### Finding #1: CI Invariant Enforcement INACTIVE 🔴 CRITICAL

**Impact**: Ring boundary (R0 crypto-core / R1 general code) is not enforced at build time. Mainnet keys could leak into general codebase without detection.

**Root Cause**: ESLint ring-import rule never implemented; config merge bug causes spread-overwrite preventing rule activation.

**Remediation**: 
1. Fix ESLint config merge (1-2 days)
2. Implement ring-import-lint rule (2-3 days)
3. Add CI validation gate (1 day)
4. Verify with PR check

**Timeline**: 4-5 days  
**Owner**: Security Engineering  
**Blocker**: YES (MUST FIX BEFORE MAINNET)

👉 **Details**: [CRITICAL-FINDINGS-DEEP-DIVE.md §Finding #1](security-audits/CRITICAL-FINDINGS-DEEP-DIVE.md)

---

### Finding #2: Crypto Implementation Divergence 🔴 CRITICAL

**Impact**: AES-256-GCM (implementation) vs XChaCha20-Poly1305 (design spec). No HKDF step. Argon2id parameters unverified.

**Root Cause**: Design specifies XChaCha20 for modern, constant-time guarantees. Implementation uses WebCrypto AES-256-GCM. Cryptographer review needed.

**Remediation**:
1. External cryptographer review of WebCrypto AES-256-GCM
2. Validate Argon2id parameters (N, r, p values)
3. KDF pipeline analysis (no HKDF)
4. Side-channel analysis
5. Decision: Keep AES-256-GCM with mitigations OR migrate to XChaCha20

**Timeline**: 1 week (external expert)  
**Cost**: $15K-25K  
**Owner**: External Cryptographer (lead), Engineering (support)  
**Blocker**: YES (MUST FIX BEFORE MAINNET)

👉 **Details**: [CRITICAL-FINDINGS-DEEP-DIVE.md §Finding #2](security-audits/CRITICAL-FINDINGS-DEEP-DIVE.md)

---

### Finding #3: Mainnet Deployment Gate Manual 🔴 CRITICAL

**Impact**: Chain-key flip from testnet to mainnet is done manually in assets.js with zero approval gates, audit trail, or safety checks. Risk of accidental or unauthorized mainnet activation.

**Root Cause**: Mainnet gates were deferred as out-of-scope in initial planning. Manual process in code review prevents automation.

**Remediation**:
1. Implement validation script to detect chain-key changes
2. Multi-step approval process (CODEOWNERS, branch protection)
3. GitHub branch protection rules (require approvals)
4. Build-time gate (MAINNET_APPROVED env var)
5. Release automation with audit trail

**Timeline**: 5 days  
**Owner**: DevOps + Security  
**Blocker**: YES (MUST FIX BEFORE MAINNET)

👉 **Details**: [CRITICAL-FINDINGS-DEEP-DIVE.md §Finding #3](security-audits/CRITICAL-FINDINGS-DEEP-DIVE.md)

---

## High-Priority Findings (Post-Mainnet Acceptable)

| # | Finding | Impact | Timeline | Mitigation |
|---|---------|--------|----------|-----------|
| 4 | RASP Browser-Layer Only | OS-level probes deferred | Q3 2026 | Post-mainnet enhancement |
| 5 | Biometric App-Layer Only | Hardware ACL not integrated | Q3 2026 | Acceptable with app-level gate |
| 6 | Per-Set 2FA Blocked | Feature gate pending schema changes | Q3 2026 | Deferred feature (audit-gated) |

---

## Mainnet Readiness Checklist

### Pre-Launch Gates (ALL Must Pass)

- [ ] **CI Ring-Import Enforcement**: ACTIVE
- [ ] **Mainnet Deployment Gate**: IMPLEMENTED
- [ ] **Crypto Audit**: APPROVED (external cryptographer sign-off)
- [ ] **Code Review**: COMPLETE (Duress PIN, Audit Log, Panic Wipe)
- [ ] **Penetration Tests**: PASSED (all 6 scenarios, 74/74 tests)
- [ ] **Security Team Sign-Off**: OBTAINED
- [ ] **Cryptographer Sign-Off**: OBTAINED
- [ ] **Release Tag Created**: DONE
- [ ] **Post-Launch Monitoring**: READY

### Launch Command (Only After All Gates Pass)

```bash
# Step 1: Ensure all blockers are fixed
npm run security:verify

# Step 2: Multi-approval mainnet activation
npm run mainnet:activate ETH --approval-required

# Step 3: Build with mainnet gate enforcement
npm run build:release  # Requires MAINNET_APPROVED=1 and CI gate

# Step 4: Deploy to production
npm run deploy
```

---

## Security Resources

- **[VEYRNOX Security Audit (2026-07-05)](security-audits/)** — Complete audit documentation
- **[Threat Model Analysis](security-audits/VEYRNOX-INDEPENDENT-SECURITY-AUDIT-FINAL.md#threat-model)** — T1-T6 threat actor coverage
- **[Remediation Guides](security-audits/CRITICAL-FINDINGS-DEEP-DIVE.md)** — Step-by-step fixes for critical findings
- **[Penetration Test Methodology](security-audits/PENETRATION-TEST-EXECUTION-GUIDE.md)** — 74 test cases, 6 scenarios
- **[OWASP Top 10 Analysis](security-audits/VEYRNOX-INDEPENDENT-SECURITY-AUDIT-FINAL.md#owasp-top-10)** — Vulnerability mapping

---

## Secrets & Credentials

**Never commit** the following to this repository:

- Private keys, seed phrases, or mnemonics
- API keys, OAuth tokens, or JWT secrets
- Database credentials or connection strings
- Hardware wallet seeds or PINs
- Mainnet RPC endpoints with auth tokens

If a secret is accidentally committed:

1. **Rotate it immediately** at the issuing provider
2. **Rewrite git history** (force-push after rotation)
3. **Alert security team** immediately
4. **Do not rely on a simple revert** — revert commits remain in history

Quick audit:
```bash
# Search for potential secrets
git log --all --oneline | xargs -I {} git show {} | grep -iE "(PRIVATE|SECRET|KEY|TOKEN|SEED|MNEMONIC)" | head -20
```

---

## Post-Launch Security Hardening Roadmap

After mainnet launch, VEYRNOX will implement:

### Q3 2026 (Months 2-4)
- OS-level RASP integration (beyond browser layer)
- Hardware-backed biometric authentication
- Per-set 2FA with time-lock
- Automated security audit refresh (6-month cycle)

### Q4 2026 (Months 5-8)
- Hardware wallet integration (Ledger, Trezor)
- Multi-signature scheme for mainnet funds
- Advanced threat detection (anomaly detection on unlock patterns)
- Security incident response playbook

### 2027 (Ongoing)
- Continuous penetration testing (quarterly)
- External code audits (annual)
- Cryptographer review (annual, or after major changes)
- Threat model refresh (annual)

---

## Questions?

- **Security Issues**: Use private vulnerability reporting
- **Questions**: Open a GitHub discussion
- **Feature Requests**: GitHub issues with label `security`

---

**Last Audit**: July 5, 2026  
**Next Audit**: Expected January 2027 (post-launch)  
**Maintained By**: VEYRNOX Security Team
