# Module 06: Machine Identity & Workload Auth

Most of the traffic inside a modern bank is machines talking to machines — batch jobs, microservices, CI/CD pipelines, and ML models — and none of them can tap a hardware key or type a password. This module covers how software proves who it is, walking the full evolution from static API keys, through OAuth client credentials, to zero-secret workload identity and mutual TLS.

🏦 Everything here follows one running example: MoneyGuard's **Fraud Detection Pipeline**, a Kubernetes-based ML system running as 50 pods that must read transaction data from MoneyGuard's private cloud storage and write risk scores to the **Core Banking Ledger** — without a single hardcoded password anywhere in the container image.

## 📚 Chapters

| # | Chapter | What it covers |
|---|---------|----------------|
| 01 | [Service Accounts and API Keys](./01_Service_Accounts_And_API_Keys.md) | What a non-human identity is, why service accounts are the #1 breach vector, the three anti-patterns to ban (shared credentials, eternal secrets, human accounts used for automation), and exactly why static API keys fail |
| 02 | [OAuth 2.0 Client Credentials Grant](./02_OAuth_Client_Credentials_Grant.md) | The machine-to-machine OAuth flow: token requests, scopes and audiences, `private_key_jwt`, sender-constrained tokens (DPoP / RFC 8705), and the Secret Zero problem it leaves behind |
| 03 | [Workload Identity Federation](./03_Workload_Identity_Federation.md) | Zero-secret identity via platform attestation — Kubernetes → cloud OIDC federation, IRSA and Managed Identities, CI/CD federation, and SPIFFE/SPIRE with SVIDs |
| 04 | [Mutual TLS (mTLS)](./04_Mutual_TLS_mTLS.md) | Two-way certificate authentication at the transport layer: the handshake, SPIFFE IDs in the SAN, service mesh vs in-app vs gateway termination, and certificate lifecycle in production |

Read them in order — each chapter fixes a weakness the previous one leaves open:

```
API keys          →  eternal bearer secret, sprawls everywhere
  ↓ Chapter 02
Client credentials →  short-lived tokens, but still a client_secret (Secret Zero)
  ↓ Chapter 03
Workload identity  →  no secret at all, but tokens are still bearer credentials
  ↓ Chapter 04
mTLS               →  credential bound to the machine; a stolen token is useless
```

## 🎯 What You'll Be Able to Do

- Choose the right machine-auth mechanism for a given trust boundary (internal service, cloud managed service, or external partner).
- Design a zero-secret Kubernetes → cloud federation, and spot the wildcard `sub` misconfiguration that silently breaks it.
- Explain the Secret Zero problem and how platform attestation resolves it.
- Roll out mTLS to a live service without downtime, and operate its certificate lifecycle.

## 📋 Prerequisites

This module assumes you already know the OAuth 2.0 basics — clients, tokens, scopes, and JWT validation. If any of that is unfamiliar, read **[Module 02 — AuthN & Federation](../02_AuthN_Federation/README.md)** first, particularly [`03_OAuth2.md`](../02_AuthN_Federation/03_OAuth2.md). Token-signature verification and key rotation are covered in [Module 04 — Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md), especially [`02_JWKS_And_Key_Rotation.md`](../04_Edge_Security_Tokens_And_Sessions/02_JWKS_And_Key_Rotation.md).

It also builds directly on the service-account primer in **[Module 01 — Core IAM Concepts](../01_Core_IAM_Concepts/README.md)** ([`07_Service_Accounts_And_NonHuman_Identity.md`](../01_Core_IAM_Concepts/07_Service_Accounts_And_NonHuman_Identity.md)). Chapter 01 here revisits that ground and goes considerably deeper.

## ➡️ What Comes Next

Module 06 is about how *machines* authenticate. **[Module 07 — Advanced Authentication & MFA](../07_Advanced_Authentication_And_MFA/README.md)** is the mirror image: how *humans* authenticate in an era where passwords and 6-digit codes are no longer enough.

---
⬅ Back to [Main README](../README.md) | 🏦 Meet the cast in [SCENARIO.md](../SCENARIO.md)
