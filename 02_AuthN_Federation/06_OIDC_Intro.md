### 1. The "Delivery" Problem (Tokens vs. Data)

In pure OAuth 2.0, when your React App says to Google, *"Hey, my `scope` is `email` and `profile`,"* Google says, *"Okay, the user agreed."* But here is the catch: **Google does not send the email address or name back to your app right then.** Instead, Google sends back an **Access Token** (an opaque, random string like `xyz123`).

* The Access Token is just a key card.
* It does not contain the string `"john@gmail.com"`.
* To actually get John's email, your backend has to take that `xyz123` token, open up a new HTTP connection, and make a completely separate request to a Google API endpoint to download the profile data.

### 2. The "Wild West" Standardization Problem (You guessed this perfectly)

Because OAuth 2.0 was designed for *authorization* (accessing APIs) and not *authentication* (logging people in), it didn't bother to create rules for how user profiles should look.

If you wanted to build a "Login with Google/Facebook/GitHub" feature using pure OAuth 2.0, you ran into a nightmare of inconsistencies:

* **The Scope Names were different:**
  * Google used `scope=email`
  * Another provider might use `scope=mail_address`

* **The API Endpoints were different:**
  * Google: `GET /oauth2/v3/userinfo`
  * Facebook: `GET /me`
  * GitHub: `GET /user`

* **The JSON Responses were different:**
  * Google returned: `{"email": "john@gmail.com", "given_name": "John"}`
  * Facebook returned: `{"mail": "john@gmail.com", "first_name": "John"}`
  * GitHub returned: `{"login": "john", "email_address": "john@gmail.com"}`

Your .NET API code would turn into a massive, tangled mess of `if/else` statements just to figure out how to extract a simple email address depending on which button the user clicked.

### 3. The Security Problem (Who is the token for?)

An Access Token is meant to be consumed by the **Resource Server** (the API), not the **Client** (your app). When an app blindly trusts an Access Token to mean "this user just logged in," it opens the door to hacking (specifically, the "Confused Deputy" token substitution attack). The app has no cryptographic proof that the user actually authenticated *for your specific app*.

---

## 4. Enter OpenID Connect (OIDC)

OpenID Connect (OIDC) is an authentication protocol built on top of OAuth2. **OIDC enables authentication** of end-users against an authorization server, which verifies the user's identity and issues an ID token, usually a JSON Web Token (JWT). This ID token contains information about the user in the form of “claims.”

### Clarification: Why do they say OIDC "enables authentication"?

Google was **already** verifying  (authenticating them) the user using password/MFA/FaceId/MFA etc. before providing AuthToken.  If Google didn't authenticate them, it wouldn't know whose data it was granting access to.

So why does the quote say OIDC "enables authentication"? It all comes down to **who is receiving the proof of that authentication**.

#### The Missing Piece: "For Whom?"

In pure OAuth 2.0, authentication happens, but it is a private secret between the User and Google.

  * Google checks the password (Authenticate the user).
  * Google says, *"Okay, I know who you are. Here is an Access Token for the React App."*
  * The React App receives the Access Token, but the token says absolutely nothing about the authentication event. The React App is basically just guessing: *"Well, Google gave me this token, so the user must have logged in."*

OIDC changes the "For Whom". When the quote says "OIDC enables authentication of end-users", it implicitly means **"OIDC enables authentication of end-users FOR THE CLIENT APP."**

#### The Bouncer Analogy

Think of Google as a Bouncer at a nightclub, and your React App as the Bartender inside.

  * **Pure OAuth 2.0:** The Bouncer checks the user's ID at the door (Authentication). The Bouncer then hands the user a blank "VIP Drink Ticket" (Access Token) and sends them inside. When the user hands the ticket to the Bartender, the Bartender knows the Bouncer let them in, but the Bartender has no idea who this person actually is. The Bartender **cannot "authenticate"** the person.
  * **OpenID Connect (OIDC):** The Bouncer checks the user's ID at the door. The Bouncer hands them the "VIP Drink Ticket" (Access Token) **AND** slaps a verified Name Tag (ID Token) on their hand (shirt) that says, *"My name is John, the Bouncer verified my ID at 9:00 PM."*

Now, when John walks up to the bar, the Bartender (your app) can look at that Name Tag and say, *"Ah, I can actually authenticate who you are."*

#### Breaking Down the Quote

Let's look at the official quote again with this new context:

> *"OIDC enables authentication of end-users **[for the Client App]** against an authorization server **[Google]**, which verifies the user's identity and issues an ID token... This ID token contains information about the user in the form of 'claims.'"*

Pure OAuth 2.0 gave you a blank key (Access Token). OIDC gives you a signed document proving exactly who turned the key (ID Token). That signed document is what "enables" your app to truly log the user in.

---

### How OIDC Solved This: The ID Token

The tech industry looked at this mess and said, *"We need a standard way to request identity data, and we need the Auth Server to hand the data directly to the app so we don't have to make extra API calls."*

OIDC sits right on top of OAuth 2.0 and introduces a few strict rules:

  1. **Standardized Scopes:** OIDC mandates standard scopes. You must include `scope=openid`. You can also add `profile` and `email`. Everyone agrees on these exact words.
  2. **The ID Token (The Payload):** This is the game-changer. When you use the `openid` scope, Google doesn't just give you an Access Token (the key). It also gives you an **ID Token**.
  3. **Data is Inside the Token:** An ID Token is a JSON Web Token (JWT). It actually contains the user's data baked right into it.

Your .NET API doesn't have to make a second trip to Google to ask for the email. It just cracks open the ID Token and reads it instantly. Furthermore, because it's OIDC, the JSON inside the token looks exactly the same whether the user logged in with Google, Microsoft, or Apple.

**Summary:** We *could* use scopes to get identity in pure OAuth, but it required extra network requests, had zero standardization across providers, and lacked basic login security. OIDC standardized the scopes and packaged the data safely into a brand new delivery vehicle: the ID Token.

---

### 5. Inside the OIDC "ID Token"

Because the ID Token is a standardized **JSON Web Token (JWT)**, your .NET API doesn't need to call Google to read it. It simply decodes the Base64 string locally.

When your .NET API decodes the ID Token, the payload looks like this:

```json
{
  "iss": "https://accounts.google.com",
  "aud": "your_photo_app_client_id_123",
  "sub": "1049384930283",
  "email": "john@gmail.com",
  "name": "John Doe",
  "iat": 1710000000,
  "exp": 1710003600
}

```

**Why this is perfectly secure:**

  * `iss` (Issuer): Proves exactly who created this token (Google).
  * `aud` (Audience): Proves this token was minted specifically for *your* Photo App, preventing the Confused Deputy attack.
  * `sub` (Subject): Google's unique, unchanging ID for this user.
  * `iat` / `exp`: Proves exactly when the user logged in and when the token expires.
---



## 6. The Flows: How We Safely Get These Tokens

### Day 0: The Setup & The Cryptography

Before your React App or .NET API can talk to Google, two things must happen: you must register your application, and you must understand the cryptography that makes the whole system trustable.

### Step 1: The Google Console Setup

You must log into the Google Cloud Developer Console and register your Photo App. Google will give you two critical pieces of information:

1. **Client ID:** (e.g., `photoapp123.apps.google.com`). This is your app's public identifier. You will place this in both your React frontend and your .NET backend.
2. **Client Secret:** (e.g., `GOCSPX-abc123superSecret`). This is a highly classified password. **This must NEVER go in your React app.** It belongs exclusively in your secure .NET backend (e.g., in `appsettings.json` or Azure Key Vault).

### Step 2: Understanding the Cryptography (Signatures vs. Hashes)

How do we know a hacker didn't intercept the communication or forge a fake token? The system relies on two different cryptographic concepts:

**A. Digital Signatures (Asymmetric Cryptography) — Used for the ID Token**
When your .NET API receives the OIDC ID Token, it needs to prove Google actually created it.

* **The Private Key:** Google has a highly secure, secret Private Key locked in their servers. When Google creates the ID Token for John Doe, it uses this Private Key to mathematically "sign" the token.
* **The Public Key:** Google publishes a set of **Public Keys** on a public web address. Anyone can download these.
* **The Verification:** Your .NET API downloads Google's Public Key. The math dictates that *only* the Public Key can successfully decode a signature made by the matching Private Key. If the math checks out, your API knows with 100% certainty the token is authentic and untampered.

**B. Hashing (SHA-256) — Used for PKCE**
A hash is a one-way mathematical function. If you run a word through the SHA-256 algorithm, it outputs a scrambled string. You cannot reverse the scrambled string back into the original word. We will see exactly how this protects mobile and frontend apps in Flow B below.

---
### OAuth 2.0 Flows

Now that we have our `Client ID`, our `Client Secret`, and an understanding of the cryptography, how do we safely transport the tokens from Google to your application?

We use **OAuth 2.0 Flows**. The flow you choose depends entirely on your architecture.

### Flow A: Authorization Code Flow (The Secure Backend Way)

**When to use it:** Use this when your authentication logic lives in a secure backend (like your .NET API) that can safely hide the static **Client Secret** you got from the Google Console.

In this flow, the React app never touches the actual tokens. It only acts as a messenger to pass a temporary "Code" to the backend.

```mermaid
sequenceDiagram
    actor User
    participant React as React App (Client UI)
    participant Google as Google (Auth Server)
    participant API as .NET API (Backend Client)

    User->>React: Click "Login with Google"
    React->>Google: Redirect to Google (scope=openid email, client_id=photoapp123)
    
    Note over User,Google: User authenticates and grants permission
    Google-->>React: Redirect back to React with temporary `code=abc123`
    
    React->>API: Send `code=abc123` to backend
    
    Note over API,Google: Backend uses its hidden Client Secret here!
    API->>Google: POST `code=abc123` + `Client Secret`
    Google-->>API: Returns ID Token & Access Token
    
    API->>API: Fetch Google Public Key & Verify Digital Signature!
    API->>API: Decode ID Token (john@gmail.com). Find user in DB.
    API-->>React: Issue local PhotoApp Session Cookie/Token

```

**Why we use a Code:** If Google just put the raw tokens in the browser URL, malicious browser extensions could steal them. The `code` acts as a temporary voucher that can *only* be redeemed securely by your backend using the `Client Secret`.

---

### Flow B: Authorization Code Flow with PKCE (The Frontend Way)

*(Pronounced "Pixy": Proof Key for Code Exchange)*

**When to use it:** Use this when your Single Page Application (React) or Mobile App talks directly to Google to get the tokens.

**The Problem:** Because React code runs in the user's browser, anyone can hit "View Source." You **cannot** put your `Client Secret` in React. But without a secret, if an attacker steals the temporary `code`, they can exchange it for tokens!

**The PKCE Solution:** PKCE creates a **dynamic, one-time secret** for every single login attempt using cryptographic hashing to replace the static Client Secret.

```mermaid
sequenceDiagram
    actor User
    participant React as React App (Public Client)
    participant Google as Google (Auth Server)
    participant API as .NET API (Resource Server)

    User->>React: Click "Login with Google"
    Note over React: 1. React generates `code_verifier` (a random string)<br/>2. React hashes it via SHA-256 into `code_challenge`
    
    React->>Google: Redirect to Google WITH the `code_challenge` (the hash)
    
    Note over User,Google: User authenticates and grants permission
    Google-->>React: Redirect back to React with temporary `code=abc123`
    
    Note over React,Google: React exchanges code WITHOUT a static Client Secret
    React->>Google: POST `code=abc123` + the original `code_verifier` (unhashed string)
    
    Google->>Google: Hashes the verifier via SHA-256. Does it match the challenge? Yes!
    Google-->>React: Returns ID Token & Access Token directly to React
    
    Note over React,API: React now has the tokens and calls the API
    React->>API: API Call: GET /photos (Header -> Bearer: ID Token)
    
    Note over API: API must now verify the token is real!
    API->>API: Fetch Google Public Key & Verify Digital Signature
    API->>API: Check internal DB roles (Can John view photos?)
    API-->>React: 200 OK (Returns Photos)

```

**How the Cryptographic Check Works:** When React sends the user to Google, it provides the `code_challenge` (the SHA-256 hashed string). When React later asks for the tokens, it provides the original, raw `code_verifier`. Google takes that raw `code_verifier` and runs it through the exact same SHA-256 algorithm on the spot. If the resulting hash perfectly matches the `code_challenge` provided in step one, Google knows with 100% certainty that the exact same React app that started the login process is the one asking for the tokens.

### How the PKCE Security Trap Works

To understand exactly why this prevents hackers from stealing your tokens, we have to look at how PKCE combines unbreakable math with local device security.

#### 1. The Device-Level Security (The Local Vault)

When your React or Mobile app generates the raw `code_verifier`, it stores it in the device's isolated local memory.

* **In Mobile Apps (iOS/Android):** Operating systems enforce **App Sandboxing**. Even if a user accidentally installed a malicious app on their phone, that evil app is physically blocked by the OS from looking into your Photo App's memory to steal the `code_verifier`.
* **In React Apps (Browsers):** Browsers enforce the **Same-Origin Policy**. The `code_verifier` is held in memory or `sessionStorage` that is strictly tied to `yourphotoapp.com`. A malicious website open in another tab cannot peek across the browser to read it.

	- Implementation: Generating & Storing Secrets in React

You do not need an external library for this; modern browsers have built-in **Web Cryptography**. Here is how to generate the secret, create the hash, and lock the secret in isolated storage.

```javascript
 Execution: Run this before redirecting to Google
async function handleLoginClick() {
    const codeVerifier = generateCodeVerifier();
    const codeChallenge = await generateCodeChallenge(codeVerifier);

    // B. Lock the raw secret inside isolated Session Storage
    // Malicious scripts in other tabs CANNOT read this.
    sessionStorage.setItem('pkce_code_verifier', codeVerifier);

    // C. Send the user to Google, passing ONLY the hashed challenge
    const googleAuthUrl = `https://accounts.google.com/o/oauth2/v2/auth?client_id=...&code_challenge=${codeChallenge}&code_challenge_method=S256`;
    window.location.href = googleAuthUrl;
}
```

#### 2. The Cryptographic Math (The One-Way Street)

The entire PKCE flow relies on one unbreakable rule of cryptography: **SHA-256 is a one-way street.** You can turn a `code_verifier` into a `code_challenge` hash in a millisecond, but all the supercomputers on Earth cannot reverse that hash back into the original text.

#### 3. The Attack Scenario: Why the Hacker Fails

Imagine a malicious network sniffer or a rogue browser extension is spying on the user's internet traffic.

1. **The Setup:** Your app generates a secret `"my_secret_123"`, locks it securely in local memory, hashes it to `"x8f9...q2p"`, and sends the user to Google.
2. **The Interception:** The user logs in, and Google redirects them back to your app with the temporary code in the URL (`code=abc123`). The hacker's network sniffer intercepts this URL and steals the code.
3. **The Trap:** The hacker races to Google's server and says, *"Hey Google! I stole `code=abc123`! Give me the Access Token!"*
4. **The Block:** Google says, *"Okay, I have a hash waiting for that code. Hand over the raw `code_verifier` so I can hash it and see if it matches."* 5.  **Game Over:** Because the hacker only intercepted network traffic, they only ever saw the mathematically irreversible **hash** (`code_challenge`). Because of App Sandboxing and Browser Isolation, they cannot reach into the user's physical device to steal the **raw** `code_verifier`. The math fails, Google rejects the request, and the hacker gets nothing.

---

#### Why `sessionStorage`?

We store the `code_verifier` in `sessionStorage` rather than `localStorage` because `sessionStorage` is strictly sandboxed to the specific browser tab. If the user closes the tab, the memory is wiped. This ensures the secret is never persisted on disk and remains inaccessible to any other origin or process.

**Summary:** PKCE is foolproof because the actual secret (`code_verifier`) is never sent over the network until the final, encrypted POST request to Google. By the time that happens, the temporary `code` has already been used and is invalid for any attacker.

---

### Flow C: The OIDC Identity Layer (The Nonce Security Check)

**When to use it:** This flow is not a separate choice from Flow B, but rather an **added security layer** within OpenID Connect (OIDC). It is mandatory when your application acts as a **Relying Party** (React App) to prevent "Replay Attacks."

#### What is a Relying Party (RP)?

In our stack, the **React App** is the **Relying Party**.

* **Why the name?** Because your application is **relying** on a third party (Google) to verify that the user is who they say they are.
* **The Relationship:** Google is the **OpenID Provider (OP)**. It "provides" the identity. Your React App "relies" on that identity to log the user into your system.

**The Problem:** An ID Token is like a digital passport. If a hacker intercepts a valid ID Token (for example, through a man-in-the-middle attack or by stealing it from a log file), they could try to "replay" that token to your .NET API to log in as that user, even though they never actually performed the login themselves.

**The Nonce Solution:** The Nonce (Number used ONCE) acts as a cryptographic "handshake" that binds your specific browser session to the specific ID Token issued by Google.

```mermaid
sequenceDiagram
    actor User
    participant React as React App (Relying Party)
    participant Google as Google (OpenID Provider)
    participant API as .NET API (Resource Server)

    User->>React: Click "Login with Google"
    Note over React: 1. React generates a random, long string (Nonce: xyz_999)
    Note over React: 2. React saves Nonce in sessionStorage
    
    React->>Google: HTTP 302 Redirect (with scope=openid & nonce=xyz_999)
    
    Note over User,Google: User enters credentials at Google
    Google->>Google: Authenticates user & copies nonce:xyz_999 into ID Token claims
    
    Google-->>React: HTTP 302 Redirect (with code=abc123)
    
    React->>Google: HTTP POST /token (Exchange code for tokens)
    Google-->>React: HTTP 200 OK (Returns ID Token with nonce:xyz_999 inside)
    
    Note over React: React pulls expected Nonce ("xyz_999") from sessionStorage
    React->>API: HTTP POST /api/auth/login <br/>(Body contains: ID_Token AND clientNonce: "xyz_999")
    
    Note over API: 1. API decodes ID Token payload<br/>2. API extracts 'nonce' claim from token ("xyz_999")
    Note over API: 3. API compares Token Nonce with the clientNonce provided in the request body
    
    alt Nonce Matches
        API-->>React: HTTP 200 OK (Identity Verified, issues App Session)
    else Nonce Mismatch
        API-->>React: HTTP 401 Unauthorized (Replay Attack Detected)
    end
```

#### The Attack Scenario: Why the Hacker Fails

Here is exactly why a Replay Attack fails when a `nonce` is implemented.

```mermaid
sequenceDiagram
    actor Hacker
    participant API as .NET API (Resource Server)

    Note over Hacker: 1. Hacker steals an old ID Token<br/>(Google sealed nonce "old_abc" inside it)
    Note over Hacker: 2. Hacker writes a script to attack the API.<br/>They don't know the original nonce, so they fake a new one ("new_xyz").
    
    Hacker->>API: HTTP POST /api/auth/login <br/>(Body: ID_Token + clientNonce: "new_xyz")
    
    Note over API: 1. API checks Token Signature (Valid - it's a real Google token)
    Note over API: 2. API opens Token and extracts the baked-in 'nonce': "old_abc"
    Note over API: 3. API reads the POST body and extracts the provided 'clientNonce': "new_xyz"
    
    Note over API: 4. API compares them: "old_abc" DOES NOT EQUAL "new_xyz"
    
    API-->>Hacker: HTTP 401 Unauthorized (Replay Attack Detected - Blocked!)

```

**Why it fails:** The hacker possesses a mathematically valid token with a perfect Google signature, but they do not possess the *context* of the current session. Because the .NET API strictly checks the `nonce` inside the token against the active session's expected `nonce` (which the hacker does not know and cannot generate), the stolen token is immediately identified as a replay and discarded.

#### How the Nonce Check Works (The Chain of Custody):

To understand who does what, here is the exact chain of custody for the Nonce:

| Step | Component | Responsibility | Technical Action |
| --- | --- | --- | --- |
| **1. Creation** | **React App** (Relying Party) | Create session ID. | Generates a random string and saves it in `sessionStorage`. |
| **2. Transport** | **Browser** | Pass Nonce to Google. | Sends the nonce in the URL during the HTTP 302 Redirect. Google doesn't process it; it just "carries" it. |
| **3. Embedding** | **Google** (OpenID Provider) | "Stitch" Nonce into token. | Places the Nonce into the JSON payload and **Digitally Signs** it. This makes the `nonce` immutable. |
| **4. Validation** | **.NET API** (Resource Server) | Verify before access. | Compares the Nonce inside the JWT to the expected session value. |

1. **Creation:** Before the user leaves your React app, you create a random string (the `nonce`). You store this string in the browser's `sessionStorage`.
2. **The Trip:** You send the `nonce` to Google. Google doesn't do anything with it except "carry it" along.
3. **The Embedding:** When Google creates your ID Token, it places your `nonce` inside the JSON payload. Crucially, Google then **Digitally Signs** the token. This makes the `nonce` immutable—it cannot be changed by anyone.
4. **The Validation:** When the ID Token reaches your .NET API, the API compares the `nonce` inside the token to the `nonce` you generated at the start. If they match, it proves this token was created *specifically* for this current login session and isn't a "replay" of an old token from an hour ago.

#### Why no one can access the Nonce?

* **Browser Isolation:** The `nonce` is stored in `sessionStorage`. Because of the **Same-Origin Policy**, a hacker's website in another tab is physically blocked by the browser from reading your app's `sessionStorage`.
* **Signature Integrity:** Even if a hacker sees the `nonce` in the ID Token, they cannot modify it. If they tried to change the `nonce` inside the token to match their own session, the **Digital Signature** would break, and your .NET API would reject the token as "Tampered."

**Summary of Security Layers:**

* **PKCE (Flow B):** Protects the **Code** (The "Voucher" for the token).
* **Nonce (Flow C):** Protects the **ID Token** (The "Passport" itself).

---

### Summary: Which Security Layers Should I Use?

The choice of flow depends on your architecture, but in a modern **OIDC** environment, you are usually layering these together.

| Architecture Setup | Recommended Flow | Security Layers | Why? |
| --- | --- | --- | --- |
| **React + .NET API (Backend handles Auth)** | **Authorization Code Flow** | Client Secret + Nonce | The .NET API is a "Confidential Client"; it can safely hide a static **Client Secret** to prove its identity to Google. |
| **React only (SPA talking to APIs)** | **Auth Code Flow + PKCE** | PKCE + Nonce | React is a "Public Client." It cannot hide a secret. **PKCE** secures the code exchange, and **Nonce** secures the user's identity. |
| **Mobile App (iOS / Android)** | **Auth Code Flow + PKCE** | PKCE + Nonce | Mobile apps can be decompiled. **PKCE** is mandatory to prevent code interception on the device. |

---

### 8. Why the "Relying Party" (React) Needs the Nonce

In our setup, the **React App** is the **Relying Party (RP)**.

* **Definition:** It is called a "Relying Party" because it **relies** on an external Identity Provider (Google) to tell it who the user is.
* **The Risk:** Without a **Nonce**, the Relying Party is vulnerable to a **Replay Attack**. An attacker could steal an old, valid ID Token and "replay" it to your .NET API. Because the token has a valid Digital Signature from Google, your API would trust it and log the attacker in.

#### How the Nonce Prevents the Attack:

1. **Generation:** The Relying Party (React) generates a unique `nonce` string and stores it in **SessionStorage**.
2. **Binding:** Google receives this `nonce` and "bakes" it into the ID Token.
3. **Validation:** When the ID Token arrives at your .NET API, the API extracts the `nonce` and ensures it matches the original value from that specific user's session.
4. **Result:** If a hacker tries to "replay" an old token, the `nonce` inside that stolen token will not match the current session's `nonce`. Your API will see the mismatch and reject the login.

---

### 8. Final Security Checklist for your .NET API

When your .NET API receives the ID Token from the React App, it must perform these three checks to be 100% secure:

1. **Signature Check:** Use the **Public Key** from Google to verify the **Digital Signature**. (Proves Google sent it).
2. **Audience Check (`aud`):** Ensure the Client ID in the token matches your app's Client ID. (Proves it was meant for *your* app).
3. **Nonce Check (`nonce`):** Ensure the nonce in the token matches the nonce generated by your React App. (Proves it's not a replay of an old session).

**Note:** Today, **PKCE** is considered so secure that it is becoming the industry standard best practice to use it **all the time**, even if you have a secure .NET backend. It adds a "Defense in Depth" layer that protects the Authorization Code even if your Client Secret were somehow compromised.

---

### Implementation: Generating the Nonce in React

*(Note: It is highly recommended to use the browser's built-in `window.crypto` for generating security strings rather than `Math.random()`, as it provides cryptographically secure randomness).*

```javascript
// 1. Generate a cryptographically secure random nonce string
function generateNonce() {
    const array = new Uint8Array(16);
    window.crypto.getRandomValues(array);
    return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('');
}

// 2. Implementation in your Login Handler
async function startLogin() {
    const nonce = generateNonce();
    
    // Store in session storage so we can check it later
    sessionStorage.setItem('auth_nonce', nonce);
    
    // Pass it to Google in the URL
    const authUrl = `https://accounts.google.com/o/oauth2/v2/auth?nonce=${nonce}&response_type=code&scope=openid email&client_id=YOUR_CLIENT_ID...`;
    window.location.href = authUrl;
}

```

### Implementation: Verifying the Nonce in .NET (Resource Server)

When your .NET API receives the token, it doesn't just check the signature; it checks the **Claims**. Here is how the Resource Server completes the Chain of Custody:

```csharp
// Inside your .NET Auth Controller
var payload = await GoogleJsonWebSignature.ValidateAsync(idToken, settings);

// 1. Digital Signature Check: Done by ValidateAsync (using Google's Public Key)
// 2. Audience Check: Done by ValidateAsync (checks your Client ID)

// 3. Nonce Check: Manually verified by the Resource Server
string expectedNonce = GetExpectedNonceFromSession(); // Retrieve the nonce you stored earlier

if (payload.Nonce != expectedNonce)
{
    // If the nonce in the token doesn't match your session, it's a replay attack!
    return Unauthorized("Security violation: Nonce mismatch.");
}

```
---
## 9. Implementation: Validating the Token & Issuing Your App Session

Once your React app (or your backend) finishes the flow, you are left holding a **Google ID Token**. You must prove it is real, and then use it to log the user into your .NET API by issuing an internal Application Token.

### Step 1: Install the Google Auth Package

Never try to write custom JWT validation code for Google tokens. Google provides an official package that automatically downloads their public keys and securely validates the signature.

```bash
dotnet add package Google.Apis.Auth
dotnet add package System.IdentityModel.Tokens.Jwt

```

### Step 2: The .NET Auth Controller

Here is the exact code where **OAuth Authorization** (Google) hands off to **Application Authorization** (Your API).

```csharp
using Google.Apis.Auth;
using Microsoft.AspNetCore.Mvc;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;

[ApiController]
[Route("api/auth")]
public class AuthController : ControllerBase
{
    // 1. Your Client ID from Google Developer Console
    private const string GOOGLE_CLIENT_ID = "YOUR_CLIENT_ID.apps.googleusercontent.com";
    
    // 2. Your internal secret for signing your OWN app tokens (Keep this safe in appsettings.json!)
    private const string INTERNAL_API_SECRET = "super_secret_key_that_is_at_least_32_characters_long!";

    [HttpPost("google-login")]
    public async Task<IActionResult> GoogleLogin([FromBody] GoogleLoginRequest request)
    {
        try
        {
            // --- A. OAUTH PHASE (Validating Google's Token) ---
            
            var settings = new GoogleJsonWebSignature.ValidationSettings()
            {
                Audience = new[] { GOOGLE_CLIENT_ID } // Prevents the Confused Deputy attack!
            };

            // Cryptographically validates the signature and expiration
            GoogleJsonWebSignature.Payload payload = await GoogleJsonWebSignature.ValidateAsync(request.IdToken, settings);

            // --- B. APPLICATION AUTH PHASE (Database & Internal Roles) ---

            string userEmail = payload.Email;

            // 1. Look up the user in YOUR database (Pseudo-code)
            // var internalUser = _dbContext.Users.FirstOrDefault(u => u.Email == userEmail);
            // if (internalUser == null) internalUser = CreateNewUser(userEmail);
            
            // Let's pretend we found the user and they are an Admin in our database
            string internalRole = "Admin"; 
            string internalUserId = "10";

            // 2. Issue YOUR internal PhotoApp Session Token
            string appToken = GenerateInternalJwt(internalUserId, userEmail, internalRole);

            return Ok(new { 
                Message = "Successfully authenticated!",
                AppToken = appToken // React will save this and send it with future requests
            });
        }
        catch (InvalidJwtException)
        {
            return Unauthorized("Invalid Google ID Token.");
        }
    }

    private string GenerateInternalJwt(string userId, string email, string role)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.ASCII.GetBytes(INTERNAL_API_SECRET);
        
        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(new[]
            {
                new Claim(ClaimTypes.NameIdentifier, userId),
                new Claim(ClaimTypes.Email, email),
                new Claim(ClaimTypes.Role, role) // This is what Application Authorization uses!
            }),
            Expires = DateTime.UtcNow.AddHours(2), // Your app session lasts 2 hours
            SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256Signature)
        };

        var token = tokenHandler.CreateToken(tokenDescriptor);
        return tokenHandler.WriteToken(token);
    }
}

public class GoogleLoginRequest
{
    public string IdToken { get; set; }
}

```

### Step 3: React Calls Your Protected API

Now that React has your **Internal App Token**, it throws the Google token away. From now on, React communicates exclusively with your `.NET API` using your token.

When React wants to delete a photo, it sends a request like this:

```javascript
fetch('https://api.photoapp.com/photos/1', {
  method: 'DELETE',
  headers: {
    'Authorization': 'Bearer YOUR_INTERNAL_APP_TOKEN'
  }
});

```

Because your internal token contains `ClaimTypes.Role = "Admin"`, your `.NET API` can now use standard `[Authorize(Roles = "Admin")]` tags on your controllers to natively enforce your application's business logic!

---

