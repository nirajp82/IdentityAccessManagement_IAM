# Module 03: Authorization & Policy Engines

Module 02 got the user through the front door and proved who they are. This module answers the harder question that follows on **every single request**: *what are they actually allowed to do?* We walk the full evolution — ACLs to RBAC to ABAC to ReBAC to a decoupled policy engine — and show exactly where each model breaks and what replaces it.

Everything here uses the same running example: **MoneyGuard Bank**, where Raj Patel designs access, Priya Sharma and Dana Cole use it, Sam Torres needs temporary access, and Eve keeps trying to get in. Meet them in [`SCENARIO.md`](../SCENARIO.md).

## 📚 Chapters

| # | Chapter | What it covers |
|---|---|---|
| 01 | [RBAC and ABAC Deep Dive](./01_RBAC_And_ABAC_Deep_Dive.md) | Why ACLs collapse, how roles fix them, what **role explosion** is, the four pillars of ABAC context (subject, action, resource, environment), how to layer RBAC as a cheap gate in front of expensive ABAC, and why fine-grained permissions must never live inside a JWT. |
| 02 | [ReBAC and Google Zanzibar](./02_ReBAC_And_Google_Zanzibar.md) | Relationship-Based Access Control: tuples (`object#relation@subject`), schema definitions, permission math, inheritance through nested folders, graph walks versus recursive SQL, SpiceDB, instant global revocation, zookies and the New Enemy Problem. |
| 03 | [Policy Engines, OPA and PBAC](./03_Policy_Engines_OPA_And_PBAC.md) | Externalising rules with **PEP / PDP / PIP / PAP**, writing Rego for Open Policy Agent, sidecar deployment and hot-path latency, caching with tenant-safe keys, hard versus soft revocation over Redis Pub/Sub, fail-closed design, multi-tenant boundary checks, and async audit logging. |

Read them in order — Chapter 3 assumes the vocabulary from Chapters 1 and 2.

## 🔗 How This Builds on Earlier Modules

- **[Module 01 — Core IAM Concepts](../01_Core_IAM_Concepts/README.md)** introduced the AAA framework and the basics of access control models in [`03_Access_Control_Models_Overview.md`](../01_Core_IAM_Concepts/03_Access_Control_Models_Overview.md), plus [least privilege](../01_Core_IAM_Concepts/04_Principle_Of_Least_Privilege.md). This module takes those same models and pushes them to production scale, adding the failure modes, the performance engineering, and the architecture.
- **[Module 02 — AuthN & Federation](../02_AuthN_Federation/README.md)** produced the token. Everything here consumes it: the JWT's `roles`, `tenant_id` and `acr` claims are the raw input to every decision in Chapters 1 and 3.

## ➡️ Where This Leads

- **[Module 04 — Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md)** picks up the revocation thread: Chapter 3 here kills a *cached decision*, while Module 04 kills the *token and session* themselves.
- **[Module 05 — Identity Lifecycle & SCIM](../05_Identity_Lifecycle_And_SCIM/README.md)** supplies the events that drive authorization data — a SCIM group change becomes a ReBAC tuple write, and a deprovisioning event becomes a hard revocation.

## 🧭 Quick Reference

| Model | Core question | Failure mode | Best for |
|---|---|---|---|
| **RBAC** | "What is your job title?" | Role explosion | Coarse entry checks, feature toggles |
| **ABAC** | "What are the circumstances?" | Spaghetti code, N+1 latency | Row-level and conditional access |
| **ReBAC** | "How are we connected?" | Needs a specialised graph database | Nested hierarchies, sharing, multi-tenancy |
| **PBAC** | "What does the rulebook say?" | Operational complexity | Enterprise-wide, regulated, versioned rules |

---
⬅ Back to [Main README](../README.md) · 🏦 [The MoneyGuard Bank Story](../SCENARIO.md)
