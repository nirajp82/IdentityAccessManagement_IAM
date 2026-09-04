# Module 11: System Design & Interview Prep

> 🎓 **The final exam for this repository.** Modules 01–10 taught the pieces. This module assembles them into one architecture, one interview technique, and one set of scenarios to test yourself against.

## 📋 Overview

Every earlier module taught a slice of identity in isolation — a login flow in one place, a provisioning protocol in another, database isolation somewhere else. That is the right way to *learn* the material. It is not how the material is ever *used*.

Real IAM failures are almost never "we implemented OAuth wrong." They are **seam failures**: the identity provider disabled the account but the gateway honoured the old token for another 55 minutes; the application filtered by tenant but the nightly report didn't; everything worked but nothing was logged, so nobody could prove it to the auditor.

Seams only become visible when you draw the whole system. That is what this module is for.

🏦 It uses the same running example as everything else — **MoneyGuard Bank**, its people (Priya, Raj, Dana, Alex, Sam, and Eve), and its systems (Workday, Entra ID, the API Gateway, the Core Banking Ledger, the Business Banking SaaS Portal, the Fraud Detection Pipeline, and Sentinel). If a name is unfamiliar, it is defined in [`SCENARIO.md`](../SCENARIO.md).

## ⚠️ Prerequisites

**This module assumes you have read modules 01–10.** It deliberately does not re-teach OIDC, SCIM, or Row-Level Security — it shows you where they *sit* and how they connect. Acronyms are still expanded on first use in each chapter for standalone readability, but the depth lives in the earlier modules.

If you are short on time, the minimum viable path before starting here is modules [01](../01_Core_IAM_Concepts/README.md) → [02](../02_AuthN_Federation/README.md) → [03](../03_Authorization_And_Policy_Engines/README.md) → [04](../04_Edge_Security_Tokens_And_Sessions/README.md), plus a skim of [08](../08_MultiTenancy_And_Data_Isolation/README.md) and [09](../09_Resilience_Observability_And_SIEM/README.md).

## 📚 Chapters

| # | Chapter | What It Covers |
|---|---------|----------------|
| 01 | [The Whiteboard Framework](./01_The_Whiteboard_Framework.md) | The 4-step system design interview method — Clarify → High-Level Components → Data Flow → Deep Dives & Trade-offs — with a clock budget, the seven-zone decomposition that works on any IAM prompt, and a worked mock: *"Design MoneyGuard's IAM system."* |
| 02 | [MoneyGuard Full Architecture Walkthrough](./02_MoneyGuard_Full_Architecture_Walkthrough.md) | 🏆 **The capstone.** One large Mermaid diagram of MoneyGuard's complete identity architecture, a zone-by-zone written narration, two requests traced end-to-end, the seven seams where real systems break, and a table mapping every component back to the module that teaches it. |
| 03 | [Interview Questions & Answers](./03_Interview_Questions_And_Answers.md) | 36 realistic IAM and security system-design questions with model answers, grouped by topic — fundamentals, federation, authorization, tokens, lifecycle, machine identity, MFA, multi-tenancy, and resilience — each with a note on what the question is really testing. |
| 04 | [Practice Exercises](./04_Practice_Exercises.md) | 12 whiteboard, diagnostic, and hands-on scenarios with hints and grading checklists — no full solutions, because recall is what interviews measure. Ends with a full 45-minute mock. |

## 🧭 Suggested Order

| If you are… | Read |
|---|---|
| **Preparing for an interview in a week** | 01 → 02 → 03 → 04, in order. Attempt exercise 12 cold on day one *and* again at the end. |
| **Consolidating what you learned in 01–10** | Start with 02 — the architecture walkthrough is the synthesis. Then 03 as a self-quiz. |
| **Interviewing tomorrow** | The cheat-sheets at the end of 01 and 02, then the rapid-fire table at the end of 03. |
| **Already comfortable, want to find gaps** | Go straight to 04. The exercises are diagnostic — whatever you stumble on tells you which module to revisit. |

## 🔑 The Ideas That Tie Everything Together

If you remember nothing else from eleven modules, remember these.

| # | Idea | Why it matters |
|---|---|---|
| 1 | **AuthN happens once per session; AuthZ happens on every request.** | Completely different scaling profiles. This one sentence organises every IAM architecture. |
| 2 | **Identity flows outward from the system of record.** | Workday → SCIM → IdP → tokens → edge → authorization → data. If accounts can be created outside that chain, no downstream control is complete. |
| 3 | **`externalId` is the anchor. Never join identities on email.** | Emails change; identifiers must not. This one decision prevents an entire genus of bug. |
| 4 | **Revocation is a two-part design.** | Short token lifetimes *plus* a distributed kill switch. Neither alone meets a 60-second requirement. |
| 5 | **Many enforcement points, one decision point.** | Many PEPs, one PDP — so permissions change in real time and there is a single place to audit. |
| 6 | **Defence in depth is literal.** | Gateway, then authorization service, then Row-Level Security. Each layer assumes the ones above it have a bug. |
| 7 | **Fail closed on anything touching money — and return 503, not 403.** | The status code is a diagnostic signal. Getting it wrong costs real incident minutes. |
| 8 | **Machines are identities too.** | With an owner, a lifecycle, least privilege, and no static secrets. In a modern bank they outnumber humans. |
| 9 | **The audit log is a control, not a byproduct.** | Structured, immutable, correlated — and it must record successes, because breaches are made of authorised actions. |
| 10 | **Least privilege is a loop, not a config.** | Grant conservatively, observe usage, tighten, recertify. |

## 📖 The Full Repository Map

Where each part of the capstone architecture is taught:

| Module | Contributes to the architecture |
|---|---|
| [01 · Core IAM Concepts](../01_Core_IAM_Concepts/README.md) | Workday as system of record, the AAA framework, Joiner-Mover-Leaver, least privilege, directories, service accounts |
| [02 · AuthN & Federation](../02_AuthN_Federation/README.md) | Entra ID, OIDC + PKCE, SAML federation with PartnerBank, Zero Trust, `state`/`nonce`/PKCE |
| [03 · Authorization & Policy Engines](../03_Authorization_And_Policy_Engines/README.md) | The AuthZ service (PDP), RBAC/ABAC/ReBAC, OPA policy bundles |
| [04 · Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md) | The API Gateway, JWT vs opaque tokens, JWKS and key rotation, the revocation list |
| [05 · Identity Lifecycle & SCIM](../05_Identity_Lifecycle_And_SCIM/README.md) | The SCIM service, JIT fallback, `externalId` anchoring, reconciliation |
| [06 · Machine Identity & Workload Auth](../06_Machine_Identity_And_Workload_Auth/README.md) | The Fraud Detection Pipeline's identity, client credentials, SPIFFE/IRSA, mTLS |
| [07 · Advanced Authentication & MFA](../07_Advanced_Authentication_And_MFA/README.md) | The MFA engine, passkeys/FIDO2, step-up and the `acr` claim, risk signals |
| [08 · Multi-Tenancy & Data Isolation](../08_MultiTenancy_And_Data_Isolation/README.md) | The Business Banking SaaS Portal, silo/pool/bridge, `tenant_id` schema, Row-Level Security |
| [09 · Resilience, Observability & SIEM](../09_Resilience_Observability_And_SIEM/README.md) | Caching and invalidation, fail-open vs fail-closed, CAP trade-offs, audit logging, Sentinel |
| [10 · IaC & Deployment Security](../10_IaC_And_Deployment_Security/README.md) | Identity configuration in Git, CI/CD approval gates, secrets management |
| **11 · This module** | **How it all fits together, and how to explain it in 45 minutes** |

---

⬅ Back to [Main README](../README.md) | 🏦 [The MoneyGuard Scenario](../SCENARIO.md) | ⬅ Previous module: [10 · IaC & Deployment Security](../10_IaC_And_Deployment_Security/README.md)
