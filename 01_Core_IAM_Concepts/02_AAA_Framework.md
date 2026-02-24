# 📘 Section : The AAA Framework

### Context: Global FinBank Enterprise System

In a monolithic .NET application, AAA often happens inside a single `web.config` or `Startup.cs`. In an enterprise bank, these are often three completely distinct systems working in harmony.

### 1. Authentication (AuthN) – "Who are you?"

**The Verification Phase.**
Authentication is the process of verifying a user's identity claims. It establishes a "Principal."

* **The Concept:** The system challenges the user to prove they are who they say they are.
* **The Bank Scenario:**
* **User:** Alice (Senior Teller).
* **Action:** Alice opens the "FinBank Core Portal" on her workstation.
* **The Challenge:** The system redirects her to the Identity Provider (IdP) (e.g., Azure AD or Okta). She provides:
1. **Something she knows:** Password.
2. **Something she has:** A YubiKey or an MFA Push to her phone.


* **The Result:** If successful, the IdP issues an **Identity Token** (ID Token). This token contains "Claims" (Name: Alice, EmployeeID: 12345, Dept: RetailBanking).


* **Developer Translation:** This is the logic that sets `HttpContext.User.Identity.IsAuthenticated = true`.

### 2. Authorization (AuthZ) – "What can you do?"

**The Decision Phase.**
Authorization is the process of verifying if the authenticated user has permission to perform a specific action on a specific resource.

* **The Concept:** Just because Alice is in the building (AuthN), doesn't mean she has the key to the vault (AuthZ).
* **The Bank Scenario:**
* **Action:** Alice tries to process a wire transfer of **$50,000**.
* **The Policy Check:** The application checks Alice’s claims against the banking policy.
* *Rule 1:* Does she have the role `Teller`? **Yes.**
* *Rule 2:* Is the amount < $10,000? **No.**


* **The Decision:** DENY. The system triggers a "Step-up Authorization" workflow requiring a Manager's approval.


* **Developer Translation:** This is your `[Authorize(Roles="Manager")]` attribute or a Policy-based requirement in .NET Core (`policy.RequireClaim("TransferLimit", "50000")`).

### 3. Accounting (Auditing) – "What did you do?"

**The Recording Phase.**
Accounting tracks the user’s consumption of resources. In a bank, this is the most critical layer for survival against lawsuits and federal audits.

* **The Concept:** Non-repudiation. If money goes missing, we must prove exactly who moved it and when.
* **The Bank Scenario:**
* **Event:** Alice views the account balance of a High Net-Worth Client.
* **The Log:** The system silently writes an immutable record to the SIEM (Security Information and Event Management) system (e.g., Splunk).
* **The Data Point:** `{ Timestamp: "2023-10-27T10:00:00Z", Actor: "Alice", Action: "View_Balance", ResourceID: "Acct_999", IP: "10.0.0.5" }`.


* **Developer Translation:** This is not just `ILogger`. This is structured audit logging, often written to a WORM (Write Once, Read Many) storage device so logs cannot be deleted by hackers.

---

### ⚔️ Authentication vs. Authorization: The Comparison

It is vital to separate these concerns in your architecture.

| Feature | Authentication (AuthN) | Authorization (AuthZ) |
| --- | --- | --- |
| **Question** | "Is this Alice?" | "Is Alice allowed to approve this loan?" |
| **Data Artifact** | **ID Token** (contains User Profile) | **Access Token** (contains Scopes/Permissions) |
| **Timing** | Happens **Once** (usually at session start). | Happens **Every Request** (API call). |
| **Protocols** | OpenID Connect (OIDC), SAML, Kerberos. | OAuth 2.0, XACML, OPA. |
| **Standard Error** | `401 Unauthorized` (You aren't logged in). | `403 Forbidden` (You are logged in, but blocked). |

---

### 🚀 Real-World Enterprise Use Case

**Scenario:** The "Suspicious Wire Transfer" at Global FinBank.

Let's watch the AAA framework handle a complex scenario step-by-step.

1. **AuthN (The Login):**
* **Step:** Bob, a Junior Teller, logs in from a coffee shop Wi-Fi.
* **System Action:** The IdP sees a "New Device" and "Risky IP." It challenges Bob with a strict biometric check (FaceID). Bob passes.
* **Status:** Authenticated.


2. **AuthZ (The Access Request):**
* **Step:** Bob attempts to access the "Swift Wire Transfer" portal.
* **System Action:** The application checks Bob’s attributes.
* *Policy:* "Access to Swift Portal requires 'Corporate Network' OR 'VPN'."
* *Context:* Bob is on public Wi-Fi.

* **Status:** **403 Forbidden**. Authorization Denied based on context (ABAC - Attribute-Based Access Control), even though he has the right Role.

3. **Accounting (The Trail):**
* **Step:** The denial is logged.
* **System Action:** The SIEM triggers an alert: *"Multiple failed access attempts to Swift Portal from external IP by user Bob."*
* **Result:** The SOC (Security Operations Center) freezes Bob's account within 2 minutes to prevent potential fraud.

---

### ❓ FAQ: Common Developer Questions

**Q1: Why do I get a 401 error when I know my password is correct?**
- **A:** In a modern IAM system, a 401 usually means the *token* is missing, expired, or invalid. You might have authenticated successfully, but if your token expired 5 minutes ago, the API sees you as "Anonymous" (401).

**Q2: Can we handle Authorization inside the Authentication step?**
- **A:** Authorization *can* be handled during authentication by embedding roles or permissions inside the ID or Access Token, but this is a **bad practice for “Day-2” operations**. Tokens are **static**—once issued, they remain valid until they expire. If access must be revoked immediately (for example, employee termination, role change, or security incident), the system cannot enforce that change while the user still holds a valid token. Policy updates and contextual rules (time, risk, transaction value) are also ignored until token renewal. In regulated environments like **banks**, **real-time authorization**—checking current permissions from a database or policy engine on every request—is safer because it enables immediate revocation and dynamic enforcement.
- **Example:**
Alice is terminated at 10:05 AM, but her token expires at 11:00 AM.
Even though HR revoked her role, she can still access systems for 55 minutes.
Real-time authorization would block her access immediately.

**Q3: Is MFA part of AuthN (Authentication) or AuthZ (Authorization)?**
- **A:** **MFA (Multi-Factor Authentication) belongs entirely to AuthN (Authentication)** because its sole purpose is to **prove who the user is** by requiring multiple independent factors (something you know, have, or are). MFA strengthens identity verification but **does not decide what the user is allowed to do**—that decision remains the responsibility of **AuthZ (Authorization)**.

However, **authorization policies can depend on authentication strength**. An AuthZ policy may require that MFA was used during login before granting access to sensitive resources. In this case, AuthZ is **not performing MFA**, but it is **checking an authentication attribute** (e.g., `mfa=true`) before allowing access. This separation keeps identity verification clean while enabling risk-based and compliance-driven access control.

- **Example (2–3 lines):**
Alice logs in using username + password + OTP (MFA) → Authentication succeeds.
Authorization policy checks `mfa=true` before allowing access to wire-transfer APIs.
Without MFA, access is denied even though Alice’s identity is valid.

