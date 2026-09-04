# Chapter 03: Interview Questions & Answers

> **Module 11 · Chapter 3 of 4** — 36 realistic IAM and security system-design questions with model answers, organised by topic.

## 📋 Overview

This is the consolidated question bank for the whole repository. Each question has:

- **The question**, phrased the way an interviewer actually asks it.
- **A model answer** — the substance, not a script to memorise.
- **🎯 What they're really testing** — the signal behind the question, so you can adapt when it's worded differently.

Answers are framed around 🏦 **MoneyGuard Bank** wherever a concrete example helps. Substitute your own employer's systems and the answers transfer unchanged.

**How to use this chapter:** read a question, *answer it out loud from memory*, then read the model answer and note what you missed. Silently reading answers builds recognition, not recall — and interviews test recall.

## 🗂️ Topics

| # | Topic | Questions |
|---|---|---|
| A | [Fundamentals & Authentication](#a--fundamentals--authentication) | 1–6 |
| B | [Federation & Protocols](#b--federation--protocols) | 7–11 |
| C | [Authorization & Policy](#c--authorization--policy) | 12–16 |
| D | [Tokens & Sessions](#d--tokens--sessions) | 17–21 |
| E | [Identity Lifecycle](#e--identity-lifecycle) | 22–25 |
| F | [Machine & Workload Identity](#f--machine--workload-identity) | 26–28 |
| G | [MFA & Advanced Authentication](#g--mfa--advanced-authentication) | 29–30 |
| H | [Multi-Tenancy & Data Isolation](#h--multi-tenancy--data-isolation) | 31–32 |
| I | [Resilience, Scale & Observability](#i--resilience-scale--observability) | 33–36 |

---

## A · Fundamentals & Authentication

### Q1. What is the difference between authentication and authorization?

**Answer.** **Authentication (AuthN)** answers *"who are you?"* — verifying a claimed identity against a credential. **Authorization (AuthZ)** answers *"what are you allowed to do?"* — evaluating whether that verified identity may perform a specific action on a specific resource.

🏦 Priya Sharma proving she is Priya by tapping her passkey is authentication. The system then deciding she may read account 8891 *because she's a teller at that branch, during business hours* is authorization.

The architecturally important consequence: **authentication happens once per session; authorization happens on every single request.** That difference drives everything. Authentication is centralised in one IdP and can afford to be expensive — it involves a human, MFA, redirects. Authorization must answer in single-digit milliseconds, ten thousand times per second, which is why it needs caching, a decoupled decision service, and careful invalidation.

> 🎯 **What they're testing:** whether you can go past the definition to the architectural consequence. Everyone gets the definition. The follow-on sentence is the signal.

---

### Q2. Walk me through the AAA framework.

**Answer.** **Authentication, Authorization, Accounting.** The third one is the one candidates forget, and it is the one auditors care about most.

| | Question | MoneyGuard component |
|---|---|---|
| **Authentication** | Who are you? | Entra ID + MFA engine |
| **Authorization** | What may you do? | AuthZ service (PDP) |
| **Accounting** | What did you actually do? | Structured audit log → Sentinel SIEM |

Accounting is not logging-for-debugging. It is the evidence layer: SOX requires that every privileged action against the Core Banking Ledger be attributable to a named human or a named workload, in a tamper-evident record. Without accounting you cannot detect an insider, cannot investigate a breach, and cannot pass an audit — you can only assert that your controls work.

> 🎯 **What they're testing:** whether you think about detection and evidence, not just prevention.

---

### Q3. Explain the principle of least privilege. How do you actually enforce it at scale?

**Answer.** Least privilege means every identity holds the minimum access needed for its function, for the minimum time.

Stating it is easy; enforcing it across 50,000 employees plus tens of thousands of workloads is a **continuous process**, not a configuration:

1. **Default deny.** No implicit access. An absent policy means "no," never "maybe."
2. **Conservative role templates.** New roles start narrow. It is far easier to grant an extra permission on request than to claw one back.
3. **Avoid wildcards.** `ledger:*` is not a permission, it is an incident waiting to happen. Enumerate actions.
4. **Observe, then tighten.** Access logs tell you which granted permissions were never used in 90 days. Remove them.
5. **Time-bound and just-in-time access.** Sam Torres the contractor gets access that *expires with his contract*, automatically. Break-glass admin rights are requested, approved, and expire in hours.
6. **Periodic recertification.** Every quarter, managers reattest that their reports still need what they hold. Unattested access is revoked, not flagged.
7. **Permission boundaries.** Even a delegated admin cannot grant more than they themselves hold — this prevents privilege escalation through delegation.

> 💡 The sentence to say: *"Least privilege isn't a one-time config; it's a loop — model, grant conservatively, observe usage from logs, tighten, and recertify on a schedule."*

> 🎯 **What they're testing:** whether you've operated a real access programme or only read the definition. The observe-and-tighten loop is the tell.

---

### Q4. What is separation of duties, and how would you enforce it technically?

**Answer.** Separation of duties (SoD) means no single identity can complete a high-risk operation alone. It defends against both insider fraud and a single compromised account.

🏦 At MoneyGuard, three concrete examples:
- The engineer who *writes* a policy change cannot be the one who *approves and applies* it — enforced by requiring a second reviewer on the pull request in the IaC pipeline.
- Raj Patel can *request* elevated ledger access but a second IAM administrator must *approve* it.
- Whoever can write to the Core Banking Ledger must not be able to modify the audit log that records those writes — different roles, different credentials, append-only storage.

Technically it's enforced three ways: **mutually exclusive role assignments** (the policy store rejects granting both `payment_initiator` and `payment_approver` to one identity), **approval workflows** on privileged grants, and **pipeline gates** in CI/CD requiring an independent approver.

> 🎯 **What they're testing:** whether you can translate a governance concept into mechanism. "We'd have a policy about it" is a weak answer; "the policy store rejects the conflicting binding" is a strong one.

---

### Q5. How do you handle the "break glass" scenario — emergency access?

**Answer.** Emergency access is where least privilege usually dies, because at 3am during an outage nobody wants to fight the access system. So you design for it deliberately rather than letting people keep standing admin rights "just in case."

The design:
- **Zero standing privilege.** Nobody holds emergency access continuously.
- **Self-service elevation with an explicit reason.** The requester states what they're doing and why.
- **Time-boxed.** Access expires automatically in 1–4 hours, not "when someone remembers."
- **Loud by default.** Elevation fires a high-priority alert to Sentinel and to a Slack channel humans read *at the moment it happens*, not in a monthly report.
- **Independently reviewed after the fact.** Every break-glass session is reviewed by someone who did not use it.
- **A genuine last-resort credential** — an offline, sealed, MFA-protected account for the case where the IdP itself is down — physically secured, with its use being an incident by definition.

The trade-off worth naming: you're accepting slightly slower incident response in exchange for eliminating permanent standing privilege. In a bank, that's the right trade.

> 🎯 **What they're testing:** operational maturity. Anyone can design the happy path; break-glass is where you show you've been on-call.

---

### Q6. A user reports they can see data they shouldn't. Walk me through your investigation.

**Answer.** Work down the layers of the diagram, because the answer is always at one specific layer, and knowing which one tells you how bad it is.

1. **Confirm and contain first.** Reproduce it, then decide whether to disable the affected access immediately. Containment beats diagnosis when data is leaking.
2. **Identity layer.** Who is this principal *actually*? Duplicate account? Did a rename create a second identity joined on email instead of `externalId`?
3. **Token layer.** Decode their token. What are the `sub`, `tenant`, `roles`, `scopes`, and `aud`? Is the token even meant for this API? A wrong `aud` accepted is an audience-confusion bug.
4. **Authorization layer.** Pull the PDP decision logs for that request's trace ID. Did the PDP say `ALLOW`? If so, the policy is wrong — a role has too much, or an ABAC condition is inverted. If the PDP said `DENY` and access still happened, the enforcement point was bypassed.
5. **Cache layer.** Was the decision served from a stale cache after a recent role change? That points at a missed invalidation event.
6. **Data layer.** Was this a code path that bypasses RLS — a batch job, a report, an export, an admin tool with a superuser connection?
7. **Blast radius.** Query the audit log for *every other* principal who could have hit the same path, and how often. One user reporting it means others may have seen it silently.
8. **Fix, then close the seam.** Fix the specific bug, then add the detection that would have caught it — a Sentinel rule for cross-tenant access patterns, or a test in CI.

> 🎯 **What they're testing:** structured debugging under pressure, and whether you contain before you investigate. The layered walk-down is the framework they want to hear.

---

## B · Federation & Protocols

### Q7. SAML 2.0 vs. OAuth 2.0 vs. OpenID Connect — when do you use each?

**Answer.** They solve three different problems and are constantly confused.

| | Purpose | Format | Use when |
|---|---|---|---|
| **SAML 2.0** | Enterprise SSO — an IdP asserts a user's identity to an application | Signed XML assertion | Integrating with established enterprise software and partner IdPs. Legacy, but battle-tested and everywhere. |
| **OAuth 2.0** | **Delegated authorization** — an app acts on a user's behalf against an API | Access token (opaque or JWT) | You need permission to *do something*, not to know who the user is. |
| **OIDC** | **Authentication**, built as a thin layer on top of OAuth 2.0 | ID Token (JWT) | You need to *log a user in* and know who they are. |

🏦 At MoneyGuard: PartnerBank federates in over **SAML** because that's what their existing IdP speaks. The Employee Portal logs Priya in with **OIDC**. When the Business Banking SaaS Portal calls an internal accounting API on Acme Corp's behalf, that's **OAuth 2.0** delegated authorization.

> 💡 **The rule of thumb:** if the app receives a token and says *"now I know who is sitting at the keyboard, so I can log them in"* — that's OIDC. If it says *"now I can read/write this API on their behalf"* — that's OAuth 2.0. If it's XML flying through a browser POST — that's SAML.

⚠️ The classic mistake is calling OAuth 2.0 an authentication protocol. It is not, and treating it as one causes real vulnerabilities: an access token proves the bearer holds a permission, not that a specific user is present.

> 🎯 **What they're testing:** the OAuth-is-not-authentication distinction. Getting this wrong is disqualifying for a security role.

---

### Q8. Walk me through the OIDC Authorization Code Flow with PKCE.

**Answer.** 🏦 Priya opens the MoneyGuard Employee Portal:

1. The portal's backend generates three secrets: a random `state`, a random `nonce`, and a **PKCE verifier**. It hashes the verifier with SHA-256 to produce the **challenge**. `state` and `nonce` go into a secure `HttpOnly` cookie; the verifier stays in server memory.
2. The browser is redirected to Entra ID with `response_type=code`, `scope=openid ...`, `state`, `nonce`, `code_challenge`, and `code_challenge_method=S256`.
3. Entra ID authenticates Priya — password plus passkey — and redirects back with a short-lived **authorization code** and the same `state`.
4. **Check 1 (CSRF):** the backend compares the `state` in the URL against the one in the cookie. Match required.
5. The backend calls the token endpoint **server-to-server**, sending the code, its client credentials, and the raw **PKCE verifier**.
6. **Check 2 (PKCE):** Entra ID hashes the verifier and compares it to the challenge from step 2. Match required.
7. Entra ID returns an **ID Token** and an **Access Token**.
8. The backend verifies the ID Token's signature using Entra ID's public key from **JWKS**, and validates `iss`, `aud`, and `exp`.
9. **Check 3 (replay):** it reads the `nonce` claim inside the ID Token and compares it to the cookie. Match required.
10. The backend issues its own `HttpOnly` session cookie. The browser never touches an OAuth token, so XSS cannot steal one.

> 🎯 **What they're testing:** whether you can name all three checks and say what each defeats. See Q9.

---

### Q9. What do `state`, `nonce`, and PKCE each protect against? Why do you need all three?

**Answer.** Three different attacks, three different mechanisms. This is the highest-density question in federation.

| Parameter | Attack it defeats | How it works |
|---|---|---|
| **`state`** | **CSRF / login fixation.** Eve completes a login halfway, captures a callback URL containing *her* code, and tricks Priya into clicking it — permanently linking Priya's session to Eve's account. | A random value stored in Priya's cookie and echoed in the URL. The backend requires them to match, proving *this browser* started *this flow*. |
| **PKCE** | **Authorization code interception.** The code travels through the browser (the "front channel") where a malicious extension or a rogue app on a phone can sniff it, then redeem it first. | The client keeps a secret verifier and sends only its hash up front. Redeeming the code requires presenting the raw verifier over the back channel — which the interceptor never saw. |
| **`nonce`** | **ID Token replay.** Eve captures a valid ID Token and presents it days later; the signature still verifies. | The value is baked *inside the signed token payload* by the IdP. The client requires it to match the current session's cookie — so yesterday's token fails today. Changing it breaks the signature. |

They're not redundant because they protect different segments: `state` protects the *session binding*, PKCE protects the *code in transit*, `nonce` protects the *token's freshness*.

⚠️ PKCE was originally specified for mobile and other public clients, but current guidance is to use it for **confidential clients too**. There's no meaningful cost and it closes the interception window regardless of client type.

> 🎯 **What they're testing:** genuine protocol depth. Almost anyone can say "PKCE is for mobile apps." Very few can say which specific attack each of the three stops.

---

### Q10. PartnerBank wants to federate into MoneyGuard via SAML. Walk me through the setup and the flow.

**Answer.** **Setup (the trust bridge), done once:**
1. PartnerBank sends MoneyGuard its **IdP metadata**: entity ID, SSO endpoint URL, and the X.509 **public** signing certificate.
2. MoneyGuard sends PartnerBank its **SP metadata**: entity ID and the **ACS URL** (Assertion Consumer Service — where assertions get POSTed).
3. Both sides agree on the attribute mapping — which SAML attribute carries the stable user identifier, the email, and the group memberships.
4. **Critically:** agree on the *immutable* identifier to anchor on. Not email. An employee ID or a directory object GUID that never changes.
5. Record the certificate expiry date and set a calendar alert. This is the number one cause of SAML outages.

**The runtime flow (SP-initiated):**
1. A PartnerBank employee hits MoneyGuard's app and enters their `@partnerbank.com` email.
2. MoneyGuard recognises the domain and generates a SAML `AuthnRequest`, redirecting the browser to PartnerBank's IdP.
3. PartnerBank authenticates them, with their own MFA.
4. PartnerBank builds a **SAML Assertion** — XML containing subject, attributes, and validity window — and signs it with its private key.
5. Because XML is too large for a URL, the IdP uses **HTTP POST binding**: it returns an HTML page with a hidden auto-submitting form that POSTs the assertion to MoneyGuard's ACS URL.
6. MoneyGuard validates the signature with PartnerBank's public certificate, checks the assertion has not expired, checks the audience restriction matches MoneyGuard, and checks the assertion ID has not been seen before (replay protection).
7. A session is created.

💡 The architectural refinement: don't teach every downstream MoneyGuard service to speak SAML. Terminate SAML at **Entra ID acting as a broker**, and let everything downstream consume a normalised OIDC token. One place understands XML; the rest of the estate stays modern.

> 🎯 **What they're testing:** whether you've actually done a federation integration. Mentioning certificate expiry and audience restriction is what proves it.

---

### Q11. What happens if PartnerBank's identity provider goes down?

**Answer.** Two different populations of users, two different outcomes:

- **Users with active sessions keep working.** Their tokens are already issued, and MoneyGuard's gateways validate signatures against **cached JWKS keys / cached certificates**, which requires no call to the IdP. They keep working until their tokens expire.
- **New logins fail.** MoneyGuard cannot authenticate a PartnerBank user without PartnerBank. We do not hold their credentials and we would not want to — that's the whole point of federation.

This is an acceptable and honest answer. The follow-up you should volunteer: *"we cannot fail open on authentication."* Letting people in because the IdP is unreachable converts an availability incident into a security incident.

What we *can* do to soften it:
- **Cache JWKS aggressively** with a long stale-serve window so a brief IdP blip doesn't invalidate every session.
- **Set token lifetimes deliberately** — 15 minutes limits revocation exposure but means a 20-minute IdP outage logs everyone out. There's a genuine trade-off between revocation speed and outage tolerance, and it should be a conscious decision, not a default.
- **Graceful degradation in the UI** — "Your identity provider is unavailable, please retry" is far better than a generic 500, because it tells PartnerBank's own help desk where to look.
- **Monitor the partner IdP** so MoneyGuard's SOC knows before PartnerBank's users start calling.

> 🎯 **What they're testing:** whether you understand that federation means outsourcing an availability dependency, and whether you'll be tempted to fail open.

---

## C · Authorization & Policy

### Q12. RBAC vs. ABAC vs. ReBAC — compare them and tell me which you'd choose.

**Answer.**

| Model | Decision basis | Strength | Weakness |
|---|---|---|---|
| **RBAC** | The subject's assigned role | Simple, and *auditable* — you can answer "who is a Teller?" with one query | Role explosion. Context-blind: it can't express "only her own branch." |
| **ABAC** | Attributes of subject, resource, action, environment | Expressive and contextual — time, location, device risk, data sensitivity | Reverse queries are hard: "who can access account 8891?" requires evaluating policy against every user. |
| **ReBAC** | The relationship graph between subject and resource | Natural for hierarchies and sharing; scales to deep nesting | Operationally heavier; usually needs a purpose-built graph store. |

🏦 At MoneyGuard I'd use **all three, layered**:
- **RBAC as the backbone.** Priya is a `Teller`. Dana Cole is an `Auditor`. This is what the compliance team reads and recertifies, and it must stay legible to non-engineers.
- **ABAC conditions on top.** `Teller` can read accounts *at their assigned branch*, *during business hours*, *from a managed device*. These conditions are where the real risk reduction lives — a role alone would have let a compromised teller account read every account in the bank.
- **ReBAC where the relationship is genuinely the permission.** In the Business Banking SaaS Portal, an Acme Corp admin may approve a payroll batch because `user → admin_of → tenant:acme → owns → batch:9931`. Expressing that in RBAC would mean a role per tenant per resource type — the definition of role explosion.

> 💡 The mature answer is *"hybrid, and here's the boundary between them."* Insisting one model wins is a junior signal — real systems are always hybrids.

> 🎯 **What they're testing:** whether you can pick per-context rather than reciting three definitions.

---

### Q13. Where do you enforce authorization — the gateway, the service, or the database?

**Answer.** **All three, at different granularities.** That's defence in depth, not redundancy.

| Layer | Granularity | 🏦 MoneyGuard example |
|---|---|---|
| **API Gateway** | Coarse | Is the token valid, unrevoked, and within rate limits? Is this principal allowed to call the `/ledger` API family at all? |
| **Service (PEP → PDP)** | Fine | May *this* principal perform `account.read` on *this specific* account, given branch and time? |
| **Database (RLS)** | Structural | Does this row's `tenant_id` match the session's tenant? The database physically will not return others. |

The reasoning is that each layer assumes the layer above it has a bug. The gateway can't do fine-grained checks because it would need to know about every resource in the bank. The service could get its query wrong. RLS catches that.

The crucial architectural point: **the enforcement points are many, but the decision point is one.** Gateway and service middleware are both PEPs calling the same PDP. That gives real-time changeability and a single place to audit "who can touch the ledger?"

⚠️ The anti-pattern is authorization logic scattered as `if (user.role == "admin")` throughout business logic. Ten services, ten implementations, ten bugs, and no way to answer an auditor's question without reading all the code.

> 🎯 **What they're testing:** the PEP/PDP distinction and whether you understand defence in depth as *independent* layers rather than the same check repeated.

---

### Q14. Your authorization service is being called 10,000 times per second. How do you make that work?

**Answer.** Four things, in order of impact:

1. **Make the service stateless and scale horizontally.** No session affinity, so any instance can answer any request behind a load balancer.
2. **Two cache layers.**
   - **L1, in-process,** with a very short TTL (a few seconds). Microsecond latency, no network hop. Absorbs the majority of traffic because access patterns are extremely repetitive — Priya reads the same account fifteen times in a session.
   - **L2, distributed (Redis),** shared across instances so a cold instance isn't cold for long.
3. **Event-driven invalidation.** This is the part people miss. When SCIM deactivates a user or Raj changes a role, an event is published and the affected cache entries are purged immediately. **The TTL is a backstop for a missed event, not the primary correctness mechanism.** If you rely on TTL alone, a demotion takes effect "eventually," which is not a sentence you can say to a regulator.
4. **Partition the policy store by tenant.** Improves locality, and limits blast radius — a hot tenant can't degrade everyone.

Two supporting details worth adding: **batch the checks** (a page listing 50 accounts should ask "which of these 50 may Priya read?" in one call, not 50), and **keep the policy language constrained** so evaluation time stays predictable. An unbounded policy language means unbounded p99.

> 🎯 **What they're testing:** whether you say "cache" and stop, or "cache *with event-driven invalidation*." The invalidation half is the whole answer.

---

### Q15. Why not put the user's permissions directly in the JWT? It would remove the authorization call entirely.

**Answer.** Because **a JWT is immutable once signed, so permissions in a token are a cache with no invalidation.**

🏦 Concretely: Raj Patel demotes Sam Torres from `Admin` to `Viewer` at 14:00. Sam's token was minted at 13:55 with `role: Admin` and a one-hour lifetime. Sam remains a full admin until 14:55 no matter what the policy store says. For a bank, that's a 55-minute unauthorised-access window created by design.

There's a second problem: JWTs are Base64-encoded, not encrypted. A permission list in a token leaks your internal authorization model to anyone who intercepts it — and to the user, who can read their own token in the browser.

The right split:
- **In the token:** stable, coarse identity claims — `sub`, `tenant`, `acr`, maybe a small set of very broad scopes. These change rarely and a short staleness window is acceptable.
- **Behind the PDP:** everything fine-grained, so it can change in real time.

⚠️ Then add the nuance, because dogma reads as junior: *"For a very high-volume, low-sensitivity read path — rendering a UI's menu items — embedding coarse claims is a legitimate optimisation, as long as the server re-checks on the actual action. The rule isn't 'never'; it's 'the sensitivity of the action determines how fresh the decision must be.'"*

> 🎯 **What they're testing:** the immutability insight, and then whether you can hold a nuanced position rather than a memorised rule.

---

### Q16. What is a policy engine like OPA, and why would you use one instead of writing policy in code?

**Answer.** A policy engine externalises authorization rules from application code into a dedicated, declarative language (Rego, for Open Policy Agent). The application sends a structured input — principal, action, resource, context — and receives a decision.

Five reasons it beats `if` statements:
1. **One place to audit.** Dana Cole can read the policy bundle instead of reading every service's source.
2. **Change without redeploying.** Updating a rule ships a policy bundle, not a release of twelve services.
3. **Testable in isolation.** Policies get unit tests with concrete inputs and expected decisions, run in CI. You can prove "a contractor can never read the ledger" as a test case.
4. **Consistency.** The same policy evaluates identically at the gateway, in a service sidecar, and in CI checks on Terraform plans.
5. **Versioned in Git.** Every rule change has an author, a reviewer, and a reason — which is exactly what SOX attestation asks for.

The trade-offs to name: a new language for the team to learn, latency if the engine is called remotely (mitigate by running it as a **sidecar** so it's a localhost call), and the risk of over-engineering — if your entire authorization model is five roles, plain code is genuinely fine.

> 🎯 **What they're testing:** whether you reach for tools proportionally, and whether you know the sidecar deployment pattern.

---

## D · Tokens & Sessions

### Q17. JWT vs. opaque tokens — what's the trade-off?

**Answer.**

| | JWT (self-contained) | Opaque (reference) |
|---|---|---|
| **Validation** | Local signature check against a cached public key. No network call. | Requires an introspection call to the IdP on every request. |
| **Latency & scale** | Excellent — scales to any RPS | Adds a network hop and makes the IdP a hard dependency on every request |
| **Revocation** | Hard — valid until `exp` unless you add a blocklist | Easy — delete it server-side, effective immediately |
| **Confidentiality** | Claims are Base64, readable by anyone holding it | Reveals nothing; it's a random string |
| **Size** | Large; sent on every request | Small |

🏦 MoneyGuard's choice: **short-lived JWTs (10–15 min) as access tokens**, validated locally at the gateway for speed, *plus* a distributed revocation list for instant kill. **Opaque refresh tokens** stored server-side, because those live for hours or days and must be genuinely revocable.

> 💡 The line to say: *"Pure statelessness is a myth in a high-security system. The real design question isn't stateless-vs-stateful — it's choosing deliberately **where** to reintroduce state, and paying for it once at the edge rather than on every hop."*

> 🎯 **What they're testing:** whether you treat JWTs as a default or as a trade-off with a known cost.

---

### Q18. If JWTs are stateless, how do you revoke one?

**Answer.** You can't revoke the token itself — a signed JWT is valid mathematics until it expires. So you make the system stop honouring it, with a layered approach:

1. **Keep lifetimes short.** 10–15 minutes bounds the worst case. This alone gets you most of the way and costs nothing but refresh traffic.
2. **A distributed revocation list at the edge.** A small, extremely fast store (Redis) holding revoked `jti` values or subject IDs. Every API Gateway instance checks it before honouring a token. Entries expire automatically when the token would have expired anyway, so the list stays tiny — it only ever holds tokens revoked *early*.
3. **Revoke the refresh token server-side**, so no new access tokens can be minted.
4. **A "not valid before" timestamp per user.** Store `sessions_invalid_before = <now>` for that subject; reject any token issued earlier. This kills *all* of a user's sessions with one write, which is what you want on account compromise.
5. **Push logout to the IdP** via OIDC back-channel logout so federated applications drop the session too.

⚠️ The mistake to avoid: checking the primary database on every request. At 10,000 RPS that's a database meltdown, and you've reinvented opaque tokens with worse performance. The revocation check must be an in-memory lookup at the edge.

🏦 This is exactly what makes MoneyGuard's 60-second offboarding requirement achievable: SCIM disables Sam in Entra ID *and* publishes a revoke event, so his live token dies in seconds instead of surviving until expiry.

> 🎯 **What they're testing:** whether you know the revocation-list pattern, and whether you'd naively hit the database.

---

### Q19. Explain JWKS and how you'd rotate signing keys without downtime.

**Answer.** **JWKS** (JSON Web Key Set) is a public endpoint — typically `/.well-known/jwks.json` — where the IdP publishes the **public** halves of its signing keys, each with a key ID (`kid`). A JWT's header carries the `kid` of the key that signed it, so a validator knows which public key to use. Validators cache the whole set, which is why token validation needs no network call and survives an IdP outage.

**Rotation without downtime** is an overlap procedure:
1. **Generate** a new key pair. Publish the new public key to JWKS alongside the old one — but keep signing with the old key.
2. **Wait** long enough for every validator's JWKS cache to refresh (say, 2× the cache TTL). Now everyone holds both keys.
3. **Switch signing** to the new key. Tokens now carry the new `kid`; validators already have it.
4. **Wait** for all tokens signed by the old key to expire.
5. **Remove** the old public key from JWKS and destroy the private key.

The rule: **publish before you use, and remove long after you stop.**

⚠️ Two operational traps. First, a validator that caches JWKS forever will break at step 3 — every validator needs a bounded TTL and a "fetch on unknown `kid`" fallback (rate-limited, so an attacker can't use unknown `kid`s to DoS your IdP). Second, if the emergency case is a *compromised* key, you skip the graceful overlap and revoke immediately — accepting that every token signed with it becomes invalid at once. That's an incident, and it's the reason to rehearse rotation while it's routine.

> 🎯 **What they're testing:** whether you've operated a key rotation. The publish-before-use ordering and the cache-TTL trap are the details that prove it.

---

### Q20. How do you design session management for the MoneyGuard Mobile App?

**Answer.** A mobile app is a **public client** — no client secret can be protected on a device someone can decompile. That drives everything.

- **Login:** Authorization Code + **PKCE**, opened in a system browser (`ASWebAuthenticationSession` / Custom Tabs), never an embedded webview. Embedded webviews let the app read the user's credentials, which defeats the point of federation and trains users to accept credential prompts inside apps.
- **Best practice: broker the code exchange through MoneyGuard's own backend.** The app hands the code and verifier to MoneyGuard's API, which exchanges them and returns its own session token. IdP tokens never live on the device.
- **Storage:** platform secure storage only — iOS Keychain, Android Keystore. Never `UserDefaults`, `SharedPreferences`, or a local file.
- **Token lifetimes:** short access tokens (~15 min) with a longer refresh token, and **refresh token rotation** — each use issues a new one and invalidates the old. If an old refresh token is ever replayed, that's proof of theft, and the whole family gets revoked.
- **Bind the session to the device** where possible — device attestation, or binding the refresh token to a key in the secure enclave, so a stolen token is useless on another device.
- **Step-up for money movement.** A valid session is not consent for a wire transfer. Biometric re-authentication is required, checked server-side via the `acr` claim.
- **Logout must reach the server** — clearing local storage alone leaves a live refresh token in existence.

> 🎯 **What they're testing:** public-client awareness, secure storage, and refresh token rotation. The rotation-detects-theft insight is a strong differentiator.

---

### Q21. Where should a browser-based single-page app store its tokens?

**Answer.** Ideally, **it shouldn't hold them at all.**

The strongest pattern is **Backend-for-Frontend (BFF)**: a server-side component performs the OIDC flow, holds the OAuth tokens server-side, and gives the browser only a `Secure`, `HttpOnly`, `SameSite` session cookie. JavaScript cannot read an `HttpOnly` cookie, so an XSS vulnerability — which is a *when*, not an *if*, on any large frontend — cannot exfiltrate a token.

If you're forced into a pure SPA with no backend:

| Location | Verdict |
|---|---|
| `localStorage` | ⚠️ Readable by any script on the page. One XSS or one compromised npm dependency and every token leaves. Common, but weak. |
| `sessionStorage` | Marginally better (per-tab, cleared on close), same XSS exposure. |
| In-memory JS variable | Better — nothing persists to disk — but lost on refresh, so you need a silent re-auth flow. |
| `HttpOnly` cookie | ✅ Best. Requires a server component and CSRF defences (`SameSite`, anti-forgery tokens). |

🏦 MoneyGuard's Employee Portal uses BFF: the .NET backend completes the code exchange and issues an encrypted `HttpOnly` session cookie. React never touches an OAuth token, so XSS token theft is structurally eliminated rather than mitigated.

> 🎯 **What they're testing:** whether you default to `localStorage` because it's convenient, or reach for BFF because you've thought about XSS.

---

## E · Identity Lifecycle

### Q22. SCIM vs. JIT provisioning — when do you use each?

**Answer.**

| | **SCIM** | **JIT** |
|---|---|---|
| **When the account appears** | Pushed ahead of time, on the HR event | Created on the user's first successful login |
| **Deprovisioning** | ✅ Explicit — a `PATCH active:false` arrives | ❌ None. The user just... stops logging in. The account lingers forever. |
| **Pre-configuration** | ✅ Admins can assign groups before day one | ❌ Nobody can find the user until they've logged in |
| **Setup cost** | Higher — an endpoint to build and operate | Nearly free — falls out of the SSO flow |

🏦 For MoneyGuard, **SCIM is the primary path**, for two reasons:

1. **Priya's day one works.** Raj can assign her to the branch group and her teller entitlements *before* she starts. With JIT, she logs in on Monday morning to an empty account and nobody can pre-configure her because she doesn't exist yet.
2. **Deprovisioning — the real reason.** JIT has *no offboarding signal at all*. When Sam's contract ends, JIT does nothing; his account and its access simply persist. Those are **zombie accounts**, and they're the single most common finding in an access audit. SCIM's `PATCH {"active": false}` is what makes the 60-second kill switch possible.

JIT remains as a **fallback** for federated partner users who will never exist in Workday — a PartnerBank employee, for example. But anyone JIT-provisioned needs a compensating control: an expiry, and inclusion in the reconciliation sweep.

> 🎯 **What they're testing:** whether you name deprovisioning as the deciding factor. Candidates who focus only on onboarding have never sat through an audit.

---

### Q23. Why should you never key user identity on email address?

**Answer.** Because **email is a mutable attribute, not an identifier.**

🏦 Priya Sharma marries and becomes Priya Rao. HR updates Workday; her email changes from `priya.sharma@moneyguard.com` to `priya.rao@moneyguard.com`. If any downstream system joined on email:
- The SCIM update either creates a **duplicate account** (now she has two, one carrying her real entitlements) or fails to match at all.
- Her audit history splits across two identities, so Dana Cole cannot produce a complete trail of her actions — a direct compliance failure.
- Worst case: her old address is recycled to a new hire, who inherits her access.

The fix is the **`externalId` anchor** — the immutable identifier from the system of record (Workday's employee number, or a directory object GUID). It never changes across renames, department moves, or email changes. Email becomes what it actually is: a display and contact attribute.

> 💡 The rule: *"`externalId` is the anchor. Email is a label. Never join on a label."*

The same principle applies in federation: use the IdP's stable subject identifier (`sub` in OIDC, a persistent NameID in SAML), not the email attribute — even though the email is right there and tempting.

> 🎯 **What they're testing:** hard-won experience. This is a bug people only truly learn once, painfully.

---

### Q24. Walk me through what happens when Priya is promoted from Teller to Loan Officer.

**Answer.** This is the **Mover** event, and it is the one organisations get wrong.

1. **Workday** records the job code change and emits a Mover event.
2. **SCIM** picks it up and computes the delta between her old and new entitlement sets.
3. **Grant** the new: loan origination system access, higher approval limits, the loan officer group.
4. **⚠️ Revoke the old:** teller-specific access, cash drawer permissions, branch till reconciliation. **This is the step that gets skipped.**
5. **Invalidate** her cached authorization decisions so the change bites immediately, not after a TTL.
6. **Re-evaluate separation of duties.** Loan Officer plus a residual Teller permission might now let one person both originate and disburse a loan — an SoD violation that neither role creates alone. The system must detect the *combination*.
7. **Notify her new manager** to attest to the resulting access.
8. **Log the whole delta** to Sentinel as an audit event.

The failure mode if step 4 is skipped is **privilege creep**. Run it for five years across a career and Raj Patel holds the union of every job he's ever had — which is precisely how one phished account becomes a bank-wide breach. Detective controls (quarterly recertification, "access anomalous versus peers" reports) catch what the preventive control missed, but the preventive control is revoke-on-move.

> 🎯 **What they're testing:** whether you volunteer the revoke step unprompted, and whether you spot the SoD re-evaluation. Almost everyone describes the grant.

---

### Q25. SCIM is a push protocol over an unreliable network. How do you handle failures and race conditions?

**Answer.** Assume every push can fail, arrive twice, or arrive out of order.

- **Idempotency.** Every operation is keyed on `externalId`, so replaying a create is a no-op rather than a duplicate. `PATCH active:false` applied twice is harmless.
- **Retries with exponential backoff**, and a **dead-letter queue** for operations that exhaust retries. Critically, a DLQ nobody watches is a liability — deprovisioning failures must page a human, because a silently failed offboarding is exactly the scenario an attacker needs.
- **Ordering.** Events carry a sequence number or version from the source. If a stale update arrives after a newer one, discard it. Otherwise a delayed "activate" can resurrect a deactivated account — the worst possible race.
- **Prioritise deprovisioning.** Deactivations jump the queue ahead of profile updates. A delayed name change is cosmetic; a delayed termination is a breach window.
- **Nightly reconciliation.** Compare Workday's full roster against every downstream directory and report the diff: accounts downstream with no active Workday owner (**orphans**), and Workday employees missing downstream (**gaps**). This is the safety net that catches every push that silently failed.
- **Monitor lag.** Alert if the time between a Workday event and its downstream application exceeds the 60-second law. You cannot claim an SLA you don't measure.

> 💡 *"SCIM is eventually consistent. Reconciliation is what converts 'eventually' into a number you can put in front of an auditor."*

> 🎯 **What they're testing:** distributed systems thinking applied to identity, and whether you know reconciliation is mandatory rather than optional.

---

## F · Machine & Workload Identity

### Q26. Why are static API keys a bad architecture, and what replaces them?

**Answer.** Five structural problems, none of which are fixable by "storing the key better":

1. **They never expire.** A key stolen today works in three years.
2. **They leak into source control.** Public GitHub is full of them; scanning bots find them within minutes of a push.
3. **They're hard to rotate.** Rotation means a coordinated redeploy of everything using it, so in practice it never happens.
4. **They're over-privileged.** Because scoping is effort, keys get broad permissions "to be safe."
5. **They're not attributable.** A shared key tells you *something* called the API, not *who*.

**The replacement, in ascending order of strength:**

| Approach | How it works | 🏦 MoneyGuard use |
|---|---|---|
| **OAuth Client Credentials** | Service trades a `client_id`/`client_secret` for a short-lived, narrowly-scoped token | Baseline for internal service-to-service calls |
| **Private Key JWT** | Service signs a short-lived assertion with a private key; server verifies with the public key. **No shared secret ever transmitted.** | Anything touching the Core Banking Ledger |
| **mTLS** | Client presents an X.509 certificate during the TLS handshake; connection is refused at the transport layer without it | Ledger APIs — a stolen token is useless from an unrecognised machine |
| **Workload identity (SPIFFE / IRSA)** | The *platform* attests the workload's identity and injects a short-lived, auto-rotating token. **No secret exists to steal.** | The Fraud Detection Pipeline's Kubernetes pods |

🏦 The Fraud Detection Pipeline gets a projected, auto-rotating identity token from the cluster proving "I am the fraud-scorer service account in the production namespace," exchanges it for a 15-minute token scoped to `transactions:read`, and calls the ledger over mTLS.

> 💡 The framing: *"This solves **secret zero** — not by storing the secret more securely, but by not having one. The workload's identity is derived from the platform, like DNA, rather than something it carries, like a password."*

> 🎯 **What they're testing:** whether you know the progression, and whether you can articulate why platform-attested identity is categorically different from a well-stored secret.

---

### Q27. How do you manage the lifecycle of non-human identities?

**Answer.** This is the gap in almost every organisation. Human identities have Workday firing lifecycle events; service accounts get created during an incident at 2am and live forever.

The programme:

1. **Every workload identity has a registered human owner and an owning team.** An identity nobody owns cannot be recertified, and will never be deleted.
2. **Created through the same IaC pipeline as everything else** — defined in Git, reviewed, applied. Never clicked into a console.
3. **Given an expiry from birth.** Default 90 days, requiring active renewal. Renewal-by-default is how immortal accounts happen.
4. **Scoped to a purpose.** `fraud-pipeline-transaction-reader`, not `fraud-service-admin`. The name should make over-privilege visibly wrong.
5. **Tied to infrastructure lifecycle.** When the Fraud Detection Pipeline is decommissioned, its identity is destroyed by the same Terraform apply that removes the deployment.
6. **Included in reconciliation and recertification** alongside human accounts. Owners attest quarterly that the identity is still needed with its current scope.
7. **Monitored for behaviour.** Baseline what each workload normally does; alert when it touches a new data class, spikes in volume, or acts from an unexpected region. Workloads are highly predictable, which makes anomaly detection unusually effective for them — far more so than for humans.

⚠️ The specific thing to call out: *"In a modern bank, machine identities outnumber human ones several times over. A least-privilege programme that covers only employees covers the minority of your attack surface."*

> 🎯 **What they're testing:** whether you extend governance thinking beyond humans. Very few candidates do.

---

### Q28. An AI agent needs to act on a user's behalf against your APIs. How do you design its IAM?

**Answer.** Treat agents as **first-class identities with stricter guardrails than humans** — they act faster, at greater volume, and with less judgement.

**Clarify first:** are they long-lived or ephemeral? Do they act autonomously or strictly on a user's behalf? What data sensitivity is in scope?

**The design:**

1. **Prefer delegation over direct authority.** The agent should act *on behalf of* a user through an OAuth-style consent flow with explicitly scoped, short-lived tokens — not hold broad standing credentials of its own. The user can see and revoke what they granted.
2. **Never exceed the delegating user's permissions.** The effective permission is the *intersection* of the agent's scope and the user's rights. An agent acting for Priya can never do more than Priya can.
3. **Purpose-built narrow roles.** `statement-summariser`, not `banking-admin`. No agent gets a role that can grant roles — that's the escalation path.
4. **Zero standing privilege.** Short-lived, scoped, revocable tokens obtained per task.
5. **Humans in the loop for high-risk actions.** The agent may *request* a money transfer or a permission change; a human approves it, and the IAM backend performs it with an SoD check. The agent never has unilateral authority over irreversible actions.
6. **Context-aware ABAC.** Environment, data sensitivity, and time factor into every decision. A test agent can never touch production customer data regardless of its nominal role.
7. **Full traceability.** Every action logs the agent ID, the delegating user, the resource, the action, the result, and the reason. The question you must be able to answer is *"who did what, why, and under whose authority?"*
8. **Behavioural baselines and rate limits.** Alert on unexpected data classes, volume spikes, or unusual timing — and cap request rate, because a looping agent can exfiltrate at machine speed.

> 🎯 **What they're testing:** whether you apply existing IAM primitives rigorously rather than inventing something novel. Delegation-over-authority and the intersection rule are the two highest-signal points.

---

## G · MFA & Advanced Authentication

### Q29. Not all MFA is equal. Rank the factors and explain phishing resistance.

**Answer.**

| Factor | Strength | Why |
|---|---|---|
| **SMS one-time code** | ⚠️ Weakest | SIM swapping, SS7 interception, and trivially phishable — the user types it into whatever site asked. |
| **TOTP authenticator app** | Moderate | No SIM attack surface, but still phishable: a proxy page collects the code and replays it within its 30-second window. |
| **Push notification** | Moderate | Phishable via **MFA fatigue** — spam approvals at 3am until someone taps. Number matching helps considerably. |
| **Passkeys / FIDO2 / WebAuthn** | ✅ Strongest | **Phishing-resistant by construction.** |

**Why passkeys are categorically different:** the credential is a key pair bound to a specific **origin** (`moneyguard.com`) and stored in hardware. When Eve sends Priya a link to `moneyguard-secure-login.com`, the browser checks the origin, finds no matching credential, and *refuses to sign*. It's not that Priya is too careful to be fooled — the protocol makes the credential unusable on the wrong site. Human vigilance is removed from the security model.

There's also nothing to steal in a breach: the server holds only public keys. And the private key never leaves the authenticator's secure hardware.

🏦 MoneyGuard's position: passkeys as the default for all employees, TOTP as an accessibility fallback, and SMS retired entirely for staff. For retail customers, SMS may persist for reach reasons — but never as the *only* factor for money movement.

> 🎯 **What they're testing:** whether you know MFA is a spectrum, and whether you can explain origin binding rather than just naming FIDO2.

---

### Q30. What is step-up authentication and where do you enforce it?

**Answer.** Step-up means requiring a **stronger, fresher** authentication for a specific high-risk action, even though the user already has a valid session. The core insight: **a valid session is not blanket consent for every possible action.**

🏦 Alex Kim has been logged into an internal tool for two hours on a password-only session. He initiates a $2M wire transfer. The API inspects the **`acr`** (Authentication Context Class Reference) claim in his token, sees only password-level assurance where the action requires hardware-key assurance, and rejects the request with a challenge. The UI prompts Alex to tap his passkey right now; a fresh token is issued with an elevated `acr`, and the transfer proceeds.

**Where to enforce it: server-side, at the API.** This is the critical detail. Prompting in the UI is a user-experience nicety — Eve does not use MoneyGuard's UI. She calls the API directly. The `acr` check must happen where the action is executed.

**What triggers it:**
- **Action sensitivity** — money movement above a threshold, changing payment details, granting permissions, bulk data export.
- **Risk signals** — new device, unusual location, impossible travel, anomalous behaviour. Adaptive rather than static.
- **Freshness** — even a strong factor goes stale. A passkey tap from four hours ago may not satisfy a wire transfer; require `auth_time` within a recent window.

The trade-off: step up too often and users habituate to approving prompts, which erodes the control. Reserve it for genuinely consequential actions.

> 🎯 **What they're testing:** the `acr` claim by name, and *server-side enforcement*. The second one is what separates a security engineer from a frontend developer.

---

## H · Multi-Tenancy & Data Isolation

### Q31. Compare siloed, pooled, and bridge tenancy models. Which would you pick for the Business Banking SaaS Portal?

**Answer.**

| Model | Structure | Isolation | Cost | Operations |
|---|---|---|---|---|
| **Siloed** | A separate database (or stack) per tenant | Strongest — physical | Highest | Painful: N databases to migrate, patch, back up, and monitor |
| **Pooled** | Shared tables, `tenant_id` on every row | Logical, enforced by RLS | Lowest | Easiest: one schema, one migration |
| **Bridge** | Pooled by default; siloed for tenants who need it | Per-tenant choice | Mixed | Moderate — two paths to maintain |

🏦 For the Business Banking SaaS Portal with hundreds of corporate clients like Acme Corp and Globex Inc, I'd choose **bridge, defaulting to pooled**:

- **Pooled for the majority.** Hundreds of small business clients cannot each justify a dedicated database. One schema means one migration, and resource efficiency for tenants with wildly uneven usage.
- **Siloed as a premium tier** for the small number of large clients with a contractual or regulatory requirement for physical separation — they pay for it, so the operational cost is funded rather than absorbed.

The decisive risk factor to name: **the noisy-neighbour and blast-radius trade-off.** Pooled means one tenant's runaway query can degrade everyone, and one authorization bug can expose everyone. That's why pooled *requires* Row-Level Security — see Q32 — and per-tenant rate limiting. Without those, "pooled" is just "shared, and hoping."

> 🎯 **What they're testing:** whether you pick based on business constraints rather than declaring one model correct, and whether you name blast radius.

---

### Q32. How do you guarantee Acme Corp can never see Globex Inc's data?

**Answer.** Three independent layers, each assuming the one above it has a bug.

**Layer 1 — the token.** Every access token carries a validated `tenant_id`, set by the IdP from the user's directory record. The client never supplies it, because anything the client supplies is attacker-controlled.

**Layer 2 — the application.** Middleware extracts the tenant from the *validated* token and sets it on the request context. Every query scopes to it. This layer will eventually have a bug — someone will write a query without the filter — which is why it isn't the last layer.

**Layer 3 — the database.** PostgreSQL **Row-Level Security**:

```sql
ALTER TABLE payroll_batches ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON payroll_batches
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

The middleware sets `app.current_tenant` at the start of every transaction. Now a developer who forgets a `WHERE tenant_id = ...` clause gets an **empty result set** instead of Globex Inc's payroll rendered in Acme Corp's browser. The application bug degrades to a functional bug rather than a breach.

⚠️ **Three things that bypass all of this, and must be handled explicitly:**
- **Background jobs, reports, and exports.** They run without a user context, so nothing sets the tenant variable. This is where multi-tenant leaks actually happen in production. Every non-request code path needs an explicit tenant scope.
- **The application's database role.** If it's a superuser or has `BYPASSRLS`, the policy silently does nothing. Verify the connecting role is subject to RLS — and test it.
- **Caches and object storage keys.** A cache key without a tenant prefix serves one tenant's data to another. Same for file paths.

**Prove it, don't assert it:** automated tests that authenticate as Acme Corp and attempt to read a known Globex Inc row, run in CI on every commit. And a Sentinel detection for any query returning rows across multiple tenant IDs.

> 🎯 **What they're testing:** defence in depth, and whether you know about the batch-job bypass. That last one is the detail that says you've operated a multi-tenant system.

---

## I · Resilience, Scale & Observability

### Q33. Your authorization service goes down. Do you fail open or fail closed?

**Answer.** **Fail closed** for anything touching money or sensitive data — if we cannot verify a permission, the answer is no. In a bank, an outage is recoverable; unauthorised access to the Core Banking Ledger is not.

But answer it with more precision than the slogan, because that's where the real signal is:

1. **Distinguish the error from the denial.** Return `503 Service Unavailable`, not `403 Forbidden`. A `403` tells the user they lack permission, sends them to the help desk, and hides an outage from the on-call engineer. The status code is a diagnostic signal, and getting it wrong costs real incident minutes.
2. **Tier the decision by risk.** Blanket fail-closed sounds rigorous but takes the whole bank offline when one cache node hiccups. For genuinely low-risk reads — a user's own profile photo — serving a slightly stale cached decision is a reasonable availability trade. For a wire transfer, never.
3. **Engineer so the choice rarely arises.** Multi-AZ deployment with automatic failover, cross-region read replicas, a stateless horizontally-scaled service, and — most importantly — **caches that keep serving during a backend outage**. A well-warmed L2 cache means most traffic survives a policy-store failure entirely.
4. **Degrade the surface, not the safety.** Put the API in read-only mode so customers can still see balances while writes are suspended. Partial service beats a blank page.
5. **Alert immediately and loudly.** A fail-closed system under load looks exactly like a mass permissions bug to users. The SOC needs to know it's an outage within seconds.

> 💡 The sentence: *"Fail closed on the ledger, because Law 4. The cost is that an authorization outage becomes a customer-facing outage — so I'd invest in caching and multi-AZ specifically so that trade-off is rarely tested."*

> 🎯 **What they're testing:** whether you can give a nuanced answer rather than the slogan, and whether you know the 503-vs-403 distinction.

---

### Q34. How does the CAP theorem apply to an IAM system?

**Answer.** Under a network partition you must choose consistency or availability — and **IAM is one of the few domains where the answer genuinely differs by operation.**

| Operation | Choose | Why |
|---|---|---|
| **Granting access** | Availability (eventual consistency is fine) | A new permission taking 30 seconds to propagate globally is a minor inconvenience. |
| **Revoking access** | **Consistency** | A revocation that hasn't propagated is an open door. This must be strongly consistent, or as close as achievable. |
| **Reading a decision** | Availability, with bounded staleness | 10,000/sec cannot be strongly consistent globally and stay fast. Cache with short TTL and event invalidation. |
| **Writing audit logs** | Availability (never block the request) | Asynchronous, buffered, at-least-once. But the buffer must be durable — losing audit data during an incident is losing the evidence. |

🏦 MoneyGuard's design: policy **writes** go to a global primary and replicate asynchronously; **reads** come from local replicas and caches. **Revocations bypass the normal path entirely** and are pushed directly to every region's revocation list, because that is the one operation where a stale read is a security incident rather than a UX blemish.

> 💡 *"Grants can be eventually consistent. Revocations cannot. Designing both on the same path is the mistake."*

> 🎯 **What they're testing:** whether you can apply a distributed-systems concept to a domain rather than reciting the theorem. The grant/revoke asymmetry is the insight.

---

### Q35. What do you log, and how would you detect an insider threat?

**Answer.** **What to log** — structured JSON, never free text, because the consumer is a machine:

| Category | Examples |
|---|---|
| **Authentication** | Login success and failure, MFA challenges, step-up events, factor used |
| **Authorization** | Decisions on sensitive resources — including `ALLOW`, not just `DENY` |
| **Lifecycle** | Account create/disable, role grants and revocations, group changes |
| **Administrative** | Policy changes, configuration changes, break-glass elevation |
| **Data access** | Reads and writes against the Core Banking Ledger |

Every event carries: actor, action, resource, result, timestamp, source IP, device, tenant, and a **trace ID** stitching one request across every component.

⚠️ **Log successes, not just failures.** The instinct is to log `DENY` because it looks like an attack. But an insider breach is a sequence of *individually authorised* actions — Dana Cole reading one customer record is normal; reading fifty thousand is not. If you log only denials, the actual breach is invisible.

**Detecting the insider** — no single event is suspicious, so detection is about **correlation and baselines** in Sentinel:

| Signal | What it catches |
|---|---|
| Volume anomaly vs. the user's own 30-day baseline | Bulk exfiltration disguised as normal work |
| Volume anomaly vs. peers in the same role | A teller behaving unlike every other teller |
| Access outside normal hours or geography | Compromised credentials, or after-hours theft |
| Breadth: accounts touched with no business relationship | Browsing rather than servicing customers |
| Sequence: privilege grant immediately followed by its use | Self-escalation |
| Behaviour change right after a Workday resignation event | The highest-value correlation available — a departing employee taking data |
| Access to audit or logging systems | Someone covering their tracks |

**Storage must be append-only and tamper-evident.** An attacker's last move is deleting the evidence — if the credentials that grant ledger access can also edit the log, the log proves nothing.

And **close the loop:** detection without a response path is an expensive way to be told you were breached. When Sentinel fires, the SOC can push the session onto the edge revocation list and kill it bank-wide in seconds.

> 🎯 **What they're testing:** the log-successes insight, the resignation correlation, and whether you connect detection to response.

---

### Q36. How would you migrate MoneyGuard from legacy AD-only authentication to modern SSO without a big-bang cutover?

**Answer.** Never big-bang an identity migration — an authentication outage locks 50,000 people out of everything simultaneously.

**Phase 1 — Discover.** Inventory every application and how it authenticates: LDAP bind, Kerberos, forms auth against AD, or a local user table. Expect the inventory to be wrong and budget for discovery. Classify by criticality and by which protocols the app can support.

**Phase 2 — Establish the new IdP alongside the old.** Stand up Entra ID and sync from AD so both directories hold the same users with the same `externalId` anchor. Nothing has moved yet; both paths work.

**Phase 3 — Migrate applications, not users.** One app at a time, lowest-risk first. Modern apps take OIDC directly; older ones sit behind an application proxy or a SAML shim. Each migration is independently reversible, which is the entire point — a failed cutover affects one app, not the bank.

**Phase 4 — Handle what cannot move.** Some 30-year-old systems will never speak OIDC. Options: leave them on AD with tightened compensating controls (network segmentation, privileged access management, enhanced monitoring), or front them with a gateway that translates. Be honest that a residual legacy estate is normal, and plan its containment rather than pretending it will vanish.

**Phase 5 — Move the source of truth.** Once the majority is on Entra ID, flip provisioning so Workday → SCIM → Entra ID is primary, and AD becomes a downstream consumer rather than the master.

**Phase 6 — Decommission, carefully.** Only after monitoring proves zero authentication traffic against the old path for a sustained period. Turning off a directory that something still silently depends on is how a migration becomes an outage.

**Throughout:** run both paths in parallel with a documented, *rehearsed* rollback for every phase; measure authentication success rate per application as the primary health metric; and communicate to the help desk before each wave, because they are the first to know when it's going wrong.

> 🎯 **What they're testing:** migration pragmatism and risk management. The strongest signals are strangler-fig incrementalism, "migrate apps not users," and acknowledging that some legacy will never move.

---

## 📝 Rapid-Fire Revision

| Q | One-line answer |
|---|---|
| AuthN vs AuthZ? | Who you are (once per session) vs what you may do (every request). |
| Third A in AAA? | Accounting — the audit trail. The one people forget. |
| OAuth 2.0 is for…? | Delegated **authorization**. Not authentication. |
| OIDC adds what to OAuth? | The **ID Token** — a signed JWT proving who logged in. |
| `state` stops? | CSRF / login fixation. |
| PKCE stops? | Authorization code interception. |
| `nonce` stops? | ID Token replay. |
| Revoke a JWT? | Short lifetime **plus** a distributed revocation list at the edge. |
| Roles in the JWT? | No — immutable token = cache with no invalidation. |
| SCIM beats JIT because? | JIT has **no deprovisioning signal**. |
| Join identities on? | `externalId`, never email. |
| Kill static API keys with? | Workload identity — platform-attested, no secret exists. |
| Phishing-resistant MFA? | Passkeys / FIDO2 — credential bound to the origin. |
| Step-up checks which claim? | `acr`, **server-side at the API**. |
| Tenant isolation last line? | Row-Level Security in the database. |
| Fail open or closed? | Closed on money. Return **503**, not 403. |
| CAP in IAM? | Grants can be eventual; **revocations must be consistent**. |
| Log denials only? | No — insider breaches are made of authorised successes. |
| Many PEPs, how many PDPs? | Many enforcement points, **one** decision point. |

---

⬅ [02 · Full Architecture Walkthrough](./02_MoneyGuard_Full_Architecture_Walkthrough.md) | [Module 11 Index](./README.md) | Next: [04 · Practice Exercises](./04_Practice_Exercises.md) ➡

⬅ Back to [Main README](../README.md) | 🏦 [The MoneyGuard Scenario](../SCENARIO.md)
