# The Comprehensive Bible: OAuth 2.0 Architecture & Implementation

## 1. Introduction: The Problem OAuth 2.0 Solves

Before modern Identity and Access Management (IAM), the internet suffered from the **Password Anti-Pattern**.

Imagine it is 2010. A user wants a new application, "PhotoApp", to import their contacts and profile details from Google. To do this, PhotoApp asks the user: *"Please enter your Google username and password here."* PhotoApp then logs into Google pretending to be the user.

**The critical problems with this approach:**

* **Over-privileged Access:** The user only wanted PhotoApp to *read* their basic profile. But with the password, PhotoApp can also *read their Gmail and delete their Google Drive*.
* **No Revocation:** The only way to stop PhotoApp is for the user to change their Google password, which breaks every other integration they have.
* **Massive Blast Radius:** If PhotoApp's database is breached, hackers gain the plaintext passwords to thousands of Google accounts.

**The Solution: OAuth 2.0 and Delegated Access**
OAuth 2.0 (RFC 6749) was created to solve this. It is a **Delegated Authorization Framework**. Instead of sharing passwords, the user is redirected to Google. Google asks the user: *"Do you want to grant PhotoApp permission to READ your profile data?"* If the user agrees, Google issues PhotoApp an **Access Token**.

Think of the Access Token as a **Valet Key** for a car. The valet key allows the driver to move the car 1 mile and park it, but it cannot open the trunk or the glovebox. OAuth 2.0 ensures applications only get the exact permissions they need, for a limited time, and can be revoked instantly without changing passwords.

* OAuth 2.0 is a widely used protocol for authorization.
* OAuth 2.0 lets the Resource Owner (the User) grant a Client (PhotoApp) access to their data (requested Scopes - email, name, etc.) on a Resource Server (Google's Profile API) — without ever sharing their passwords with the Client. The password is only ever given directly to the Authorization Server (Google).
* This is the core of how apps like Twitter or Spotify let users sign in with Google or Facebook.

---

## 2. The Big Misconception: Two Types of "Authorization"

The word **“authorization”** is used in two very different ways in IAM, which often causes confusion:

* **A. OAuth Authorization (What Google Does):**
*“Did this user give this app (React App) permission to access their Google profile data?”*
OAuth doesn’t care **how** the user logged in—MFA, password, FaceID, etc.—it only ensures the Authorization Server (Google) securely verified their identity and issued a token.
* **B. Application Authorization (What Your API Does):**
*“What can this user do inside my app?”*
Google does **not** manage your app’s roles or business logic. It’s entirely up to your application (.NET API) to check the user’s role (Admin, View-Only, etc.) and decide what actions they’re allowed to perform, like deleting a photo.

**🔥 The Golden Rule:**
Whenever you hear the term "OAuth 2.0", it is referring STRICTLY to "A" (Getting permission from Google). OAuth packs its bags and goes home the moment that token is issued; it has absolutely nothing to do with "B" (what the user can do inside your app like delete photos, add photos etc).

**Key Features:**

* Focused on authorization, not authentication methods.
* Uses access tokens to delegate permissions.
* Based on JSON over HTTP.
* Designed for web, mobile, and API-based applications.

---

## 3. The Core Concept: What "Authorization" Actually Means Here

To clear up any confusion right from the start, we need to define exactly what is being "authorized" in this flow.

In OAuth 2.0, **authorization only defines how a client (your React App) gets permission from Google to access a resource hosted by Google (like the user's profile data).**

When a user clicks "Login with Google" in your app, here is exactly what happens regarding authorization:

1. Google verifies the user has access to their own Google account (Authentication).
2. Google asks the user: *"Do you want to allow this React App to read your Google profile data?"*
3. The user clicks "Yes."
4. **That is the authorization.** Google is authorizing the client app (React App) to read the user's profile data from their Google account.

It only defines how the React App gets permission to access a resource hosted by Google. That is where OAuth's job ends.

**Crucial Distinction:** This does **not** mean authorizing what the user can do inside your Photo App. Google is not authorizing the user to "upload a photo" or "delete a photo" in your system. Application-level permissions are handled entirely by your own .NET API after the OAuth process is finished.

---

## 4. Application Stack & System Roles

To build a scalable IAM system, you must strictly define the boundaries of the protocol's components.

### Application Stack:

* **Frontend:** React App (Client)
* **Backend:** .NET Photo API (Resource Server)
* **Storage:** Database / Cloud Storage (photos saved here)
* **Login Provider:** Google (Authorization Server)
* **Standard:** OAuth 2.0 (Note: We are not using OIDC yet).

### System Roles (Very Important)

| OAuth Role | Actual Component | Definition |
| --- | --- | --- |
| **Resource Owner** | The User | The entity capable of granting access to a protected resource. They handle *Authentication* (proving who they are) and *Consent* (delegating access). |
| **Client** | React App | The application making requests on behalf of the Resource Owner. |
| **Authorization Server** | Google | The server that authenticates the user, obtains consent, and issues the tokens (e.g., `accounts.google.com`). |
| **Resource Server** | .NET Photo API | The API hosting the protected data (Photos) and internal app logic, which accepts and validates internal App Tokens. |

### Advanced Role Breakdowns (Internal vs External)

* **Human Consumer (External):** Alice wants to use PhotoApp. She is the Resource Owner of her Google profile. She authenticates at Google and consents to delegate access to PhotoApp.
* **Human Employee (Internal):** Bob, a PhotoApp Content Moderator. He logs into the internal **Admin Portal** (Client). He is the Resource Owner of his corporate identity and authorizes the Admin Portal to call internal APIs on his behalf.
* **Non-Human Entity (Machine-to-Machine):** A background "Thumbnail Generator Microservice". In the **Client Credentials Flow**, no human is present. The microservice acts as its own Resource Owner to get an internal token from your .NET Auth service to process images.

---

## 5. Where Photos Are Actually Stored & Protected Resources

In this example, photos are stored in **YOUR system**. Google does NOT store your photos here.

| Component | Responsibility |
| --- | --- |
| **.NET API** | Handles upload/download logic |
| **Database/Storage** | Stores photo files (e.g., AWS S3, Azure Blob, Local DB) |

The **resource** in OAuth terminology (for your backend) is: `User Photos`. They are accessible through your backend API.

Example endpoints:

* `POST /photos` → upload photo
* `GET /photos` → list photos
* `GET /photos/{id}` → view photo

---

## 6. What OAuth Actually Solves (and What It Doesn't)

**What the OAuth Protocol DOES solve:**

* **Passwordless Delegation:** It acts as a secure rulebook. The user types their password directly into Google's secure site (not your app). Google then hands your app an Access Token. Your app never sees the user's password.
* **Standardized Permission:** It defines exactly how that Access Token is securely issued to web, mobile, and API-based applications.

**What the OAuth Protocol does NOT solve (The "Under the Hood" Details):**

* **The specific method of authentication:** OAuth requires that Google authenticates the user, but it doesn't care *how*. It doesn't matter if Google asks for a password, fingerprint, SMS code, or YubiKey.
* **How permissions are stored:** It doesn't know if your .NET API uses SQL, MongoDB, or what your database tables look like.
* **How APIs implement business logic:** It has absolutely no idea what an `upload_photo` action is inside your specific app.

### Your API Permissions Are Separate

Your system defines internal permissions like:

| Action | Permission |
| --- | --- |
| **Upload photo** | allowed |
| **View photo** | allowed |
| **Delete photo** | restricted to owner |

These permissions are stored in **your database** (e.g., in a `Users` or `UserRoles` table) and enforced by your .NET API.

---

## 7. Core Concepts: Tokens, Scopes, Claims, and JWTs

### Tokens

* **Access Token:** The “Valet Key.” A credential used by the Client application to securely access an API. It represents *delegated authorization* (what the client can do), not *authentication* (who the user is).
* **Refresh Token:** A long-lived credential used by the Client to obtain new Access Tokens when they expire, avoiding frequent user reauthentication.
* **ID Token (OIDC Extension):** A JWT that contains verified user identity information (email, name). *OAuth 2.0 handles authorization; OpenID Connect adds authentication via this token.*

### Scopes and Claims

* **Scopes:** Defined at the OAuth level. They represent the *permissions* the client is requesting (e.g., `profile`, `email`, or internally `photos:write`).
* **Claims:** Key-value pairs inside a token asserting facts (e.g., `sub` for user ID, `exp` for expiration).

### JWT Structure and Validation

A JSON Web Token (JWT) consists of three Base64-URL encoded parts: `Header.Payload.Signature`.

* **Header:** Defines the algorithm (e.g., `RS256`) and Key ID (`kid`).
* **Payload:** Contains the claims (scopes, expiration, issuer).
* **Signature:** Cryptographic math proving the token was created by the server and hasn't been tampered with.

#### How JWT Signatures are Used (Step-by-Step API Validation):

1. **Phase 1: Signing:** The Auth Server (Google, or your internal .NET Auth service) creates the Header/Payload. It hashes them (SHA-256) and encrypts that hash with its highly guarded **Private Key**. This is the Signature.
2. **Phase 2: Verification:** Your .NET API Gateway receives the JWT. It downloads the Auth Server's **Public Key**.
3. **The Math:** The Gateway hashes the Header/Payload itself (Hash A). It uses the Public Key to decrypt the Signature on the token (Hash B).
4. **The Result:** If Hash A == Hash B, the token is perfectly intact and genuinely issued by the trusted server. The request is allowed through instantly without querying a database.

> **Digital Signatures vs. Encryption**
> * **Encryption (Keeping a Secret):** Lock it with a Public Key → Only recipient can unlock with Private Key.
> * **Signature (Proving Who You Are):** Sign it with a Private Key → Everyone can verify it with a Public Key.
> 
> 

### Extensions

* **PKCE (Proof Key for Code Exchange):** Cryptographic extension preventing malicious apps from intercepting the Authorization Code.
* **Token Introspection (RFC 7662):** An endpoint (`/introspect`) where an API queries the Auth Server to check if an opaque token is active.
* **Token Revocation (RFC 7009):** An endpoint (`/revoke`) to invalidate tokens.

---

## 8. Phase 0: The Prerequisite Setup (Client Registration)

Before your React App or .NET API can talk to Google, they must establish trust.

1. **The Introduction:** You go to the Google Cloud Console and register "PhotoApp".
2. **The Strict Return Address (`redirect_uri`):** You register the exact URL where users should be sent back (e.g., `https://photoapp.com/callback`). If a request asks to redirect anywhere else, Google instantly blocks it.
3. **The ID Badge (`client_id`):** Google gives PhotoApp a public identifier (e.g., `photoapp_client_123`).
4. **The Secret Password (`client_secret`):** Google gives your .NET backend a highly confidential password to prove its identity during server-to-server token exchanges. *(Note: This is never placed in the React frontend).*

---

## 9. Full Flow Step-by-Step & Deep Dive

Here is the exact step-by-step flow, mapping the high-level concepts to the underlying HTTP payloads.

1. **User opens React app:** Navigates to `https://photoapp.com` and clicks *Login with Google*.
2. **Redirect to Google (The Authorization Request):** App redirects to Google's OAuth endpoint.
```http
GET /o/oauth2/v2/auth?
  response_type=code
  &client_id=photoapp_client_123
  &redirect_uri=https://photoapp.com/callback
  &scope=profile email
  &state=random_state_88291
  &code_challenge=E9Melhoa2... (PKCE Hint)
  &code_challenge_method=S256 HTTP/1.1
Host: accounts.google.com

```


3. **Google authenticates user:** User proves their identity to Google (e.g., email, password, MFA).
4. **User grants OAuth permission:** Google asks, *"Allow PhotoApp to access your profile?"* User clicks *Allow*.
5. **Authorization code returned:** Google redirects back to your React app with a temporary, single-use code.
```http
HTTP/1.1 302 Found
Location: https://photoapp.com/callback?code=SplxlOBeZQ...&state=random_state_88291

```


6. **Backend exchanges code for token (The Private Backroom):** React sends this code to your .NET API. Your .NET API opens a direct, encrypted, hidden tunnel to Google to trade the code for the Token.
```http
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Authorization: Basic cGhvdG9hcHBfY2xpZW50... (client_id:client_secret)

grant_type=authorization_code
&code=SplxlOBeZQ...
&redirect_uri=https://photoapp.com/callback
&code_verifier=the_raw_secret_password_from_pkce

```


7. **Backend identifies the user:** Google validates the `client_secret` and PKCE `code_verifier`, then returns the Google Access Token and ID Token. Your backend decodes the ID Token to see the user is `john@gmail.com` and finds/creates this user in **your database**.
8. **Your backend creates an application session:** Your API issues its *own* token (an internal JWT) representing the user's session in PhotoApp, returning it to React.
9. **React calls your Photo API:** React uses your internal token to make requests.
```http
POST /photos HTTP/1.1
Host: api.photoapp.com
Authorization: Bearer <Your_Internal_App_JWT>

```


10. **Your API checks permissions:** Your .NET API checks its own database: *Who is this user? Can they upload?* and allows or denies the request.

---

## 10. Architecture & Flow Diagram

The following diagram illustrates exactly where **OAuth Authorization** ends and where **Application Authorization** begins, incorporating PKCE and internal token handoffs.

```mermaid
sequenceDiagram
    actor User
    participant React as React App (Client)
    participant Google as Google (OAuth Server)
    participant API as .NET API (Resource Server)
    participant DB as Database (Storage & Roles)

    User->>React: Click "Login with Google"
    React->>React: Generate PKCE verifier & challenge
    React->>Google: Redirect for Authentication (+ challenge)
    
    Note over User,Google: 1. OAUTH AUTHORIZATION (Delegated Access)<br/>Answers: "Did the user let this app access their Google profile?"
    Google-->>React: Return Auth Code via redirect
    React->>API: Send Auth Code & PKCE verifier
    API->>Google: POST /token (Code + Secret + Verifier)
    Google-->>API: Returns Google Tokens (Identity: john@gmail)
    
    Note over API,DB: 2. APPLICATION AUTHORIZATION (IAM / Permissions)<br/>Answers: "What actions can John perform inside Photo App?"
    API->>DB: Lookup internal user roles for john@gmail
    DB-->>API: Returns role: "Standard User"
    API-->>React: Issue Internal Application Session Token (JWT)
    
    User->>React: Click "Delete Photo"
    React->>API: DELETE /photos/1 (with App Token)
    API->>API: Check internal rules: Can Standard User delete?
    API-->>React: 403 Forbidden (Action blocked by your backend)

```

---

## 11. Clear Responsibility Table

| System | Responsibility |
| --- | --- |
| **Google** | Authenticate user (Passwords, 2FA, etc.) |
| **Google** | Issue OAuth token (Confirm user gave consent to the client) |
| **React App** | Start OAuth flow & display UI |
| **.NET API** | Identify user via Google's Token |
| **.NET API** | Enforce App Permissions & Roles (Can upload? Can delete?) |
| **Storage** | Store actual photo files |

### One Simple Sentence That Clears Everything

OAuth only answers:

> **"Did the user allow this application to access their Google data?"**

It does **NOT** answer:

> **"What actions can the user perform inside your Photo Application?"** (That is handled by your .NET backend logic).

---

## 12. System Architecture: Scaling PhotoApp

When PhotoApp handles 50,000 requests per second on New Year's Eve, your .NET API cannot query the database for every single image upload to ask, *"Is this token valid?"* The database would melt down.

To achieve zero downtime, PhotoApp divides its architecture into three main layers:

1. **The Edge Layer (The Bouncers):** Consists of a Web Application Firewall (WAF) and the .NET API Gateway. Every request passes through here. The Gateway uses its CPU to verify the JWT digital signatures locally, without ever touching the database. It uses **Redis** (an in-memory cache) to hold a blocklist of revoked tokens.
2. **The IAM Control Plane (The Passport Office):** Your internal authentication microservice and distributed database. It only wakes up when a user logs in or refreshes a session.
3. **The Resource Servers (The Vaults):** The backend .NET microservices (e.g., `ImageProcessingService`, `StorageService`). They blindly trust the API Gateway.

**Scenario: The Stolen Laptop (Redis Blocklist in Action)**
Bob, a PhotoApp moderator, gets his laptop stolen. The thief has a valid JWT for the Admin Portal that doesn't expire for 10 minutes.

1. An IT Admin clicks "Revoke" in the system.
2. The control plane instantly pushes Bob's JWT ID (`jti`) to the **Redis Blocklist** at the Edge.
3. The thief tries to delete a photo. The API Gateway sees the token ID on the Redis "Wanted" list and instantly rejects it with `401 Unauthorized`, killing the token mid-flight without slowing down the rest of the system.

---

## 13. Security Considerations: How Hackers Exploit OAuth 2.0

When an OAuth 2.0 implementation is breached, it is almost never because a hacker "cracked the math." It is because a developer forgot to lock a specific door.

### Attack 1: CSRF (Cross-Site Request Forgery)

* **The Attack:** A hacker starts a login flow with *their own* Google account, pauses it, and tricks a victim into clicking the callback URL. The victim's PhotoApp session gets permanently linked to the hacker's Google account.
* **The Mitigation (`state` parameter):** React generates a random `state` string, saves it in a cookie, and passes it to Google. When the code comes back, React verifies the URL's `state` matches the cookie. The hacker's trap fails because the victim's browser doesn't have the hacker's `state` cookie.

### Attack 2: Authorization Code Interception

* **The Attack:** A malicious browser extension (or fake mobile app) intercepts the `photoapp.com/callback?code=123` redirect, steals the code, and trades it for a token.
* **The Mitigation (PKCE):** Proof Key for Code Exchange (PKCE) forces the React app to generate a temporary secret password (`code_verifier`) and send a hash of it (`code_challenge`) to Google on step 1. When trading the code, the backend must provide the raw password. The hacker's extension has the code, but lacks the raw password trapped in React's memory.

### Attack 3: Token Leakage

* **The Attack:** Putting tokens directly in URLs (the old "Implicit Flow") leaves them in browser histories, Wi-Fi logs, and Google Analytics.
* **The Mitigation:** Never use the Implicit Flow. Always use the **Authorization Code Flow**, where tokens are delivered securely server-to-server and passed via the `Authorization: Bearer` HTTP header.

### Attack 4: Token Theft via XSS (Cross-Site Scripting)

* **The Attack:** You store your internal App JWT in the browser's `localStorage`. A hacker posts a photo comment containing malicious JavaScript. When viewed, the script reads `localStorage` and steals the token.
* **The Mitigation (The BFF Pattern):** The React app never sees the token. Your .NET backend (acting as a Backend-For-Frontend proxy) holds the JWT in memory and issues an encrypted **`HttpOnly` Cookie** to React. Browsers block JavaScript from reading `HttpOnly` cookies, completely neutralizing XSS token theft.

---

## 14. Trade-Offs and Design Decisions

| Decision | Option 1 | Option 2 | Recommendation for PhotoApp |
| --- | --- | --- | --- |
| **Flow Selection** | **Auth Code + PKCE** (Highest Security) | **Client Credentials** (Machine-to-Machine) | Use **Auth Code + PKCE** for React/Mobile users. Use **Client Credentials** for background microservices. |
| **Token Type** | **JWT / Value Token** (Stateless, Fast math check) | **Opaque / Reference Token** (Requires DB lookup on every call) | **JWTs** at the API edge for massive scale, combined with a Redis blocklist for revocation. |
| **Refresh Strategy** | **Standard Expiration** (Lives for 30 days) | **Refresh Token Rotation (RTR)** (One-time use tickets) | **RTR**. If a thief reuses an old refresh token, the server detects the theft and instantly detonates the entire session. |

---

## 15. Integration Examples & FAQ

While the mechanics are universal, different Identity Providers use different URLs. If you swap Google for another provider, update your endpoints:

* **Okta / Auth0:** `https://{domain}.okta.com/oauth2/default/v1/authorize`
* **Microsoft Entra ID:** `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize`
* **AWS Cognito:** `https://{prefix}.auth.{region}.amazoncognito.com/oauth2/authorize`
* **Keycloak:** `https://{domain}/realms/{realm}/protocol/openid-connect/auth`

### Frequently Asked Questions

**Q: What is the difference between OAuth 2.0 and OpenID Connect (OIDC)?**
**A:** OAuth 2.0 is for *Authorization* (delegated access to APIs, using the Access Token). OIDC is a layer built on top of OAuth 2.0 for *Authentication* (verifying user identity, using the ID Token).

**Q: When should I use JWT vs opaque tokens?**
**A:** Use **JWTs** for high-scale architectures where APIs need to validate tokens locally without database latency. Use **Opaque tokens** for ultra-high-security environments where the ability to instantly revoke a token directly at the database level is more critical than performance.

**Q: How do I secure refresh tokens in a web app?**
**A:** Never send them to the browser's JavaScript memory. Keep them in the backend database (BFF pattern) and link them to the user via an `HttpOnly` cookie. Always implement **Refresh Token Rotation**.
