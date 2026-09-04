# OAuth 2.0 — Delegated Authorization

## Overview

**OAuth 2.0** (RFC 6749) is a **delegated authorization framework**. It lets one application call an API *on behalf of a user* without ever seeing that user's password.

Note the word carefully: **authorization**, not authentication. OAuth 2.0 answers *"has this user allowed this application to do this specific thing?"* It does **not** answer *"who is this user?"* — that is what OpenID Connect adds on top, and it is covered in [Chapter 04 — OIDC](./04_OIDC.md).

🏦 **The MoneyGuard scenario for this chapter:** Priya Sharma opens the **MoneyGuard Employee Portal**. The portal needs to read customer balances from the **Core Banking Ledger** through the **API Gateway**. Priya never gives the portal her password — she authenticates directly at **MoneyGuard Entra ID**, which then issues the portal a scoped, short-lived **access token** good only for `ledger.read`.

Think of the access token as a **valet key**. A valet key starts the car and moves it a mile; it does not open the trunk or the glovebox. That is precisely the point of OAuth: minimum permission, limited time, revocable instantly, without changing anyone's password.

## Why This Matters

Before OAuth, systems used the **password anti-pattern**: an application asked you for your password to another service and then logged in pretending to be you.

🏦 Imagine **Acme Corp** (one of MoneyGuard's business banking clients) wants its accounting software to pull its MoneyGuard account balances every night. Without OAuth, Acme's finance manager types their MoneyGuard username and password into the accounting tool. Now:

- **Over-privileged access** — they wanted read-only balances, but the tool can now also initiate wire transfers.
- **No revocation** — the only way to cut the tool off is to change the MoneyGuard password, which breaks every other integration.
- **Huge blast radius** — if the accounting vendor is breached, Eve gets working banking credentials for hundreds of businesses.

With OAuth 2.0, Acme's finance manager is redirected to MoneyGuard, authenticates there, and sees a consent screen: *"Acme Accounting wants to **view account balances**. Allow?"* MoneyGuard then issues the tool a token limited to `accounts.read`, expiring in 15 minutes, revocable with one click, and **the password never leaves MoneyGuard**.

Internally the same mechanism protects the crown jewels: the Employee Portal holds a token scoped to `ledger.read` and literally cannot post a wire transfer, even if Eve compromises the portal.

## Key Concepts

### The four roles

| Role | Definition | In the MoneyGuard walkthrough |
|---|---|---|
| **Resource Owner** | The entity that owns the data and can grant access to it | Priya (or Acme's finance manager) |
| **Client** | The application asking for access on the owner's behalf | MoneyGuard Employee Portal |
| **Authorization Server** | Authenticates the owner, gets consent, issues tokens | MoneyGuard Entra ID at `login.moneyguard.com` |
| **Resource Server** | The API holding the protected data | Core Banking Ledger, fronted by `api.moneyguard.com` |

The Resource Owner does **two distinct things** during a flow:
1. **Authenticates** — proves who they are to the authorization server (password + MFA). The client never sees credentials.
2. **Consents** — grants the specific scopes the client asked for (clicks *Allow*).

> 💡 The authorization server and the resource server are sometimes the same system, sometimes not, and sometimes not even the same company. At MoneyGuard they are deliberately separate: Entra ID mints tokens; the API Gateway validates them.

### The artifacts

| Artifact | What it is | Lifetime at MoneyGuard |
|---|---|---|
| **Authorization code** | A single-use voucher returned via browser redirect, exchanged for tokens over a back channel | ~60 seconds |
| **Access token** | The credential the client presents to the API. Format-agnostic in the spec; a JWT at MoneyGuard | 15 minutes |
| **Refresh token** | Long-lived credential used to get a new access token without re-prompting the user | Hours to days, rotating |
| **Scope** | The permission being requested, e.g. `ledger.read`, `wires.write` | Per request |
| **Claim** | A key-value fact inside a token, e.g. `sub`, `exp`, `roles` | Per token |

### Client types

| | **Confidential client** | **Public client** |
|---|---|---|
| Can keep a secret? | Yes — it has a server | No — code is on the user's device |
| Examples | Employee Portal backend, Fraud Detection Pipeline | MoneyGuard Mobile App, a browser-only SPA |
| Authenticates with | `client_id` + `client_secret` (or private key JWT / mTLS) | `client_id` + **PKCE** only |

⚠️ Never ship a `client_secret` inside a mobile app or JavaScript bundle. Anyone can extract it. Public clients rely on PKCE instead ([Chapter 05](./05_State_Nonce_And_PKCE.md)).

## Detailed Notes

### Concept 1 — The two meanings of "authorization"

This confuses almost everyone the first time.

| | **OAuth authorization** | **Application authorization** |
|---|---|---|
| The question | "Did Priya allow the Employee Portal to call the ledger API with scope `ledger.read`?" | "Is Priya allowed to view *this particular* VIP account?" |
| Who decides | MoneyGuard Entra ID (authorization server) | MoneyGuard's own API + policy engine (OPA) |
| Where it is stored | Client registration + user consent | MoneyGuard's Identity Store and Rego policies |
| Which module | This one | [Module 03 — Authorization & Policy Engines](../03_Authorization_And_Policy_Engines/README.md) |

> **🔥 The golden rule:** OAuth 2.0 packs its bags and goes home the moment the token is issued. Everything after that — roles, record-level permissions, business rules — is your application's job.

### Concept 2 — What OAuth does and does not standardize

**Does:**
- How a client obtains a token without seeing the user's password.
- The shape of the authorization request, the code exchange, and the token response.
- How scopes are requested and consent is expressed.

**Does not:**
- **How** the user authenticated. Password? Passkey? SMS code? OAuth does not care; it only waits for the authorization server to say "done."
- **What format** the access token takes. Opaque random string or JWT — both are spec-compliant.
- **What your API does with it.** OAuth has never heard of a "wire transfer."

### Concept 3 — The grant types (flows)

| Grant | Use it for | MoneyGuard example | Status |
|---|---|---|---|
| **Authorization Code + PKCE** | Any flow with a human and a browser | Employee Portal, Mobile App, Acme's accounting tool | ✅ The default. Use it. |
| **Client Credentials** | Machine-to-machine, no human present | Fraud Detection Pipeline → Core Banking Ledger | ✅ Correct for workloads |
| **Refresh Token** | Renewing an expired access token | Every long-lived session | ✅ With rotation |
| **Device Authorization** | Input-constrained devices | A branch signage display or ATM-style kiosk | ✅ Where applicable |
| **Implicit** | (returned tokens directly in the URL fragment) | — | ❌ Deprecated — tokens leak via URLs |
| **Resource Owner Password Credentials (ROPC)** | (client collects the password itself) | — | ❌ Deprecated — it *is* the password anti-pattern |

### Concept 4 — Client registration (the prerequisite nobody sees)

Nothing works until Raj registers the client at MoneyGuard Entra ID. This is where the **redirect URI allowlist** is created, and that allowlist is a load-bearing security control.

```mermaid
sequenceDiagram
    autonumber
    participant Alex as Alex - Backend Engineer
    participant IdP as MoneyGuard Entra ID

    Note over Alex, IdP: One-time setup, before any user logs in

    Alex->>IdP: 1. Register application "MoneyGuard Employee Portal"
    Alex->>IdP: 2. Declare exact redirect URI<br/>https://portal.moneyguard.com/callback
    Note over IdP: Entra ID stores the allowlist.<br/>Codes are ONLY ever sent to these URLs.
    IdP-->>Alex: 3. Issue client_id = portal-web-prod
    IdP-->>Alex: 4. Issue client_secret (confidential clients only)
    Note over Alex: Secret goes into the vault,<br/>never into source control or the browser.
```

⚠️ Without a strict redirect URI allowlist, Eve manipulates the authorization request to send the code to `https://evil.example/callback`. With the allowlist, Entra ID rejects the request outright. Register **exact** URIs — no wildcards, no open path prefixes.

### Concept 5 — Access tokens and how an API validates one

A **JWT (JSON Web Token)** has three Base64URL parts: `header.payload.signature`.

**How the authorization server signs it:**
1. Build the plaintext header and payload.
2. Hash them (SHA-256) → **Hash A**.
3. Encrypt Hash A with the **private key** → that is the signature.

**How the API Gateway verifies it — with no database call:**
1. Fetch and cache MoneyGuard's public keys from the JWKS endpoint (once per hour).
2. Hash the received header and payload itself → **Hash B**.
3. Decrypt the signature with the **public key** → reveals **Hash A**.
4. If Hash A == Hash B, the token is authentic and unmodified.

The gateway must then check the claims — **all four, every time**:

| Claim | Check | Why |
|---|---|---|
| `iss` | Is this from `https://login.moneyguard.com/...`? | Rejects tokens from a different issuer |
| `aud` | Is this token meant for `api://moneyguard-ledger`? | Prevents a token issued for another API being replayed here |
| `exp` | Is it still in date? | Limits the stolen-token window |
| `scp` / `scope` | Does it carry the required permission? | Enforces least privilege |

⚠️ **The identity trap.** A JWT access token usually contains a `sub` claim, which makes it *look* like proof of who the user is. It is not. It is an authorization artifact. A resource server must never treat an access token as proof of authentication — that is the confused-deputy vulnerability OIDC was created to close.

💡 Signing key rotation and the mechanics of JWKS are covered in [Module 04 — Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md).

### Concept 6 — Refresh token rotation

```mermaid
sequenceDiagram
    autonumber
    participant Portal as Employee Portal backend
    participant IdP as MoneyGuard Entra ID
    participant DB as Token store

    Note over Portal, DB: Access token has expired
    Portal->>IdP: POST /token (grant_type=refresh_token, refresh_token=RT_A)
    IdP->>DB: Is RT_A valid and unused?
    IdP->>DB: Invalidate RT_A, issue RT_B
    IdP-->>Portal: New access token + RT_B

    Note over Portal, DB: Later, Eve tries the stolen RT_A
    Portal->>IdP: POST /token (refresh_token=RT_A)
    IdP->>DB: RT_A already consumed - REUSE DETECTED
    IdP->>DB: Revoke the entire token family including RT_B
    IdP-->>Portal: 400 invalid_grant, session terminated
```

Reuse detection turns a stolen refresh token from a silent long-term compromise into a loud, immediate alarm.

### Concept 7 — Where the token lives in the browser

⚠️ Storing an access token in `localStorage` means a single XSS bug anywhere on the page — including in a third-party npm dependency — hands Eve a working token. `localStorage.getItem('access_token')` is one line of injected JavaScript.

✅ **The Backend-for-Frontend (BFF) pattern** is MoneyGuard's answer:

1. The browser app never performs the token exchange itself.
2. A thin backend (the BFF) completes the code exchange and holds the tokens server-side.
3. The BFF returns only an opaque session identifier in a cookie flagged `HttpOnly; Secure; SameSite=Strict`.
4. Browsers are hard-wired to forbid JavaScript from reading an `HttpOnly` cookie, so XSS cannot steal it.
5. On each call the BFF swaps the cookie for the real token and forwards the request.

### Concept 8 — Machine-to-machine: the Client Credentials grant

🏦 The **Fraud Detection Pipeline** scores transactions in near real time and needs to read from the Core Banking Ledger. There is no human, so there is no MFA and no consent screen — the workload authenticates *as itself*.

```mermaid
sequenceDiagram
    autonumber
    participant FD as Fraud Detection Pipeline
    participant IdP as MoneyGuard Entra ID
    participant PEP as Sidecar proxy (PEP)
    participant LED as Core Banking Ledger

    FD->>IdP: POST /token<br/>grant_type=client_credentials, scope=ledger.read
    IdP->>IdP: Authenticate the service principal
    IdP-->>FD: Access token, expires_in 300, scope ledger.read
    FD->>PEP: GET /v1/ledger/transactions + Bearer token
    PEP->>PEP: Verify signature, aud, exp, and scope
    PEP->>LED: Forward authorized request
    LED-->>FD: 200 OK
```

Note the deliberately tiny 5-minute lifetime and the read-only scope: if the pipeline is compromised, Eve still cannot write to the ledger.

## Workflow

🏦 Priya logs into the Employee Portal and the portal reads a balance — the full Authorization Code flow with PKCE:

```mermaid
sequenceDiagram
    autonumber
    actor Priya as Priya (Resource Owner)
    participant Portal as Employee Portal (Client)
    participant IdP as MoneyGuard Entra ID<br/>(Authorization Server)
    participant API as API Gateway -> Core Banking Ledger<br/>(Resource Server)

    Priya->>Portal: Opens portal.moneyguard.com
    Portal->>Portal: Generate state + PKCE code_verifier,<br/>hash verifier into code_challenge
    Portal-->>Priya: 302 to /authorize?client_id&scope=ledger.read&state&code_challenge
    Priya->>IdP: Follows redirect
    IdP-->>Priya: Login page, then MFA challenge
    Priya->>IdP: Credentials + MFA (client never sees these)
    Note over IdP: Consent: internal apps are pre-authorized<br/>by MoneyGuard admins, so no prompt
    IdP-->>Priya: 302 back to /callback?code=...&state=...
    Priya->>Portal: GET /callback?code=...&state=...
    Portal->>Portal: Verify returned state matches stored state
    Portal->>IdP: POST /token (code + client_secret + code_verifier)
    IdP->>IdP: Verify client secret, verify SHA256(code_verifier) == code_challenge
    IdP-->>Portal: access_token + refresh_token
    Portal->>API: GET /v1/accounts/8842/balance + Bearer access_token
    API->>API: Verify signature, iss, aud, exp, scope=ledger.read
    API-->>Portal: 200 OK, balance
    Portal-->>Priya: Renders the balance
```

## Architecture Diagram

```mermaid
flowchart TD
    subgraph Owners["Resource Owners"]
        PRIYA["Priya - employee"]
        ACME["Acme Corp finance manager"]
    end

    subgraph ClientApps["Clients"]
        PORTAL["Employee Portal backend<br/>confidential - has client_secret"]
        MOBILE["MoneyGuard Mobile App<br/>public - PKCE only"]
        ACCT["Acme Accounting tool<br/>confidential third party"]
        FRAUD["Fraud Detection Pipeline<br/>client credentials"]
    end

    subgraph AS["Authorization Server - MoneyGuard Entra ID"]
        AUTHZ["/oauth2/v2.0/authorize<br/>front channel, browser"]
        TOKEN["/oauth2/v2.0/token<br/>back channel, server to server"]
        JWKS["/discovery/v2.0/keys<br/>public signing keys"]
        REVOKE["/oauth2/v2.0/revoke"]
        STORE["Identity Store + client registrations<br/>redirect URI allowlist"]
    end

    subgraph RS["Resource Servers"]
        GW["API Gateway - PEP<br/>api.moneyguard.com"]
        OPA["Open Policy Agent - PDP"]
        LEDGER["Core Banking Ledger"]
        REDIS[("Redis: revoked jti")]
    end

    PRIYA --> PORTAL
    PRIYA --> MOBILE
    ACME --> ACCT
    PORTAL --> AUTHZ
    MOBILE --> AUTHZ
    ACCT --> AUTHZ
    PORTAL --> TOKEN
    MOBILE --> TOKEN
    ACCT --> TOKEN
    FRAUD --> TOKEN
    AUTHZ --> STORE
    TOKEN --> STORE
    JWKS -.cached hourly.-> GW
    PORTAL --> GW
    MOBILE --> GW
    FRAUD --> GW
    GW <--> OPA
    GW <--> REDIS
    GW --> LEDGER
```

## Step-by-Step Process

1. **Register the client.** Alex registers the Employee Portal; Entra ID returns `client_id=portal-web-prod` and a `client_secret` stored in the vault. The exact redirect URI is allowlisted.
2. **Client prepares the request.** The portal generates a random `state`, a random PKCE `code_verifier`, and `code_challenge = BASE64URL(SHA256(code_verifier))`. Both secrets stay server-side.
3. **Front-channel redirect.** The browser is sent to `/authorize` with `response_type=code`, `client_id`, `redirect_uri`, `scope`, `state`, `code_challenge`, and `code_challenge_method=S256`.
4. **Authentication.** Entra ID collects Priya's password and MFA. The client never sees them.
5. **Consent.** External clients (Acme's tool) show a consent screen listing the requested scopes. Internal first-party apps are usually **pre-authorized** by MoneyGuard admins, so Priya sees no prompt.
6. **Code returned.** Entra ID redirects to the registered `redirect_uri` with a single-use `code` (valid ~60 s) and the original `state` echoed back unchanged.
7. **State check.** The portal compares the returned `state` with the one it stored. Mismatch → abort. (This is the CSRF defence; see [Chapter 05](./05_State_Nonce_And_PKCE.md).)
8. **Back-channel exchange.** The portal's backend POSTs to `/token` with `grant_type=authorization_code`, the `code`, the `redirect_uri`, its client credentials, and the raw `code_verifier`. This connection is server-to-server; the browser never sees it.
9. **Server-side verification.** Entra ID checks the client secret, checks the code has not been used, checks `redirect_uri` matches the original request, and checks `SHA256(code_verifier) == code_challenge`.
10. **Tokens issued.** An `access_token` (15 min) and a `refresh_token` are returned.
11. **API call.** The portal calls `api.moneyguard.com` with `Authorization: Bearer <access_token>`.
12. **Resource server validation.** The gateway verifies the signature against cached JWKS keys, checks `iss`, `aud`, `exp`, and `scp`, checks `jti` against the Redis blocklist, then asks OPA for the business decision.
13. **Renewal.** At the 15-minute mark the portal silently exchanges the rotating refresh token for a new access token — no user interaction.
14. **Revocation.** On logout or on a security event, the portal calls `/revoke` for the refresh token and the control plane pushes the access token's `jti` to Redis.

## Commands and Examples

**1. The authorization request (front channel — this is a browser redirect, not a `curl`)**

```http
GET /moneyguard.com/oauth2/v2.0/authorize
  ?response_type=code
  &client_id=portal-web-prod
  &redirect_uri=https%3A%2F%2Fportal.moneyguard.com%2Fcallback
  &scope=ledger.read%20accounts.read
  &state=Kx7mQp2vR9tL4nZs
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGZQ5A9Djk
  &code_challenge_method=S256 HTTP/1.1
Host: login.moneyguard.com
```

**2. The callback (front channel)**

```http
HTTP/1.1 302 Found
Location: https://portal.moneyguard.com/callback?code=0.AXoAmS7...UtGf&state=Kx7mQp2vR9tL4nZs
```

**3. The token exchange (back channel — server to server)**

```bash
curl -X POST https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u "portal-web-prod:$CLIENT_SECRET_FROM_VAULT" \
  -d "grant_type=authorization_code" \
  -d "code=0.AXoAmS7...UtGf" \
  -d "redirect_uri=https://portal.moneyguard.com/callback" \
  -d "code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"
```

Response:

```json
{
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "ledger.read accounts.read",
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Im1nLTIwMjYtMDIifQ.eyJpc3MiOi...",
  "refresh_token": "0.AXoAr7Kd9Qm...ZzQ1"
}
```

⚠️ Notice there is **no `id_token`** here — this is pure OAuth 2.0. The portal has permission to call the API but has received no standardized statement of who Priya is. That gap is exactly what [Chapter 04 — OIDC](./04_OIDC.md) fixes.

**4. The decoded access token payload**

```json
{
  "iss": "https://login.moneyguard.com/moneyguard.com/v2.0",
  "aud": "api://moneyguard-ledger",
  "sub": "8f14e45f-ea1b-4f3d-9c22-7a1e0d6b91c4",
  "azp": "portal-web-prod",
  "scp": "ledger.read accounts.read",
  "roles": ["teller"],
  "branch": "BR-0142",
  "jti": "d0f4c7a1-2b55-4a10-91cf-3c9d0e77b2aa",
  "iat": 1772539200,
  "exp": 1772540100
}
```

**5. Calling the resource server**

```bash
curl -i https://api.moneyguard.com/v1/accounts/8842/balance \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Im1nLTIwMjYtMDIifQ..."
```

```http
HTTP/1.1 200 OK
{"accountId":"8842","available":10420.55,"currency":"USD"}
```

Insufficient scope — the portal tries to post a wire with a `ledger.read` token:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope", scope="wires.write"
```

**6. Client Credentials — the Fraud Detection Pipeline**

```bash
curl -X POST https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=svc-fraud-detection" \
  -d "client_secret=$VAULT_SECRET" \
  -d "scope=ledger.read"
```

```json
{
  "token_type": "Bearer",
  "expires_in": 300,
  "scope": "ledger.read",
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Im1nLTIwMjYtMDIifQ..."
}
```

**7. Refreshing**

```bash
curl -X POST https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u "portal-web-prod:$CLIENT_SECRET_FROM_VAULT" \
  -d "grant_type=refresh_token" \
  -d "refresh_token=0.AXoAr7Kd9Qm...ZzQ1"
```

**8. Revoking**

```bash
curl -X POST https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/revoke \
  -u "portal-web-prod:$CLIENT_SECRET_FROM_VAULT" \
  -d "token=0.AXoAr7Kd9Qm...ZzQ1" \
  -d "token_type_hint=refresh_token"
```

**9. Introspection — for opaque tokens (RFC 7662)**

```bash
curl -X POST https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/introspect \
  -u "api-ledger:$RS_SECRET" -d "token=<opaque_token>"
```

```json
{ "active": true, "scope": "ledger.read", "client_id": "portal-web-prod",
  "username": "priya.sharma@moneyguard.com", "exp": 1772540100 }
```

## Best Practices

- **Authorization Code + PKCE for everything with a human**, confidential clients included. PKCE costs nothing and protects the code even if the client secret leaks.
- **Never use Implicit or ROPC.** Both are formally discouraged; ROPC is literally the password anti-pattern wearing an OAuth badge.
- **Exact-match redirect URIs.** No wildcards, no `*` path suffixes.
- **Short access tokens (5–15 min), rotating refresh tokens.**
- **Request the narrowest scopes** the feature actually needs, and request them at the moment they are needed rather than all at once at login.
- **Validate `iss`, `aud`, `exp`, and scope on every request** at the resource server. A valid signature alone is not enough.
- **Use the BFF pattern** for browser apps so JavaScript never touches a token.
- **Store `client_secret` in a vault**, inject at runtime, and rotate on a schedule. It never belongs in Git, a container image, or a frontend bundle.
- **Prefer stronger client authentication than a shared secret** where available: private-key JWT (`client_assertion`) or mutual TLS.
- **Emit an audit event** for every token issued, refreshed, and revoked, with `client_id`, `sub`, scopes, and `jti`.

## Common Mistakes

- ⚠️ **Using OAuth 2.0 as a login protocol.** An access token proves delegation, not identity. Use OIDC.
- ⚠️ **Trusting the `sub` in an access token as authentication.** The confused-deputy trap.
- ⚠️ **Skipping the `aud` check.** A token legitimately issued for a low-value API is then accepted by the ledger API.
- ⚠️ **Putting tokens in URLs** — `/dashboard?token=eyJ...`. URLs land in browser history, server logs, proxy logs, and `Referer` headers. Use the `Authorization` header.
- ⚠️ **Tokens in `localStorage`.** One XSS and every session is gone.
- ⚠️ **Omitting `state`.** Login-CSRF: Eve gets Alice's browser to redeem *Eve's* code, silently linking Alice's session to Eve's account.
- ⚠️ **Reusing an authorization code.** Codes are single-use for a reason; a second redemption should invalidate the whole token family.
- ⚠️ **Overly broad scopes** — asking for `wires.write` at login because a rarely used screen might need it later.
- ⚠️ **Long-lived non-rotating refresh tokens.** A 30-day static refresh token is a 30-day breach.
- ⚠️ **`client_secret` in a mobile app or SPA.** It is not a secret once it ships.

## Pro Tips

- 💡 **Front channel vs back channel** is the mental model that unlocks OAuth. The **front channel** is the browser — visible, taggable, loggable, therefore only safe to carry a *code*. The **back channel** is server-to-server — private, therefore where *tokens* travel. Say this in an interview and you have explained why the Authorization Code flow exists.
- 💡 **Why a code at all?** Because if the authorization server put tokens directly in the redirect URL, they would be exposed to browser history, extensions, and referrer leakage. The code is a one-time voucher that is useless without the client's credentials and PKCE verifier.
- 💡 **Scopes are coarse; policy is fine.** `ledger.read` says "this app may read ledger data." Whether *this particular user* may read *this particular VIP account* is a policy-engine decision. Do not try to encode row-level permissions in scopes.
- 💡 **JWT vs opaque is a real trade-off.** JWTs scale (local validation, no round trip) but are hard to revoke; opaque tokens revoke instantly but need an introspection call per request. MoneyGuard uses JWTs plus a Redis `jti` blocklist to get most of both.
- 💡 **Read the `WWW-Authenticate` header** on a 401/403. A good resource server tells you exactly which scope you were missing.
- 💡 Every provider uses different endpoint paths — Entra ID `/{tenant}/oauth2/v2.0/authorize`, Okta `/oauth2/default/v1/authorize`, Keycloak `/realms/{realm}/protocol/openid-connect/auth`, Cognito `/oauth2/authorize`, CyberArk `/OAuth2/Authorize/{app-id}`. Never hardcode them; read `/.well-known/openid-configuration`.

## Real-World Use Cases

1. **🏦 Internal delegated access.** The Employee Portal calls the Core Banking Ledger with a token scoped `ledger.read`. Even a fully compromised portal cannot move money.
2. **🏦 Third-party delegated access.** Acme Corp authorizes its accounting software to read balances with `accounts.read`. Acme can revoke that grant from MoneyGuard's business banking settings without changing any password, and MoneyGuard can see exactly which client requested what.
3. **🏦 Machine-to-machine.** The Fraud Detection Pipeline uses Client Credentials to get a 5-minute `ledger.read` token. No human, no MFA, minimal blast radius.
4. **🏦 Mobile banking.** The MoneyGuard Mobile App is a public client using Authorization Code + PKCE, with the refresh token stored in the device's hardware-backed keystore (iOS Keychain / Android Keystore).
5. **🏦 Just-in-time privilege.** A database administrator holds no standing production scope. On approval they receive a token carrying `db.admin` with a two-hour expiry; when it expires, access ends with no manual cleanup.
6. **🏦 Peak-load resilience.** On a payday spike the gateway validates tens of thousands of tokens per second using cached public keys and pure CPU math — the identity database sees no traffic at all.

## Key Takeaways

- OAuth 2.0 is **delegated authorization**, not authentication.
- Four roles: **Resource Owner, Client, Authorization Server, Resource Server.**
- **Authorization Code + PKCE** is the answer for every human-facing flow; **Client Credentials** for workloads. Implicit and ROPC are dead.
- The **authorization code** travels the front channel (browser); **tokens** travel the back channel (server to server).
- The **redirect URI allowlist** and the **`state` parameter** are load-bearing security controls, not boilerplate.
- Resource servers must check **`iss`, `aud`, `exp`, and scope** — a valid signature alone proves nothing about intent.
- **Short access tokens + rotating refresh tokens + a `jti` blocklist** make a stateless system revocable.
- Never store tokens in `localStorage`; use the **BFF pattern** with `HttpOnly` cookies.
- OAuth ends when the token is issued. What a user may do *inside* MoneyGuard is [Module 03](../03_Authorization_And_Policy_Engines/README.md)'s job.

## Glossary

| Term | Definition |
|---|---|
| **OAuth 2.0** | RFC 6749 framework for delegated authorization. |
| **Resource Owner** | The user (or service) who owns the protected data. |
| **Client** | The application requesting access on the owner's behalf. |
| **Authorization Server** | Issues tokens after authenticating the owner and obtaining consent. |
| **Resource Server** | The API that holds protected data and validates access tokens. |
| **Authorization Code** | Single-use voucher returned via redirect, exchanged for tokens. |
| **Access Token** | Credential presented to the resource server; short-lived. |
| **Refresh Token** | Long-lived credential used to obtain new access tokens. |
| **Scope** | A named permission requested by the client, e.g. `ledger.read`. |
| **Claim** | A key-value assertion inside a token. |
| **Grant type** | The specific flow used to obtain a token. |
| **Front channel** | Communication routed through the user's browser (redirects). |
| **Back channel** | Direct server-to-server communication. |
| **Confidential client** | A client that can protect a secret (has a backend). |
| **Public client** | A client that cannot protect a secret (mobile app, SPA). |
| **PKCE** | Proof Key for Code Exchange — protects the code from interception. |
| **JWKS** | JSON Web Key Set — the published public keys used to verify token signatures. |
| **Introspection (RFC 7662)** | Endpoint where a resource server asks whether an opaque token is active. |
| **Revocation (RFC 7009)** | Endpoint for invalidating a token. |
| **BFF (Backend-for-Frontend)** | Server that holds tokens so browser JavaScript never does. |
| **Confused deputy** | Attack where a component is tricked into misusing its authority — here, treating an access token as proof of identity. |

## Revision Notes

- OAuth 2.0 = **delegated authorization**. "Did the user allow this app to do X?" Never "who is the user?"
- Roles: **Owner, Client, Authorization Server, Resource Server.**
- Flows: **Auth Code + PKCE** (humans) | **Client Credentials** (machines) | **Refresh** (renewal). Implicit & ROPC = ❌.
- Front channel carries the **code**; back channel carries the **tokens**.
- Setup artifacts: `client_id`, `client_secret` (confidential only), **exact redirect URI allowlist**.
- Token response fields: `access_token`, `token_type`, `expires_in`, `scope`, `refresh_token`.
- Resource server checks: **signature, `iss`, `aud`, `exp`, scope** (+ `jti` blocklist).
- Defaults: access **15 min**, M2M **5 min**, refresh **rotating with reuse detection**.
- Storage: **BFF + `HttpOnly; Secure; SameSite=Strict`**, never `localStorage`.
- Missing piece: no standardized identity → go to [OIDC](./04_OIDC.md).

## Interview Questions

**1. Is OAuth 2.0 an authentication protocol? Explain.**
No. OAuth 2.0 is an authorization framework. It defines how a client obtains a scoped, time-limited access token to call an API on a user's behalf. Authentication does happen — the authorization server verifies the user — but the *result* of that authentication is never communicated to the client in a standardized, verifiable way. The client only receives an access token, which is an instruction to the resource server, not a statement to the client. Using it as a login signal creates the confused-deputy vulnerability, which is precisely why OpenID Connect was layered on top.

**2. Why does the Authorization Code flow use a code instead of returning the token directly?**
Because the redirect back to the client travels through the browser — the front channel — where the URL is visible in browser history, extensions, referrer headers, and server logs. Returning a token there (the old Implicit flow) leaks it. A code is a single-use, short-lived voucher that is worthless without the client's credentials and PKCE verifier, and it is redeemed over a private back-channel HTTPS call the browser never sees. The token therefore never appears in a URL.

**3. Walk me through what the resource server must validate on an incoming access token.**
First, verify the signature using the issuer's public key from the JWKS endpoint, selecting the key by `kid` and caching keys locally. Then check `iss` matches the expected issuer, `aud` matches this specific API, `exp` (and `nbf` if present) against current time with minimal clock skew, and that the token carries the scope required by the endpoint being called. At MoneyGuard the gateway additionally checks `jti` against a Redis revocation blocklist. Only after all that does it hand the decoded claims to the policy engine for the business decision.

**4. When would you use Client Credentials instead of Authorization Code?**
When there is no human resource owner. A background workload like the Fraud Detection Pipeline is acting on its own behalf, not on behalf of a user, so there is nobody to authenticate or to show a consent screen to. It authenticates directly with its own credentials — ideally a private-key JWT or mTLS rather than a shared secret — and receives a token scoped to what that workload needs. The correct hardening is a very short lifetime, a single narrow scope, and rotating the credential from a vault.

**5. How do you handle token revocation with stateless JWTs?**
Layered. Keep access tokens short (5–15 minutes) so most revocation needs are satisfied by expiry alone. For refresh tokens, use the standard `/revoke` endpoint and enforce rotation with reuse detection so a stolen token detonates the session. For immediate access-token revocation, maintain a distributed blocklist keyed by `jti` in Redis, checked by the gateway in sub-millisecond time and expired automatically at the token's original `exp` so the list stays small. For broader events, push signals like `password_changed` or `account_disabled` to enforcement points and reject tokens issued before that timestamp.

**6. What is PKCE and why does it matter even for confidential clients?**
PKCE (Proof Key for Code Exchange) has the client generate a random `code_verifier`, send only its SHA-256 hash (`code_challenge`) in the authorization request, and present the raw verifier at the token exchange. The authorization server re-hashes and compares. It was created for public clients that cannot hold a secret, but it matters for confidential clients too: it is defence in depth. If a `client_secret` leaks from a config file or a log, an attacker who also intercepts a code still cannot redeem it, because the verifier never left the legitimate client's memory. Current OAuth security best practice recommends PKCE on every authorization code flow.

**7. JWT or opaque access tokens — how do you choose?**
JWTs are self-contained, so a resource server validates them locally with cached public keys: no network hop, no shared state, and they scale to enormous request rates. The cost is revocation — you cannot un-issue a JWT, only blocklist it. Opaque tokens are random strings that mean nothing without an introspection call to the authorization server, which gives instant revocation and keeps claims private, but adds latency and a hard dependency on the authorization server's availability. MoneyGuard uses JWTs at the edge for scale plus a Redis `jti` blocklist to recover most of the revocation benefit.

**8. Acme Corp says their accounting tool "just needs the user's MoneyGuard password to log in nightly." What do you tell them?**
That this is the password anti-pattern and MoneyGuard will not support it. The tool would gain the user's full authority — including wire transfers — with no scoping, no expiry, and no way to revoke it except a password change that breaks everything else. The correct pattern is a registered OAuth client with a consent-based Authorization Code grant limited to `accounts.read`, storing a rotating refresh token so the nightly job can renew without a human. Acme can revoke the grant from MoneyGuard's settings, MoneyGuard can see exactly which client made which call, and no MoneyGuard credential is ever stored by the vendor.

## Practice Exercises

1. **Construct the URL.** Write the complete `/authorize` URL the MoneyGuard Mobile App would use to request `accounts.read` and `transfers.write` as a public client with redirect URI `moneyguard://oauth/callback`. Include every parameter it must send, and state which one you must *not* include and why.
2. **Choose the grant.** For each, pick the grant type and justify it: (a) the Employee Portal reading balances for Priya; (b) the Fraud Detection Pipeline scoring transactions overnight; (c) Acme Corp's accounting tool pulling statements nightly with no human present after the initial setup; (d) a branch lobby display showing queue times.
3. **Break it deliberately.** List four changes you could make to MoneyGuard's OAuth configuration that would each individually allow Eve to obtain a valid access token for Priya. For each, name the exact control that currently prevents it.
4. **Scope design.** Design the scope list for the Core Banking Ledger API covering: reading balances, reading transaction history, initiating a wire, and approving a wire. Explain why "approve" must be a separate scope from "initiate", and state which of these the Employee Portal should be allowed to request at all.
5. **Trace the theft.** Eve steals the Employee Portal's refresh token at 09:00. The portal legitimately refreshes at 09:12. Write the minute-by-minute sequence of what happens with rotation and reuse detection enabled, and what happens without it. State the exact moment MoneyGuard detects the compromise in each case.

---
⬅ Back to [Module 02 index](./README.md) | ➡ Next: [OpenID Connect](./04_OIDC.md) | 🏦 [MoneyGuard scenario](../SCENARIO.md)
