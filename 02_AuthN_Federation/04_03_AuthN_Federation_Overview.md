## Authentication & Federation in Zero-Trust IAM

### 1. Where It Fits in the Zero-Trust Architecture

A common point of confusion is drawing the line between the **Identity Provider (IdP)** and the **IAM Control Plane (Authorization Server)**.

* **The IdP (Authentication & Federation):** This is the realm of **OIDC** and **SAML**. The IdP's only job is to answer the question: *"Who is this user, and can they prove it?"* It handles the login screens, the MFA prompts, and federating trust out to partner directories (like Azure AD).
* **The IAM Control Plane (Authorization):** This is the realm of **OAuth2**. Once the IdP authenticates the user, the IAM Control Plane takes over. It maps the user to their internal roles (e.g., `WealthManager`), generates the JWT Access Token (the cryptographic ticket), and issues Refresh Tokens.

**Summary:** OIDC/SAML log you in. OAuth2 gives your client app the permissions to access the APIs.

### 2. Build vs. Buy: The IAM Crossroads

When implementing this layer, organizations must decide whether to build their own Identity/Authorization Server (e.g., using open-source libraries like IdentityServer4 or Spring Security) or buy an Enterprise Enterprise IAM solution.

#### The "Build" Scenario (Custom Identity Server)

* **Use Case:** You are a highly specialized tech company (like Netflix or GitHub) with millions of consumers, unique login flows (e.g., passwordless magic links tied to custom hardware), and a massive engineering team to maintain cryptographic security updates.
* **Trade-offs:** * *Pros:* Ultimate flexibility, no vendor lock-in, zero per-user licensing costs.
* *Cons:* You are responsible for patching zero-day crypto vulnerabilities. If you implement OAuth2 or SAML incorrectly, your entire bank is breached. High engineering maintenance overhead.

#### The "Buy" Scenario (Reference: CyberArk Workforce Identity / CIAM)

* **Use Case (MoneyGuard):** A highly regulated global bank. MoneyGuard cannot risk a custom-built login screen having an injection flaw. They purchase **CyberArk Workforce Identity** for employee SSO and **CyberArk Customer Identity (CIAM)** for the consumer mobile app.
* **How it fits:** CyberArk acts as *both* the IdP (handling MFA, Biometrics, and SAML federation with corporate clients) *and* the IAM Control Plane (issuing the OAuth2 JWTs and managing the Identity Store).
* **Trade-offs:**
* *Pros:* Bank-grade security out-of-the-box, turnkey MFA, SOC2/FedRAMP compliance, guaranteed SLA, native integration with Privileged Access Management (PAM) for database admins.
* *Cons:* Vendor lock-in. Licensing costs scale with the number of users. Customizing the UI of the login screens has strict boundaries.

### 3. The Token Lifecycle: Refresh & Revocation

#### Refresh Tokens

JWT Access Tokens should have a short lifespan (e.g., 15 minutes) to limit the damage if stolen. But we cannot force Alice to log in with MFA every 15 minutes.

* **The Mechanism:** When the IAM Control Plane (e.g., CyberArk) issues the 15-minute Access Token, it also issues a **Refresh Token** (valid for, say, 12 hours).
* **The Flow:** When the Access Token expires, the Client App (Teller App backend) silently sends the Refresh Token to `auth.moneyguard.com/oauth2/token`. CyberArk verifies the Refresh Token, ensures Alice hasn't been fired in the last 15 minutes, and issues a *new* 15-minute Access Token.
* **Security Best Practice (Refresh Token Rotation):** Every time a Refresh Token is used, CyberArk invalidates it and issues a *new* Refresh Token alongside the new Access Token. If an attacker steals a Refresh Token, the moment they try to use it, CyberArk detects a "reuse" anomaly and instantly locks Alice's account.

#### Token Revocation

Because Access Tokens (JWTs) are mathematically verified locally by the API Gateway (PEP), they cannot simply be "deleted" from a central server.

* **Standard OAuth2 Revocation Endpoint:** The client app can send a POST request to `/oauth2/revoke` to invalidate a Refresh Token, forcing the user to log in again.
* **The Zero-Trust Active Revocation (Redis):** As detailed in our architecture, if an Access Token is stolen and the Admin clicks "Revoke", the IAM Control Plane pushes the token's `jti` (JWT ID) to a high-speed Redis blocklist. The API Gateway checks this list in <1ms before allowing the request, effectively revoking a stateless token mid-flight.

---

## OAuth 2.0: The Authorization Standard

#### 1. Introduction & Flow

OAuth 2.0 is an **Authorization** framework. It was designed to allow a user to grant a third-party application access to their resources *without* handing over their password.

**The Authorization Code Flow (with PKCE):**
This is the industry-standard flow for web and mobile apps.

1. **Request:** The Client App redirects the user to the IAM Control Plane requesting specific `scopes` (e.g., `scope=wires:write`).
2. **Consent:** The user logs in (handled by the IdP) and clicks "Approve".
3. **The Code:** The IAM Control Plane redirects the user back to the Client App with a single-use Authorization Code.
4. **The Exchange:** The Client App's backend securely contacts the IAM Control Plane, authenticates itself (`client_secret`), and exchanges the code for an Access Token.
5. **PKCE (Proof Key for Code Exchange):** For public clients (like mobile apps that can't hold a `client_secret`), PKCE uses cryptographic hashing (a code verifier and challenge) during steps 1 and 4 to prove that the app exchanging the code is the exact same app that requested it.

#### 2. Trade-offs

* **Pros:** Highly flexible. Secures APIs without exposing user credentials to client apps. Granular scopes limit application power.
* **Cons:** **OAuth2 is NOT an authentication protocol.** It does not tell the client app *who* logged in; it only gives the app a token to access an API. Misusing OAuth2 for login leads to severe security vulnerabilities (which is why OIDC was invented).

#### 3. Use Cases

* **Machine-to-Machine (M2M):** The `Fraud Service` using the Client Credentials Grant to request a token to talk to the `Ledger Service` (No human involved).
* **Delegated Access:** Alice allowing a third-party tax software to read her MoneyGuard account balance `scope=accounts:read`.

---

## OpenID Connect (OIDC): The Identity Layer

#### 1. Introduction & Flow

Because OAuth 2.0 didn't solve the "Who is this user?" problem, OpenID Connect (OIDC) was built directly on top of OAuth 2.0 to provide **Authentication**.

It uses the exact same flows (like the Authorization Code Flow), but introduces a new, critical element: the **ID Token**.

**The OIDC Flow Addition:**

1. The Client App requests the `openid` scope (e.g., `scope=openid profile email wires:write`).
2. During the Token Exchange (Step 4 of OAuth2), the IAM Control Plane returns **two** tokens:
* **The Access Token:** Meant for the API Gateway (contains scopes).
* **The ID Token:** A JWT meant *strictly for the Client App*. It contains identity claims (e.g., `name: Alice`, `email: alice@moneyguard.com`, `auth_time`).

#### 2. Trade-offs

* **Pros:** Standardizes identity. The Client App can decode the ID Token to display "Welcome, Alice!" without having to query a backend database. Built on modern JSON and REST standards.
* **Cons:** Requires strict validation by the client app (verifying the `nonce`, `aud`, and `iss` of the ID token) to prevent token substitution attacks.

#### 3. Use Cases

* **Modern Web/Mobile App Login:** When Alice logs into the MoneyGuard mobile app, OIDC authenticates her (giving the app her profile via the ID Token) and authorizes the app to make wire transfers on her behalf (via the Access Token).
* **Social SSO:** "Log in with Google" or "Log in with Apple" are strictly OIDC implementations.

---

## SAML 2.0: Enterprise Federation

#### 1. Introduction & Flow

Security Assertion Markup Language (SAML) is the grandfather of enterprise federation. Built on XML, it allows two disparate organizations to establish a severe, cryptographic trust relationship.

In SAML, the Client App is called the **Service Provider (SP)**, and the authentication server is the **Identity Provider (IdP)**.

**SP-Initiated Flow:**

1. **The Request:** "Acme Corp" HR employee goes to `corporate.moneyguard.com` (The SP).
2. **The Redirect (SAML AuthnRequest):** MoneyGuard redirects the browser to Acme Corp's IdP (e.g., Azure AD) with an XML document asking for authentication.
3. **Authentication:** The user logs into Azure AD.
4. **The Assertion (SAML Response):** Azure AD generates an XML document (The Assertion) stating: *"This is Bob, he is in HR, and I have cryptographically signed this statement."* This is posted back to MoneyGuard via the user's browser.
5. **Validation:** MoneyGuard verifies the XML signature using Acme Corp's public certificate and grants Bob a session.

#### 2. Trade-offs

* **Pros:** Extremely battle-tested. Standard for B2B Enterprise SSO. Supports deep, complex XML assertions mapping entire organizational structures.
* **Cons:** Uses heavy, legacy XML format. Not designed for mobile apps or modern SPA (Single Page Applications) like React/Angular. Prone to XML Signature Wrapping (XSW) attacks if the parser is not perfectly configured.

#### 3. Use Cases

* **B2B Corporate Federation:** A Fortune 500 company mandates that all third-party SaaS tools (like MoneyGuard's payroll portal) integrate with their legacy Active Directory via SAML.
* **Legacy App Integration:** Connecting older, on-premise banking software that does not understand modern JSON/OIDC into the central CyberArk Identity system.
