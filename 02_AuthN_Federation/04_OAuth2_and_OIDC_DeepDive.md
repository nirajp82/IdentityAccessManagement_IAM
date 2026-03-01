# The Bible: OAuth 2.0 & OpenID Connect (OIDC) at MoneyGuard

## 1. The Origin Problem (Why do we need this?)

To understand OAuth 2.0 and OIDC, you must understand the two distinct problems they were built to solve: **The External Password Problem** and **The Internal Zero-Trust Problem**.

### The External Problem (The Origin of OAuth 2.0)

Imagine it is 2010. A MoneyGuard customer wants to use a third-party budgeting app called **BudgetPro**. BudgetPro says, *"Give us your MoneyGuard username and password, and our servers will log in as you to read your transactions."*

* **The Danger:** By giving BudgetPro their password, the user gives them full control. BudgetPro could maliciously wire money out of the account. Furthermore, if BudgetPro is hacked, the hacker gets the plaintext passwords to thousands of MoneyGuard accounts.
* **The Solution:** **OAuth 2.0 (Delegated Authorization).** Instead of asking for a password, BudgetPro redirects the user to MoneyGuard. MoneyGuard asks the user, *"Do you want to allow BudgetPro to READ your transactions?"* MoneyGuard then issues BudgetPro a highly restricted **Access Token** (like a hotel keycard that only opens one specific door).

### The Internal Problem (The Origin of OIDC)

Fast forward to the era of Zero-Trust. MoneyGuard realizes that if OAuth 2.0 is great for protecting APIs from *external* apps, it should be used for *internal* apps too. We don't want the internal `Teller Portal` to blindly trust the internal `Wire Transfer Microservice`.

* **The Danger:** Developers started using OAuth 2.0 Access Tokens to "log users in" to internal apps. This was a fatal flaw. An Access Token is just a keycard; it proves you have *permission*, but it does not contain your *identity*.
* **The Solution:** **OpenID Connect (OIDC).** Built directly on top of OAuth 2.0, OIDC introduces the **ID Token** (an ID Badge). Now, when an internal MoneyGuard app asks for a token, it gets two: an Access Token (to talk to the API) and an ID Token (to know exactly who the user is).

---

## 2. Core Concepts: The Two Tokens

When a client application completes the OIDC flow, it receives two JSON Web Tokens (JWTs). It is critical to understand their distinct roles.

| Feature | ID Token (OIDC) | Access Token (OAuth 2.0) |
| --- | --- | --- |
| **Analogy** | A Driver's License (Identity) | A Hotel Keycard (Permission) |
| **Intended Audience** | The Client Application (e.g., Teller Web App) | The Resource Server (e.g., API Gateway) |
| **What it contains** | User claims: `name`, `email`, `auth_time` | Authorization claims: `scopes`, `roles` |
| **How it is used** | The Client app decodes it to render the UI ("Welcome, Alice!"). | The Client app attaches it to the HTTP `Authorization: Bearer` header. |

---

## 3. The Master Flow: Authorization Code with PKCE

Whether the application is internal or external, the industry standard is the **Authorization Code Flow with PKCE** (Proof Key for Code Exchange).

### The Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    
    box Client Application
    participant App as Client Backend<br>(teller.moneyguard.com)
    participant Browser as User's Browser
    end
    
    box Central IAM System
    participant IDP as idp.moneyguard.com<br>(AuthN)
    participant Token as auth.moneyguard.com<br>(AuthZ)
    end

    Note over App, Browser: 1. Initiation
    Browser->>App: User clicks "Login"
    App->>Browser: HTTP 302 Redirect to IdP
    
    Note over Browser, IDP: 2. Authentication
    Browser->>IDP: GET /authorize?scope=openid profile wires:write
    IDP-->>Browser: Shows Login Screen (MFA)
    Browser->>IDP: User submits credentials
    
    Note over Browser, App: 3. The Callback (The "Code")
    IDP->>Browser: HTTP 302 Redirect to App Callback with ?code=xyz
    Browser->>App: GET /callback?code=xyz
    
    Note over App, Token: 4. Secure Token Exchange (Backend)
    App->>Token: POST /token (code, client_id, client_secret)
    Token-->>App: Returns JSON: {id_token, access_token, refresh_token}
    
    Note over App: 5. Validation & Session
    App->>App: Validate ID Token signature
    App->>Browser: Set Secure HttpOnly Cookie

```

### The Deep-Dive HTTP Payloads

**Step 2: The Auth Request (Browser -> IdP)**
The `openid` scope tells the server to generate an ID Token. The `wires:write` scope requests API permissions.

```http
GET /authorize?
  response_type=code
  &client_id=app_teller_prod_01
  &redirect_uri=https://teller.moneyguard.com/callback
  &scope=openid profile email wires:write
  &state=abc123xyz
  &nonce=random_nonce_987 HTTP/1.1
Host: idp.moneyguard.com

```

**Step 4: The Token Exchange (Client Backend -> IAM Control Plane)**
The Client App proves its identity using Basic Auth (`client_id` and `client_secret`) and trades the temporary code for tokens.

```http
POST /oauth2/token HTTP/1.1
Host: auth.moneyguard.com
Authorization: Basic YXBwX3RlbGxlcl9wcm9kXzAxOnN1cGVyX2h0...
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=xyz123&redirect_uri=https://teller.moneyguard.com/callback

```

**Step 4 (Response): The Tokens Arrive**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...", 
  "token_type": "Bearer",
  "expires_in": 900, 
  "refresh_token": "def456uvw",
  "scope": "wires:write",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtp..." 
}

```

---

## 4. Real-World Use Cases at MoneyGuard

Why and when do we use this? Here is the breakdown of Internal vs. External usage.

### Use Case A: Internal First-Party App (The Teller Portal)

* **Scenario:** A bank employee logs into the internal `teller.moneyguard.com` dashboard to view a customer profile and initiate a wire.
* **Why OIDC is used:** MoneyGuard operates a Zero-Trust network. Just because the Teller App is hosted on the corporate intranet doesn't mean the API Gateway trusts it.
* **How it works:** The Teller App uses the exact OIDC flow described above. It receives the **ID Token** to know the employee's name and email for the frontend UI. It receives the **Access Token** containing the `roles: ["teller"]` claim. When the employee clicks "Submit Wire", the Teller App attaches the Access Token to the API request, proving to the API Gateway (PEP) that a legally authenticated employee is driving the request.

### Use Case B: External Third-Party App (Plaid / Fintech)

* **Scenario:** A MoneyGuard customer wants to link their checking account to an external budgeting app like Plaid.
* **Why OAuth 2.0 is used:** We must allow Plaid to read the user's balance *without* giving Plaid the user's banking password.
* **How it works:** This is pure **OAuth 2.0 Delegated Authorization**. Plaid redirects the user to MoneyGuard. MoneyGuard asks the user, *"Do you want to grant Plaid access to read your account balance?"* * **The Difference:** Plaid asks for `scope=accounts:read`. Plaid **does not** ask for the `openid` scope. Therefore, the IAM Control Plane gives Plaid an Access Token, but it *does not* give Plaid an ID Token. Plaid does not need to know the user's SSN or full profile; it only needs the keycard to the transaction API.

---

## 5. Implementation in .NET Applications

How does this actually look in code? In the MoneyGuard ecosystem, we build APIs and Web Apps using **ASP.NET Core**.

### A. The Client App (`teller.moneyguard.com`)

The Web App must implement OIDC to get the tokens, and it must use the **BFF (Backend-For-Frontend)** pattern to store them securely in an HttpOnly cookie (never in `localStorage`).

*In `Program.cs` of the Teller Web App:*

```csharp
// Add Authentication Services
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie(options => 
{
    // The BFF Pattern: Store the tokens securely in an encrypted, HttpOnly cookie
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
})
.AddOpenIdConnect(options =>
{
    options.Authority = "https://auth.moneyguard.com"; // IAM Control Plane
    options.ClientId = "app_teller_prod_01";
    options.ClientSecret = builder.Configuration["Oidc:ClientSecret"];
    
    options.ResponseType = "code"; // Use Authorization Code Flow
    options.UsePkce = true;        // Enable PKCE for security
    
    // Request both Identity and API scopes
    options.Scope.Add("openid");
    options.Scope.Add("profile");
    options.Scope.Add("wires:write");
    
    // Save tokens in the cookie so the backend can use them for API calls later
    options.SaveTokens = true; 
});

```

### B. The API Gateway / Microservice (`api.moneyguard.com`)

The API Gateway does not care about OIDC or logging users in. It only cares about validating the incoming **Access Token**.

*In `Program.cs` of the API Gateway:*

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
.AddJwtBearer(options =>
{
    // Where to fetch the JWKS Public Keys to verify the cryptographic signature
    options.Authority = "https://auth.moneyguard.com";
    
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = "https://auth.moneyguard.com",
        
        ValidateAudience = true,
        ValidAudience = "api.moneyguard.com", // Ensure token is meant for this API
        
        ValidateLifetime = true // Reject expired tokens
    };
});

```

---

## 6. Directory Services & JML (Joiner, Mover, Leaver)

How does the IAM Control Plane actually know the employee's title to put it into the tokens? It integrates with the HR and Directory workflow.

1. **The Joiner (HR Source of Truth):** HR enters a new employee, Bob, into Workday as a "Wealth Manager".
2. **SCIM Provisioning:** Workday uses the SCIM (System for Cross-domain Identity Management) API to push Bob's profile into MoneyGuard's Active Directory (the Identity Store).
3. **Just-In-Time Claims Mapping:** Bob logs into the Teller App. In Step 4 (Token Exchange), the IAM Control Plane looks up Bob in Active Directory. It reads his group membership. The IAM engine runs a rule: *"If AD Group = WealthManager, inject `roles: ["wealth_manager"]` into the Access Token."*
4. **The Leaver (Termination):** Bob quits. HR disables his Workday profile. Active Directory immediately disables his account. He can no longer authenticate at the IdP. To kill his *active* 15-minute Access Token, MoneyGuard uses Token Revocation (detailed below).

---

## 7. Token Mechanics: Refresh, Caching, & Revocation

### The Refresh Token Flow

Access Tokens must be short-lived (e.g., 15 minutes) to minimize damage if stolen. However, we cannot force an employee to perform MFA every 15 minutes.

* **The Solution:** In Step 4, the IAM server also issues a `refresh_token` (valid for 12 hours).
* **The Execution:** At minute 14, the Teller App Backend silently sends the `refresh_token` to `auth.moneyguard.com/oauth2/token`. The IAM server checks if Bob's account is still active in Active Directory. If yes, it returns a brand new 15-minute Access Token without bothering Bob.

### JWKS Caching (Gateway Performance)

The API Gateway uses the `Microsoft.AspNetCore.Authentication.JwtBearer` library. This library automatically fetches the Public Keys from `https://auth.moneyguard.com/.well-known/jwks.json` and caches them in RAM. When Bob makes an API call, the Gateway verifies the token's cryptographic signature locally in microseconds. It does not make a network call to the IAM server.

### Token Revocation (The Redis Blocklist)

If Bob's laptop is stolen while he has an active 15-minute Access Token, the IAM Admin clicks "Revoke Session".
Because the API Gateway validates tokens locally (caching), it doesn't know the session was revoked.

* **The Zero-Trust Fix:** The IAM Control Plane publishes the token's unique ID (`jti` claim) to a distributed **Redis Blocklist**.
* The API Gateway checks this Redis cache on every single request. If the `jti` is found in Redis, the Gateway instantly returns a `401 Unauthorized`, neutralizing the stolen token mid-flight.

---

## 8. Frequently Asked Questions (FAQ)

**Q: I am building a React Single Page Application (SPA). Should I use the Implicit Flow and store the Access Token in `localStorage`?**
**A:** **Absolutely Not.** The Implicit Flow is officially deprecated. Storing JWTs in `localStorage` makes them highly vulnerable to Cross-Site Scripting (XSS) attacks. You must use the BFF (Backend-For-Frontend) pattern, where a lightweight .NET backend handles the tokens and issues a secure `HttpOnly` cookie to the React app.

**Q: Can I put authorization rules directly in the ID Token?**
**A:** No. The ID Token is strictly for identity (who you are). The Access Token is for authorization (what you can do). Mixing them violates the protocol design and can lead to confused-deputy attacks at the API layer.

**Q: Why do we need the `state` and `nonce` parameters in the Auth Request?**
**A:** The `state` parameter prevents Cross-Site Request Forgery (CSRF) by ensuring the callback response matches the exact browser session that started the login. The `nonce` parameter is injected into the ID Token by the IdP; the Client App checks it to prevent Token Replay attacks.
