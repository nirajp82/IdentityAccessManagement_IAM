# 🏦 The MoneyGuard Bank Story (Read This First)

Every chapter in this repository teaches its concept using **one running example**: a fictional global retail and commercial bank called **MoneyGuard Bank**. Using the same company, people, and systems everywhere means once you learn a name here, it means the same thing in every other chapter — no re-learning a new fictional startup for every topic.

If a chapter uses a name, system, or acronym you don't recognize, it's probably defined here.

## The Company

**MoneyGuard Bank** is a mid-size global bank with:
- ~50,000 employees (tellers, loan officers, IT staff, auditors)
- Millions of retail customers using web + mobile banking
- A growing **Business Banking SaaS Portal** used by corporate clients (small businesses that bank with MoneyGuard) to manage payroll and invoicing
- A mix of **old on-premise systems** (30 years of legacy) and **new cloud systems** (actively being modernized) — this mix is exactly why IAM is hard in the real world, and exactly why MoneyGuard is a useful teaching example.

## The Cast of Characters

| Name | Role | Used to illustrate... |
|---|---|---|
| **Priya Sharma** | New bank teller | Onboarding ("Joiner"), everyday employee access |
| **Raj Patel** | Identity & Access (IAM) Administrator | How access is designed, granted, and audited |
| **Dana Cole** | Security Auditor / Compliance Officer | Audit trails, least privilege, compliance (SOX/GDPR) |
| **Alex Kim** | Backend Software Engineer | Building login flows, APIs, service-to-service auth |
| **Sam Torres** | Short-term Contractor | Temporary access, least privilege, just-in-time access |
| **Eve** | External Attacker (industry-standard placeholder name for a bad actor) | Threats: credential stuffing, token theft, phishing |

## The Systems

| System | What it is | Used in |
|---|---|---|
| **Workday** | HR system of record — the trigger for hiring, role changes, and terminations | Module 01, 05 |
| **MoneyGuard Active Directory (AD)** | Legacy on-premise identity directory (LDAP/Kerberos) for internal, older apps | Module 01 |
| **MoneyGuard Entra ID** (Azure AD) | Modern cloud identity provider (IdP) — the "front door" for SSO | Module 01, 02 |
| **MoneyGuard Employee Portal** | Internal web app employees log into (an OIDC Relying Party / SAML Service Provider) | Module 02, 03, 04 |
| **MoneyGuard Mobile App** | Customer-facing mobile banking app | Module 02, 04, 07 |
| **Core Banking Ledger** | The most sensitive backend system — the "crown jewels" (account balances, transactions) | Module 03, 06, 08 |
| **Business Banking SaaS Portal** | Multi-tenant web app used by corporate customers (e.g. **Acme Corp** and **Globex Inc**, two of MoneyGuard's business clients) | Module 08 |
| **Fraud Detection Pipeline** | A Kubernetes-based ML system that scores transactions for fraud in near real-time — MoneyGuard's example of machine-to-machine, non-human identity | Module 06 |
| **API Gateway** | The single front door for all API traffic; validates tokens, rate-limits, enforces policy at the edge | Module 03, 04, 09 |
| **MoneyGuard SOC / SIEM ("Sentinel")** | Security Operations Center tooling that watches logs for suspicious activity | Module 09 |
| **PartnerBank** | An external partner bank that federates into MoneyGuard via SAML for a joint mortgage product | Module 02 |

## Why a Bank?

Banking is a great teaching domain for IAM because:
- **High stakes** — a mistake means real financial loss or regulatory fines, so every access decision has a clear "why."
- **Real compliance drivers** — SOX, GDPR, and PCI-DSS give concrete, realistic reasons for audit trails and least privilege.
- **Everyone understands the vocabulary** — tellers, branches, accounts, and loans are familiar, so we can spend our energy on the *IAM* concept, not on explaining the business domain.

---
⬅ Back to [Main README](./README.md)
