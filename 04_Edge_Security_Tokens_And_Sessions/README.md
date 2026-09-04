# Module 04: Edge Security, Tokens & Sessions

Module 02 issued the token and Module 03 decided what it was allowed to do. This module is about the token's *life in transit*: what format it should take on each side of your perimeter, how services prove it is genuine without calling anyone, and — the hardest problem in the whole stack — how you take it back the moment something goes wrong.

Everything here runs on the **MoneyGuard Bank** example: Alex Kim building the mobile login flow, Priya Sharma's session on a shared branch terminal, Raj Patel rotating signing keys on a Tuesday night, and Eve trying to hold on to a stolen token for fifty more minutes. Meet them in [`SCENARIO.md`](../SCENARIO.md).

## 📚 Chapters

| # | Chapter | What it covers |
|---|---|---|
| 01 | [JWT vs Opaque Tokens](./01_JWT_Vs_Opaque_Tokens.md) | Value versus reference tokens, why a JWT must never reach a browser or phone, the **Phantom Token Pattern**, BFF with `HttpOnly` cookies, introspection (RFC 7662), **DPoP** sender-constrained tokens, **Token Exchange** (RFC 8693) for blast-radius control, and why RS256 rather than HS256. |
| 02 | [JWKS and Key Rotation](./02_JWKS_And_Key_Rotation.md) | The `kid` header claim, the `/.well-known/jwks.json` key set, the **three-phase zero-downtime rotation** (publish → switch → wait → retire), JWKS caching with rate-limited refresh on an unknown `kid`, HSM-held private keys, and emergency rotation. |
| 03 | [Token Revocation and Session Management](./03_Token_Revocation_And_Session_Management.md) | Why stateless tokens cannot be revoked, the **Redis blocklist at the edge** (`sub` / `sid` / `jti`), **refresh token rotation with reuse detection**, doing logout properly (RFC 7009, RP-initiated and back-channel), **CAEP / Shared Signals**, and sliding versus absolute session expiry. |

Read them in order — Chapter 2 assumes the RS256 and JWKS groundwork from Chapter 1, and Chapter 3 assumes both.

## 🔗 How This Builds on Earlier Modules

- **[Module 02 — AuthN & Federation](../02_AuthN_Federation/README.md)** covered OAuth 2.0, OIDC and SAML — the protocols that *mint* the token. This module picks it up the instant it is issued. (The JWKS and key-rotation material that previously lived in Module 02 now lives here, in [Chapter 2](./02_JWKS_And_Key_Rotation.md), alongside the token formats it protects.)
- **[Module 03 — Authorization & Policy Engines](../03_Authorization_And_Policy_Engines/README.md)** decided *what* the token holder may do. Chapter 3 of Module 03 killed a cached **authorization decision** with a Redis Pub/Sub kill switch; Chapter 3 here kills the **credential and the session** themselves, one layer lower.
- **[Module 01 — Core IAM Concepts](../01_Core_IAM_Concepts/README.md)** established least privilege, which is the reasoning behind short token lifetimes and per-hop token exchange.

## ➡️ Where This Leads

- **[Module 05 — Identity Lifecycle & SCIM](../05_Identity_Lifecycle_And_SCIM/README.md)** supplies the trigger: when a SCIM `PATCH { "active": false }` arrives for a terminated employee, it must call straight into the revocation path built in Chapter 3.
- **Module 06 — Machine Identity & Workload Auth** applies the same token mechanics to non-human callers such as the Fraud Detection Pipeline.

## 🧭 Quick Reference

| Question | Answer |
|---|---|
| What does a public client get? | An **opaque** token, or an `HttpOnly` cookie in front of a BFF |
| What do internal services get? | A short-lived, audience-scoped **JWT** signed with RS256 |
| How does a service know a JWT is genuine? | The `kid` header → the cached **JWKS** public key → local signature check |
| How long should an access token live? | 5–15 minutes; 60 seconds for exchanged service-to-service tokens |
| How do you revoke instantly? | A **Redis blocklist at the gateway**, written by a pushed CAEP event |
| How do you rotate signing keys safely? | **Publish → switch → wait out max token lifetime → retire** |
| What is the golden phrase? | **Opaque outside, JWT inside. Stateless inside, stateful at the edge.** |

---
⬅ Back to [Main README](../README.md) · 🏦 [The MoneyGuard Bank Story](../SCENARIO.md)
