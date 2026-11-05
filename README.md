# 🔐 Identity & Access Management (IAM) — The Complete Field Guide

## https://identiflows.dev/ - A Visual Reference for Identity Flows

A from-scratch, beginner-friendly course on Identity & Access Management — the concepts, protocols, and system-design tradeoffs a Software/Staff Engineer is expected to know. It's written as personal study notes and reference material, organized so it's still useful when you come back to it in six months.

Every chapter teaches its concept through **one running example** instead of a different toy company each time. Read **[`SCENARIO.md`](./SCENARIO.md)** first — it introduces **MoneyGuard Bank**, the cast of characters (Priya, Raj, Dana, Alex, Sam, and the attacker Eve), and the systems they use. Once you know that cast, every chapter in every module speaks the same language.

## 📁 How This Repo Is Organized

The course is split into **11 numbered modules**, each its own folder with a `README.md` index and a set of numbered chapters (`01_...md`, `02_...md`, ...). Modules build on each other roughly in order — Module 01 assumes nothing, Module 11 assumes you've read everything before it — but each module's `README.md` also says what it builds on and what it feeds into, so you can jump around.

Every standard chapter (Modules 01–10) follows the same template: **Overview → Why This Matters → Key Concepts → Detailed Notes → Workflow → Architecture Diagram → Step-by-Step Process → Commands and Examples → Best Practices → Common Mistakes → Pro Tips → Real-World Use Cases → Key Takeaways → Glossary → Revision Notes → Interview Questions → Practice Exercises**. Module 11 is the capstone and deliberately breaks that mold — it's a whiteboard framework, a full architecture walkthrough, and a bank of interview Q&A and exercises.



## 📖 Table of Contents

| # | Module | What You'll Learn |
|---|--------|--------------------|
| 01 | [Core IAM Concepts](./01_Core_IAM_Concepts/README.md) | What IAM is, the AAA framework, access control models at a glance, least privilege, the Joiner-Mover-Leaver lifecycle, identity repositories, service accounts |
| 02 | [AuthN & Federation](./02_AuthN_Federation/README.md) | Zero Trust architecture, SAML 2.0, OAuth 2.0, OpenID Connect, and the `state`/`nonce`/PKCE parameters that keep those flows safe |
| 03 | [Authorization & Policy Engines](./03_Authorization_And_Policy_Engines/README.md) | RBAC and ABAC in depth, ReBAC and Google Zanzibar, policy engines (OPA) and PBAC |
| 04 | [Edge Security, Tokens & Sessions](./04_Edge_Security_Tokens_And_Sessions/README.md) | JWT vs opaque tokens, JWKS and signing key rotation, token revocation and session management |
| 05 | [Identity Lifecycle & SCIM](./05_Identity_Lifecycle_And_SCIM/README.md) | JIT provisioning vs SCIM, the SCIM protocol in depth, handling directory-sync race conditions |
| 06 | [Machine Identity & Workload Auth](./06_Machine_Identity_And_Workload_Auth/README.md) | Service accounts and API keys, the OAuth Client Credentials grant, workload identity federation (SPIFFE/IRSA), mutual TLS |
| 07 | [Advanced Authentication & MFA](./07_Advanced_Authentication_And_MFA/README.md) | MFA fundamentals, phishing-resistant MFA (WebAuthn/FIDO2/passkeys), step-up authentication |
| 08 | [Multi-Tenancy & Data Isolation](./08_MultiTenancy_And_Data_Isolation/README.md) | Tenancy models (siloed/pool/bridge), database schema design and PostgreSQL Row-Level Security |
| 09 | [Resilience, Observability & SIEM](./09_Resilience_Observability_And_SIEM/README.md) | CAP theorem and fail-open vs fail-closed design, caching and invalidation, audit logging, SIEM and threat detection |
| 10 | [IaC & Deployment Security](./10_IaC_And_Deployment_Security/README.md) | Infrastructure-as-Code for identity (Terraform/OpenTofu, GitOps), secrets management (Vault) |
| 11 | [System Design & Interview Prep](./11_System_Design_And_Interview_Prep/README.md) | A whiteboard framework, MoneyGuard's full architecture walked end-to-end, 36 interview Q&A, and 12 practice exercises |

## 🧭 Suggested Reading Paths

- **Learning IAM from scratch:** read Modules 01 → 11 in order. Each module's README tells you what it assumes and what comes next.
- **Studying for a system-design or Staff/Senior IAM interview:** skim Modules 01–02 for vocabulary, then read Modules 03, 06, 08, and 09 for the topics interviewers probe hardest, then finish with Module 11.
- **Looking up one thing:** go straight to the module README that owns the topic — every module README lists its chapters and a one-line summary of each.

## 🏷️ Conventions Used Throughout

- 🏦 marks a walkthrough using the MoneyGuard Bank scenario.
- ⚠️ marks a common mistake or security pitfall.
- 💡 marks a pro tip.
- All diagrams are [Mermaid](https://mermaid.js.org/) and render directly on GitHub — no extra tooling needed to view them.

---
*Start with [`SCENARIO.md`](./SCENARIO.md), then [Module 01: Core IAM Concepts](./01_Core_IAM_Concepts/README.md).*
