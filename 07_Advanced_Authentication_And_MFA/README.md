# Module 07: Advanced Authentication & MFA

[Module 06](../06_Machine_Identity_And_Workload_Auth/README.md) covered how *machines* prove their identity without passwords. This module is the mirror image: how *humans* prove theirs in an era where passwords are already leaked, 6-digit codes can be relayed by a live phishing proxy, and a stolen session cookie bypasses the login entirely.

It walks from the basics of multi-factor authentication, through phishing-resistant WebAuthn/FIDO2, to step-up authentication that re-verifies a user at the moment a dangerous action is attempted.

🏦 The running examples follow MoneyGuard's people: **Priya Sharma** (teller, whose password is already in a breach dump), **Raj Patel** (IAM administrator, the bank's most valuable human account), **Sam Torres** (contractor on a personal device), and **Eve**, who runs a real-time reverse proxy on `moneyguerd.com`.

## 📚 Chapters

| # | Chapter | What it covers |
|---|---------|----------------|
| 01 | [MFA Fundamentals](./01_MFA_Fundamentals.md) | The three factor categories, how TOTP and push actually work, MFA fatigue and number matching, adaptive/risk-based authentication, recovery design, and the `acr`/`amr`/`auth_time` claims |
| 02 | [Phishing-Resistant MFA: WebAuthn & FIDO2](./02_Phishing_Resistant_MFA_WebAuthn_FIDO2.md) | Adversary-in-the-Middle phishing step by step, WebAuthn registration and assertion, origin binding and RP IDs, signature counters, and FIDO2 vs WebAuthn vs passkeys |
| 03 | [Step-Up Authentication](./03_StepUp_Authentication.md) | Protecting actions rather than logins: `acr` strength plus `auth_time` freshness, the RFC 9470 `insufficient_user_authentication` challenge, `max_age` and `prompt=login`, and transaction binding (PSD2 dynamic linking) |

Read them in order — each chapter answers the weakness the previous one exposes:

```
Passwords          →  already leaked and reused
  ↓ Chapter 01
MFA (TOTP / push)  →  stops bulk attacks, but a live proxy relays the code
  ↓ Chapter 02
WebAuthn / FIDO2   →  unphishable login, but the session afterwards is still a bearer credential
  ↓ Chapter 03
Step-up auth       →  re-verify strength AND freshness at the moment of the risky action
```

## 🎯 What You'll Be Able to Do

- Rank authentication factors correctly and explain *why* SMS, TOTP, and plain push are all phishable.
- Explain an Adversary-in-the-Middle attack and the exact cryptographic reason WebAuthn stops it.
- Design a phishing-resistant rollout whose recovery path isn't weaker than the credential it replaces.
- Enforce step-up on the resource server using `acr` and `auth_time`, with transaction binding for payments.

## 📋 Prerequisites

This module assumes the AAA framework — Authentication, Authorization, and Accounting — and the basics of identity lifecycle. If those are unfamiliar, read **[Module 01 — Core IAM Concepts](../01_Core_IAM_Concepts/README.md)** first, particularly [`02_AAA_Framework.md`](../01_Core_IAM_Concepts/02_AAA_Framework.md).

Chapter 03 also assumes you can read a JWT and understand OAuth authorization requests. If you need that grounding, see **[Module 02 — AuthN & Federation](../02_AuthN_Federation/README.md)**, especially [`04_OIDC.md`](../02_AuthN_Federation/04_OIDC.md).

## 🔗 Related Modules

- **[Module 06 — Machine Identity & Workload Auth](../06_Machine_Identity_And_Workload_Auth/README.md)** — the machine-side counterpart. Note the symmetry: WebAuthn binds a human credential to an *origin*, while mTLS binds a machine credential to a *host*. Both defeat credential theft the same way.
- **[Module 04 — Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md)** — session lifetime and revocation, the gap step-up exists to cover.

---
⬅ Back to [Main README](../README.md) | 🏦 Meet the cast in [SCENARIO.md](../SCENARIO.md)
