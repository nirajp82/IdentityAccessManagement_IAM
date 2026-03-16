# MoneyGuard: Enterprise Zero-Trust IAM Architecture Master Document

## Overview

This document outlines the highly scalable Zero-Trust Identity and Access Management (IAM) architecture for **MoneyGuard**, a global bank. To secure highly sensitive financial data, this design strictly separates **Authentication (AuthN)**, **Authorization (AuthZ) management**, and **Policy Enforcement**. This ensures that every single request is explicitly verified before it reaches our core banking systems.

---

### Architecture Diagram (Fully Detailed Flow)

```mermaid
sequenceDiagram
    autonumber

    %% Define Architectural Groupings (Boxes)
    box Client Environment
    participant US as 👤 User Browser & Client App<br>(teller.moneyguard.com)
    end

    box Identity Provider (IdP)
    participant IDP as idp.moneyguard.com<br>(MFA, OIDC, Federation)
    end

    box IAM Control Plane
    participant IAM as auth.moneyguard.com<br>(Identity Store, Token Svc)
    end

    box Enforcement Layer
    participant PEP as api.moneyguard.com<br>API Gateway (PEP)
    participant PDP as Open Policy Agent<br>(PDP)
    end

    box Protected Resources
    participant RES as Core Microservices<br>(/v1/wire-transfers)
    end

    %% Step-by-Step Flow
    Note over US, IDP: Phase 1: Authentication
    US->>IDP: 1. User accesses app. Browser redirects to idp.moneyguard.com/authorize
    IDP-->>US: 2. User logs in & passes MFA. IdP redirects back with ?code=xyz
    
    Note over US, IAM: Phase 2: Secure Token Exchange
    US->>IAM: 3. App Backend calls POST /oauth2/token (using code, client_id, client_secret)
    
    rect rgb(60, 60, 60)
    IAM->>IDP: 3b. Backend Federation Check: IAM verifies 'code=xyz' directly with IdP
    IDP-->>IAM: Valid. Code belongs to Alice.
    end
    
    IAM-->>US: 4. IAM generates & returns signed JWT Access Token to Client App
    
    Note over US, RES: Phase 3: Zero-Trust API Call & Enforcement
    US->>PEP: 5. App makes API Call to POST /v1/wires with JWT (Bearer Token)
    
    rect rgb(60, 60, 60)
    PEP->>PDP: 5b. PEP verifies JWKS signature. Hands to PDP to evaluate ABAC rules.
    PDP-->>PEP: ALLOW Decision
    end
    
    PEP->>RES: 6. Gateway forwards authorized request
    RES-->>US: 200 OK (Wire Processed Successfully)

```

## 1. Architecture Modules in Detail

### Module 1: [ Users / Services ]

The initiators of requests. In the MoneyGuard ecosystem, these fall into three categories:

* **Customers:** Accessing the MoneyGuard Mobile App or consumer web portal.
* **Employees:** Tellers, Wealth Managers, or Admins accessing internal dashboards (e.g., `teller.moneyguard.com`).
* **Machine Identities:** Internal microservices (e.g., the Fraud Detection service) needing to communicate with the Core Ledger, or authorized third-party fintech apps.

### Module 2: [ Identity Provider (IdP) ]

The front door. The IdP handles **Authentication (AuthN)**—proving *who* is logging in. It does not care about what they are allowed to do, only that their credentials are valid.

* **OIDC / OAuth2 / SAML:** MoneyGuard uses OIDC for modern web/mobile logins and SAML for legacy banking software.
* **MFA (Multi-Factor Authentication):** For a bank, this is mandatory. The IdP enforces a second factor, such as an SMS code for consumers or a hardware YubiKey for MoneyGuard database administrators.
* **Federation:** Allows large corporate clients to securely access MoneyGuard's portal using their existing client's company logins (e.g., Azure AD). By establishing this trust relationship, MoneyGuard outsources user verification and password management directly to the client's own IT system. This means MoneyGuard won't manage client usernames and passwords; they are managed entirely by the client.
  
### Module 3: [ IAM Control Plane ]

The brain. Once the IdP authenticates the user, the IAM Control Plane takes over to handle **Authorization (AuthZ)**—determining what the user is allowed to do and giving them the cryptographic "ticket" (token) to do it.

* **Identity Store:** The central database linking the authenticated user to their MoneyGuard profile (e.g., associating Alice with her "Wealth Manager" profile and department attributes).
* **Role & Policy Engine (RBAC / ABAC):**
* *RBAC (Role-Based):* Grants access based on static roles (e.g., Alice is a `WealthManager`).
* *ABAC (Attribute-Based):* Grants access based on dynamic context (e.g., Alice can only approve transfers *if* she is accessing from a registered MoneyGuard IP address *and* it is during business hours).


* **Token Service:** Generates secure JSON Web Tokens (JWT) that encapsulate the user's identity, scoped permissions, and an expiration time.
* **Lifecycle (SCIM):** SCIM (System for Cross-domain Identity Management) automates provisioning. When HR fires an employee in Workday, SCIM instantly communicates with `iam.moneyguard.com` to revoke their roles, ensuring they cannot generate new tokens.

### Module 4: [ Enforcement Layer ]

The muscle. This layer physically sits in front of MoneyGuard's backend network. It assumes every network request is hostile until proven otherwise.

* **API Gateway / Sidecar (PEP):** Hosted at `api.moneyguard.com`. This is the Policy Enforcement Point. When a request comes in, the gateway extracts the JWT from the `Authorization: Bearer <token>` HTTP header. For internal service-to-service traffic, this is handled by a proxy sidecar (like Envoy) running alongside the microservice.
* **Policy Decision Point (PDP):** The gateway pauses the request and hands the token to the PDP (a lightning-fast, local engine like Open Policy Agent). The PDP checks its rules and definitively answers: *"Is this user allowed to do X on resource Y right now?"*

### Module 5: [ Core Banking Microservices ]

The ultimate destination. These are internal APIs (e.g., `/v1/wire-transfers`). Because they sit behind the Enforcement Layer, the developers building these services do not need to write complex authentication code. They trust the gateway.

---

## 2. Deep Dive: Redirection and Payload Flows (Human Users)

A common point of confusion is exactly **who redirects to whom**. In a standard, secure OAuth2/OIDC flow (Authorization Code Flow), the IdP does *not* talk directly to the IAM Control Plane. The **Client App (User's Browser)** acts as the middleman.

**Step A: The Initial Request (Client -> IdP)**
Alice opens `teller.moneyguard.com`. The app sees she isn't logged in and redirects her browser to the IdP.

* **HTTP Request:**

```http
GET /authorize?client_id=teller_app&response_type=code&redirect_uri=https://teller.moneyguard.com/callback&scope=openid profile
Host: idp.moneyguard.com

```

**Step B: The Authentication & Redirect (IdP -> Client)**
Alice enters her password and passes MFA. The IdP verifies her. The IdP then redirects Alice's browser *back to the client app*, giving it a temporary, single-use "Authorization Code".

* **HTTP Response:**

```http
HTTP/1.1 302 Found
Location: https://teller.moneyguard.com/callback?code=SplxlOBeZQQYbYS6WxSbIA

```

**Step C: The Token Exchange (Client Backend -> IAM Control Plane)**
Now the Client App has a `code`. It needs a real token. The Client App's backend makes a secure, behind-the-scenes call to the IAM Control Plane to trade the code for a JWT.

* **HTTP Request:**

```http
POST /oauth2/token
Host: auth.moneyguard.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https://teller.moneyguard.com/callback&client_id=teller_app&client_secret=super_secret_key

```

**Step D: The Token Delivery (IAM Control Plane -> Client Backend)**
The IAM Control Plane verifies the code, checks Alice's RBAC/ABAC rules in the Identity Store, and generates the JWT Access Token.

* **HTTP Response:**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "id_token": "eyJhbGciOiJIUzI1NiIs..."
}

```

The Client App now uses this `access_token` to authenticate API calls to the Enforcement Layer.

---

## 3. Secure Frontend Token Storage (Preventing XSS)

Once the Single Page Application (like a React or Angular app at `teller.moneyguard.com`) receives the `access_token` in Step D, it must be stored securely.

### The Vulnerability: `localStorage`

Many tutorials show storing the JWT in `localStorage` or `sessionStorage`. **For a bank like MoneyGuard, this is strictly prohibited.** If the frontend application has a Cross-Site Scripting (XSS) vulnerability, that JavaScript can easily run `localStorage.getItem('access_token')` and steal the token, leading to a complete account takeover.

### The Solution: Backend-For-Frontend (BFF) & HttpOnly Cookies

MoneyGuard utilizes the **BFF Pattern**:

1. The frontend React app does not handle the OAuth flow directly. Instead, it communicates with a lightweight backend proxy (the BFF).
2. The BFF handles Step C (Token Exchange) with the IAM Control Plane.
3. When the BFF receives the JWT, it does *not* send the raw token to the React app. Instead, it wraps the token in a tightly locked-down HTTP Cookie.

The HTTP Cookie must have the following flags set by the server:

* `HttpOnly`: This is the most critical flag. It tells the browser, *"Do not let JavaScript read this cookie under any circumstances."*
* `Secure`: Ensures the cookie is only sent over HTTPS.
* `SameSite=Strict`: Prevents the browser from sending the cookie in cross-site requests, mitigating Cross-Site Request Forgery (CSRF) attacks.

When the React app wants to call `api.moneyguard.com/v1/wires`, it makes the request, and the browser *automatically* attaches the secure cookie containing the JWT.

---

## 4. Deep Dive: PEP vs. PDP Explained

In enterprise security, the Enforcement Layer is broken into two components defined by the XACML (eXtensible Access Control Markup Language) framework. They work as a pair.

* **PEP (Policy Enforcement Point):** This is your **API Gateway** at `api.moneyguard.com` (or a sidecar proxy like Envoy). It is the "muscle." Its only job is to intercept incoming traffic, pause the connection, ask for permission, and then strictly enforce whatever answer it gets. *It does not know the security rules.*
* **PDP (Policy Decision Point):** This is the **Rule Engine** (like Open Policy Agent). It sits deep inside your secure internal network, right next to the PEP to ensure < 1ms latency. It is the "brain." Its only job is to evaluate complex rules against the user's token and say **ALLOW** or **DENY**. *It does not handle network traffic or talk to the user.*

### The VIP Portfolio Example

**Analogy:** Think of the PEP as the security guard at a building, and the PDP as the master guest list. The security guard stops you and asks for your ID. The guard then checks with the guest list to see if you are actually allowed into the specific room you requested.

1. **The PEP Intercepts:** Alice, a Wealth Manager, tries to view a VIP client portfolio by sending a `GET` request to `/v1/vip-portfolios/777`. The PEP grabs her request, rips out the JWT Token, and pauses the connection.
2. **The PEP Asks:** The PEP sends a rapid, internal message to the PDP: *"Hey PDP, I have a user here. Here is her JWT token. She wants to perform a `GET` on `/v1/vip-portfolios/777`. Is this allowed?"*
3. **The PDP Calculates:** The PDP evaluates the "Master Rulebook". It checks: Does she have the `WealthManager` role? Yes. Is she on a corporate IP? Yes. Is it between 9 AM and 5 PM? Yes. The PDP responds to the PEP with `{"allow": true}`.
4. **The PEP Enforces:** Because the PDP said yes, the PEP opens the door and forwards Alice's request to the database. If she had tried at 11:00 PM on a Saturday, the PDP would have said `false`, and the PEP would have immediately killed the connection with a `403 Forbidden` error.

---

### 5. Deep Dive: Cryptographic Trust (How Systems Validate Tokens)

#### A. How the IAM Control Plane Validates the Source (Client Authentication)

When the Client App (the backend server for `teller.moneyguard.com`) asks for a token in Step C, it doesn't just ask nicely. It must mathematically prove its identity to `auth.moneyguard.com`.

* **Client Registration (The Setup):** Long before Alice ever logs in, the developers of the Teller App registered their application with the IAM Control Plane. The IAM system generated a unique `client_id` (like a username, e.g., `app_teller_prod_01`) and a highly secure `client_secret` (like a 64-character password known *only* to the Teller App's backend server).
* **The Secure POST Request (The Check):** When exchanging the code, the Teller App backend makes a secure, server-to-server POST request to the IAM Control Plane. It sends its ID and Secret using an `Authorization: Basic` header (which is a base64-encoded string of the ID and Secret).

**The Exact HTTP Request:**

```http
POST /oauth2/token HTTP/1.1
Host: auth.moneyguard.com
# This header proves the request is coming from the legitimate Teller App backend
Authorization: Basic YXBwX3RlbGxlcl9wcm9kXzAxOnN1cGVyX3NlY3JldF9wYXNzd29yZF8xMjM=
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https://teller.moneyguard.com/callback

```

* **The Result:** The IAM Control Plane checks its database. Because the `client_secret` perfectly matches the `client_id` for the Teller App, `auth.moneyguard.com` knows with 100% certainty that the request came from the legitimate app server, not a hacker's laptop.

#### B. How the IAM Control Plane Validates the Code (IdP Trust)

The IAM Control Plane (`auth.moneyguard.com`) just received the code `SplxlOBeZQQYbYS6WxSbIA` from the Teller App. The IAM Control Plane did *not* generate this code—the IdP (`idp.moneyguard.com`) did.

To verify it, MoneyGuard uses a **Backend Federation Check** (Server-to-Server Trust).

1. **The Code arrives at the IAM Control Plane:** `auth.moneyguard.com` pauses the token generation process. It looks at the code and says, *"I need to verify this with the IdP."*
2. **The Hidden Backend Call:** The IAM Control Plane makes a highly secure, private API call over the internal bank network directly to the Identity Provider's backend API.
```http
POST /api/v1/verify-code HTTP/1.1
Host: internal-idp.moneyguard.local
Authorization: Bearer <iam_internal_system_token>

{
  "code": "SplxlOBeZQQYbYS6WxSbIA"
}

```


3. **The IdP Responds:** The IdP (`idp.moneyguard.com`) checks its own short-term database. It replies to the IAM Control Plane:
```json
{
  "valid": true,
  "authenticated_user": "alice.smith@moneyguard.com",
  "mfa_passed": true,
  "time_issued": "2026-02-28T17:00:00Z"
}

```


4. **The IAM Control Plane takes over:** Now that `auth.moneyguard.com` has cryptographic proof from the IdP that the code is real and belongs to Alice, it throws the code away. It looks up Alice in its own Identity Store, pulls her `WealthManager` roles, generates the final JWT, and sends it back to the Teller App.

*(Note: In some cloud-native architectures, instead of making an API call, the IdP and the IAM Control Plane might share a highly secure, internal Redis database cache. The IdP writes the code to Redis, and the IAM Control Plane reads it directly from Redis to verify it instantly).*


### C. How the PDP Validates Context

The **PEP and PDP work as a relay team**.

1. **The PEP does the math:** The API Gateway verifies the cryptographic signature, expiration time, and issuer. If any fail, it drops the request.
2. **The PDP evaluates business logic:** Once the PEP knows the token is mathematically authentic, it hands the decoded JSON payload to the PDP. The PDP trusts the token is real and focuses purely on evaluating the business rules (RBAC/ABAC).

### Cryptographic Sequence Diagram

```mermaid
sequenceDiagram
    participant App as Client App
    participant IAM as auth.moneyguard.com<br>(IAM Control Plane)
    participant PEP as api.moneyguard.com<br>(API Gateway)
    participant PDP as Policy Decision Point

    Note over IAM,PEP: Background Process (Once per hour)
    PEP->>IAM: GET /.well-known/jwks.json
    IAM-->>PEP: Returns Public Keys (Cached in memory)

    Note over App,IAM: 1. Token Generation Phase
    App->>IAM: POST /token (client_id, client_secret)
    IAM->>IAM: Validates Client Credentials
    IAM->>IAM: Signs JWT with PRIVATE KEY
    IAM-->>App: Returns JWT

    Note over App,PEP: 2. API Request Phase
    App->>PEP: GET /v1/wires + Bearer JWT
    
    Note over PEP,PDP: 3. Validation Phase
    PEP->>PEP: Math Check: Verify JWT Signature using Cached PUBLIC KEY
    PEP->>PEP: Rule Check: Is token expired? Is Issuer correct?
    PEP->>PDP: Send decoded JSON payload & HTTP request details
    PDP->>PDP: Business Check: Evaluate Rego ABAC Policies
    PDP-->>PEP: {"allow": true}
    
    Note over PEP: 4. Enforcement
    PEP->>Backend: Forward Request to Microservice

```

---

## 6. Deep Dive: How the IAM Control Plane Knows User Roles & Attributes

A common gap in understanding IAM is assuming the IAM Control Plane "magically" knows who is a Wealth Manager, or what department they work in. In a massive bank like MoneyGuard, this data originates entirely outside the IAM system.

It works via a pipeline of **Provisioning (SCIM)** and **Claims Mapping**.

### Step 1: The Source of Truth (HR Systems)

The absolute master record for any employee is the Human Resources system (e.g., Workday or SAP SuccessFactors). When Alice is hired, HR creates her profile:

* `Job Title:` Senior Wealth Manager
* `Department:` Private Banking
* `Cost Center:` 8472

### Step 2: Provisioning via SCIM

The HR system is securely linked to the IAM Control Plane (`iam.moneyguard.com`) via a protocol called **SCIM (System for Cross-domain Identity Management)**.
Whenever HR updates a profile, Workday instantly sends a JSON payload to the IAM Control Plane's SCIM API.

*Workday's POST request to MoneyGuard IAM:*

```json
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "userName": "alice@moneyguard.com",
  "active": true,
  "title": "Senior Wealth Manager",
  "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User": {
    "department": "Private Banking",
    "manager": {"value": "26118915-6090-4610-87e4-49d8ca9f808d"}
  }
}

```

### Step 3: The Identity Store

The IAM Control Plane receives this SCIM payload and saves it into its internal database, called the **Identity Store** (often backed by Active Directory, LDAP, or a cloud directory). The IAM system now physically holds Alice's attributes.

### Step 4: Just-In-Time Claims Mapping

Fast forward to Alice logging into the Teller App. In Step C of the OAuth flow, the app requests a token from the `Token Service`. Here is what happens in the millisecond before the JWT is generated:

1. The Token Service looks up `alice@moneyguard.com` in the Identity Store.
2. It pulls her raw attributes (`title: "Senior Wealth Manager"`, `department: "Private Banking"`).
3. The Token Service uses a **Claims Mapper** (a set of rules defined by IAM admins) to translate HR titles into application roles.
* *Mapper Rule:* `IF Title CONTAINS "Wealth Manager" THEN Add Role "wealth_manager"`
* *Mapper Rule:* `IF Department == "Private Banking" THEN Add Attribute "clearance_level: 3"`


4. The Token Service injects these mapped results into the final JWT payload (called "claims"), signs the token cryptographically, and hands it back to the Client App.

Now, when Alice makes an API call, the PDP reads the `wealth_manager` claim inside her token, entirely unaware of the HR systems that originally facilitated it.

---

## 7. Policy Enforcement: Open Policy Agent (Rego)

When the API Gateway (PEP) intercepts a request to `api.moneyguard.com/v1/wires`, it sends the parsed JWT to the Open Policy Agent (PDP). Here is how MoneyGuard writes an ABAC policy in **Rego** to evaluate Alice's wire transfer request:

```rego
package moneyguard.api.wires

import future.keywords.in
import future.keywords.if

# 1. SECURE BY DEFAULT: Deny all requests unless explicitly allowed
default allow := false

# 2. THE ALLOW RULE: Evaluates to true ONLY IF all statements inside are true
allow if {
    # Match the exact API route and HTTP method
    input.request.method == "POST"
    input.request.path == "/v1/wires"

    # RBAC Check: Does the user's JWT contain the required role?
    "wealth_manager" in input.token.payload.roles

    # ABAC Check 1: Is the user originating from a trusted Corporate IP network?
    is_corporate_ip(input.request.source_ip)

    # ABAC Check 2: Dynamic payload validation. 
    input.request.body.amount <= 100000
}

# 3. HELPER FUNCTION: Define what constitutes a "Corporate IP"
is_corporate_ip(ip) if {
    corporate_ips := ["10.50.0.0/16", "192.168.100.0/24"]
    ip in corporate_ips
}

```

---

## 8. .NET Implementation Example (Enforcement Layer)

Here is how MoneyGuard's Enforcement Layer (built on **ASP.NET Core**) intercepts the incoming API request, extracts the JWT, and calls the Open Policy Agent (PDP) before allowing the request to hit the core banking controllers.

```csharp
using System.Net.Http;
using System.Text;
using System.Text.Json;
using Microsoft.AspNetCore.Http;

public class OpaMiddleware
{
    private readonly RequestDelegate _next;
    private readonly HttpClient _httpClient;
    
    // The local URL where Open Policy Agent is running (usually a sidecar on localhost)
    private const string OpaUrl = "http://localhost:8181/v1/data/moneyguard/api/wires/allow";

    public OpaMiddleware(RequestDelegate next, HttpClient httpClient)
    {
        _next = next;
        _httpClient = httpClient;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 1. Extract the Bearer Token from the request
        var authHeader = context.Request.Headers["Authorization"].ToString();
        var token = authHeader.StartsWith("Bearer ") ? authHeader.Substring(7) : null;

        // 2. Build the Input payload for OPA
        var opaInput = new
        {
            input = new
            {
                request = new
                {
                    method = context.Request.Method,
                    path = context.Request.Path.Value,
                    source_ip = context.Connection.RemoteIpAddress?.ToString()
                },
                token = token // OPA will decode this JWT on its end
            }
        };

        // 3. Ask OPA for the Policy Decision (The PDP Check)
        var content = new StringContent(JsonSerializer.Serialize(opaInput), Encoding.UTF8, "application/json");
        var response = await _httpClient.PostAsync(OpaUrl, content);
        
        if (response.IsSuccessStatusCode)
        {
            var responseStream = await response.Content.ReadAsStreamAsync();
            var opaResult = await JsonSerializer.DeserializeAsync<OpaResponse>(responseStream);

            // 4. Enforce the Decision
            if (opaResult != null && opaResult.Result)
            {
                // ALLOWED: Pass the request down the pipeline
                await _next(context);
                return;
            }
        }

        // DENIED: Short-circuit the pipeline and return a 403 Forbidden
        context.Response.StatusCode = StatusCodes.Status403Forbidden;
        await context.Response.WriteAsync("Access Denied by Policy Decision Point (PDP).");
    }
}

public class OpaResponse
{
    [System.Text.Json.Serialization.JsonPropertyName("result")]
    public bool Result { get; set; }
}

```

---

## 9. End-to-End Example Flow Summary (Human User)

1. **Initiation:** Alice opens `wealth.moneyguard.com`.
2. **Authentication (IdP):** She is redirected to `idp.moneyguard.com`, logs in, and passes MFA. The IdP redirects her browser back to the app with a temporary Auth Code (`?code=xyz`).
3. **Authorization (IAM):** The web app backend sends the Auth Code to `auth.moneyguard.com` along with its `client_id` and `client_secret`. The IAM Control Plane performs a **backend federation check** directly with the IdP to ensure the code is legitimate. Once verified, it looks up Alice in the Identity Store, assigns her the `wealth_manager` role, and returns a signed JWT Access Token.
4. **The 1st API Call (Wire Transfer):** Alice clicks "Send Wire". Her browser sends a POST request to `api.moneyguard.com/v1/wires` with her secure HttpOnly cookie containing the JWT.
5. **Enforcement (PEP/PDP):** The .NET API Gateway middleware intercepts the request. It mathematically verifies the JWT signature against its locally cached public JWKS keys. It then hands the decoded token to the OPA PDP to evaluate the Rego policy. The PDP checks the token claims, the IP, and the amount, and responds: `{"result": true}`.
6. **Execution:** The Gateway forwards the request to the internal Wire Transfer Microservice, which safely processes the transfer.
7. **The 2nd API Call (Account Balance):** A few minutes later, Alice navigates to a new page to view client funds. Her browser sends a GET request to `api.moneyguard.com/v1/account-balances` using the *same* active JWT.
8. **Stateless Enforcement (Bypassing IdP/IAM):** The API Gateway intercepts this second request. It does *not* contact the IAM Control Plane or the IdP. It instantly verifies the JWT signature and expiration locally. It asks the PDP to evaluate the specific rules for the `/v1/account-balances` endpoint. The PDP allows it, and the Gateway instantly routes the request to the Account Microservice.
----

## 10. Deep Dive: Machine-to-Machine (M2M) Zero-Trust Flow

What happens when there is no human involved? For example, when the internal `Mobile Deposit Service` needs to record a transaction in the `Core Ledger Service`.

**Key Difference:** A machine cannot pass an MFA prompt. Therefore, we **bypass the IdP completely** and use the **OAuth2 Client Credentials Grant**. The microservice talks directly to the IAM Control Plane.

### The M2M Architecture Sequence

```mermaid
sequenceDiagram
    participant MDS as Mobile Deposit Service
    participant IAM as auth.moneyguard.com (IAM Control Plane)
    participant PEP as Sidecar Proxy (PEP at Ledger)
    participant CLS as Core Ledger Service

    Note over MDS,IAM: 1. Request Machine Token
    MDS->>IAM: POST /oauth2/token (client_id, client_secret)
    IAM-->>MDS: Returns JWT (Scopes: ledger:write)
    
    Note over MDS,CLS: 2. Make Authenticated Internal Call
    MDS->>PEP: POST /v1/ledger/deposit + Bearer JWT
    
    Note over PEP,CLS: 3. Local Zero-Trust Enforcement
    PEP->>PEP: Local PDP Evaluates JWT Signature & Scopes
    PEP->>CLS: Forward Authorized Request
    CLS-->>MDS: 200 OK (Deposit Recorded)

```

### The M2M HTTP Payloads

**Step 1: The Token Request**
The `Mobile Deposit Service` backend makes a secure POST request to the IAM Control Plane. It authenticates itself using its unique Service Account credentials (`client_id` and `client_secret`).

* **HTTP Request:**
```http
POST /oauth2/token
Host: auth.moneyguard.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=svc_mobile_deposit&client_secret=super_secure_vault_secret&scope=ledger:write

```



**Step 2: The Token Response**
The IAM Control Plane verifies the service account exists, is active, and is allowed to request the `ledger:write` scope.

* **HTTP Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 300, 
  "scope": "ledger:write"
}

```



*(Notice this token is highly restricted and short-lived—often just 5 minutes for M2M communication).*

**Step 3: Service-to-Service Enforcement**
When the `Mobile Deposit Service` calls the `Core Ledger Service`, the Sidecar Proxy (PEP) running next to the Ledger catches the request. The local Open Policy Agent (PDP) runs a Rego policy that specifically checks: *"Does this JWT contain the `ledger:write` scope, and is the `client_id` strictly `svc_mobile_deposit`?"*

---

## 8. Real-World MoneyGuard Use Cases

To further clarify how this architecture applies to the real world, here are three distinct business scenarios broken down step-by-step:

### Use Case A: Corporate Client Federation (B2B SaaS)

**Scenario:** "Acme Corp" uses MoneyGuard to process their company payroll. Acme wants its HR employees to log into MoneyGuard without creating new passwords.

* **Implementation:** MoneyGuard's IdP uses **Federation**. When an Acme employee enters `alice@acme.com` at `corporate.moneyguard.com`, MoneyGuard recognizes the domain and redirects the login request directly to Acme's own Microsoft Azure AD.
* **Security Benefit:** If Alice is fired from Acme, Acme disables her Microsoft account. She immediately loses access to MoneyGuard because she can no longer pass the federated authentication step. MoneyGuard never stores her password.

### Use Case B: Zero-Trust Internal Microservices

**Scenario:** A developer writes a new internal `Analytics Service`. Because it is inside the corporate firewall, the developer assumes it can just pull data directly from the `Core Ledger Service` database.

* **Implementation:** Under Zero-Trust, the firewall means nothing. The `Core Ledger Service` forces the `Analytics Service` to use the **M2M Flow (Section 7)**. If the `Analytics Service` does not possess a valid JWT issued by `auth.moneyguard.com` containing the exact `ledger:read` scope, the Sidecar Proxy drops the request.
* **Security Benefit:** If a hacker breaches the `Analytics Service`, they cannot easily pivot to drain funds from the Ledger, because the Analytics Service's Service Account is not authorized to write data.

### Use Case C: Automated Employee Offboarding via SCIM

**Scenario:** A MoneyGuard Wealth Manager quits.

* **Implementation:** Human Resources updates the employee's status to "Terminated" in Workday. Workday triggers a **SCIM** (System for Cross-domain Identity Management) webhook call to `iam.moneyguard.com`.
* **Security Benefit:** The IAM Control Plane instantly strips all roles from the user's Identity Store profile. If the user tries to log in, they will be rejected.

---

## 9. Frequently Asked Questions (FAQ)

**Q: Why separate the IdP from the IAM Control Plane?**
**A:** Separation of concerns. The IdP specializes in the complex, heavily regulated world of authentication (passwords, biometrics, hardware keys). The Control Plane specializes in your specific business logic for authorization (who gets to do what within your specific bank apps).


**Q: What is the difference between an API Gateway (PEP) and a PDP?**

**A:** A simple way to understand this is to think of the API Gateway as the security guard at the entrance of a building, and the PDP as the master guest list. The security guard (API Gateway) stops you at the door and checks your ID. However, instead of deciding on their own, the guard consults the guest list (PDP) to determine whether you are actually allowed to enter the specific room you’re requesting access to. The guard enforces the decision, but the guest list decides the rules.

```mermaid
sequenceDiagram
    participant U as User / Client App
    participant G as API Gateway (PEP)
    participant D as Policy Decision Point (PDP)
    participant S as Backend Service

    U->>G: HTTP Request + JWT Token
    note right of G: PEP intercepts request\nExtracts token & context

    G->>D: Authorization Request\n(subject, action, resource, context)
    note right of D: PDP evaluates RBAC/ABAC policies\n(token, roles, attributes)

    alt Access Allowed
        D-->>G: ALLOW
        note right of G: PEP enforces decision
        G->>S: Forward request
        S-->>G: Response
        G-->>U: 200 OK + Data
    else Access Denied
        D-->>G: DENY
        G-->>U: 403 Forbidden
    end
```

In enterprise security architectures, these two components work together as a pair and are formally defined by the XACML (eXtensible Access Control Markup Language) security model. The Policy Enforcement Point (PEP) is typically implemented as an API Gateway or sidecar proxy. It acts as the “muscle” of the system. Its sole responsibility is to intercept incoming traffic, pause the request, ask for an authorization decision, and then strictly enforce whatever decision it receives. The PEP does not understand business rules or authorization logic.

The Policy Decision Point (PDP), on the other hand, is the “brain” of the system. It is usually implemented as a rule engine, such as Open Policy Agent (OPA). The PDP evaluates complex authorization rules against the user’s identity, roles, attributes, and request context. It returns a simple decision—ALLOW or DENY—but does not handle network traffic, user interaction, or request routing.

From an architectural placement perspective, the API Gateway acting as the PEP sits at the very edge of the internal network, for example at `api.moneyguard.com`. It faces the public internet and intercepts every incoming request from web or mobile clients. The PDP sits deeper inside the secure internal network, often deployed very close to the gateway, sometimes even as a sidecar container in the same cluster. This close proximity ensures that authorization checks can be performed with extremely low latency, often under one millisecond, over the local network.

To understand how these components interact in practice, consider a concrete example from a fictional financial platform called MoneyGuard. In this scenario, Alice is a Wealth Manager attempting to view a highly sensitive VIP client portfolio. She clicks a button in her application, which sends a `GET` request to `api.moneyguard.com/v1/vip-portfolios/777`. Her request includes a JWT access token.

When the request arrives, the API Gateway immediately intercepts it. Acting as the enforcement point, the gateway extracts Alice’s JWT token and pauses the request. At this stage, the gateway does not inspect authorization rules or attempt to determine whether Alice is allowed to view VIP portfolios. It simply recognizes that it must request a decision from the PDP.

The gateway then sends a fast, internal authorization request to the PDP containing all relevant context. This includes Alice’s JWT token, the HTTP method (`GET`), the requested resource (`/v1/vip-portfolios/777`), and contextual attributes such as her IP address (`192.168.1.50`). Conceptually, the gateway is asking, “Is this user allowed to perform this action on this resource under these conditions?”

The PDP receives this information and evaluates it against the authorization policies defined by the security team. These policies may include multiple checks. For example, the PDP verifies that Alice’s token includes the `WealthManager` role, confirms that the token is still valid, checks that the request originates from a trusted corporate IP address, and ensures that the access attempt occurs during approved business hours, such as between 9 AM and 5 PM on a weekday. After evaluating all rules, the PDP returns a simple JSON response indicating whether access is allowed.

If the PDP responds with `{"allow": true}`, the API Gateway enforces the decision by forwarding the request to the VIP Portfolio microservice, which retrieves the data and returns it to Alice. If, however, the PDP responds with `{"allow": false}`—for example, if Alice attempted access at 11:00 PM on a Saturday—the gateway immediately terminates the request and returns an HTTP 403 Forbidden response. In this case, the backend microservice is never contacted and remains fully protected.

Separating the PEP and PDP provides significant architectural benefits. Embedding all authorization rules directly into the API Gateway would quickly become unmanageable, especially in systems with hundreds of microservices and complex, attribute-based policies. API Gateways are optimized for high-throughput request routing and traffic management, not for evaluating complex authorization logic. By isolating policy evaluation in the PDP and enforcement in the PEP, the system remains fast, scalable, and easier to maintain. Security teams can update policies centrally without redeploying gateways, while the gateway continues to focus exclusively on enforcing decisions at high performance.


**Q: Why doesn't the Wire Transfer Microservice handle its own security?**
**A:** If we have 500 microservices at MoneyGuard, we don't want 500 different engineering teams writing their own security logic. By centralizing it at the Enforcement Layer (`api.moneyguard.com` and sidecar proxies), we ensure bank-grade security is uniformly applied, audited, and updated in one place.

**Q: What happens if `idp.moneyguard.com` goes down?**
**A:** Users cannot log in to get *new* tokens. However, because JWTs are stateless and self-contained, users who already have a valid token (e.g., one that lasts for 15 minutes) can continue working until that token expires. The API Gateway/PDP can validate existing tokens locally without calling the IdP.

**Q: Why use ABAC over RBAC for banking?**
**A:** RBAC (Roles) is too broad. If Alice has the `WealthManager` role, she could theoretically transfer money anytime, anywhere. ABAC (Attributes) allows MoneyGuard to say: *"Alice can transfer money (Role), BUT only from a bank-issued laptop (Attribute 1), during 9 AM - 5 PM (Attribute 2), and for her assigned clients only (Attribute 3)."*

**Q: How does the backend service know who made the request if it doesn't handle auth?**
**A:** The Enforcement Layer passes the validated JWT token (or a stripped-down, sanitized version of it) down to the downstream Microservice in the HTTP headers. The Microservice reads this header to know "Alice made this request" strictly for logging or database audit purposes.

---

## 10. Deep Dive: Token Revocation & Continuous Access Evaluation

A critical challenge with JWTs is that they are stateless. If a JWT has an expiration time (`exp`) set for 15 minutes, it is valid for those 15 minutes. Even if an administrator clicks "Revoke Access" in the IAM dashboard, the API Gateway (PEP) has no way of knowing that, because it verifies the JWT mathematically using a public key, without calling the central server.

If an attacker steals Alice's active JWT, how does MoneyGuard stop them before the 15 minutes are up?

### Solution 1: The Distributed Blocklist (Deny List)

MoneyGuard implements a high-speed distributed cache (e.g., **Redis**) attached directly to the API Gateway.

1. **The Revocation Event:** An Admin detects suspicious activity on Alice's account and clicks "Lock Account".
2. **Updating the List:** The IAM Control Plane immediately writes Alice's unique JWT ID (the `jti` claim inside the token payload) to the Redis Blocklist cluster.
3. **The Enforcement Check:** In the .NET API Gateway (from Section 5), before the middleware even calls the OPA PDP, it makes a sub-millisecond check against Redis: *"Is this `jti` in the Blocklist?"*
4. **Denial:** If the `jti` is found, the API Gateway immediately returns a `401 Unauthorized`, neutralizing the stolen token instantly. The `jti` is kept in Redis only until the token's original `exp` time passes, keeping the database small and fast.

### Solution 2: Continuous Access Evaluation (CAE)

For broader security events, MoneyGuard uses CAE based on the **Shared Signals Framework (SSF)**.

Rather than waiting for the API Gateway to ask if a token is valid, the IdP and IAM Control Plane *push* critical security events to the Enforcement Layer via Webhooks or an Event Stream (like Apache Kafka).

**Example Events:**

* `account_disabled`
* `password_changed`
* `high_risk_ip_detected`

If the API Gateway receives a `password_changed` event for Alice, it can instantly flush all active sessions and block any JWTs bearing her user ID (`sub` claim) that were issued prior to the timestamp of the password change, forcing an immediate re-authentication.

---
## 11. Use Case: Just-In-Time (JIT) Access & Privileged Identity Management (Zero Standing Privileges)

**Question:** *How do we eliminate standing admin privileges in the MoneyGuard ecosystem and implement Just-In-Time (JIT) access for both Human users and Machine Identities when elevated permissions are required?*

**Detailed Answer:**
Modern Zero-Trust IAM systems operate on the principle of **Zero Standing Privileges (ZSP)**. This means a user or service has zero administrative privileges by default. High privileges exist only when absolutely necessary, are explicitly approved, and automatically evaporate when the task is complete. 

Achieving this requires different architectural approaches for Humans (using OIDC and PAM brokers) versus Machines (using SPIFFE/Client Credentials and dynamic OPA policies).

### A. JIT Access for Human Identities (OIDC & PAM Integration)

For human users (e.g., a MoneyGuard Database Admin needing to run a migration), JIT relies on an external workflow engine (ITSM like ServiceNow) and short-lived tokens.

1. **Zero Privileges by Default:** The admin logs into the MoneyGuard portal via the IdP (`idp.moneyguard.com`). Their baseline JWT contains standard user roles. They cannot access production databases.
2. **The Request:** The admin navigates to the Privileged Access Management (PAM) portal and requests the `db_admin_prod` role, linking a valid Jira/ServiceNow ticket for the migration.
3. **Approval & Step-Up Auth:** A manager approves the request. The PAM broker temporarily adds the admin to the elevated group in the Identity Store. The portal immediately forces the admin's browser to perform a **Step-Up Authentication** via OIDC (prompting for a YubiKey MFA tap).
4. **Time-Bound Token Issuance:** The IAM Control Plane issues a *new* JWT. This token contains the elevated `"role": "db_admin_prod"` claim, but it is issued with a strict, aggressively short expiration time (`exp` set to exactly 2 hours).
   * The Golden Ticket (JWT): The system issues a new JSON Web Token (JWT). This is a digital pass that says: "This is the Admin, they have DB rights, but this pass self-destructs in 2 hours."
5. **Access:** The Admin presents this token to the Database Gateway. The Gateway sees the valid "DB Admin" role and the timestamp, and lets them in.
6. **Automatic Revocation:** After 2 hours, the JWT organically expires. The PAM broker automatically strips the role from the Identity Store. If the admin tries to make another API call, the PEP gateway rejects the expired token. No manual cleanup is required.

### JIT Access Architecture Flow (Human Example)

```mermaid
sequenceDiagram
    autonumber
    
    participant Admin as Database Admin
    participant PAM as PAM / ITSM Portal
    participant IAM as auth.moneyguard.com (IAM)
    participant PEP as api.moneyguard.com (PEP)

    Note over Admin, IAM: Phase 1: Baseline Access
    Admin->>IAM: Standard Login
    IAM-->>Admin: Returns Baseline JWT (No Admin Rights)
    
    Note over Admin, IAM: Phase 2: The JIT Request
    Admin->>PAM: Request 'db_admin' for Ticket #REQ-998 (2 Hours)
    PAM->>PAM: Manager Approves Request
    PAM->>IAM: API Call: Add User to 'db_admin' group temporarily
    
    Note over Admin, IAM: Phase 3: Step-Up & Elevation
    PAM-->>Admin: Prompt Step-Up Auth (MFA Required)
    Admin->>IAM: Re-Authenticate with YubiKey
    IAM-->>Admin: Returns Elevated JWT (Expires in 2 Hours)
    
    Note over Admin, PEP: Phase 4: Execution & Expiration
    Admin->>PEP: Execute DB Migration + Elevated JWT
    PEP-->>Admin: 200 OK (Allowed by PDP)
    
    Note over Admin, IAM: 2 Hours Later...
    PAM->>IAM: API Call: Remove User from 'db_admin' group
    Admin->>PEP: Execute DB Migration + Elevated JWT
    PEP-->>Admin: 401 Unauthorized (Token Expired)
```

### B. JIT Access for Machine Identities (M2M / Dynamic Policies)

In the Human (Part A) scenario, the Admin gets a special "High-Clearance License" for two hours. In this Machine (Part B) scenario, the script keeps its standard ID card, but the guard at the gate is handed a temporary memo that says: *"For the next hour only, let anyone with a 'Backup-Script' ID pass through."* Once that hour is up, the guard shreds the memo and goes back to the old rulebook where that ID is blocked—even if the script’s ID card is still valid.

### Updated JIT Flow for Machine Identities

1. **Zero Privileges by Default:** The backup workload (Machine) boots up with a basic ID token. If it tries to access the Ledger, the **Policy Decision Point (PDP)** looks at its permanent rulebook and says "No."
2. **The Request:** A "Manager" service (like Kubernetes) tells the IAM system that it's time for the scheduled backup.
3. **Dynamic Policy Injection (The "Law" Change):** The IAM system sends a temporary "Rego" policy—a snippet of code—directly to the PDP's active memory. This effectively updates the local laws on the fly.
4. **Enforcement:** The workload tries the same API call. This time, the PDP sees the temporary "Law" and grants access.
5. **Automatic Revocation:** At 3:01 AM, the temporary rule expires and disappears from the PDP’s memory. The "Law" reverts to its original state, and access is instantly cut off.

```mermaid
sequenceDiagram
    participant W as Backup Workload (Machine)
    participant K as K8s / Orchestrator
    participant IAM as IAM Control Plane
    participant PDP as PDP (The "Guard")
    participant DB as Ledger Service

    Note over PDP: Rulebook: "Deny Backup-Script"
    
    W->>PDP: 01:50 AM - Request Access
    PDP-->>W: 403 Forbidden (Law says No)

    Note over K, IAM: 02:00 AM - Backup Window Triggered
    K->>IAM: Start Backup Window
    IAM->>PDP: Inject Temp Law: "Allow Backup-Script until 03:00"
    
    W->>PDP: 02:05 AM - Request Access
    Note right of PDP: Checks Temp Law: ALLOWED
    PDP->>DB: Access Granted
    
    Note over PDP: 03:01 AM - Temp Law Expires/Self-Shreds
    
    W->>PDP: 03:05 AM - Request Access
    Note right of PDP: Back to Default Rule: DENIED
    PDP-->>W: 403 Forbidden

```

---
