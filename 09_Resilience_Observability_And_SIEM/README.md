# Module 09: Resilience, Observability & SIEM

Identity is the dependency every other system has, which makes it the one that must never fall over — and the one whose failures must always be visible. This module covers what happens when the authorization engine breaks (CAP theorem, fail-open vs. fail-closed), how caching keeps identity fast without letting revoked permissions survive, how to log security events without leaking the credentials you're trying to protect, and how a SIEM turns scattered logs into detected attacks.

🏦 The running examples are MoneyGuard's **API Gateway**, its policy engine, and the **SOC/SIEM ("Sentinel")** — with Priya, Raj, Alex, Dana, and Eve playing their usual parts.

## 📂 Chapters

| # | Chapter | What It Covers |
|---|---------|----------------|
| 01 | [CAP Theorem & Fail-Open vs. Fail-Closed](./01_CAP_Theorem_And_FailOpen_vs_FailClosed.md) | Consistency vs. availability in identity systems; why grants can be eventually consistent but revocations cannot; the fail-open/fail-closed comparison; graceful degradation, circuit breakers, and the revocation gap. |
| 02 | [Caching Strategies & Invalidation](./02_Caching_Strategies_And_Invalidation.md) | The L1/L2/read-replica ladder; cache-aside; TTL as a security parameter; tenant-scoped cache keys; event-driven invalidation vs. TTL backstop; stampedes, jitter, and coalescing. |
| 03 | [Audit Logging & Observability](./03_Audit_Logging_And_Observability.md) | Application vs. audit vs. security-event streams; structured logging and log injection; correlation IDs and distributed tracing; what to log and what never to log; policy-engine decision-log masking; WORM storage and hash chaining. |
| 04 | [SIEM & Threat Detection](./04_SIEM_And_Threat_Detection.md) | The collect→normalize→enrich→detect→respond pipeline; signature vs. UEBA vs. threat-intel detection; the ten core identity detections; writing and tuning rules; SOAR playbooks and their guardrails; retention and cost. |

## 🎯 Prerequisites

- **[Module 04 — Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md)** is the main prerequisite. This module assumes you know what a JWT is, what the `jti` and `sub` claims are, how token revocation and blocklists work, and why token lifetime matters — all of which are covered there.
- **[Module 03 — Authorization & Policy Engines](../03_Authorization_And_Policy_Engines/README.md)** helps, since the policy decision point (PDP) is the dependency whose failure Chapter 1 is about.
- **[Module 08 — Multi-Tenancy & Data Isolation](../08_MultiTenancy_And_Data_Isolation/README.md)** matters for the `tenant_id` that must appear in every cache key, log line, and detection rule.

New to the repo? Read [`SCENARIO.md`](../SCENARIO.md) first to meet MoneyGuard Bank and its cast.

## 🔑 The One-Paragraph Summary

Identity systems choose availability for authentication and consistency for revocation, and they fail **closed** on anything touching money — but the mature answer is graceful degradation through a cache ladder with a per-resource declared failure mode, so reads survive an outage while writes do not. Caching makes that possible, provided TTL is treated as a security parameter and event-driven invalidation is the primary mechanism with TTL as the backstop. Everything that happens is recorded as structured, sanitized audit events — `jti` and `sub`, never the raw token or the person's name — correlated by a trace ID and written to append-only storage. Those events feed a SIEM that normalizes across sources and detects the patterns no single service can see, with automated playbooks that respond in seconds and guardrails that stop the automation itself becoming an attack surface.

---

📖 [Main README](../README.md) | 🏦 [The MoneyGuard Story](../SCENARIO.md)

⬅ Previous module: [08 — Multi-Tenancy & Data Isolation](../08_MultiTenancy_And_Data_Isolation/README.md) | ➡ Next module: [10 — IaC & Deployment Security](../10_IaC_And_Deployment_Security/README.md)
