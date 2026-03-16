# The Comprehensive Bible: OAuth 2.0 Architecture & Implementation

## 1. Introduction: The Problem OAuth 2.0 Solves

Before modern Identity and Access Management (IAM), the internet suffered from the **Password Anti-Pattern**.

Imagine it is 2010. A user wants a third-party application, "PhotoApp", to track their profile and contacts from "Google". To do this, PhotoApp asks the user: *"Please enter your Google username and password here."* PhotoApp then logs into Google pretending to be the user.

**The critical problems with this approach:**

* **Over-privileged Access:** The user only wanted PhotoApp to *read* their profile. But with the password, PhotoApp can also *read their Gmail and delete their Google Drive*.
* **No Revocation:** The only way to stop PhotoApp is for the user to change their Google password, which breaks every other integration they have.
* **Massive Blast Radius:** If PhotoApp's database is breached, hackers gain the plaintext passwords to thousands of Google accounts.

**The Solution: OAuth 2.0 and Delegated Access**
OAuth 2.0 (RFC 6749) was created to solve this. It is a **Delegated Authorization Framework**. Instead of sharing passwords, the user is redirected to Google. Google asks the user: *"Do you want to grant PhotoApp permission to READ your profile data?"* If the user agrees, Google issues PhotoApp an **Access Token**.

Think of the Access Token as a **Valet Key** for a car. The valet key allows the driver to move the car 1 mile and park it, but it cannot open the trunk or the glovebox. OAuth 2.0 ensures applications only get the exact permissions they need, for a limited time, and can be revoked instantly without changing passwords.

* OAuth 2.0 is a widely used protocol for authorization.
* OAuth 2.0 lets the Resource Owner (the User) grant a Client (PhotoApp) access to their data (requested Scopes - email, name etc.) on a Resource Server (Google's Profile API) — without ever sharing their passwords with the Client. The password is only ever given directly to the Authorization Server (Google).
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

### Why is it called an "Authorization" Framework?

- In OAuth 2.0, the word **authorization** refers to the act of delegation.
- It is called an authorization framework because the **User (Resource Owner)** is explicitly *authorizing* the **Client (React App)** to access the User's own profile data hosted by Google.
- When a user clicks "Login with Google", the framework simply provides a secure mechanism for the User to say to Google: *"I authorize this React App to read my profile data on my behalf."* Because the core action is the user granting permission, OAuth 2.0 is strictly an authorization protocol, not an authentication protocol.

---
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

## 4. Application Stack & Deep-Dive System Roles

To build a scalable IAM system, you must strictly define the boundaries of the protocol's components.

### Application Stack:

* **Frontend:** React App (Client)
* **Backend:** .NET Photo API (Resource Server)
* **Storage:** Database / Cloud Storage (photos saved here)
* **Login Provider:** Google (Authorization Server)
* **Standard:** OAuth 2.0 (Note: We are not using OIDC yet).

### Roles

* **Resource Owner:** The entity capable of granting access to a protected resource (e.g., Alice, the user).
    * **Dual Responsibilities of the Resource Owner:** For OAuth 2.0 to function securely, the Resource Owner performs two distinct actions during the authorization flow:
       * **Authentication:** Proves *who they are* to the Authorization Server (logging in with password and MFA). The Client application never sees user credentials.
       * **Consent (Delegation):** Explicitly grants permission (Scopes) to the Client application (clicking **Allow**).

    * **Types of Resource Owners (PhotoApp Context):**
        * **Human Consumer (External):** Alice wants to use PhotoApp. She is the Resource Owner of her Google profile. She authenticates at `accounts.google.com` and consents to delegate `profile:read` access to PhotoApp.
        * **Human Employee (Internal):** Bob, a PhotoApp Moderator. He logs into the internal **Admin Portal**. Bob is the Resource Owner of his *own corporate identity and session*. He authorizes the Admin Portal to call the internal .NET APIs on his behalf.
        * **Non-Human Entity (Machine-to-Machine):** A background Thumbnail Generator Microservice. In the **Client Credentials Flow**, no human is present. The microservice *acts as the Resource Owner* of its own data and authenticates itself to the Auth Server to obtain a token.
* **Client:** The application making requests on behalf of the Resource Owner (e.g., React App).
* **Authorization Server (IAM):** The server that authenticates the user, obtains consent, and issues tokens (e.g., Google).
* **Resource Server:** The API hosting the protected data (e.g., .NET Photo API).

---

## 5. Where Photos Are Actually Stored & Protected Resources

In this example, photos are stored in **YOUR system**. Google does NOT store your photos here.

| Component | Responsibility |
| --- | --- |
| **.NET API** | Handles upload/download logic |
| **Database/Storage** | Stores photo files (e.g., AWS S3, Azure Blob, Local DB) |

The **resource** in OAuth terminology (for your backend) is: `User Photos`. They are accessible through your backend API (which is called the **Resource Server**).

Example endpoints:

* `POST /photos` → upload photo
* `GET /photos` → list photos
* `GET /photos/{id}` → view photo

---

## 6. What OAuth Actually Solves (and What It Doesn't)

OAuth 2.0 is a widely used **protocol** (a set of rules) for **delegated authorization**.

**What the OAuth Protocol DOES solve:**

* **Passwordless Delegation:** It acts as a secure rulebook. The user types their password directly into Google's secure site (not your app). Google then hands your app an Access Token. Your app never sees the user's password.
* **Standardized Permission:** It defines exactly how that Access Token is securely issued to web, mobile, and API-based applications.

**What the OAuth Protocol does NOT solve (The "Under the Hood" Details):**

* **The specific method of authentication:** OAuth requires that Google authenticates the user, but it doesn't care *how*. It doesn't matter if Google asks for a password, fingerprint, SMS code, or YubiKey. OAuth just waits for Google to say, *"Authentication successful, here is the token."*
* **How permissions are stored:** It doesn't know if your .NET API uses SQL, MongoDB, or what your database tables look like.
* **How APIs implement business logic:** It has absolutely no idea what an `upload_photo` or `delete_photo` action is inside your specific app. That is entirely up to your backend.

### Your API Permissions Are Separate

Your system defines internal permissions like:

| Action | Permission |
| --- | --- |
| **Upload photo** | allowed |
| **View photo** | allowed |
| **Delete photo** | restricted to owner |

These permissions are stored in **your database** (e.g., in a `Users`, `Photos`, or `UserRoles` table) and enforced by your .NET API.

---

## 7. Tokens, Scopes, Claims & JWT Verification Math

### Tokens

* **Access Token:** It a temporary security credential used to authenticate a user or application and authorize access to specific resources and APIs. The “Valet Key.” A credential used by the Client application to securely access the Resource Server.
   * **Purpose (Authorization, Not Authentication):** Represents *delegated authorization*. It communicates exactly *what* the client is allowed to do, but it is **not** intended to prove *who* the user is.
   * **Format Agnostic:** OAuth 2.0 does not define the structure of an Access Token. It may be an opaque string or a structured JSON Web Token (JWT).
   * **The Identity Trap (Architectural Warning):** In modern enterprise systems, Access Tokens are commonly JWTs and include a subject identifier (`sub`). However, **Resource Servers must treat Access Tokens strictly as authorization artifacts and must never use them as proof of user authentication.**
   * **Strict Validation Rules:** An API must validate `iss` (Trusted issuer?), `aud` (Meant for this API?), `exp` (Still valid?), and `scope` (Has required permissions?).

* **Refresh Token:** A long-lived credential used by the Client to obtain new Access Tokens when they expire. *(Problem solved: Allows Access Tokens to remain short-lived for security while avoiding frequent user reauthentication).*
* **ID Token (OIDC Extension):** A JWT that contains verified user identity information (subject, name, email). *(Problem solved: OAuth 2.0 handles authorization, while OpenID Connect adds authentication).*

### Scopes and Claims

* **Scopes:** Defined at the OAuth 2.0 level. They represent the *permissions* the client is requesting (e.g., `photos.upload, openid, contacts.readonly`).
* **Claims:** Key-value pairs inside a token asserting facts (e.g., `sub, email, given_name, iat, exp` for user ID).

### JWT Structure and Validation (The Local Math)

A JSON Web Token (JWT) consists of three Base64-URL encoded parts: `Header.Payload.Signature`.

* **Header:** Defines the algorithm (e.g., `RS256`) and Key ID (`kid`).
* **Payload:** Contains the claims (scopes, expiration, issuer).
* **Signature:** Cryptographic math proving the token was created by the Authorization Server and hasn't been tampered with.

**How the JWT signature is used (Step-by-Step without database calls):**

* **Phase 1: The Auth Server "Signs" the Token**
	* Google creates the plaintext JWT Header and Payload.
	* **The Hash:** Google runs that plaintext through a SHA-256 algorithm to create a unique string (let's call it **Hash A**).
	* **The Signature:** Google encrypts **Hash A** using its tightly guarded **Private Key**. This encrypted hash becomes the Signature attached to the back of the token.

* **Phase 2: The API Gateway "Verifies" the Token**
	* Your .NET API receives the JWT. It downloads Google's **Public Key**.
	* **Step 1 (The Gateway's Hash):** The .NET API takes the plaintext Header and Payload from the token and runs them through SHA-256 itself to create its own hash (**Hash B**).
	* **Step 2 (The Decryption):** The .NET API applies Google's **Public Key** to the encrypted Signature. Because it was encrypted with the Private Key, the Public Key successfully decrypts it, revealing Google's original hash (**Hash A**).
	* **Step 3 (The Final Check):** The API asks: **Does Hash B exactly equal Hash A?** If yes, the token is perfectly intact and genuinely issued by Google.

> #### **The Golden Rule: Signatures vs. Encryption**
> 
> 
> * **Data Encryption (Keeping a Secret):** "Lock it -> Only recipient can unlock." You encrypt with a public key; only the recipient's private key can read it. (Analogy: A locked box).
> * **Digital Signatures (Proving Who You Are):** "Sign it -> Everyone can verify." You sign with your private key; anyone can use your public key to verify it. (Analogy: A wax seal).
> 
> 

### PKCE, Introspection, and Revocation

* **PKCE (Proof Key for Code Exchange):** A cryptographic extension for the Authorization Code flow. *(Prevents malicious apps from intercepting the Authorization Code).*
* **Token Introspection (RFC 7662):** An endpoint (`/introspect`) where an API queries if an opaque token is active.
* **Token Revocation (RFC 7009):** An endpoint (`/revoke`) to invalidate tokens.

---

## 8. Deep Dive: Redirection and Payload Flows

To understand OAuth 2.0, you must track exactly how the browser redirects the user and how the backend servers whisper to each other securely.

### Phase 0: The Prerequisite Setup (Client Registration)

**The Problem:** Google cannot just hand out "Valet Keys" to any website. If Google doesn't have a strict list of safe return addresses, a hacker could trick Google into sending the Auth Code to `hacker-site.com/callback`!

**The Solution:** You register PhotoApp in the Google Cloud Console.

1. **The Introduction:** You register "PhotoApp."
2. **The Strict Return Address (`redirect_uri`):** You register the exact URL where users are sent back (e.g., `https://photoapp.com/callback`).
3. **The ID Badge (`client_id`):** Google gives PhotoApp a public identifier (`photoapp_client_99`).
4. **The Secret Password (`client_secret`):** Google gives your .NET backend a highly confidential password to prove its identity during Server-to-Server exchanges.

```mermaid
sequenceDiagram
    autonumber
    participant App as PhotoApp Developers
    participant IAM as Google Cloud Console

    Note over App, IAM: Phase 0: One-time setup before any user logs in
    
    App->>IAM: 1. The Introduction (Register App Name)
    App->>IAM: 2. Set Strict Return Address (redirect_uri: https://photoapp.com/callback)
    
    Note over IAM: Google saves the allowed<br/>URL to prevent routing attacks
    
    IAM-->>App: 3. Issue ID Badge (client_id: photoapp_client_99)
    IAM-->>App: 4. Issue Secret Password (client_secret)
    
    Note over App: PhotoApp .NET Backend stores the<br/>client_secret securely in its vault.

```

### Scenario A: The External Application (PhotoApp / React)

This is the full flow step-by-step, including the exact HTTP payloads sent between your React App, Google, and the .NET Backend.

```mermaid
sequenceDiagram
    autonumber
    participant Alice as Alice (Browser)
    participant React as PhotoApp Backend
    participant Google as accounts.google.com (Auth Server)
    participant GoogleAPI as Google Profile API (Resource Server)

    Alice->>React: 1. Clicks "Connect Google Profile"
    React->>React: Generates PKCE Secret & Hash
    React->>Alice: Redirects browser to Google
    Alice->>Google: 2. GET /authorize (Sends PKCE Hash)
    Google-->>Alice: Prompts Login & Consent Screen
    Alice->>Google: 3. Authenticates & Clicks "Allow"
    Google->>Alice: Redirects browser back to PhotoApp
    Alice->>React: 4. GET /callback?code=123 (Delivers Code)
    React->>Google: 5. POST /token (Sends Code + PKCE Secret)
    Google->>Google: Verifies PKCE Secret matches Hash
    
    %% THE DIFFERENCE IS HERE: Only Access Token is returned
    Google-->>React: 6. Returns Access Token (No ID Token)
    
    %% THE API CALL IS TO GOOGLE, NOT .NET
    React->>GoogleAPI: 7. GET /v1/profile + Bearer Access Token
    GoogleAPI->>GoogleAPI: Verifies Token
    GoogleAPI-->>React: 8. Returns Profile Data

```

#### Step 1 & 2: The Authorization Request (Browser to Auth Server)

Alice clicks "Login with Google". React generates the PKCE secret password, creates the "hint" (`code_challenge`), and redirects Alice to Google.

```http
GET /o/oauth2/v2/auth?
  response_type=code
  &client_id=photoapp_client_99
  &redirect_uri=https://photoapp.com/callback
  &scope=profile email
  &state=random_state_88291
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGZxkVj...
  &code_challenge_method=S256 HTTP/1.1
Host: accounts.google.com

```

#### Step 3: Authentication and Consent (The Human Element)

Google looks at the browser and shows Alice a login screen. After she authenticates, Google shows a **Consent Screen**: *"PhotoApp wants to read your profile. Do you allow this?"* Alice clicks **Allow**.

#### Step 4: The Callback (Auth Server to Browser to Client)

Google generates a temporary "Authorization Code" (valid for 60 seconds). It redirects Alice's browser back to PhotoApp.

```http
HTTP/1.1 302 Found
Location: https://photoapp.com/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=random_state_88291

```

#### Step 5: The Token Exchange (The Private Backroom)

React sends this code to your .NET backend. The backend opens a direct, encrypted tunnel to Google to trade the code for the Tokens.

```http
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic cGhvdG9hcHBfY2xpZW50Xzk5OnNlY3JldF9wYXNzd29yZA==

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https://photoapp.com/callback
&code_verifier=the_raw_secret_password_generated_in_step_1

```

**Line-by-Line Translation:**

1. **The ID Badge:** `Authorization: Basic cGhvdG...` (.NET mashes the `client_id` and `client_secret` to prove its identity).
2. **The Action:** `grant_type=authorization_code` ("I am here to trade a code for a token").
3. **The Receipt:** `&code=Splxl...` ("Here is the code Alice gave me").
4. **The PKCE Secret:** `&code_verifier=the_raw_secret...` (.NET hands over the **unscrambled, raw password**. Google hashes it and checks if it matches the Hint from Step 1).

#### Step 6 & 7: The Delivery & Identity

Google responds with the Tokens. Your .NET backend identifies the user (`john@gmail.com`), finds them in your DB, and creates an internal application session (issuing its own token to React).

#### Step 8 & 9: The API Call (.NET Application Authorization)

React calls the .NET API with your internal token. The API checks its database: *Who is this user? Do they own this photo?* and allows or denies the request.

### Scenario B: The Internal Application (Admin Portal)

If Bob (an employee) logs into an internal `Admin Portal` using an internal Auth Server, the flow is identical. However:

1. **First-Party Trust (Skipping Consent):** Because the Admin Portal is a first-party app, the internal Auth Server skips the Consent Screen. It assumes Bob naturally consents.
2. **Elevated Scopes:** It requests scopes like `photos:delete:all`.
3. **The BFF Pattern:** The token is hidden in a secure proxy to protect it from browser hackers.

---

## 9. Deep Dive FAQs: PKCE & Interception

**Q: What is the "Interception Attack"?**
**A:** React and Mobile apps are **"Public Clients."** You cannot put a `client_secret` inside them because anyone can read the source code.
**The Stolen Ticket (The Flashlight App):**

1. Alice clicks "Login".
2. Google sends the Auth Code back via a URL: `photoapp://callback?code=123`.
3. A malicious "Free Flashlight" app on her phone intercepts this URI.
4. The Flashlight app runs to Google and says, *"Here is Alice's code! Give me the Token!"* Because there is no `client_secret`, Google cannot tell the Flashlight app from the real PhotoApp.

**Q: How does PKCE solve this?**
**A:** PKCE lets the app create a **temporary, one-time password**.

> **The Coat Check PIN Analogy:**
> Dropping your coat off gets you a ticket (Auth Code). If a pickpocket steals the ticket, they take your coat.
> **PKCE is adding a PIN.** You tell the attendant a secret PIN. If the pickpocket steals the ticket, the attendant asks for the PIN. The pickpocket doesn't know it, and your coat is safe.

**The Math Trap:** React scrambles the password (hash) and leaves it as a hint with Google. When the Flashlight app steals the code and takes it to Google, Google demands the raw, unscrambled password. The Flashlight app doesn't have it (because it's trapped in React's local memory). The hacker is blocked.

**Q: How does Google know how to unscramble it?**
**A:** React sends an instruction manual in Step 1: `&code_challenge_method=S256`. This tells Google: *"I scrambled this using the universal SHA-256 algorithm."* Google uses the same algorithm to verify the math later.

---

## 10. System Architecture: Scaling PhotoApp

When PhotoApp handles 50,000 requests per second on Black Friday, your .NET API cannot query the database for every single image upload to ask, *"Is this token valid?"* The database would melt down.

To achieve zero downtime, PhotoApp divides its architecture into three main layers.

```mermaid
flowchart TD
    subgraph EdgeLayer [1. The Edge Layer - The Bouncers]
        WAF[Web Application Firewall]
        GW[.NET API Gateway]
        Redis[(Redis In-Memory Cache)]
    end

    subgraph IAMLayer [2. IAM Control Plane - The Passport Office]
        AuthZ[Internal .NET Auth Server]
        DB[(Distributed Database)]
    end

    subgraph BackendLayer [3. Resource Servers - The Vaults]
        Microservices[Photo Microservices]
    end

    Internet((Internet)) --> WAF
    WAF --> GW
    GW <-->|Check Public Keys & Blocklist locally| Redis
    GW -.->|Fetch new Public Keys| AuthZ
    AuthZ <-->|Read/Write Refresh Tokens| DB
    GW ==>|Forward authorized traffic| Microservices

    style EdgeLayer fill:#f9f9f9,stroke:#333,stroke-width:2px
    style IAMLayer fill:#e6f3ff,stroke:#0066cc,stroke-width:2px
    style BackendLayer fill:#e6ffe6,stroke:#009933,stroke-width:2px

```

### Layer 1: The Edge Layer (The Bouncers)

* **WAF:** Blocks DDoS attacks.
* **API Gateway:** Every API request passes through here. It uses its CPU to do the "Wax Seal Math" (validating JWT signatures) without talking to the database.
* **Redis:** Holds Public Keys and a **Revocation Blocklist**.

### Layer 2: The IAM Control Plane (The Passport Office)

* **Auth Server & DB:** Only wakes up when a user logs in or refreshes a session. Never stores Access Tokens, only Refresh Tokens.

### Scenario 1: Black Friday Traffic Spike (The "Stateless" Scale)

50,000 users check their photos at once. The Gateway pulls the Public Key from Redis (0.001 ms) and verifies all 50,000 signatures locally via CPU math. The database experiences **zero** traffic spike.

### Scenario 2: The Stolen Laptop (The Redis Blocklist)

Bob's laptop is stolen. The thief has a valid JWT.

1. IT clicks "Revoke".
2. The Control Plane instantly pushes Bob's JWT ID (`jti`) to the **Redis Blocklist**.
3. The thief clicks "Delete Photo". The API Gateway checks Redis, sees Bob's token on the "Wanted" list, and immediately returns `401 Unauthorized`.

---

## 11. Security Considerations: How Hackers Exploit OAuth 2.0

When an OAuth 2.0 implementation is breached, it is because a developer forgot to lock a specific door.

### Attack 1: CSRF (Cross-Site Request Forgery) on the Callback

**The Bait and Switch:** Hacker Mallory logs into PhotoApp and starts a Google connection. She pauses the flow, copies the callback URL (`photoapp.com/callback?code=MALLORYS_CODE`), and tricks Alice into clicking it. PhotoApp receives the code, thinks Alice sent it, and permanently links Alice's PhotoApp account to Mallory's Google account.
**The Mitigation (`state` parameter):** React generates a random `state` (e.g., `XYZ_123`) and saves it in Alice's browser cookie. It sends this to Google. When the code returns, React ensures the URL's `state` matches the cookie. Because Alice's computer does not have Mallory's cookie, the handshake fails, and the attack is blocked.

### Attack 2: Authorization Code Interception

**The Flashlight App:** Solved entirely by **PKCE** (as detailed in Section 9).

### Attack 3: Token Leakage (The "Postcard" Problem)

If you put an Access Token in a URL (`/dashboard?token=eyJhb...`), it is visible to everyone like a postcard (saved in histories, Wi-Fi logs, analytics).
**The Mitigation:** Never use the deprecated "Implicit Flow". Always use the **Authorization Code Flow**, delivering tokens server-to-server and passing them in the `Authorization: Bearer` HTTP header (a sealed envelope).

### Attack 4: Token Theft via XSS (Cross-Site Scripting)

If you save an Access Token in a browser's `localStorage`, a hacker can post a malicious comment with hidden JavaScript. When Alice views it, the script steals the token from `localStorage`.
**The Mitigation (The BFF Pattern):**

1. The React app **never** sees the tokens.
2. The .NET backend (BFF) handles the OAuth flow and keeps the tokens in its memory.
3. The BFF issues a traditional, encrypted **`HttpOnly` Cookie** to React.
4. Browsers are hardcoded to physically prevent JavaScript from reading an `HttpOnly` cookie. The XSS attack is completely neutralized.

---

## 12. Trade-Offs and Design Decisions

| Decision | Option 1 | Option 2 | Recommendation |
| --- | --- | --- | --- |
| **Flow Selection** | **Auth Code + PKCE** (Human Web/Mobile Apps). Keeps tokens safe from the browser. | **Client Credentials** (Machine-to-Machine). Microservices using vault passwords. | Use **Auth Code + PKCE** for React. Use **Client Credentials** for internal microservices. |
| **Token Type** | **JWT / Value Token** (Stateless, fast math check, scales infinitely). | **Opaque / Reference Token** (Requires DB lookup on every call, instant revocation). | **JWTs** at the API edge for scale, combined with a Redis blocklist for revocation. |
| **Refresh Strategy** | **Standard Expiration** (Refresh token lives 30 days. High risk if stolen). | **Refresh Token Rotation (RTR)** (One-time use tickets). | **RTR**. Every refresh issues a new token. If a thief reuses an old token, the server detects the theft and instantly detonates the entire session. |

---

## 13. Clear Responsibility Table & The Golden Rule

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

> **"What actions can the user perform inside your Photo Application?"** (That is handled by your backend logic).

---

## 14. Integration Examples (Real-World Identity Providers)

While the mechanics are universal, different IdPs use different URLs for their "Passport Office."

**1. CyberArk Identity**

* **Authorize:** `https://{tenant}.id.cyberark.cloud/OAuth2/Authorize/{app-id}`
* **Token:** `https://{tenant}.id.cyberark.cloud/OAuth2/Token/{app-id}`
* **Keys:** `https://{tenant}.id.cyberark.cloud/OAuth2/Keys/{app-id}`

**2. Okta / Auth0**

* **Authorize:** `https://{domain}.okta.com/oauth2/default/v1/authorize`
* **Token:** `https://{domain}.okta.com/oauth2/default/v1/token`

**3. Microsoft Entra ID (Azure AD)**

* **Authorize:** `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize`
* **Token:** `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token`

**4. AWS Cognito**

* **Authorize:** `https://{prefix}.auth.{region}.amazoncognito.com/oauth2/authorize`
* **Token:** `https://{prefix}.auth.{region}.amazoncognito.com/oauth2/token`

**5. Keycloak**

* **Authorize:** `https://{domain}/realms/{realm}/protocol/openid-connect/auth`
* **Token:** `https://{domain}/realms/{realm}/protocol/openid-connect/token`

---

## 15. Reference Diagrams

### Diagram 1: Client Credentials Flow (M2M)

```mermaid
sequenceDiagram
    autonumber
    participant S1 as Thumbnail Microservice (Client)
    participant AS as .NET Auth Server
    participant S2 as Storage Microservice (API)

    S1->>AS: POST /token (client_id, secret, grant_type=client_credentials)
    AS->>AS: Authenticate Service Account
    AS-->>S1: Return Access Token
    S1->>S2: POST /v1/storage/upload + Bearer Token
    S2->>S2: Validate Token & Scopes
    S2-->>S1: 200 OK

```

### Diagram 2: Token Issuance and Refresh (Rotation)

```mermaid
sequenceDiagram
    autonumber
    participant C as PhotoApp Backend
    participant AS as Auth Server
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

## 16. Frequently Asked Questions (FAQ)

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
