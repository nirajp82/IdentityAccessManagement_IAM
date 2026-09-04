# OpenID Connect (OIDC) — The Identity Layer on Top of OAuth 2.0

## Overview

**OpenID Connect (OIDC)** is a thin authentication layer built directly on top of OAuth 2.0. It uses the exact same flows, the same endpoints, and the same redirects — and adds one thing that changes everything: a standardized, signed statement about **who logged in**, delivered directly to the application, called the **ID Token**.

OAuth 2.0 answers *"may this app call that API?"* OIDC answers *"who is the person sitting in front of this app, and can I prove it?"*

🏦 **The MoneyGuard scenario for this chapter:** Priya Sharma logs into the **MoneyGuard Employee Portal**, and a customer logs into the **MoneyGuard Mobile App**. In both cases the app must display "Welcome, Priya" and create a session for the *right person*, with cryptographic proof that **MoneyGuard Entra ID** actually authenticated that person, right now, *for this specific application*.

If you read only one line of this chapter: **OIDC = OAuth 2.0 + ID Token + standardized claims + a discovery document.**

## Why This Matters

Before OIDC, teams built "log in with X" using raw OAuth 2.0, and hit three separate walls.

### Wall 1 — The delivery problem

In pure OAuth, when the portal asks for `scope=email profile`, the authorization server replies with an **access token** — an instruction to an API, not information. It does **not** contain `priya.sharma@moneyguard.com`. To learn who the user is, the portal must open a *second* HTTP connection to a profile API and download it. Extra latency, extra failure mode, extra code.

### Wall 2 — The standardization problem

OAuth never defined what identity looks like, so every provider invented its own vocabulary:

| Provider | Scope name | Profile endpoint | JSON field for email |
|---|---|---|---|
| Provider A | `email` | `/oauth2/v3/userinfo` | `"email"` |
| Provider B | `mail_address` | `/me` | `"mail"` |
| Provider C | *(implicit)* | `/user` | `"email_address"` |

Alex's login code turns into a tangle of `if/else` branches just to extract an email address.

### Wall 3 — The security problem

An access token is meant for the **resource server**, not the **client**. When an app treats "I received an access token" as "this user logged in", it has no cryptographic proof that the token was minted *for it* or that the user authenticated *just now*. That is the **confused deputy / token substitution** attack: Eve presents an access token she legitimately obtained for a different application, and the app happily logs her in as its owner.

### The bouncer analogy

Think of MoneyGuard Entra ID as a bouncer, and the Employee Portal as the bartender inside.

- **Pure OAuth 2.0:** The bouncer checks ID at the door, then hands the guest a blank drink ticket (access token). The bartender sees a valid ticket but has **no idea who this person is** — only that *someone* was let in.
- **OIDC:** The bouncer checks ID, hands over the drink ticket (access token) **and** a tamper-evident name badge (ID token) reading *"This is Priya Sharma, I verified her with MFA at 14:02, and this badge was made specifically for the Employee Portal."*

Now the bartender can genuinely authenticate the guest. That is what "OIDC enables authentication" means: it enables authentication **for the client application**.

## Key Concepts

| OIDC term | Meaning | MoneyGuard equivalent |
|---|---|---|
| **OP (OpenID Provider)** | The IdP that authenticates users and issues ID tokens | MoneyGuard Entra ID |
| **RP (Relying Party)** | The application that *relies* on the OP to say who the user is | Employee Portal, MoneyGuard Mobile App |
| **ID Token** | A signed JWT containing identity claims, issued **to the RP** | The core of this chapter |
| **`openid` scope** | The mandatory scope that turns an OAuth flow into an OIDC flow | Always present |
| **UserInfo endpoint** | Standard API returning additional claims, called with the access token | `/oidc/userinfo` |
| **Discovery document** | `/.well-known/openid-configuration` — machine-readable config | Removes all hardcoded endpoints |
| **`nonce`** | Random value bound into the ID token to prevent replay | Mandatory at MoneyGuard |
| **Standard claims** | Agreed names: `sub`, `email`, `name`, `given_name`, `picture`… | Identical across every OP |

## Detailed Notes

### Concept 1 — ID Token vs Access Token (the single most important table in this module)

| | **ID Token** | **Access Token** |
|---|---|---|
| Introduced by | OpenID Connect | OAuth 2.0 |
| Answers | "Who is the user?" | "What may this client do?" |
| **Audience (`aud`)** | The **client** (`portal-web-prod`) | The **API** (`api://moneyguard-ledger`) |
| Intended consumer | The RP application | The resource server |
| Format | Always a **JWT** | Anything — JWT or opaque |
| Contains | Identity claims: `sub`, `email`, `name`, `auth_time`, `nonce` | Permission claims: `scp`, `roles`, `aud` |
| Sent to APIs? | ❌ **Never** | ✅ Yes, in `Authorization: Bearer` |
| Typical lifetime | Minutes — it is consumed once, at login | 5–15 minutes, used repeatedly |

⚠️ **Two mirror-image mistakes, both common, both serious:**
- **Sending the ID token to an API as a bearer token.** Its `aud` is the client, not the API. A correctly implemented API rejects it — and one that accepts it has an audience-confusion bug.
- **Using the access token to identify the user in the app.** That is the confused-deputy hole OIDC exists to close.

### Concept 2 — What OIDC actually adds to OAuth

Five concrete additions, nothing more:

1. **The `openid` scope.** Include it and you get OIDC behaviour; omit it and you have plain OAuth.
2. **The ID token** in the token response, alongside the access token.
3. **Standard claims and standard scopes** — `profile`, `email`, `address`, `phone` map to a fixed set of claim names, identical across every conformant provider.
4. **The UserInfo endpoint** — one standard place to fetch extra claims with the access token.
5. **Discovery and metadata** — `/.well-known/openid-configuration` plus a `jwks_uri`, so an RP can configure itself.

Everything else — the redirect, the code, the token exchange, PKCE, `state` — is inherited unchanged from OAuth 2.0.

### Concept 3 — Inside the ID token

```json
{
  "iss": "https://login.moneyguard.com/moneyguard.com/v2.0",
  "aud": "portal-web-prod",
  "sub": "8f14e45f-ea1b-4f3d-9c22-7a1e0d6b91c4",
  "name": "Priya Sharma",
  "preferred_username": "priya.sharma@moneyguard.com",
  "email": "priya.sharma@moneyguard.com",
  "email_verified": true,
  "nonce": "n-0S6_WzA2Mj",
  "auth_time": 1772539195,
  "amr": ["pwd", "mfa"],
  "acr": "urn:moneyguard:loa:2",
  "iat": 1772539200,
  "exp": 1772539800
}
```

| Claim | Why it exists |
|---|---|
| `iss` | Who minted this token. Must equal the expected issuer exactly. |
| `aud` | Who it was minted **for**. Must equal this RP's `client_id` — this is the anti-confused-deputy control. |
| `sub` | The OP's permanent, unique, never-reused ID for this user. **This is the value you key your database on**, not the email. |
| `nonce` | Binds the token to *this* login attempt. Must match what the RP generated. |
| `auth_time` | *When* the user actually authenticated — not when the token was issued. Essential for step-up and `max_age`. |
| `amr` | Authentication Methods References: how they authenticated (`pwd`, `mfa`, `hwk`, `face`). |
| `acr` | Authentication Context Class Reference: the assurance level reached. |
| `exp` / `iat` | Validity window. |

💡 **Key `sub`, never `email`.** People change their email address; `sub` never changes. Keying on email is how account-takeover bugs get written.

### Concept 4 — The five checks an RP must perform on an ID token

Do all five. Skipping any one of them is a real vulnerability.

| # | Check | What it prevents |
|---|---|---|
| 1 | **Signature** verifies against the OP's JWKS public key (selected by `kid`) | Forged tokens |
| 2 | **`iss`** exactly equals the expected issuer string | Tokens from a rogue or wrong provider |
| 3 | **`aud`** equals this client's `client_id` (and `azp` if multiple audiences) | Token substitution / confused deputy |
| 4 | **`exp`** is in the future, `iat` is sane (small clock skew allowed) | Expired-token replay |
| 5 | **`nonce`** matches the value this RP generated and stored for this session | ID token replay |

Plus, when you asked for it: **`auth_time`** within `max_age`, and **`acr`/`amr`** meeting your step-up requirement.

### Concept 5 — The `nonce` chain of custody

The `nonce` is the OIDC-specific defence, and its journey confuses people because the authorization code does **not** carry it. Here is exactly where it lives at each moment:

| Phase | Component | Action | Where the `nonce` is | Where the code/token is |
|---|---|---|---|---|
| 1. Creation | RP (Portal) | Generates a random `nonce` | Stored server-side in the login session | Not created yet |
| 2. Redirect | Browser | Carries the user to the OP | Sent in the `/authorize` URL as `&nonce=...` | Not created yet |
| 3. Code issued | OP | User authenticates; OP mints a code | **Memorized by the OP**, tied to this login attempt | `code=...` returned via redirect |
| 4. Exchange | RP backend | POSTs the code to `/token` | Still in the RP's session | Code sent over the back channel |
| 5. ID token minted | OP | Validates the code, builds the ID token | **Retrieved and embedded as a claim**, then signed | ID token returned |
| 6. Validation | RP | Compares token `nonce` with session `nonce` | Compared. Mismatch → reject | Signature verified via JWKS |

**The "aha" moment is phase 3:** the authorization code is just an opaque coat-check ticket carrying nothing. The OP remembers your `nonce` server-side and only stitches it into the ID token at phase 5 — after which the signature makes it immutable.

Why it works: Eve may hold a genuine, correctly signed ID token from an earlier session, but its baked-in `nonce` belongs to *that* session. She cannot alter it without breaking the signature, and she cannot guess the current session's `nonce`. See [Chapter 05](./05_State_Nonce_And_PKCE.md) for the full attack walkthrough.

### Concept 6 — OIDC response types (which flow to use)

| `response_type` | Flow | Verdict |
|---|---|---|
| `code` | **Authorization Code** (+ PKCE) — code on the front channel, all tokens on the back channel | ✅ Use this. Always. |
| `id_token` / `id_token token` | **Implicit** — tokens delivered in the URL fragment | ❌ Deprecated. Tokens leak through URLs, history, and extensions. |
| `code id_token` | **Hybrid** — ID token early on the front channel, tokens later | ⚠️ Legacy niche (some older frameworks). Avoid for new work. |

MoneyGuard uses `response_type=code` with PKCE for the Employee Portal, the Mobile App, and everything else. No exceptions.

### Concept 7 — Where does the RP put the token?

Same problem as OAuth, same answer.

⚠️ Storing an ID token or access token in `localStorage` means one XSS bug — even one in a transitive npm dependency — hands Eve a working credential. A bearer token has no device binding: MoneyGuard's API cannot distinguish Priya's browser from Eve's laptop.

✅ **Preferred: the BFF (Backend-for-Frontend) pattern.**

```mermaid
sequenceDiagram
    autonumber
    participant Browser as Portal frontend (browser)
    participant BFF as Portal BFF (server)
    participant OP as MoneyGuard Entra ID (OP)
    participant API as API Gateway

    Browser->>BFF: 1. Click "Sign in"
    BFF->>OP: 2. Authorization Code + PKCE flow (server side)
    OP-->>BFF: 3. id_token + access_token + refresh_token
    Note over BFF: Validates ID token, keeps ALL tokens server side
    BFF-->>Browser: 4. Set-Cookie: session=...; HttpOnly; Secure; SameSite=Strict
    Note over Browser: JavaScript physically cannot read an HttpOnly cookie
    Browser->>BFF: 5. GET /api/accounts (cookie auto-attached)
    BFF->>API: 6. Forward with Authorization: Bearer <access_token>
    API-->>Browser: 7. 200 OK
```

✅ **Alternative for native apps and pure SPAs with no backend: DPoP** (Demonstrating Proof-of-Possession). The client generates a non-extractable key pair in the browser or secure enclave, the OP stamps the token with that public key's thumbprint, and the client signs a small proof JWT (`htm`, `htu`, `iat`, `jti`) for every API call. A stolen token is then useless — the thief cannot produce the proof because the private key never left the device.

The MoneyGuard Mobile App stores its refresh token in the platform keystore (iOS Keychain / Android Keystore) and holds the access token only in memory.

### Concept 8 — Sessions, logout, and step-up

- **The ID token is not a session.** It is consumed once, at login, to establish the RP's own session. Do not re-present it later.
- **`max_age` and `prompt`.** `prompt=login` forces re-authentication; `max_age=300` requires the user to have authenticated in the last 5 minutes, verified against `auth_time`. This is how MoneyGuard forces a fresh authentication before a high-value wire transfer.
- **Step-up with `acr_values`.** Requesting `acr_values=urn:moneyguard:loa:3` asks the OP for phishing-resistant MFA; the RP verifies the returned `acr`/`amr` actually meets the bar. (Covered in depth in Module 07.)
- **RP-initiated logout.** Redirect to the `end_session_endpoint` with `id_token_hint` and `post_logout_redirect_uri` so the OP session ends too, not just the local one.

### Concept 9 — OAuth 2.0 vs OIDC, side by side

This is the comparison the chapter exists to make.

| | **OAuth 2.0** ([Chapter 03](./03_OAuth2.md)) | **OpenID Connect** (this chapter) |
|---|---|---|
| Purpose | **Authorization** — delegated API access | **Authentication** — proving who logged in |
| Question answered | "May this app call that API on the user's behalf?" | "Who is this user, and did they just authenticate?" |
| Specification | RFC 6749 (2012) | OIDC Core 1.0 (2014), built on RFC 6749 |
| Trigger | Any scope | The **`openid`** scope |
| Primary artifact | Access token | **ID token** (plus the access token) |
| Token audience | The API | The **client application** |
| Token format | Unspecified — JWT or opaque | Always a **JWT** |
| Identity data | Not standardized; extra API call needed | Standard claims delivered inside the token |
| Discovery | Not standardized | `/.well-known/openid-configuration` |
| Replay defence | `state` (CSRF) | `state` **plus `nonce`** |
| User info | Provider-specific endpoint | Standard `/userinfo` endpoint |
| Logout | Not defined | `end_session_endpoint` defined |
| Use for | API access, machine-to-machine, third-party delegation | Login / SSO for web, mobile, and SPAs |
| MoneyGuard uses it for | Portal → Ledger API, Fraud Pipeline → Ledger | Portal login, Mobile App login |

**And all three protocols in this module:**

| | **SAML 2.0** | **OAuth 2.0** | **OIDC** |
|---|---|---|---|
| Type | Authentication + federation | Authorization | Authentication |
| Format | XML | JSON | JSON / JWT |
| Artifact | Signed assertion | Access token | ID token |
| Transport | Browser redirect + form POST | Redirect + back-channel HTTPS | Redirect + back-channel HTTPS |
| Mobile-friendly | ❌ | ✅ | ✅ |
| Gives an API token | ❌ | ✅ | ✅ |
| Discovery | Metadata XML | — | `.well-known` JSON |
| Replay defence | `InResponseTo`, `NotOnOrAfter`, audience | `state` | `state` + `nonce` |
| Best for | B2B federation with PartnerBank, legacy apps | API access, M2M | Modern login |

💡 The deep symmetry: a **SAML assertion** and an **OIDC ID token** do the same job — a signed statement about a user, issued by an IdP, consumed by an app. XML envelope vs JWT envelope.

## Workflow

🏦 Priya logs into the MoneyGuard Employee Portal with OIDC — the full Authorization Code flow with PKCE and `nonce`:

```mermaid
sequenceDiagram
    autonumber
    actor Priya
    participant Portal as Employee Portal (RP)
    participant OP as MoneyGuard Entra ID (OP)
    participant API as API Gateway -> Core Banking Ledger

    Priya->>Portal: Opens portal.moneyguard.com
    Note over Portal: Generate state, nonce, and PKCE code_verifier<br/>Store all three in the server-side login session
    Portal-->>Priya: 302 /authorize?scope=openid profile email ledger.read<br/>&state&nonce&code_challenge&code_challenge_method=S256
    Priya->>OP: Follows redirect
    OP-->>Priya: Login page + MFA challenge
    Priya->>OP: Password + MFA approval
    Note over OP: Authenticates Priya.<br/>Memorizes the nonce against this login attempt.
    OP-->>Priya: 302 /callback?code=...&state=...
    Priya->>Portal: GET /callback?code=...&state=...
    Portal->>Portal: Check returned state == stored state
    Portal->>OP: POST /token (code, client_secret, code_verifier)
    OP->>OP: Verify client + SHA256(code_verifier) == code_challenge
    OP-->>Portal: id_token + access_token + refresh_token
    Note over Portal: Validate ID token:<br/>1 signature via JWKS<br/>2 iss<br/>3 aud == portal-web-prod<br/>4 exp<br/>5 nonce == stored nonce
    Portal->>Portal: Look up user by sub, create local session
    Portal-->>Priya: Set-Cookie HttpOnly session; "Welcome, Priya"
    Priya->>Portal: Views a customer balance
    Portal->>API: GET /v1/accounts/8842/balance + Bearer ACCESS token
    Note over API: Access token, never the ID token
    API-->>Portal: 200 OK
    Portal-->>Priya: Renders the balance
```

## Architecture Diagram

```mermaid
flowchart TD
    subgraph Users["Users"]
        PRIYA["Priya - employee"]
        CUST["Customer"]
    end

    subgraph RPs["Relying Parties"]
        PORTAL["Employee Portal<br/>confidential RP + BFF"]
        MOBILE["MoneyGuard Mobile App<br/>public RP, PKCE + keystore"]
    end

    subgraph OP["OpenID Provider - MoneyGuard Entra ID"]
        WK["/.well-known/openid-configuration<br/>discovery"]
        AUTHZ["/oauth2/v2.0/authorize"]
        TOK["/oauth2/v2.0/token<br/>returns id_token + access_token"]
        UINFO["/oidc/userinfo"]
        JWKS["/discovery/v2.0/keys<br/>JWKS public keys"]
        LOGOUT["/oauth2/v2.0/logout<br/>end_session_endpoint"]
        MFA["MFA + passkeys"]
        DIR["Identity Store<br/>sub, name, email, roles"]
    end

    subgraph APIs["Resource Servers"]
        GW["API Gateway - PEP"]
        LEDGER["Core Banking Ledger"]
    end

    SIEM["MoneyGuard SIEM - Sentinel"]

    PRIYA --> PORTAL
    CUST --> MOBILE
    PORTAL --> AUTHZ
    MOBILE --> AUTHZ
    AUTHZ --> MFA --> DIR
    PORTAL --> TOK
    MOBILE --> TOK
    TOK --> DIR
    JWKS -.ID token verification.-> PORTAL
    JWKS -.ID token verification.-> MOBILE
    JWKS -.access token verification.-> GW
    WK -.self-configuration.-> PORTAL
    PORTAL -->|Bearer ACCESS token| GW
    MOBILE -->|Bearer ACCESS token| GW
    GW --> LEDGER
    PORTAL --> UINFO
    PORTAL --> LOGOUT
    AUTHZ -.login events.-> SIEM
    GW -.access events.-> SIEM
```

## Step-by-Step Process

1. **Discover.** The RP fetches `/.well-known/openid-configuration` once at startup and caches the endpoints and `jwks_uri`. Nothing is hardcoded.
2. **Register.** Alex registers the RP: `client_id`, exact redirect URI, allowed scopes, and (confidential clients only) a `client_secret` in the vault.
3. **Begin login.** The RP generates three random values — `state`, `nonce`, `code_verifier` — and stores them in the server-side login session, keyed to this browser.
4. **Redirect to `/authorize`** with `response_type=code`, `scope=openid profile email ledger.read`, `state`, `nonce`, `code_challenge`, and `code_challenge_method=S256`.
5. **The OP authenticates** the user: password, MFA, passkey — whatever policy demands. The RP never sees a credential.
6. **Consent.** First-party MoneyGuard apps are pre-authorized; third-party clients see the scope list and must approve.
7. **Code returned** to the registered redirect URI, with `state` echoed back unchanged.
8. **Verify `state`** against the stored value. Mismatch → abort the login.
9. **Back-channel token exchange** at `/token`, sending the code, client credentials, and the raw `code_verifier`.
10. **Receive `id_token` + `access_token` (+ `refresh_token`).**
11. **Validate the ID token — all five checks:** signature (JWKS, by `kid`), `iss`, `aud`, `exp`, `nonce`. Add `auth_time`/`acr` checks if you requested them.
12. **Resolve the user by `sub`.** Look up (or JIT-create) the local profile keyed on `sub`, and attach MoneyGuard roles from the Identity Store.
13. **Create the RP's own session** — an `HttpOnly; Secure; SameSite=Strict` cookie. The ID token's job is now finished; discard it.
14. **Call APIs with the *access* token**, never the ID token.
15. **Optionally call `/userinfo`** with the access token for claims that were too large or too fresh to embed in the ID token.
16. **Logout** by clearing the local session *and* redirecting to `end_session_endpoint` with `id_token_hint`.

## Commands and Examples

**1. Discovery**

```bash
curl -s https://login.moneyguard.com/moneyguard.com/v2.0/.well-known/openid-configuration | jq
```

```json
{
  "issuer": "https://login.moneyguard.com/moneyguard.com/v2.0",
  "authorization_endpoint": "https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/authorize",
  "token_endpoint":         "https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/token",
  "userinfo_endpoint":      "https://login.moneyguard.com/moneyguard.com/oidc/userinfo",
  "jwks_uri":               "https://login.moneyguard.com/moneyguard.com/discovery/v2.0/keys",
  "end_session_endpoint":   "https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/logout",
  "response_types_supported": ["code"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email", "offline_access", "ledger.read"],
  "code_challenge_methods_supported": ["S256"],
  "claims_supported": ["sub", "iss", "aud", "exp", "iat", "nonce", "auth_time",
                       "name", "preferred_username", "email", "email_verified", "acr", "amr"]
}
```

**2. The authorization request — note `scope=openid`, `nonce`, and PKCE**

```http
GET /moneyguard.com/oauth2/v2.0/authorize
  ?response_type=code
  &client_id=portal-web-prod
  &redirect_uri=https%3A%2F%2Fportal.moneyguard.com%2Fcallback
  &scope=openid%20profile%20email%20ledger.read
  &state=Kx7mQp2vR9tL4nZs
  &nonce=n-0S6_WzA2Mj
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGZQ5A9Djk
  &code_challenge_method=S256 HTTP/1.1
Host: login.moneyguard.com
```

Same URL for the MoneyGuard Mobile App, with a native redirect URI and **no** client secret anywhere:

```
moneyguard://oauth/callback
```

**3. The token response — now with an `id_token`**

```json
{
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "openid profile email ledger.read",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Im1nLTIwMjYtMDIiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOi...",
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Im1nLTIwMjYtMDIifQ.eyJhdWQiOiJhcGk6...",
  "refresh_token": "0.AXoAr7Kd9Qm...ZzQ1"
}
```

**4. The decoded ID token — header and payload**

```json
// header
{ "alg": "RS256", "kid": "mg-2026-02", "typ": "JWT" }
```

```json
// payload
{
  "iss": "https://login.moneyguard.com/moneyguard.com/v2.0",
  "aud": "portal-web-prod",
  "sub": "8f14e45f-ea1b-4f3d-9c22-7a1e0d6b91c4",
  "name": "Priya Sharma",
  "preferred_username": "priya.sharma@moneyguard.com",
  "email": "priya.sharma@moneyguard.com",
  "email_verified": true,
  "nonce": "n-0S6_WzA2Mj",
  "auth_time": 1772539195,
  "amr": ["pwd", "mfa"],
  "acr": "urn:moneyguard:loa:2",
  "iat": 1772539200,
  "exp": 1772539800
}
```

Compare `aud` here (`portal-web-prod`, the client) with `aud` in the access token from [Chapter 03](./03_OAuth2.md) (`api://moneyguard-ledger`, the API). Different audiences, different jobs.

**5. The JWKS the RP uses to verify the signature**

```bash
curl -s https://login.moneyguard.com/moneyguard.com/discovery/v2.0/keys | jq
```

```json
{
  "keys": [
    { "kty": "RSA", "use": "sig", "kid": "mg-2026-02", "alg": "RS256",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx4...", "e": "AQAB" },
    { "kty": "RSA", "use": "sig", "kid": "mg-2026-01", "alg": "RS256",
      "n": "qDi7Tx4DhNvPQsl1ofxxc2ePQFcmeuvTZgQ2mB6dEEjbcx4jfV0Q...", "e": "AQAB" }
  ]
}
```

The `kid` in the token header selects the key. Two keys are published during rotation — see [Module 04](../04_Edge_Security_Tokens_And_Sessions/README.md).

**6. Generating `nonce` and PKCE in the MoneyGuard Mobile App's web view / SPA**

```javascript
// Cryptographically secure randomness - never Math.random()
function randomString(bytes = 32) {
  const a = new Uint8Array(bytes);
  crypto.getRandomValues(a);
  return btoa(String.fromCharCode(...a))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

async function sha256Base64Url(input) {
  const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(input));
  return btoa(String.fromCharCode(...new Uint8Array(digest)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

async function startLogin() {
  const state        = randomString();
  const nonce        = randomString();
  const codeVerifier = randomString();
  const codeChallenge = await sha256Base64Url(codeVerifier);

  // sessionStorage is tab-scoped and origin-isolated; wiped when the tab closes
  sessionStorage.setItem('mg_state', state);
  sessionStorage.setItem('mg_nonce', nonce);
  sessionStorage.setItem('mg_pkce_verifier', codeVerifier);

  const p = new URLSearchParams({
    response_type: 'code',
    client_id: 'moneyguard-mobile',
    redirect_uri: 'https://mobile.moneyguard.com/callback',
    scope: 'openid profile email accounts.read',
    state, nonce,
    code_challenge: codeChallenge,
    code_challenge_method: 'S256'
  });
  window.location.href =
    `https://login.moneyguard.com/moneyguard.com/oauth2/v2.0/authorize?${p}`;
}
```

**7. Validating the ID token server-side (.NET, Employee Portal)**

```csharp
// Endpoints and keys come from discovery - never hardcode them
var config = await new ConfigurationManager<OpenIdConnectConfiguration>(
        "https://login.moneyguard.com/moneyguard.com/v2.0/.well-known/openid-configuration",
        new OpenIdConnectConfigurationRetriever())
    .GetConfigurationAsync(CancellationToken.None);

var parameters = new TokenValidationParameters
{
    ValidateIssuer           = true,
    ValidIssuer              = "https://login.moneyguard.com/moneyguard.com/v2.0",
    ValidateAudience         = true,
    ValidAudience            = "portal-web-prod",        // the CLIENT, not the API
    ValidateLifetime         = true,
    ClockSkew                = TimeSpan.FromMinutes(2),
    ValidateIssuerSigningKey = true,
    IssuerSigningKeys        = config.SigningKeys,
    ValidAlgorithms          = new[] { "RS256" }          // never accept "none" or HS256 here
};

var principal = new JwtSecurityTokenHandler()
    .ValidateToken(idToken, parameters, out var validated);

// Check 5: the nonce - NOT done for you by the library
var tokenNonce    = ((JwtSecurityToken)validated).Payload.TryGetValue("nonce", out var n) ? n?.ToString() : null;
var expectedNonce = HttpContext.Session.GetString("mg_nonce");
if (string.IsNullOrEmpty(tokenNonce) || tokenNonce != expectedNonce)
    return Unauthorized("Nonce mismatch - possible ID token replay.");

// Resolve the user by the immutable subject identifier
var subject = principal.FindFirst("sub")!.Value;
var user    = _users.FindBySubject(subject) ?? _users.JitProvision(principal);

// The ID token's job is done. Issue MoneyGuard's own session cookie.
Response.Cookies.Append("mg_session", _sessions.Create(user.Id), new CookieOptions
{
    HttpOnly = true, Secure = true, SameSite = SameSiteMode.Strict,
    Expires = DateTimeOffset.UtcNow.AddHours(2)
});
```

**8. UserInfo — extra claims, fetched with the *access* token**

```bash
curl -s https://login.moneyguard.com/moneyguard.com/oidc/userinfo \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Im1nLTIwMjYtMDIifQ..."
```

```json
{ "sub": "8f14e45f-ea1b-4f3d-9c22-7a1e0d6b91c4",
  "name": "Priya Sharma",
  "email": "priya.sharma@moneyguard.com",
  "email_verified": true,
  "department": "Retail Branch Operations",
  "branch": "BR-0142" }
```

⚠️ Always confirm the `sub` returned by `/userinfo` matches the `sub` in the ID token before merging the claims.

**9. Forcing fresh authentication before a high-value wire**

```
GET /oauth2/v2.0/authorize?...&prompt=login&max_age=300&acr_values=urn:moneyguard:loa:3
```

Then verify the returned `auth_time` is within 300 seconds and `acr` is `urn:moneyguard:loa:3`. Requesting it is not enough — you must check the response.

**10. Logout**

```http
GET /moneyguard.com/oauth2/v2.0/logout
  ?id_token_hint=eyJhbGciOiJSUzI1NiIs...
  &post_logout_redirect_uri=https%3A%2F%2Fportal.moneyguard.com%2Fsigned-out HTTP/1.1
Host: login.moneyguard.com
```

## Best Practices

- **Always `response_type=code` with PKCE.** Never Implicit, never Hybrid for new work.
- **Perform all five ID token checks**, and remember the `nonce` check is usually the one your library does *not* do for you.
- **Pin the signing algorithm** to `RS256` (or ES256). Explicitly reject `alg: none` and reject HMAC algorithms for tokens that should be asymmetrically signed.
- **Key your user table on `sub` + `iss`**, never on email.
- **Use discovery**; never hardcode endpoints or paste public keys into config.
- **Cache JWKS with a `kid`-miss refresh**: if a token arrives with an unknown `kid`, refetch once (rate-limited) rather than failing outright.
- **Do not send the ID token to APIs.** Send the access token.
- **Request the smallest scope set**, and prefer `/userinfo` over stuffing large claim sets into the ID token.
- **Use BFF + `HttpOnly` cookies** for browser apps; **platform keystore + DPoP** for native apps.
- **Implement RP-initiated logout** so the OP session ends too.
- **Use `acr_values`, `max_age`, and `prompt` for step-up** on high-risk actions — and verify `acr`/`amr`/`auth_time` in the response.
- **Log `sub`, `client_id`, `acr`, `amr`, and `auth_time`** to Sentinel for every login.

## Common Mistakes

- ⚠️ **Skipping the `nonce` check** because the library validated everything else. It usually does not validate `nonce`. This leaves ID token replay wide open.
- ⚠️ **Not checking `aud`.** A perfectly valid ID token issued for a different MoneyGuard app then logs someone into the portal.
- ⚠️ **Sending the ID token as the API bearer token.** Wrong audience, wrong purpose.
- ⚠️ **Using the access token to identify the user.** The exact confused-deputy hole OIDC exists to close.
- ⚠️ **Keying accounts on `email`.** Emails change and can be reassigned; `sub` cannot.
- ⚠️ **Trusting `email_verified: false`** as a verified identity, then merging accounts on it.
- ⚠️ **Accepting `alg: none`** or letting the token's own header choose the verification algorithm.
- ⚠️ **Treating the ID token as a long-lived session token.** It is a one-time login artifact.
- ⚠️ **Only clearing the local cookie on logout.** The OP session survives, so the next "login" silently re-authenticates with no prompt.
- ⚠️ **Requesting `acr_values` but never checking the returned `acr`.** Asking is not enforcing.
- ⚠️ **Tokens in `localStorage`.** Still true, still fatal.

## Pro Tips

- 💡 **`openid` is the switch.** Add that one scope and an OAuth flow becomes an OIDC flow, and an ID token appears. Remove it and it silently reverts. Many "why is there no ID token?" bugs are exactly this.
- 💡 **Two tokens, two audiences.** If you can say instantly which audience each token carries and who consumes it, you understand OIDC.
- 💡 **`auth_time` is not `iat`.** `iat` is when the token was minted (possibly from a silent refresh); `auth_time` is when the human actually proved who they were. Step-up logic must use `auth_time`.
- 💡 **`amr` is a gift for risk engines.** `["pwd"]` versus `["hwk","mfa"]` lets MoneyGuard require a hardware key for wires while allowing password-plus-push for balance viewing.
- 💡 **The ID token's audience being the client is why it must never go to an API** — and it is the cleanest one-sentence answer to "what is the difference between an ID token and an access token?" in an interview.
- 💡 **Decode, do not trust, in debugging.** Paste a JWT into a decoder to inspect claims, but never let production code skip signature verification because "we already looked at it."
- 💡 In an interview, note that an OIDC ID token and a SAML assertion are the same idea in different envelopes — that framing shows you understand federation rather than memorized two protocols.

## Real-World Use Cases

1. **🏦 Employee Portal SSO.** Priya signs in once at MoneyGuard Entra ID and reaches the portal, the HR tool, and the ticketing system without retyping a password. The portal keys her account on `sub` and reads her name straight from the ID token.
2. **🏦 Mobile banking login.** The MoneyGuard Mobile App is a public OIDC client using Authorization Code + PKCE, opening the login in a system browser (not an embedded web view), storing the refresh token in the platform keystore.
3. **🏦 Step-up before a wire.** A customer initiates a $50,000 transfer. The app restarts the OIDC flow with `prompt=login`, `max_age=120`, and `acr_values` demanding phishing-resistant MFA, then verifies `acr` and `auth_time` before submitting.
4. **🏦 Just-in-time provisioning.** A newly hired employee logs in for the first time; the portal has no local record, so it JIT-creates a profile from the ID token's `sub`, `name`, and `email`, then attaches roles from the SCIM-populated Identity Store.
5. **🏦 Contractor session limits.** Sam Torres's client is configured with a short `max_age` and no `offline_access` scope, so his sessions cannot be silently extended with a refresh token.
6. **🏦 Audit trail.** Every login writes `sub`, `client_id`, `acr`, `amr`, `auth_time`, and source IP to Sentinel, so Dana can prove which authentication strength protected any given transaction.

## Key Takeaways

- **OIDC = OAuth 2.0 + `openid` scope + ID token + standard claims + discovery.**
- The **ID token is for the client** (`aud` = `client_id`); the **access token is for the API** (`aud` = API identifier). Never swap them.
- Validate the ID token with **all five checks**: signature, `iss`, `aud`, `exp`, `nonce`.
- The **`nonce`** prevents ID token replay and is the check libraries most often leave to you.
- Key users on **`sub`**, not email.
- Use **`response_type=code` + PKCE** everywhere; Implicit is dead.
- **Discovery** (`/.well-known/openid-configuration`) means never hardcoding endpoints or keys.
- **`auth_time`, `acr`, and `amr`** are how you implement and verify step-up authentication.
- Store tokens with **BFF + `HttpOnly` cookies** (web) or **keystore + DPoP** (native), never `localStorage`.
- OIDC is what you use to *log people in*; [OAuth 2.0](./03_OAuth2.md) is what you use to *call APIs*; [SAML](./02_SAML.md) is what you use when the partner requires XML.

## Glossary

| Term | Definition |
|---|---|
| **OIDC (OpenID Connect)** | Authentication layer built on OAuth 2.0, adding the ID token and standard claims. |
| **OP (OpenID Provider)** | The identity provider that authenticates users and issues ID tokens. |
| **RP (Relying Party)** | The application relying on the OP to identify the user. |
| **ID Token** | Signed JWT of identity claims, audienced to the client. |
| **`openid` scope** | The scope that activates OIDC behaviour in an OAuth flow. |
| **`sub` (subject)** | The OP's permanent unique identifier for a user. |
| **`aud` (audience)** | Who the token was minted for. |
| **`iss` (issuer)** | Who minted the token. |
| **`nonce`** | Random value bound into the ID token to prevent replay. |
| **`auth_time`** | Timestamp of the actual user authentication event. |
| **`acr`** | Authentication Context Class Reference — the assurance level reached. |
| **`amr`** | Authentication Methods References — how the user authenticated. |
| **UserInfo endpoint** | Standard endpoint returning additional claims, called with the access token. |
| **Discovery document** | `/.well-known/openid-configuration`, the OP's machine-readable metadata. |
| **JWKS (`jwks_uri`)** | Published public keys used to verify token signatures. |
| **`kid`** | Key ID in a JWT header, selecting which JWKS key to verify with. |
| **Hybrid flow** | `response_type=code id_token`; legacy, avoid. |
| **`prompt` / `max_age`** | Parameters forcing re-authentication or bounding its age. |
| **`end_session_endpoint`** | The OP endpoint for RP-initiated logout. |
| **DPoP** | Demonstrating Proof-of-Possession — binds a token to a client-held private key. |
| **JIT provisioning** | Creating a local user record on first successful login. |

## Revision Notes

- **OIDC = OAuth + `openid` scope + ID token.** Same flows, same endpoints, one new artifact.
- **ID token → client** (`aud` = client_id). **Access token → API** (`aud` = API). Never swap.
- Five ID token checks: **signature, `iss`, `aud`, `exp`, `nonce`.**
- `nonce` chain: RP generates → sent in `/authorize` → OP memorizes → embedded in the ID token at `/token` → RP compares. The **code carries no nonce**.
- Key on **`sub`**, never email. `sub` + `iss` together is the truly unique key.
- Flows: **`code` + PKCE ✅**, Implicit ❌, Hybrid ⚠️.
- Standard claims: `sub`, `name`, `email`, `email_verified`, `auth_time`, `acr`, `amr`, `nonce`.
- Standard endpoints via discovery: `authorization`, `token`, `userinfo`, `jwks_uri`, `end_session`.
- Step-up: `prompt=login`, `max_age`, `acr_values` — and **verify the response**.
- Storage: **BFF + HttpOnly cookie** (web), **keystore + DPoP** (native).

## Interview Questions

**1. What is the difference between OAuth 2.0 and OpenID Connect?**
OAuth 2.0 is an authorization framework: it produces an access token that lets a client call an API on a user's behalf, and it deliberately says nothing to the client about who that user is. OIDC is an authentication layer on the same machinery: adding the `openid` scope makes the authorization server also return an ID token — a signed JWT, audienced to the client, containing standard identity claims such as `sub`, `email`, `auth_time`, and `nonce`. OIDC also standardizes the UserInfo endpoint, a discovery document, and logout. In one line: OAuth gets your app permission; OIDC tells your app who the user is.

**2. What is the difference between an ID token and an access token?**
Audience and consumer. The ID token's `aud` is the client application, and it exists so the client can establish who logged in; it is always a JWT and is consumed once, at login. The access token's `aud` is the API, and it exists so the resource server can decide whether to serve the request; it may be a JWT or an opaque string and is presented repeatedly in the `Authorization` header. Sending an ID token to an API is wrong because the audience does not match; using an access token to identify the user in your app is wrong because it is an authorization artifact, not an authentication statement.

**3. How do you validate an ID token, and which check is most often forgotten?**
Verify the signature using the OP's JWKS key selected by the token's `kid`, with the algorithm pinned to RS256. Then check `iss` equals the expected issuer exactly, `aud` equals your `client_id`, and `exp`/`iat` are within a small clock skew. The fifth check — and the one most often forgotten because most libraries do not do it for you — is `nonce`: the value inside the token must equal the random value your application generated and stored for this specific login session. If you requested `max_age` or `acr_values`, you must also verify `auth_time` and `acr`.

**4. What attack does the `nonce` prevent, and how?**
ID token replay. Suppose Eve obtains a genuine, correctly signed ID token from an earlier session or from a log file. Without a `nonce`, she can inject it into a fresh login attempt; the signature is valid, the issuer is right, and the audience is right, so a naive relying party accepts it and logs her in as the victim. The `nonce` breaks this: the relying party generates a fresh random value per login, the OP embeds it in the ID token and signs it, and the relying party rejects any token whose `nonce` does not match the current session's stored value. Eve cannot change the `nonce` without invalidating the signature, and cannot guess the current one.

**5. Why does `sub` matter more than `email`?**
`sub` is the OpenID Provider's permanent, unique, never-reused identifier for a user; `email` is a mutable attribute. If you key accounts on email, then a person changing their surname and email address becomes a stranger to your system, and — worse — if an organization ever reassigns an old address, the new holder inherits the old account. The correct key is the pair `(iss, sub)`: the issuer plus the subject, since `sub` is only guaranteed unique within one issuer. Email should be stored as a display and contact attribute, and only trusted when `email_verified` is true.

**6. Why is the Implicit flow deprecated, and what replaced it?**
Implicit returned tokens directly in the URL fragment so browser-only apps could avoid a back-channel call. That put credentials into a place that leaks: browser history, referrer headers, extensions, and crash logs, with no client authentication and no way to bind the response to the request. It was replaced by Authorization Code with PKCE, which returns only a single-use code on the front channel and exchanges it for tokens over a direct HTTPS call, with PKCE proving that the app redeeming the code is the same app that started the flow. Modern browsers' support for cross-origin requests removed the original reason Implicit existed.

**7. How would you implement step-up authentication for a $50,000 wire transfer?**
Restart the OIDC flow with `prompt=login` to force a fresh authentication, `max_age` set to a small window such as 120 seconds, and `acr_values` naming the required assurance level — for MoneyGuard, one that means phishing-resistant MFA. The critical half is on the way back: verify that the returned ID token's `auth_time` is inside the window and that `acr`/`amr` actually reflect the required method. Requesting a level is a hint; enforcement happens only when the relying party checks the response. The resulting authentication context should then be carried into the authorization decision and logged for audit.

**8. A developer says the login "works" but skips the `state` and `nonce` parameters. What do you tell them?**
That it works only in the absence of an attacker. Without `state`, the application is open to login CSRF: an attacker can get a victim's browser to redeem the attacker's authorization code, silently binding the victim's session to the attacker's account so that everything the victim subsequently enters lands in the attacker's profile. Without `nonce`, the application is open to ID token replay with a genuinely signed but stale or substituted token. Both parameters cost a few lines — generate a cryptographically random value, store it server-side against the session, and compare on return — and both are mandatory in MoneyGuard's implementation standard.

## Practice Exercises

1. **Compare the two tokens.** Take the ID token and access token payloads in this chapter and build a table of every claim, marking for each: which token it appears in, who consumes it, and what would break if it were missing. Then explain what a resource server should do if it receives the ID token instead of the access token.
2. **Write the validator.** In pseudocode or your language of choice, implement the five ID token checks for the MoneyGuard Employee Portal. Include JWKS fetching with `kid` selection and a rate-limited refresh on unknown `kid`, algorithm pinning, and the `nonce` comparison. Then add the two extra checks needed for a step-up flow.
3. **Design the mobile login.** Specify the complete OIDC configuration for the MoneyGuard Mobile App: client type, `response_type`, redirect URI strategy, which values are generated per login, where each is stored on the device, and where the refresh token lives. Justify why a system browser is used rather than an embedded web view.
4. **Break the audience.** MoneyGuard has two OIDC clients: `portal-web-prod` and `moneyguard-mobile`. Describe step by step how Eve could abuse a missing `aud` check to log into the portal using an ID token she legitimately obtained for the mobile app, and state exactly which line of validation stops her.
5. **Protocol choice.** For each, choose SAML, OAuth 2.0, or OIDC and defend it in two sentences: (a) PartnerBank loan officers reaching the joint mortgage portal; (b) a customer signing into the MoneyGuard Mobile App; (c) the Fraud Detection Pipeline reading the ledger; (d) the Employee Portal calling the Core Banking Ledger on Priya's behalf; (e) a 2003 on-premise treasury application that only understands XML.

---
⬅ Back to [Module 02 index](./README.md) | ➡ Next: [state, nonce & PKCE](./05_State_Nonce_And_PKCE.md) | 🏦 [MoneyGuard scenario](../SCENARIO.md)
