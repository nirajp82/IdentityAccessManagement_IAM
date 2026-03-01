# The Comprehensive Bible: OAuth 2.0 Architecture & Implementation

## 1. Introduction: The Problem OAuth 2.0 Solves

Before modern Identity and Access Management (IAM), the internet suffered from the **Password Anti-Pattern**.

Imagine it is 2010. A user wants a third-party application, "BudgetApp", to track their expenses from their bank, "MoneyGuard". To do this, BudgetApp asks the user: *"Please enter your MoneyGuard username and password here."* BudgetApp then logs into the bank pretending to be the user.

**The critical problems with this approach:**

* **Over-privileged Access:** The user only wanted BudgetApp to *read* transactions. But with the password, BudgetApp can also *wire money*.
* **No Revocation:** The only way to stop BudgetApp is for the user to change their bank password, which breaks every other integration they have.
* **Massive Blast Radius:** If BudgetApp's database is breached, hackers gain the plaintext passwords to thousands of bank accounts.

**The Solution: OAuth 2.0 and Delegated Access**
OAuth 2.0 (RFC 6749) was created to solve this. It is a **Delegated Authorization Framework**. Instead of sharing passwords, the user is redirected to the bank. The bank asks the user: *"Do you want to grant BudgetApp permission to READ your transactions?"* If the user agrees, the bank issues BudgetApp an **Access Token**.

Think of the Access Token as a **Valet Key** for a car. The valet key allows the driver to move the car 1 mile and park it, but it cannot open the trunk or the glovebox. OAuth 2.0 ensures applications only get the exact permissions they need, for a limited time, and can be revoked instantly without changing passwords.

---

## 2. Core Concepts and Protocols

To build a scalable IAM system, you must strictly define the boundaries of the protocol's components.

### Roles

* **Resource Owner:**
  The entity capable of granting access to a protected resource (e.g., Alice, the bank customer).

  * **Dual Responsibilities of the Resource Owner:**
    For OAuth 2.0 to function securely, the Resource Owner performs two distinct actions during the authorization flow:

    * **Authentication:** Proves *who they are* to the Authorization Server (for example, by logging in with username, password, and MFA). The Client application never sees user credentials.
    * **Consent (Delegation):** Explicitly grants permission (Scopes) to the Client application (for example, clicking **Allow** when asked to share specific data).

  * **Types of Resource Owners (MoneyGuard Context):**

    * **Human Consumer (External):**

      * **Example:** Alice, a retail banking customer.
      * **Scenario:** Alice wants to use an external fintech app (“BudgetApp”). Alice is the Resource Owner of her checking account transaction history. She authenticates at `idp.moneyguard.com` and consents to delegate `transactions:read` access to BudgetApp.
    * **Human Employee (Internal):**

      * **Example:** Bob, a MoneyGuard Wealth Manager.
      * **Scenario:** Bob logs into the internal **Teller Portal**. Bob is the Resource Owner of his *own corporate identity and session*. He authorizes the Teller Portal (the Client) to call the Core Ledger APIs on his behalf using his assigned employee roles.
    * **Non-Human Entity (Machine-to-Machine):**

      * **Example:** Fraud Detection Microservice.
      * **Scenario:** In the **Client Credentials Flow**, no human is present. In this edge case, the Client application *also acts as the Resource Owner*. The microservice owns its own data and processes and authenticates itself to the Authorization Server using a `client_secret` to obtain an Access Token.

* **Client:** The application making requests on behalf of the Resource Owner (e.g., BudgetApp, or MoneyGuard's own Mobile App).
* **Authorization Server (IAM):** The server that authenticates the Resource Owner, obtains their consent, and issues the tokens (e.g., `auth.moneyguard.com`).
* **Resource Server:** The API hosting the protected data, which accepts and validates the Access Token (e.g., `api.moneyguard.com`).

### Tokens

* **Access Token:**
  The “Valet Key.” A credential used by the Client application to securely access the Resource Server (API).

  * **Purpose (Authorization, Not Authentication):**
    Represents *delegated authorization*. It communicates exactly *what* the client is allowed to do (for example, read transactions), but it is **not** intended to prove *who* the user is.

  * **Format Agnostic:**
    OAuth 2.0 does not define the structure or format of an Access Token. It may be a random opaque string or a structured JSON Web Token (JWT).

  * **The Identity Trap (Architectural Warning):**
    In modern enterprise systems like MoneyGuard, Access Tokens are commonly implemented as JWTs and may include a subject identifier (`sub`) or other user-related claims. However, **Resource Servers must treat Access Tokens strictly as authorization artifacts and must never use them as proof of user authentication.**

  * **Strict Validation Rules:**
    When a Resource Server receives an Access Token, authorization decisions must be based *entirely* on validated security claims, including:

    * `iss` — Was the token issued by a trusted Authorization Server?
    * `aud` — Was the token minted specifically for this API?
    * `exp` — Is the token still within its valid lifetime?
    * `scope` / `roles` — Does the token contain the exact permissions required for the requested operation?

  * **Golden Rule:**
    An API must never grant access or assume a valid user session solely based on the presence of identity-related claims inside an Access Token.

* **Refresh Token:**
  A long-lived credential used by the Client to obtain new Access Tokens when they expire.
  *Problem solved: Allows Access Tokens to remain short-lived for security while avoiding frequent user reauthentication.*

* **ID Token (OIDC Extension):**
  A JSON Web Token (JWT) that contains verified user identity information (for example, subject, name, email).
  *Problem solved: OAuth 2.0 handles authorization, while OpenID Connect adds authentication by introducing a standardized, verifiable identity token.*

### Scopes and Claims

* **Scopes:** Defined at the OAuth 2.0 level. They represent the *permissions* the client is requesting (e.g., `transactions:read`, `wires:write`). *Problem solved: Enforces the Principle of Least Privilege.*
* **Claims:** Key-value pairs inside a token asserting facts about the token or user (e.g., `sub` for user ID, `exp` for expiration). *Problem solved: Allows Resource Servers to make stateless authorization decisions.*

### JWT Structure and Validation

A JSON Web Token (JWT) consists of three Base64-URL encoded parts: `Header.Payload.Signature`.

* **Header:** Defines the algorithm (e.g., `RS256`) and Key ID (`kid`).
* **Payload:** Contains the claims (scopes, expiration, issuer).
* **Signature:** Cryptographic math proving the token was created by the Authorization Server and hasn't been tampered with.
  * Problem solved: Allows APIs to validate tokens locally without querying the IAM database.*
  * **How the JWT signature used (Step-by-Step):**
    Here is exactly how an API Gateway validates an incoming token without ever making a network call to the database.

    * **Phase 1: The Auth Server "Signs" the Token**

      * When Alice logs in, the Authorization Server (IAM) creates the JWT Header and Payload (e.g., `{"name": "Alice", "scopes": "wires:write"}`).
      * **The Hash:** The Auth Server runs that plaintext Header and Payload through a SHA-256 algorithm. This creates a unique, irreversible string of characters (let's call it **HashX**).
      * **The Signature:** The Auth Server takes **HashX** and encrypts it using its tightly guarded **Private Key**. This encrypted hash is attached to the end of the token. This is the **Signature**.

    * **Phase 2: The API Gateway "Verifies" the Token**

      * The Client App sends this JWT to the API Gateway (`api.moneyguard.com`). The Gateway has never seen this token before, but it downloaded the Auth Server's **Public Key** this morning.
      * **Step A (The Gateway's Hash):** The Gateway looks at the plaintext Header and Payload on the token and runs them through the SHA-256 algorithm itself. It holds this new hash in memory. Let's call it **Hash A**.
      * **Step B (The Public Key Decryption):** The Gateway takes the encrypted **Signature** off the back of the token and applies the Auth Server's **Public Key** to it **(Decrypt it)**. Because it was encrypted with the Private Key, the Public Key successfully decrypts it, revealing the original hash that the Auth Server calculated. Let's call this decrypted hash **Hash B**.
      * **Step C (The Final Check):** The Gateway asks: **Does Hash A exactly equal Hash B?**

    * **The Result**

      * If a hacker tampered with the token (e.g., changed `wires:read` to `wires:write`), the Gateway's **Hash A** will be completely different. The comparison fails.
      * If a hacker tried to fake the signature, the Public Key in Step B will fail to decrypt it, spitting out garbage math. The comparison fails.
      * If **Hash A == Hash B**, the API Gateway knows the token is perfectly intact and genuinely issued by the Auth Server. It lets the request through to the backend microservice immediately.

      #### **The Golden Rule: Signatures vs. Encryption**
      
      ##### ** Data Encryption (Keeping a Secret)**
      
      * **Goal:** Send a hidden message only the recipient can read.
      * **How it works:**
      
        1. You encrypt the message using the **recipient’s public key**.
        2. Only the recipient’s **private key** can decrypt it.
      * **Result:** The message stays secret — even if someone intercepts it, they can’t read it.
      
      **Analogy:** You put a letter in a locked box and give the key only to the recipient.
      
      ##### ** Digital Signatures (Proving Who You Are)**
      
      * **Goal:** Prove you wrote the message and that it hasn’t been changed.
      * **How it works:**
      
        1. You create a **hash** (a fingerprint) of the message.
        2. You “sign” the hash with your **private key**.
        3. Anyone can use your **public key** to verify that the signature matches the message.
      * **Result:** If it matches, it proves the holder of the private key created the message, and the message is intact.
      
      **Analogy:** You sign a document with your personal seal. Anyone can check the seal to confirm it’s truly from you and hasn’t been tampered with.
      
      ✅ **Quick Memory Trick:**
      
      * **Encryption:** “Lock it → Only recipient can unlock.”
      * **Signature:** “Sign it → Everyone can verify.”

### PKCE, Introspection, and Revocation

* **PKCE (Proof Key for Code Exchange):** A cryptographic extension for the Authorization Code flow. *Problem solved: Prevents malicious apps from intercepting the Authorization Code and exchanging it for a token.*
* **Token Introspection (RFC 7662):** An endpoint (`/introspect`) where an API can query the Auth Server to check if an opaque token is active.
* **Token Revocation (RFC 7009):** An endpoint (`/revoke`) where a client tells the Auth Server to invalidate a refresh or access token.

---

## 3. Deep Dive: Redirection and Payload Flows

To understand OAuth 2.0, you must track exactly how the browser redirects the user and how the backend servers whisper to each other securely.

We will look at two distinct flows: an external third-party app, and an internal first-party app. Both use the **Authorization Code Flow with PKCE**, but the user experience and trust levels differ.

### Scenario A: The External Application (BudgetApp)

**The Setup:** Alice (the Resource Owner) wants to use BudgetApp (the Client) to track her spending. BudgetApp needs permission to pull data from MoneyGuard's APIs (the Resource Server).

```mermaid
sequenceDiagram
    autonumber
    participant Alice as Alice (Browser)
    participant BudgetApp as BudgetApp Backend
    participant Auth as auth.moneyguard.com (IAM)
    participant API as api.moneyguard.com (Resource)

    Alice->>BudgetApp: 1. Clicks "Connect Bank"
    BudgetApp->>BudgetApp: Generates PKCE Secret & Hash
    BudgetApp->>Alice: Redirects browser to MoneyGuard
    Alice->>Auth: 2. GET /authorize (Sends PKCE Hash)
    Auth-->>Alice: Prompts Login & Consent Screen
    Alice->>Auth: 3. Authenticates & Clicks "Allow"
    Auth->>Alice: Redirects browser back to BudgetApp
    Alice->>BudgetApp: 4. GET /callback?code=123 (Delivers Code)
    BudgetApp->>Auth: 5. POST /token (Sends Code + PKCE Secret)
    Auth->>Auth: Verifies PKCE Secret matches Hash
    Auth-->>BudgetApp: 6. Returns Access Token (Valet Key)
    BudgetApp->>API: 7. GET /transactions + Bearer Token
    API->>API: Verifies JWT Signature Locally
    API-->>BudgetApp: 8. Returns Bank Data

```

#### Step 1 & 2: The Authorization Request (Browser to Auth Server)

Alice clicks "Connect Bank" in BudgetApp. BudgetApp generates the PKCE secret password in its backend, creates the "hint" (`code_challenge`), and redirects Alice's browser to the MoneyGuard Auth Server.

* **What it solves:** This tells the bank *who* is asking for access and *what* they want, while securely dropping off the PKCE locked box.

```http
GET /authorize?
  response_type=code
  &client_id=budgetapp_client_99
  &redirect_uri=https://budgetapp.com/callback
  &scope=transactions:read
  &state=random_state_88291
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGZxkVj...
  &code_challenge_method=S256 HTTP/1.1
Host: auth.moneyguard.com

```

#### Step 3: Authentication and Consent (The Human Element)

MoneyGuard's Auth Server pauses the technical flow. It looks at the browser and shows Alice a login screen.
After she enters her password and MFA, the Auth Server shows a **Consent Screen**: *"BudgetApp wants to read your transactions. Do you allow this?"* Alice clicks **Allow**.

#### Step 4: The Callback (Auth Server to Browser to Client)

The Auth Server generates a temporary, single-use "Authorization Code" (valid for 60 seconds). It redirects Alice's browser back to BudgetApp, attaching the code to the URL.

```http
HTTP/1.1 302 Found
Location: https://budgetapp.com/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=random_state_88291

```

#### Step 5: The Token Exchange (Server to Server)

BudgetApp's backend extracts the code from the URL. It immediately opens a secure, hidden backend connection to MoneyGuard's Auth Server to trade the code for the Access Token.

* **What it solves:** This is where the **PKCE check** happens. BudgetApp sends the raw `code_verifier` (the secret password). The Auth Server hashes it and ensures it matches the `code_challenge` from Step 1. If a malicious app stole the code in Step 4, they would fail here because they don't know the secret password.

```http
POST /token HTTP/1.1
Host: auth.moneyguard.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic YnVkZ2V0YXBwX2NsaWVudF85OTpzZWNyZXRfcGFzc3dvcmQ=

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https://budgetapp.com/callback
&code_verifier=the_raw_secret_password_generated_in_step_1

```

#### Step 6: The Delivery of the Valet Key

The Auth Server successfully verifies the math. It issues the JWT Access Token and a Refresh Token back to the BudgetApp server.

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "8xLOxBtZp8",
  "scope": "transactions:read"
}

```

#### Step 7: The API Call (Client to Resource Server)

BudgetApp now has the Valet Key. In the background, it calls the MoneyGuard API to fetch Alice's data. It attaches the Access Token to the `Authorization` header. The API Gateway verifies the JWT digital signature locally (the math check) and returns the financial data.

```http
GET /v1/transactions HTTP/1.1
Host: api.moneyguard.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...

```

---

### Scenario B: The Internal Application (Teller Portal)

**The Setup:** Bob (the Resource Owner) is a MoneyGuard employee. He arrives at work and opens the internal `Teller Portal` (the Client) on his laptop to process wire transfers on the `Core Ledger API` (the Resource Server).

The flow for the internal Teller Portal is technically **identical** to the BudgetApp flow (it still uses the Auth Code with PKCE). However, because this is an internal Zero-Trust environment, a few business rules change.

#### Key Differences in the Internal Flow:

1. **First-Party Trust (Skipping the Consent Screen):** When Bob is redirected to `auth.moneyguard.com` (Step 2), he still has to log in with his employee credentials and MFA. However, the Auth Server **skips the Consent Screen**. Because the Teller Portal is an official, first-party MoneyGuard application, the IAM system assumes Bob naturally consents to using it.
2. **Elevated Scopes:** BudgetApp only requested `transactions:read`. The internal Teller Portal will request highly sensitive scopes like `wires:write` and `accounts:admin`.
3. **The Backend-For-Frontend (BFF) Pattern:**
The Teller Portal is likely a Single Page Application (like React). To keep the Access Token safe from browser hackers, the React app doesn't hold the token. The token is held by the Teller Portal's backend proxy. The proxy issues Bob's browser a secure, encrypted HTTP Cookie. When Bob clicks "Send Wire", the browser sends the Cookie, the proxy swaps the Cookie for the Access Token, and forwards the request to the API Gateway.

By using the exact same OAuth 2.0 PKCE flow for both BudgetApp and the Teller Portal, MoneyGuard ensures that every single request hitting its API Gateway has a mathematically verifiable "Valet Key," regardless of whether the request came from an external startup or an internal employee laptop.


---

## 4. System Architecture and Components

A highly available, multi-region OAuth 2.0 architecture separates the IAM Control Plane from the API Enforcement Layer.

### Architecture Components:

1. **Global Load Balancer / WAF:** Routes traffic to the nearest geographic region and blocks DDoS attacks.
2. **API Gateway (Resource Server / PEP):** The entry point for all API traffic (`api.moneyguard.com`). It validates JWT signatures locally using cached Public Keys (JWKS).
3. **Authorization Server (IAM):** The cluster handling `/authorize` and `/token` endpoints. It is compute-heavy (cryptographic signing).
4. **Token Storage Database:** A highly available, multi-region database (e.g., Cassandra, DynamoDB, CockroachDB). *Crucially, JWT Access Tokens are NOT stored here.* Only **Refresh Tokens** and **Client Configurations** are stored here.
5. **Distributed Cache (Redis):** Serves two purposes:
* *JWKS Cache:* Stores the Auth Server's public keys at the edge.
* *Revocation Blocklist:* Stores the IDs (`jti`) of revoked JWTs. The API Gateway checks this in <1ms.



### Diagram: System Flow at Scale

```mermaid
flowchart TD
    subgraph Edge [Edge Layer]
        GW[API Gateway / PEP]
        Redis[(Redis: Blocklist & JWKS)]
    end

    subgraph IAM [IAM Control Plane]
        AuthZ[Auth Server Nodes]
        DB[(Distributed DB: Refresh Tokens)]
    end

    subgraph Backend [Resource Servers]
        Core[Core Banking APIs]
    end

    %% Edge internal connections
    GW <-->|1. Sub-millisecond local check| Redis

    %% Cross-boundary connections
    GW <-->|2. Fetch public keys periodically| AuthZ
    AuthZ <-->|3. Read/Write long-lived tokens| DB
    
    %% Traffic flow
    GW ==>|4. Forward authorized traffic| Core

    %% Styling to make it look like an architecture diagram
    style Edge fill:#f9f9f9,stroke:#333,stroke-width:2px
    style IAM fill:#e6f3ff,stroke:#0066cc,stroke-width:2px
    style Backend fill:#e6ffe6,stroke:#009933,stroke-width:2px

```

---

## 5. Security Considerations

OAuth 2.0 is powerful but highly susceptible to implementation errors.

### Common Attacks & Mitigations

| Attack | Description | Mitigation |
| --- | --- | --- |
| **CSRF (Cross-Site Request Forgery)** | Attacker forces a logged-in user to submit the attacker's authorization code, binding the victim's session to the attacker's account. | Always use the **`state`** parameter. The client generates a random string, stores it in a cookie, and compares it to the `state` returned in the callback. |
| **Authorization Code Interception** | A malicious mobile app registers the same custom URI scheme (`myapp://`) to steal the auth code. | **PKCE.** Even if the code is stolen, the attacker cannot exchange it without the `code_verifier` secret held by the legitimate app. |
| **Token Leakage** | Tokens leak into browser history, server logs, or third-party analytics via the `Referer` header. | **Never pass tokens in URLs.** Use POST requests for token exchange, and send Access Tokens strictly in the `Authorization: Bearer` header. |
| **Token Replay** | An attacker intercepts a valid token and reuses it against an API. | Use **TLS 1.2+** everywhere. Keep Access Token lifetimes extremely short (5-15 mins). |

### Secure Token Storage (The BFF Pattern)

Single Page Applications (React/Angular) should **never** store tokens in `localStorage` due to Cross-Site Scripting (XSS) risks.

* **Best Practice:** Implement a **Backend-For-Frontend (BFF)**. The backend server performs the OAuth flow, holds the tokens in memory, and issues a secure, encrypted `HttpOnly`, `SameSite=Strict` cookie to the browser.

---

## 6. Trade-Offs and Design Decisions

### Flow Comparison

| Flow | Use Case | Security Level | Why use it? |
| --- | --- | --- | --- |
| **Auth Code + PKCE** | Web Apps, SPAs, Mobile Apps | Highest | Keeps tokens off the browser; protects against intercepted codes. |
| **Client Credentials** | Machine-to-Machine (M2M) | High | No human involved. Services use Vault-stored secrets to get tokens. |
| **Device Code** | Smart TVs, CLI tools | Medium | Best for devices with no keyboard. User approves via a secondary device (phone). |
| *Implicit Flow* | *Legacy SPAs* | *Deprecated* | *Never use this.* Tokens are sent in the URL, highly vulnerable to leakage. |

### Token Type: JWT vs. Opaque

* **JWT (Value Token):** Best for high-scale microservices. APIs validate mathematically without hitting the database. *Trade-off:* Hard to revoke instantly.
* **Opaque (Reference Token):** Best for ultra-sensitive systems with low traffic. *Trade-off:* Creates massive database bottlenecks because APIs must call `/introspect` on every single request.

### Refresh Strategies

* **Absolute Expiration:** The Refresh Token dies after 30 days, forcing a hard login.
* **Sliding Expiration (Idle Time):** The token lives for 30 days, but expires if unused for 7 days.
* **Refresh Token Rotation (RTR):** *Best Practice.* Every time a Refresh token is used, it is invalidated and a new one is issued. This instantly detects and mitigates stolen refresh tokens.

---

## 7. Integration Examples (Real-World Identity Providers)

While the protocol is standard, vendor URLs differ based on their multi-tenant architecture. Here is how you configure the endpoints for leading IdPs.

**1. CyberArk Identity (Workforce / CIAM)**

* *Authorize:* `https://{tenant-id}.id.cyberark.cloud/OAuth2/Authorize/{application-id}`
* *Token:* `https://{tenant-id}.id.cyberark.cloud/OAuth2/Token/{application-id}`
* *Keys (JWKS):* `https://{tenant-id}.id.cyberark.cloud/OAuth2/Keys/{application-id}`
* *Note:* CyberArk uniquely scopes its OAuth 2.0 endpoints specifically to the Custom OAuth2 Server Application ID created in your Identity Administration portal, allowing for highly granular token policies per application.

**2. Okta / Auth0**

* *Authorize:* `https://{your-domain}.okta.com/oauth2/default/v1/authorize`
* *Token:* `https://{your-domain}.okta.com/oauth2/default/v1/token`
* *Keys (JWKS):* `https://{your-domain}.okta.com/oauth2/default/v1/keys`

**3. Microsoft Entra ID (Azure AD)**

* *Authorize:* `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/authorize`
* *Token:* `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`
* *Note:* Azure AD scopes often look like URIs (e.g., `api://moneyguard/.default`).

**4. AWS Cognito**

* *Authorize:* `https://{domain-prefix}.auth.{region}.amazoncognito.com/oauth2/authorize`
* *Token:* `https://{domain-prefix}.auth.{region}.amazoncognito.com/oauth2/token`

**5. Keycloak (Open Source)**

* *Authorize:* `https://{domain}/realms/{realm-name}/protocol/openid-connect/auth`
* *Token:* `https://{domain}/realms/{realm-name}/protocol/openid-connect/token`

---

## 8. Diagrams

### Diagram 1: Authorization Code Flow with PKCE

```mermaid
sequenceDiagram
    autonumber
    participant U as User Browser
    participant C as Client Backend (BFF)
    participant AS as Authorization Server
    participant RS as Resource Server (API)

    C->>C: Generate PKCE verifier & challenge
    U->>AS: GET /authorize (client_id, challenge)
    AS-->>U: Prompt Login & Consent
    U->>AS: User authenticates
    AS->>U: Redirect to /callback?code=xyz
    U->>C: Deliver auth code
    C->>AS: POST /token (code, verifier, client_secret)
    AS->>AS: Validate code & verifier match
    AS-->>C: Return Access Token & Refresh Token
    C->>RS: API Call + Header: "Authorization: Bearer <token>"
    RS->>RS: Validate JWT Signature (Local Math)
    RS-->>C: 200 OK (Protected Data)

```

### Diagram 2: Client Credentials Flow (M2M)

```mermaid
sequenceDiagram
    autonumber
    participant S1 as Fraud Microservice (Client)
    participant AS as Authorization Server
    participant S2 as Ledger Microservice (API)

    S1->>AS: POST /token (client_id, client_secret, grant_type=client_credentials)
    AS->>AS: Authenticate Service Account
    AS-->>S1: Return Access Token
    S1->>S2: POST /v1/ledger/alert + Bearer Token
    S2->>S2: Validate Token & Scopes
    S2-->>S1: 200 OK

```

### Diagram 3: Token Issuance and Refresh (Rotation)

```mermaid
sequenceDiagram
    autonumber
    participant C as Client Backend
    participant AS as Authorization Server
    participant DB as Auth Database

    Note over C, DB: Access Token Expires
    C->>AS: POST /token (grant_type=refresh_token, refresh_token=RT_A)
    AS->>DB: Check if RT_A is valid & not revoked
    AS->>DB: Invalidate RT_A
    AS->>DB: Generate RT_B
    AS-->>C: Return New Access Token + New RT_B
    Note over C, DB: If attacker tries to use stolen RT_A later...
    C->>AS: POST /token (refresh_token=RT_A)
    AS->>DB: RT_A flagged as ALREADY USED!
    AS->>DB: Security Breach Detected -> Revoke RT_B immediately
    AS-->>C: 400 Bad Request (Session Terminated)

```

---

## 9. Frequently Asked Questions (FAQ)

**Q: What is the difference between OAuth 2.0 and OpenID Connect (OIDC)?**
**A:** OAuth 2.0 is for *Authorization* (delegated access to APIs). It uses the Access Token. OIDC is a layer built on top of OAuth 2.0 for *Authentication* (verifying user identity). It introduces the ID Token (JWT) so the client application knows exactly who logged in.

**Q: When should I use JWT vs opaque tokens?**
**A:** Use **JWTs** for high-scale, microservice architectures where APIs need to validate tokens locally without adding latency or database overhead. Use **Opaque tokens** for legacy systems, monoliths, or ultra-high-security environments where the ability to instantly revoke a token directly at the database level is more important than network performance.

**Q: How do I secure refresh tokens in a web or mobile app?**
**A:** For web apps, never send them to the browser; keep them in the backend database (BFF pattern). For mobile apps, store them in the hardware-backed secure enclave (iOS Keychain or Android Keystore). Always implement **Refresh Token Rotation** on the server side to detect and mitigate theft.

**Q: How do PKCE and CSRF protections work?**
**A:** CSRF protection uses the `state` parameter to ensure the callback response corresponds to the exact browser session that started the flow. PKCE protects the Authorization Code from interception by requiring the client to prove it holds a secret (`code_verifier`) that matches the hashed challenge sent in the initial request.

**Q: How to handle token revocation and expiration in distributed systems using JWTs?**
**A:** Because JWT validation is stateless, you cannot simply delete them from a database.

1. Keep JWT expirations extremely short (e.g., 5 minutes).
2. Implement a push-based distributed blocklist (e.g., Redis). When a session is revoked, push the token's unique ID (`jti`) to Redis. API Gateways check this fast, in-memory cache before trusting the JWT signature.
