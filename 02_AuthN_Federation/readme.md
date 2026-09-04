# Module 02: Authentication (AuthN) & Federation

This module covers the **front door** of Identity and Access Management: how a system proves *who* someone is, and how one organization safely trusts another organization's identity system. It walks through Zero Trust as the architectural frame, then the three protocols that implement it in practice — **SAML 2.0** for enterprise and partner federation, **OAuth 2.0** for delegated API access, and **OpenID Connect (OIDC)** for modern login — and finishes with the `state`, `nonce`, and PKCE parameters that keep those flows safe from real attacks.

Every chapter is grounded in the same running example: **MoneyGuard Bank** — Priya at the branch counter, Alex building the login flows, Raj configuring the trust relationships, Dana auditing the result, and **Eve** trying to break in. Read [`SCENARIO.md`](../SCENARIO.md) first if you have not met the cast.

## 📚 Chapters

| # | Chapter | What it covers |
|---|---|---|
| 01 | [Zero Trust Architecture Overview](./01_Zero_Trust_Architecture_Overview.md) | Why "never trust, always verify" makes identity the new perimeter — the IdP vs token-plane split, PEP/PDP enforcement, federation, token lifecycles, and how MoneyGuard's whole AuthN architecture fits together |
| 02 | [SAML 2.0](./02_SAML.md) | XML-based enterprise SSO and B2B federation, walked through MoneyGuard's joint mortgage product with **PartnerBank**: metadata exchange, bindings, assertions, signature validation, and the attacks to defend against |
| 03 | [OAuth 2.0](./03_OAuth2.md) | Delegated authorization — the four roles, Authorization Code + PKCE, Client Credentials, scopes, access and refresh tokens, and why OAuth alone is **not** a login protocol |
| 04 | [OpenID Connect (OIDC)](./04_OIDC.md) | The identity layer on top of OAuth 2.0: the ID token, standard claims, discovery, UserInfo, step-up authentication — plus a direct OAuth-vs-OIDC comparison table |
| 05 | [`state`, `nonce` & PKCE](./05_State_Nonce_And_PKCE.md) | The three security parameters, framed around **MoneyGuard Mobile App** login: CSRF, ID token replay, and authorization code interception — what each one stops and who verifies it |

Read them in order. Chapter 01 gives you the map; 02–04 are the protocols; 05 is the hardening that applies to 03 and 04.

## 🧭 How This Module Fits

**Builds on → [Module 01: Core IAM Concepts](../01_Core_IAM_Concepts/README.md)**
Module 01 established the vocabulary this module assumes: the AAA framework (Authentication, Authorization, Accounting), the difference between AuthN and AuthZ, least privilege, identity repositories, and the Joiner-Mover-Leaver lifecycle. This module takes the "Authentication" leg of AAA and shows how it is actually implemented over the wire between systems that do not share a database.

**Feeds into → [Module 03: Authorization & Policy Engines](../03_Authorization_And_Policy_Engines/README.md)**
Every chapter here ends the same way: a validated identity and a signed token. Module 03 picks up from there and answers the *next* question — given that we know who this is, what are they allowed to do? RBAC, ABAC, ReBAC, and the policy engines (OPA) that make those decisions.

**Feeds into → [Module 04: Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md)**
This module produces tokens; Module 04 is about living with them: JWT vs opaque tokens, JWKS and signing key rotation, revocation, and session management at the API Gateway.

Two topics that used to live in this folder now live elsewhere, because they apply across the whole stack rather than only to login:

- **JWKS and signing key rotation** → [Module 04 — Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md)
- **Audit logging and observability** → [Module 09 — Resilience, Observability & SIEM](../09_Resilience_Observability_And_SIEM/README.md)

## 🏷️ Conventions

- 🏦 marks a MoneyGuard walkthrough.
- ⚠️ marks a common mistake or security pitfall.
- 💡 marks a pro tip.
- All diagrams are [Mermaid](https://mermaid.js.org/) and render directly on GitHub.

---
🏠 [Main README](../README.md) | 🏦 [The MoneyGuard Bank Story](../SCENARIO.md) | ⬅ [Module 01: Core IAM Concepts](../01_Core_IAM_Concepts/README.md) | ➡ [Module 03: Authorization & Policy Engines](../03_Authorization_And_Policy_Engines/README.md)
