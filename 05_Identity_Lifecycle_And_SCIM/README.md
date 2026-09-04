# Module 05: Identity Lifecycle & SCIM

Module 01 introduced the **Joiner–Mover–Leaver** lifecycle as a concept. This module is the engineering answer to it: the actual protocol that carries a hire, a role change or a termination from an HR system into every downstream application — automatically, and without the user having to do anything.

The recurring example is **MoneyGuard Bank**'s **Business Banking SaaS Portal**, where corporate clients **Acme Corp** (running Entra ID) and **Globex Inc** (running Okta) need their own staff provisioned, entitled, and — most importantly — **deprovisioned** without MoneyGuard ever being told by a human. Meet the full cast in [`SCENARIO.md`](../SCENARIO.md).

## 📚 Chapters

| # | Chapter | What it covers |
|---|---|---|
| 01 | [JIT Provisioning vs SCIM](./01_JIT_Provisioning_Vs_SCIM.md) | Pull versus push provisioning, JIT's two failure modes (**the ghost user** and **the deprovisioning gap**), why zombie accounts survive SSO being blocked, the SCIM handshake (base URL + per-tenant bearer token), a head-to-head comparison, and why production runs both mechanisms together. |
| 02 | [SCIM Protocol Deep Dive](./02_SCIM_Protocol_Deep_Dive.md) | RFC 7643/7644 in practice: the `/Users`, `/Groups` and discovery endpoints, the `User` and `Group` schemas, `externalId` versus `id`, the three `PatchOp` dialects real providers actually send, filtering and `ListResponse` pagination, the error schema and status-code contracts, and wiring group changes into the authorization graph. |
| 03 | [Handling Sync Race Conditions](./03_Handling_Sync_Race_Conditions.md) | The fast lane (SSO) versus the slow lane (batched SCIM): day-one lockouts, **upsert** on `(tenant_id, external_id)`, idempotency and side-effect guards, out-of-order delivery that silently reverses terminations, buffering dangling group memberships, tombstones, and the nightly **reconciliation sweep**. |

Read them in order — Chapter 3 assumes the protocol detail from Chapter 2.

## 🔗 How This Builds on Earlier Modules

- **[Module 01 — Core IAM Concepts](../01_Core_IAM_Concepts/README.md)** established the **Joiner–Mover–Leaver** lifecycle in [`05_IAM_Lifecycle_JML.md`](../01_Core_IAM_Concepts/05_IAM_Lifecycle_JML.md). This module does **not** re-teach JML — it assumes it and goes straight to the protocol that implements it.
- **[Module 02 — AuthN & Federation](../02_AuthN_Federation/README.md)** provides the SSO that JIT provisioning rides on. SCIM synchronises accounts; it does not authenticate anyone, so SAML or OIDC has to work first.
- **[Module 03 — Authorization & Policy Engines](../03_Authorization_And_Policy_Engines/README.md)** consumes what SCIM produces: a `PATCH /Groups` membership change becomes a **ReBAC relationship tuple** ([Chapter 2](../03_Authorization_And_Policy_Engines/02_ReBAC_And_Google_Zanzibar.md)), and a deprovisioning event fires the **Pub/Sub kill switch** that evicts cached authorization decisions ([Chapter 3](../03_Authorization_And_Policy_Engines/03_Policy_Engines_OPA_And_PBAC.md)).
- **[Module 04 — Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md)** is what makes deprovisioning real. A SCIM `active: false` must call straight into the [revocation path](../04_Edge_Security_Tokens_And_Sessions/03_Token_Revocation_And_Session_Management.md) — otherwise the account is flagged inactive while the person's access token stays valid for another hour.

## 🧭 Quick Reference

| Question | Answer |
|---|---|
| What triggers JIT? | The user's first SSO login — a **pull** |
| What triggers SCIM? | A hire, role change or termination in the IdP — a **push** |
| Which one can offboard? | **Only SCIM.** A leaver never logs in, so JIT never fires |
| What do you match users on? | **`externalId`** — never email, never `userName` |
| Where does the tenant come from? | The **SCIM bearer token**, never the request body |
| How fast is SCIM? | Batched — Entra ID syncs roughly every **40 minutes** |
| What does `active: false` mean? | A **security event**: disable + kill sessions + revoke API keys + delete tuples + evict caches + cancel jobs + audit |
| What catches whatever the push dropped? | A **nightly reconciliation sweep**, disabling orphans (never deleting them) |

---
⬅ Back to [Main README](../README.md) · 🏦 [The MoneyGuard Bank Story](../SCENARIO.md)
