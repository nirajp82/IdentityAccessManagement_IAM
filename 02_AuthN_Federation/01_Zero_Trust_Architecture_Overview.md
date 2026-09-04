# Zero Trust Architecture Overview

## Overview

**Zero Trust Architecture (ZTA)** is a security model built on one blunt rule: **"never trust, always verify."**

In the old model — often called **castle-and-moat** — a company drew a network boundary (the moat). Anything inside the boundary was trusted. If you were plugged into a desk at a MoneyGuard branch, the Core Banking Ledger assumed you were friendly.

Zero Trust deletes the idea of "inside." Every single request — from Priya the teller at a branch counter, from the MoneyGuard Mobile App on a customer's phone, from a background service, or from Eve the attacker on a stolen laptop — must prove **who it is** and **what it is allowed to do**, on every call.

That "prove who it is" half is **Authentication (AuthN)**, and it is what this entire module is about. This chapter is the map: it shows where AuthN and federation sit inside MoneyGuard's Zero Trust architecture, and how the four protocol chapters that follow (SAML, OAuth 2.0, OIDC, and the `state`/`nonce`/PKCE security parameters) plug into that map.

## Why This Matters

🏦 MoneyGuard Bank has ~50,000 employees, millions of retail customers, thirty years of legacy on-premise systems, and a brand-new cloud estate. Its identity problem looks like this on any given Tuesday:

- **Priya Sharma** (new teller) logs into the **MoneyGuard Employee Portal** from a branch PC.
- A **customer** opens the **MoneyGuard Mobile App** on public airport Wi-Fi.
- A **PartnerBank** loan officer — an employee of a *completely different company* — needs into MoneyGuard's joint mortgage portal, and MoneyGuard must never see or store their password.
- **Sam Torres** (contractor) needs read-only access for six weeks and then must lose it automatically.
- **Eve** is running credential-stuffing attacks against the login page around the clock.

There is no network boundary that separates those five cases. A firewall cannot tell Priya from Eve — both arrive as HTTPS traffic. **Identity is the new perimeter**, and the login/federation layer is where that perimeter is actually enforced.

Regulators agree. SOX and PCI-DSS both require MoneyGuard to prove *who* did *what* to financial records. You cannot produce that proof unless every request carries a verifiable identity.

## Key Concepts

| Concept | Plain-English meaning | Who does it at MoneyGuard |
|---|---|---|
| **AuthN (Authentication)** | "Who are you, and can you prove it?" | MoneyGuard Entra ID |
| **AuthZ (Authorization)** | "What are you allowed to do?" | API Gateway + Open Policy Agent |
| **Accounting / Audit** | "What did you actually do?" | MoneyGuard SIEM ("Sentinel") |
| **IdP (Identity Provider)** | The system that verifies credentials and issues proof of login | MoneyGuard Entra ID |
| **Authorization Server** | The system that issues access tokens for APIs | MoneyGuard Entra ID (same product, different job) |
| **RP / SP (Relying Party / Service Provider)** | The app that trusts the IdP instead of storing passwords | Employee Portal, Mobile App, joint mortgage portal |
| **PEP (Policy Enforcement Point)** | The "muscle" that blocks or forwards a request | API Gateway at `api.moneyguard.com` |
| **PDP (Policy Decision Point)** | The "brain" that answers ALLOW or DENY | Open Policy Agent (OPA) |
| **Federation** | Trusting *another organization's* IdP to authenticate its own people | MoneyGuard ↔ PartnerBank via SAML |
| **Token** | A short-lived, signed credential the client presents on every call | JWT access token, ~15 minutes |

## Detailed Notes

### Concept 1 — Castle-and-Moat vs Zero Trust

| | Castle-and-Moat (old) | Zero Trust (MoneyGuard today) |
|---|---|---|
| Trust boundary | The corporate network | Nothing. Every request is untrusted. |
| Check happens | Once, at the VPN | Every single API call |
| Blast radius of a breach | Attacker inside the network reaches everything | Attacker still needs a valid, scoped, unexpired token per resource |
| Works for mobile/SaaS? | Badly | Yes — there is no "inside" to be outside of |

The three practical Zero Trust rules MoneyGuard follows:

1. **Verify explicitly** — authenticate and authorize using identity, device, location, and time, on every request.
2. **Least privilege** — issue tokens with the narrowest scope and shortest lifetime that still gets the job done.
3. **Assume breach** — design as if Eve is already inside the network, because one day she will be.

### Concept 2 — The Two Identity Planes (the most common point of confusion)

People blur "IdP" and "authorization server" together. Keeping them separate makes the whole module click:

- **The Identity Provider (AuthN & Federation)** — the realm of **OIDC** and **SAML**. Its only job: *"Who is this user, and can they prove it?"* It owns login screens, MFA prompts, passkeys, and outbound trust to partner directories like PartnerBank's IdP.
- **The Authorization / Token layer (AuthZ)** — the realm of **OAuth 2.0**. Once identity is established, this layer maps the person to MoneyGuard roles (e.g. `teller`, `wealth_manager`), mints the signed JWT access token, and issues refresh tokens.

> **One-line summary:** OIDC and SAML *log you in*. OAuth 2.0 *gives your app permission to call APIs*.

At MoneyGuard both roles are served by **MoneyGuard Entra ID** at `login.moneyguard.com`, but they are genuinely different responsibilities and are usually configured, tested, and audited separately.

### Concept 3 — PEP vs PDP (the enforcement pair)

These two terms come from the XACML (eXtensible Access Control Markup Language) model, and they always work as a pair.

- **PEP — Policy Enforcement Point.** MoneyGuard's **API Gateway** at `api.moneyguard.com`, plus sidecar proxies (Envoy) next to internal services. It intercepts traffic, pulls the JWT out of the `Authorization: Bearer` header, verifies the signature mathematically, pauses the request, asks for a decision, and enforces the answer. **It does not know the business rules.**
- **PDP — Policy Decision Point.** **Open Policy Agent (OPA)**, deployed right beside the gateway so a decision takes under a millisecond. It evaluates the rules and answers `{"allow": true}` or `{"allow": false}`. **It never touches network traffic.**

💡 Analogy: the PEP is the security guard at the door; the PDP is the guest list. The guard checks your ID and enforces the answer, but the list decides who gets in.

Why split them? MoneyGuard has hundreds of microservices. Embedding policy logic in every gateway means every policy change is a redeploy. With a separate PDP, Raj updates one policy bundle and every enforcement point picks it up.

### Concept 4 — Federation: why MoneyGuard never stores a PartnerBank password

**Federation** means MoneyGuard *outsources* the act of verifying a user to another organization's identity system, and then trusts a cryptographically signed statement about the result.

🏦 For the joint mortgage product, PartnerBank loan officers need access to MoneyGuard's mortgage portal. Two options:

| Option | What happens | Verdict |
|---|---|---|
| Create MoneyGuard accounts for PartnerBank staff | MoneyGuard stores 400 extra passwords, must handle their password resets, and — worst of all — when PartnerBank fires someone, MoneyGuard finds out weeks later, if ever | ❌ |
| **Federate via SAML** | PartnerBank's IdP authenticates its own people and sends MoneyGuard a signed assertion. MoneyGuard stores zero credentials. | ✅ |

The security win is the offboarding story: the moment PartnerBank disables an account in its own directory, that person can no longer produce a valid assertion, so MoneyGuard access dies instantly and automatically. This is covered in depth in [Chapter 02 — SAML](./02_SAML.md).

### Concept 5 — Choosing the protocol

| | **SAML 2.0** | **OAuth 2.0** | **OIDC** |
|---|---|---|---|
| Answers | "Who is this user?" (AuthN) | "What may this app do?" (AuthZ) | "Who is this user?" (AuthN) |
| Data format | XML | JSON over HTTPS | JSON over HTTPS |
| Year | 2005 | 2012 (RFC 6749) | 2014 |
| Core artifact | Signed XML **Assertion** | **Access Token** | **ID Token** (a JWT) |
| Best at | B2B enterprise SSO with legacy partners | Delegated API access, machine-to-machine | Modern web + mobile login |
| Bad at | Mobile apps, SPAs, APIs | Login — it is **not** an authentication protocol | Nothing much; it is the modern default |
| MoneyGuard uses it for | PartnerBank federation, legacy banking software | Employee Portal → Core Banking Ledger API; service-to-service | Employee Portal login, Mobile App login |

⚠️ The single most expensive mistake in this table: **using OAuth 2.0 alone as a login protocol.** An access token proves an app was granted permission; it does not prove *to your app* who logged in. That gap is exactly why OIDC exists. See [Chapter 04 — OIDC](./04_OIDC.md).

### Concept 6 — The token lifecycle

Zero Trust means tokens are short-lived and revocable.

- **Access token: ~15 minutes.** Short enough that a stolen token is nearly worthless, because there is no time to use it.
- **Refresh token: hours to days.** Held only by a confidential backend or a device's secure storage. When the access token expires, the client silently trades the refresh token for a new access token — no MFA prompt, no interruption for Priya.
- **Refresh token rotation.** Every use of a refresh token invalidates it and returns a *new* one. If Eve steals a refresh token and uses it, the legitimate client's next refresh presents an already-used token — MoneyGuard detects the reuse and kills the entire session.
- **Revocation.** A JWT is verified locally by the gateway with a public key, so it cannot simply be "deleted" server-side. MoneyGuard pushes the token's `jti` (JWT ID) onto a Redis blocklist that the gateway checks in under a millisecond, and separately exposes the standard OAuth `/oauth2/revoke` endpoint for refresh tokens.

### Concept 7 — Build vs Buy

| | **Build** (custom identity server) | **Buy** (Entra ID, Okta, CyberArk, Auth0) |
|---|---|---|
| Fits | Companies with unique login flows and a large security engineering team | Regulated enterprises — including MoneyGuard |
| Pros | Total flexibility, no per-user licensing, no vendor lock-in | Bank-grade crypto maintained for you, turnkey MFA, SOC 2 / FedRAMP evidence, SLA |
| Cons | **You** patch every crypto zero-day. One SAML or OAuth implementation bug breaches the bank. | Licensing scales with headcount; login-screen customization has hard limits |

MoneyGuard **buys**. A bank cannot afford a homegrown XML signature parser with a signature-wrapping flaw in it.

## Workflow

Priya opens the Employee Portal and views a customer's balance. Every Zero Trust component appears exactly once:

```mermaid
sequenceDiagram
    autonumber
    actor Priya as Priya (Teller)
    participant Portal as MoneyGuard Employee Portal
    participant IdP as MoneyGuard Entra ID<br/>login.moneyguard.com
    participant PEP as API Gateway (PEP)<br/>api.moneyguard.com
    participant PDP as Open Policy Agent (PDP)
    participant Ledger as Core Banking Ledger

    Priya->>Portal: Opens portal.moneyguard.com
    Portal-->>Priya: 302 Redirect to /authorize (no session yet)
    Priya->>IdP: GET /oauth2/v2.0/authorize
    IdP-->>Priya: Password prompt + MFA challenge
    Priya->>IdP: Credentials + MFA approval
    IdP-->>Priya: 302 Redirect back with ?code=...&state=...
    Priya->>Portal: GET /callback?code=...&state=...
    Portal->>IdP: POST /oauth2/v2.0/token (code + client_secret + PKCE verifier)
    IdP-->>Portal: id_token + access_token + refresh_token
    Note over Portal: Validates ID token: signature, iss, aud, exp, nonce
    Portal-->>Priya: Sets HttpOnly session cookie
    Priya->>PEP: GET /v1/accounts/8842/balance
    PEP->>PEP: Verify JWT signature with cached JWKS public key
    PEP->>PDP: {subject, roles, action, resource, ip, time}
    PDP-->>PEP: {"allow": true}
    PEP->>Ledger: Forward authorized request
    Ledger-->>Priya: 200 OK, balance returned
```

## Architecture Diagram

```mermaid
flowchart TD
    subgraph Clients["1 - Identities"]
        EMP["Employees<br/>Priya, Raj, Dana, Sam"]
        CUST["Customers<br/>MoneyGuard Mobile App"]
        PART["PartnerBank staff<br/>external org"]
        SVC["Workloads<br/>Fraud Detection Pipeline"]
    end

    subgraph IdPLayer["2 - Identity Provider - AuthN"]
        ENTRA["MoneyGuard Entra ID<br/>OIDC + OAuth2 + SAML + MFA"]
        AD["MoneyGuard Active Directory<br/>legacy LDAP / Kerberos"]
        PBIDP["PartnerBank IdP<br/>SAML federation"]
    end

    subgraph Control["3 - Token and Identity Control Plane - AuthZ"]
        STORE["Identity Store<br/>roles and attributes"]
        TOKEN["Token Service<br/>signs JWTs, JWKS endpoint"]
        SCIM["SCIM sync from Workday"]
    end

    subgraph Enforce["4 - Enforcement Layer"]
        GW["API Gateway - PEP<br/>api.moneyguard.com"]
        OPA["Open Policy Agent - PDP"]
        REDIS[("Redis blocklist<br/>revoked jti")]
    end

    subgraph Resources["5 - Protected Resources"]
        LEDGER["Core Banking Ledger"]
        MICRO["Other microservices"]
    end

    SIEM["MoneyGuard SIEM - Sentinel"]

    EMP --> ENTRA
    CUST --> ENTRA
    PART --> PBIDP --> ENTRA
    SVC --> TOKEN
    AD -.sync.-> ENTRA
    ENTRA --> STORE --> TOKEN
    SCIM --> STORE
    TOKEN -.JWKS public keys.-> GW
    EMP --> GW
    CUST --> GW
    GW <--> OPA
    GW <--> REDIS
    GW --> LEDGER
    GW --> MICRO
    GW -.logs.-> SIEM
    ENTRA -.logs.-> SIEM
```

## Step-by-Step Process

1. **Trust is established first, once.** Raj registers each application: for OIDC apps he creates a client with a `client_id`, an allowed `redirect_uri`, and (for confidential clients) a `client_secret`. For PartnerBank he exchanges SAML metadata. No user has logged in yet.
2. **A request arrives with no session.** Priya hits `portal.moneyguard.com`; the portal sees no session cookie.
3. **Redirect to the IdP.** The portal sends her browser to `login.moneyguard.com/oauth2/v2.0/authorize` with `state`, `nonce`, and a PKCE `code_challenge`.
4. **AuthN happens.** Entra ID checks her password and enforces MFA. MoneyGuard's app never sees her password.
5. **Authorization code returns.** The IdP redirects back with a single-use `code` and the echoed `state`.
6. **Back-channel token exchange.** The portal's backend POSTs the code plus its client credentials and PKCE `code_verifier` to `/token`, over a direct server-to-server connection the browser never sees.
7. **Claims mapping.** Before signing, the token service looks up Priya in the Identity Store — populated by SCIM from Workday — and maps `Job Title = Teller` to the role claim `teller`.
8. **Tokens issued.** An `id_token` (who she is, for the portal) and an `access_token` (what the portal may call, for the API) come back.
9. **The API call.** The portal calls `api.moneyguard.com` with `Authorization: Bearer <access_token>`.
10. **Enforcement.** The gateway (PEP) verifies the signature against cached JWKS keys, checks the `jti` against the Redis blocklist, and asks OPA (PDP) for a decision including context: role, source IP, time of day, and amount.
11. **Forward or reject.** ALLOW forwards to the Core Banking Ledger; DENY returns `403 Forbidden` and the ledger is never contacted.
12. **Everything is logged.** Both the login event and the authorization decision are shipped to Sentinel so Dana can reconstruct the whole chain during an audit.

## Commands and Examples

**1. Discovery — every OIDC provider publishes its own configuration**

```bash
curl https://login.moneyguard.com/moneyguard.com/v2.0/.well-known/openid-configuration
```

```json
{
  "issuer": "https://login.moneyguard.com/moneyguard.com/v2.0",
  "authorization_endpoint": "https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/authorize",
  "token_endpoint":         "https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/token",
  "jwks_uri":               "https://login.moneyguard.com/moneyguard.com/discovery/v2.0/keys",
  "revocation_endpoint":    "https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/revoke",
  "response_types_supported": ["code"],
  "scopes_supported": ["openid", "profile", "email", "ledger.read", "ledger.write"]
}
```

**2. A MoneyGuard access token payload (decoded JWT)**

```json
{
  "iss": "https://login.moneyguard.com/moneyguard.com/v2.0",
  "aud": "api://moneyguard-ledger",
  "sub": "8f14e45f-ea1b-4f3d-9c22-7a1e0d6b91c4",
  "preferred_username": "priya.sharma@moneyguard.com",
  "roles": ["teller"],
  "branch": "BR-0142",
  "scp": "ledger.read",
  "jti": "d0f4c7a1-2b55-4a10-91cf-3c9d0e77b2aa",
  "iat": 1772539200,
  "exp": 1772540100
}
```

Note `exp - iat = 900` seconds — a 15-minute token.

**3. Calling a protected resource through the PEP**

```bash
curl -i https://api.moneyguard.com/v1/accounts/8842/balance \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Im1nLTIwMjYtMDIifQ..."
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"accountId":"8842","available":10420.55,"currency":"USD"}
```

Same call at 11 PM on a Saturday from a non-corporate IP:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{"error":"access_denied","reason":"policy: outside_business_hours"}
```

**4. The OPA (Rego) policy behind that decision**

```rego
package moneyguard.ledger

import future.keywords.if
import future.keywords.in

# Secure by default: everything is denied unless a rule says otherwise
default allow := false

allow if {
    input.request.method == "POST"
    input.request.path == "/v1/wires"

    # RBAC: the JWT must carry the role
    "wealth_manager" in input.token.payload.roles

    # ABAC: context must also be right
    is_corporate_ip(input.request.source_ip)
    input.request.body.amount <= 100000
}

is_corporate_ip(ip) if {
    net.cidr_contains("10.50.0.0/16", ip)
}
```

**5. Revoking a stolen token immediately**

```bash
# Raj clicks "Lock account" -> the control plane pushes the jti to Redis
redis-cli SETEX "blocklist:jti:d0f4c7a1-2b55-4a10-91cf-3c9d0e77b2aa" 900 "revoked"
```

The gateway checks that key before every forward, so a stolen token dies in under a second instead of waiting out its 15 minutes.

## Best Practices

- **Authenticate at the edge, authorize at the edge, and log everything.** Backend microservices should not each reimplement security.
- **Keep access tokens short** (5–15 minutes) and refresh tokens rotating.
- **Always use the Authorization Code flow with PKCE**, even for confidential clients. It is free defense-in-depth.
- **Never let browser JavaScript hold a raw token.** Use the Backend-for-Frontend (BFF) pattern with an `HttpOnly; Secure; SameSite=Strict` cookie.
- **Federate rather than duplicate.** If another organization already owns the identity, trust their IdP; do not create shadow accounts.
- **Default deny in policy.** Rego, XACML, and every other policy language should start from `default allow := false`.
- **Feed every AuthN and AuthZ event to the SIEM** with a correlation ID, so Dana can trace one request across five systems.
- **Buy your identity platform** unless identity *is* your product.

## Common Mistakes

- ⚠️ **Treating the VPN as the security boundary.** "It's on the internal network" is not an authorization decision.
- ⚠️ **Using an OAuth access token as proof of login.** It answers "what may this app do", not "who is this person". This is the classic confused-deputy setup.
- ⚠️ **Long-lived access tokens** (hours or days) "for convenience". You have just handed Eve a long window.
- ⚠️ **Storing tokens in `localStorage`.** One XSS bug anywhere on the page reads them all.
- ⚠️ **Skipping `state`, `nonce`, or PKCE** because "the flow works without them." It works right up until it is attacked. See [Chapter 05](./05_State_Nonce_And_PKCE.md).
- ⚠️ **Putting policy logic inside every microservice.** Five hundred teams will implement it five hundred slightly different ways, and Dana will find every one of them.
- ⚠️ **Trusting a signed token without checking `aud` and `iss`.** A perfectly valid token issued for a *different* audience is still an attack.

## Pro Tips

- 💡 **Draw the two planes on the whiteboard first.** In an interview, sketching "IdP = AuthN, token service = AuthZ, gateway = PEP, OPA = PDP" before writing any protocol detail instantly signals you understand the architecture, not just the acronyms.
- 💡 **JWKS caching is what makes Zero Trust affordable.** The gateway fetches public keys once an hour and then verifies 50,000 tokens per second with pure local CPU math — zero database traffic.
- 💡 **If the IdP goes down, existing sessions survive.** Nobody can get a *new* token, but the gateway validates existing JWTs locally. Say this out loud in an interview; it demonstrates you understand statelessness.
- 💡 **Use `jti` from day one.** Without a unique token ID, you cannot build a revocation blocklist later without reissuing your whole token format.
- 💡 **Zero Standing Privileges.** Nobody should hold admin rights by default. Grant them just-in-time, time-boxed to the change ticket, and let the token expire rather than relying on someone remembering to clean up.

## Real-World Use Cases

1. **🏦 Corporate federation (B2B).** A PartnerBank loan officer opens MoneyGuard's joint mortgage portal, is bounced to PartnerBank's own IdP, and returns with a signed SAML assertion. MoneyGuard stores no PartnerBank credential. When PartnerBank offboards them, access ends the same hour.
2. **🏦 Automated offboarding.** HR marks Sam Torres "contract ended" in Workday. SCIM pushes the change to the Identity Store within minutes; his roles are stripped and his refresh token is revoked. His current access token dies at its 15-minute mark.
3. **🏦 Machine-to-machine.** The Fraud Detection Pipeline needs to read from the Core Banking Ledger. No human, no MFA — it uses the OAuth 2.0 Client Credentials grant to get a 5-minute token scoped to `ledger.read` only. If that service is compromised, Eve still cannot write to the ledger.
4. **🏦 Just-in-time admin access.** A database administrator has zero standing production rights. They request `db_admin_prod` against ticket REQ-998, a manager approves, they re-authenticate with a hardware key, and receive a JWT that expires in exactly two hours. No manual cleanup.
5. **🏦 Stolen laptop.** Priya's laptop is taken with a live session. Raj revokes it; the `jti` lands in Redis; the very next API call from that laptop returns `401 Unauthorized`.

## Key Takeaways

- Zero Trust means **no implicit trust from network location** — verify identity and policy on every request.
- **Identity is the perimeter.** AuthN (this module) is the front door; AuthZ (Module 03) is the room key.
- **OIDC and SAML log you in; OAuth 2.0 lets your app call APIs.** Never substitute one for another.
- The enforcement pair is **PEP (gateway, muscle) + PDP (OPA, brain)**.
- **Federation** means trusting another organization's IdP so you never store their passwords — and so their offboarding becomes your offboarding.
- **Short tokens + rotation + a revocation blocklist** is how a stateless system stays revocable.
- Related material lives in neighbouring modules: **JWKS and signing key rotation** are covered in [Module 04 — Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md), and **audit logging and observability** are covered in [Module 09 — Resilience, Observability & SIEM](../09_Resilience_Observability_And_SIEM/README.md).

## Glossary

| Term | Definition |
|---|---|
| **ZTA (Zero Trust Architecture)** | Security model where no request is trusted based on network location; every request is verified explicitly. |
| **AuthN (Authentication)** | Proving *who* an identity is. |
| **AuthZ (Authorization)** | Deciding *what* an authenticated identity may do. |
| **IdP (Identity Provider)** | The system that authenticates users and issues proof of authentication. |
| **SP (Service Provider)** | SAML term for the application that consumes the IdP's assertion. |
| **RP (Relying Party)** | OIDC term for the same idea — the app relying on the IdP. |
| **PEP (Policy Enforcement Point)** | The component that intercepts and enforces — MoneyGuard's API Gateway. |
| **PDP (Policy Decision Point)** | The rule engine that answers ALLOW/DENY — Open Policy Agent. |
| **JWT (JSON Web Token)** | A signed, self-contained token of the form `header.payload.signature`. |
| **JWKS (JSON Web Key Set)** | The public-key document a resource server fetches to verify JWT signatures. |
| **jti** | The unique ID claim inside a JWT, used for revocation blocklists. |
| **Federation** | Trusting an external organization's IdP to authenticate its own users. |
| **MFA (Multi-Factor Authentication)** | Requiring more than one kind of proof (password + phone/hardware key). |
| **SCIM (System for Cross-domain Identity Management)** | The standard protocol for pushing user create/update/delete between systems. |
| **RBAC / ABAC** | Role-Based / Attribute-Based Access Control. |
| **BFF (Backend-for-Frontend)** | A small server that holds tokens on behalf of a browser app so JavaScript never touches them. |
| **Rego** | The policy language used by Open Policy Agent. |
| **Zero Standing Privileges (ZSP)** | The principle that nobody holds elevated rights by default. |

## Revision Notes

- Zero Trust in three words: **never trust, always verify**. In three rules: verify explicitly, least privilege, assume breach.
- Two identity planes: **IdP = AuthN (OIDC/SAML)**, **token service = AuthZ (OAuth 2.0)**.
- Enforcement pair: **PEP = gateway/muscle**, **PDP = OPA/brain**.
- Protocol picker: **SAML** for legacy B2B partners, **OIDC** for modern login, **OAuth 2.0** for API access and machine-to-machine.
- Token defaults: access **15 min**, refresh **rotating**, revocation via **`jti` blocklist in Redis**.
- Federation kills shadow accounts and makes partner offboarding instant.
- Never: `localStorage` tokens, OAuth-as-login, long-lived access tokens, policy inside every service.
- Next: [SAML](./02_SAML.md) → [OAuth 2.0](./03_OAuth2.md) → [OIDC](./04_OIDC.md) → [state/nonce/PKCE](./05_State_Nonce_And_PKCE.md).

## Interview Questions

**1. What is Zero Trust, and how is it different from a traditional perimeter model?**
Zero Trust removes implicit trust based on network location. In the traditional castle-and-moat model, the check happened once at the VPN and everything inside the network was trusted. In Zero Trust, every request is authenticated and authorized on its own merits, every time, using identity plus context (device, IP, time). The three operating rules are verify explicitly, enforce least privilege, and assume breach. Practically, it means MoneyGuard's Core Banking Ledger treats a request from a branch PC exactly as suspiciously as one from the public internet.

**2. Where does authentication sit in a Zero Trust architecture, and what sits next to it?**
Authentication sits at the front of the chain, at the Identity Provider. It answers "who are you" and produces a signed artifact — a SAML assertion or an OIDC ID token. Immediately after it, a token/authorization layer maps that identity to internal roles and mints an OAuth access token. Downstream, an enforcement pair (PEP at the gateway, PDP in the policy engine) uses that token to decide each individual request. Authentication alone grants nothing; it only starts the chain.

**3. Explain PEP and PDP, and why they are separated.**
The PEP is the enforcement muscle — MoneyGuard's API Gateway or an Envoy sidecar. It intercepts the request, validates the JWT signature, asks for a decision, and enforces it. The PDP is the brain — Open Policy Agent — which evaluates RBAC and ABAC rules and returns ALLOW or DENY. They are separated so policy can change centrally without redeploying hundreds of gateways, and so gateways stay optimized for throughput while the policy engine stays optimized for rule evaluation.

**4. When would you choose SAML over OIDC in 2026?**
Only when the other side requires it. SAML is XML-based, designed for browser-based enterprise SSO, and is still the lingua franca for B2B federation with established enterprises and legacy on-premise software. MoneyGuard uses SAML for PartnerBank because PartnerBank's IdP already speaks it and rebuilding their side is not MoneyGuard's decision. For anything MoneyGuard controls — the Employee Portal, the Mobile App, APIs — OIDC is the correct choice because SAML has no good story for mobile apps, single-page apps, or API access tokens.

**5. If a JWT is stateless and verified locally, how do you revoke it?**
Three complementary answers. First, keep it short-lived (5–15 minutes) so revocation is mostly a non-issue. Second, maintain a distributed blocklist: on revocation, push the token's `jti` into Redis and have the gateway check it in sub-millisecond time until the token's original `exp` passes. Third, use Continuous Access Evaluation: the IdP pushes events like `password_changed` or `account_disabled` to enforcement points, which then reject any token for that subject issued before the event timestamp.

**6. What happens to MoneyGuard if the Identity Provider goes down?**
Nobody can obtain a *new* token — new logins and refreshes fail. But users who already hold a valid access token keep working until it expires, because the gateway verifies signatures locally against cached JWKS public keys and does not call the IdP per request. That is a deliberate resilience property of stateless tokens. The mitigation for the login outage itself is a multi-region, highly available IdP, and a documented decision about whether the gateway fails open or fails closed if it cannot refresh its key cache — for a bank, fail closed.

**7. Why is federation better than creating accounts for partner-company employees?**
Because identity lifecycle ownership stays with the company that actually knows the truth. MoneyGuard has no way to learn that PartnerBank fired someone yesterday. With federation, PartnerBank disables the account in its own directory and the person can no longer produce a valid assertion, so MoneyGuard access ends automatically. It also removes hundreds of passwords from MoneyGuard's breach surface and eliminates a whole class of password-reset support tickets.

**8. How do you enforce Zero Trust for a service with no human behind it?**
The service gets its own identity — a service principal with a `client_id` and either a secret from a vault or, better, a workload identity credential — and uses the OAuth 2.0 Client Credentials grant to obtain a very short token (about 5 minutes) scoped narrowly, for example `ledger.read` only. A sidecar PEP in front of the callee validates that token's signature, scope, and client ID before forwarding. The MFA step is replaced by cryptographic proof of the workload's identity, and the least-privilege scope means a compromised service still cannot write.

## Practice Exercises

1. **Map the planes.** On one page, draw MoneyGuard's flow for Priya logging into the Employee Portal and reading a balance. Label exactly which box performs AuthN, which mints the token, which is the PEP, and which is the PDP. Then mark where an audit log entry is emitted at each step.
2. **Pick the protocol.** For each of these, choose SAML, OAuth 2.0, or OIDC and justify in one sentence: (a) PartnerBank loan officers reaching the joint mortgage portal; (b) the MoneyGuard Mobile App logging a customer in; (c) the Fraud Detection Pipeline reading the ledger; (d) a 2003 on-premise treasury application that only speaks XML.
3. **Design the revocation path.** Eve steals Priya's active access token at 10:00:00. Raj notices at 10:00:20. Write the exact sequence of systems touched and state the worst-case number of seconds Eve retains access, both with and without a Redis `jti` blocklist.
4. **Write the policy.** Extend the Rego snippet in this chapter so that Sam Torres (a contractor with the role `contractor`) may call `GET /v1/accounts/{id}/balance` but never `POST /v1/wires`, and only between 09:00 and 17:00 on weekdays. State which inputs the PEP must supply to the PDP for your rule to work.
5. **Argue build vs buy.** Write a half-page recommendation to MoneyGuard's CISO on whether to build a custom identity server for the Mobile App. Include at least two risks specific to a regulated bank and one scenario in which building would actually be the right call.

---
⬅ Back to [Module 02 index](./README.md) | 🏦 [MoneyGuard scenario](../SCENARIO.md) | 🏠 [Main README](../README.md)
