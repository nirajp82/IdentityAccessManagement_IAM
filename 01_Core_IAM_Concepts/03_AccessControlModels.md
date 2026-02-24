# 📘 The Evolution of Access Control Models (RBAC, ABAC, PBAC)

### The Context: MoneyGuard Banking Platform

In a simple application, authorization is easy. You add a tag like `[Authorize(Roles = "Admin")]` to your code, and you are done. But in an enterprise system like **MoneyGuard**, a global bank, authorization must handle regulatory rules, remote work constraints, and highly sensitive financial data. Hardcoding roles simply won't survive the scale.

In system design, authorization is an evolution. You do not just pick one model; you combine them as your system grows in complexity.

---

## 1️⃣ RBAC (Role-Based Access Control)

**The Mental Model:** The VIP Badge. Your title determines your access.

RBAC is the baseline. It assumes that authorization is determined entirely by a person's **job function**. Instead of giving Alice permission to look at accounts, you give the *Teller* role permission to look at accounts, and you make Alice a Teller.

* **How it Works:** The IAM team creates roles like **Teller** or **BranchManager** in an identity provider (like Azure AD). When Alice is hired, she is assigned the Teller role and instantly inherits all permissions tied to that role.
* **The .NET Reality:** This is perfect for high-level, coarse-grained checks. Can this person access the application at all?
```csharp
[Authorize(Roles = "Teller")]
public IActionResult ViewAccount(Guid accountId) { ... }

```


* **The Breaking Point ("Role Explosion"):** RBAC fails when access requires *context*. If New York Tellers cannot view VIP accounts, and work-from-home Tellers cannot approve loans, you end up creating hyper-specific roles like `NY_Teller` or `WFH_Teller`. Soon, you have thousands of overlapping roles, making the system impossible to audit.

---

## 2️⃣ ABAC (Attribute-Based Access Control)

**The Mental Model:** The Security Guard with a Checklist. Context matters.

ABAC fixes Role Explosion. Instead of just asking *who* the user is, ABAC asks *under what conditions* access should be allowed. It makes decisions in real-time by evaluating three pillars:

1. **Subject Attributes:** Data about the user (e.g., Department, Clearance Level).
2. **Resource Attributes:** Data about the thing being accessed (e.g., Data Classification, Dollar Amount).
3. **Environment Attributes:** Data about the current situation (e.g., Time of Day, IP Address).

* **How it Works:** Alice (a Teller) tries to view a VIP account while working from home. The ABAC rule says: *Allow access ONLY IF the user is a Teller, the resource is a standard account, AND the environment is the corporate office network.* Because Alice is at home looking at a VIP account, the system denies access. No special `WFH_Teller` role is needed.

---

## 3️⃣ PBAC (Policy-Based Access Control)

**The Mental Model:** The Central Command Center. Externalized logic.

ABAC tells you *what* to evaluate (attributes). PBAC tells you *where* to evaluate it. As applications grow into distributed microservices, you don't want complex ABAC logic duplicated across 50 different APIs. PBAC extracts the authorization logic out of your code entirely and puts it into a centralized Policy Engine.

* **How it Works:** Regulators introduce a rule: *Transactions over $10k need two approvals, but only if the risk score is high.* Instead of hardcoding this in C#, MoneyGuard writes this as "Policy-as-Code" in a central engine. The API simply pauses the request, asks the engine for permission, and enforces the "Allow/Deny" response.

---

## ⚔️ The Quick Comparison Guide

| Feature | RBAC (Role-Based) | ABAC (Attribute-Based) | PBAC (Policy-Based) |
| --- | --- | --- | --- |
| **Decision Driver** | Static job titles | Real-time context | Externalized rule engines |
| **Superpower** | Simple and easy to set up | Highly flexible and granular | Easy to audit and govern globally |
| **Achilles Heel** | Role explosion | Code sprawl if not managed | High operational complexity |
| **Best Used For** | Basic app login and feature toggles | Row-level data access | Complex enterprise & regulatory rules |

---

## 📘 Part 2: Mapping These Models to OAuth2 & OIDC

A massive trap engineers fall into is trying to cram *all* authorization logic into a single token. Tokens should carry **stable identity context**. Dynamic authorization decisions must be made at runtime.

* **RBAC in Tokens (The Perfect Match):** Roles are stable. They fit perfectly into token claims.
```json
{ "sub": "user-123", "roles": ["Teller"], "groups": ["Branch-NY"] }

```


* **ABAC in Tokens (The Split Responsibility):** You **cannot** put ABAC fully into a token because environment and resource data change constantly. The Token carries *Subject Attributes*, but the Application must fetch *Resource Attributes* from the DB and evaluate *Environment Attributes* at runtime.
* **PBAC in Tokens (Identity Only):** The token simply tells the application *who* is making the request. The application takes that token data, gathers the rest of the context, and asks the central Policy Engine for permission.

### ⚠️ Common System Design Pitfalls to Avoid

1. **Bloating the Token:** Do not encode fine-grained permissions (like `can_edit_account_12345`) directly into tokens. It breaks size limits.
2. **Misusing Scopes:** OAuth2 scopes (like `read:accounts`) are for delegating access to clients, not a replacement for user permissions.
3. **Stale Context:** Relying on long-lived tokens to make sensitive data-access decisions ignores that the user's risk profile may have changed 10 minutes ago.

---

## 📘 Part 3: Practical Implementation Guide

Let's implement a strict banking rule:

> *A **Teller** (Subject) can only view a **VIP Account** (Resource) if they are connected from the **Corporate Network** (Environment).*

### Level 1: ABAC in .NET (Code-Level Authorization)

In this approach, the logic lives inside the .NET application using Custom Authorization Handlers.

**1. The Resource & Requirement:**

```csharp
public class BankAccount { public bool IsVip { get; set; } }

public class CorporateNetworkVipAccessRequirement : IAuthorizationRequirement { }

```

**2. The ABAC Handler (The Brain):**

```csharp
public class VipAccountAccessHandler : AuthorizationHandler<CorporateNetworkVipAccessRequirement, BankAccount>
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    public VipAccountAccessHandler(IHttpContextAccessor httpContextAccessor) => _httpContextAccessor = httpContextAccessor;

    protected override Task HandleRequirementAsync(AuthorizationHandlerContext context, CorporateNetworkVipAccessRequirement req, BankAccount resource)
    {
        // 1. SUBJECT ATTRIBUTE
        if (!context.User.IsInRole("Teller")) return Task.CompletedTask; 

        // 2. RESOURCE ATTRIBUTE
        if (!resource.IsVip) { context.Succeed(req); return Task.CompletedTask; }

        // 3. ENVIRONMENT ATTRIBUTE
        var ip = _httpContextAccessor.HttpContext?.Connection?.RemoteIpAddress?.ToString();
        if (ip == "192.168.1.100") { context.Succeed(req); } // All context conditions met!

        return Task.CompletedTask;
    }
}

```

**3. The Clean Controller:**
The API controller fetches the data and asks the .NET Authorization Service to evaluate the handler.

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetAccount(Guid id)
{
    var account = await _repository.GetAccountByIdAsync(id);
    var authResult = await _authService.AuthorizeAsync(User, account, "StrictVipPolicy");

    if (!authResult.Succeeded) return Forbid(); 
    return Ok(account);
}

```

---

### Level 2: PBAC with Open Policy Agent (Externalized Authorization)

If MoneyGuard grows to 50 microservices, writing C# handlers everywhere is unmaintainable. PBAC extracts the "Brain" completely out of the app into a tool like **Open Policy Agent (OPA)**.

**1. The OPA Policy (Written in Rego):**
The security team writes Policy-as-Code. The .NET app no longer knows the rules.

```rego
package moneyguard.accounts
default allow = false

# Rule: Tellers can access VIP accounts ONLY from the corporate IP
allow {
    input.user.role == "Teller"
    input.resource.is_vip == true
    input.environment.ip_address == "192.168.1.100"
}

```

**2. The Streamlined .NET Application:**
The .NET app gathers the context, builds a JSON payload, and asks OPA for a `true/false` decision. No complex C# handlers are needed.

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetAccount(Guid id)
{
    var account = await _repository.GetAccountByIdAsync(id);
    
    // Build the Context Payload
    var context = new {
        user = new { role = User.FindFirst("role")?.Value },
        resource = new { is_vip = account.IsVip },
        environment = new { ip_address = HttpContext.Connection.RemoteIpAddress?.ToString() }
    };

    // Ask OPA for the decision
    bool isAllowed = await _opaClient.EvaluatePolicyAsync("moneyguard/accounts/allow", context);

    if (!isAllowed) return Forbid(); 
    return Ok(account);
}

```

**Why this is the Ultimate Evolution:**

* **Decoupled Security:** The security team can update IPs in the Rego file, and it takes effect bank-wide instantly. No code redeployments.
* **Universal Language:** .NET, Java, and Python microservices all send the same JSON to OPA.
* **Centralized Auditing:** Auditors can view all company access rules in one centralized policy repository.
---
# 🛡️ The Master Guide to IAM Authorization: RBAC, ABAC, & PBAC

Before we dive into the rules, let's establish a mental anchor. Imagine securing a highly sensitive bank vault.

* **RBAC (Role-Based) is the Front Door Gatekeeper:** It only looks at your ID badge's job title. *(“Are you a Manager? Yes. You may enter the building.”)*
* **ABAC (Attribute-Based) is the Vault Guard:** It looks at the specific context of what you are trying to do right now. *(“Are you accessing a local account, during your shift, on a secure network? Yes. Proceed.”)*
* **PBAC (Policy-Based) is the Chief Compliance Judge:** It holds the master lawbook for the whole bank. *(“Does corporate policy require two managers to approve a transfer of this size? Yes. I will hold this until the second manager signs.”)*

---

## 📖 The Real-World Use Case: High-Value Bank Transfer

*Let's see exactly why we need all three models working together to prevent system failure.*

**The Scenario:**

* **Who:** Alice is a **Branch Manager** at MoneyGuard.
* **What:** She wants to approve a **$25,000 wire transfer**.
* **Context:** This action is **high-risk**, regulated, and involves multiple microservices.

### Step 1️⃣ RBAC — “Can Alice even open this door?”

**Problem it solves:** RBAC answers the fundamental question: *Does Alice’s job function allow access to this feature at all?*

* **Rule Example:** Only users with the role `Manager` can approve transfers.
* **How it works:**
* Alice’s role is in her identity token (`roles: ["Manager"]`).
* The application checks the token → “Yes, Alice is a Manager” → continue.
* If Alice were a Teller → RBAC blocks immediately.


* **✅ Benefit:** Simple, fast, auditable, coarse-grained decision. It saves the system from doing heavy calculations for unauthorized people.

### Step 2️⃣ ABAC — “Is Alice allowed for this specific transfer?”

**Problem it solves:** RBAC cannot handle **conditional access based on dynamic attributes**. For example, we want to:

* Only approve transfers from the **corporate network**.
* Only approve transfers **during business hours**.
* Only approve transfers **up to $50,000**.
If we tried to handle these in RBAC, we’d need thousands of “niche roles” like `Manager_NY_CorpNetwork_BusinessHours`. This causes **Role Explosion**, which is impossible to maintain.

**ABAC solution:** Evaluate **attributes at runtime**:

* **User Attribute:** region = NY, clearance = L2
* **Resource Attribute:** amount = 25,000
* **Environment Attribute:** network = corporate, time = 2:30 PM
* **Decision:** Alice passes ABAC rules → allowed to approve this transfer **contextually**.
* **✅ Benefit:** Fine-grained, contextual, flexible. No role explosion.

### Step 3️⃣ PBAC — “Does the organization allow this action right now?”

**Problem ABAC doesn’t solve:** While ABAC can check context per user/resource/environment, it doesn’t:

1. **Centralize complex regulatory rules** that affect multiple services.
2. **Version, audit, or govern policies** for compliance.
3. **Handle multi-step or multi-actor approvals**.

**Example Problem:** MoneyGuard has a rule: *Transfers > $10,000 require **two manager approvals**, MFA authentication, and a risk score below a threshold.* ABAC alone can’t easily check “two approvals across different users” or enforce organization-wide compliance rules. Encoding this logic in each service’s code would lead to **duplicated, inconsistent, brittle logic**.

**PBAC Solution:**

* Policy is defined centrally in a **policy engine**.
* The Application sends **user attributes + resource + environment** to the engine.
* The Engine returns **allow/deny + reason**.
* Policy can be updated centrally without redeploying services.
* **Example:** Alice approves the first $25k transfer → PBAC says “pending second approval required”. Bob approves the second → PBAC evaluates MFA, risk score, time → PBAC says “allow execute”.
* **✅ Benefit:** Centralized governance, auditable/versioned policies, and handles **multi-step or regulated workflows** across services.

### ✅ The Mix-and-Match Story (Why we need all three)

1. **RBAC** → “Alice is a Manager, so she can open the feature” ✅
2. **ABAC** → “Alice is allowed under current context (branch, network, time, transaction amount)” ✅
3. **PBAC** → “Regulatory policies and multi-actor rules allow this action right now” ✅

**The Golden Rule of System Design:**

* *Without PBAC:* ABAC alone cannot handle multi-step approvals, enterprise-wide governance, or audit/versioning.
* *Without ABAC:* RBAC alone cannot handle conditional, context-based access.
* *Without RBAC:* Every access decision would require evaluating full attribute sets for every user → inefficient and harder to reason about.

---

## ❓ Part 1: The Core Models (FAQ)

**Q1: If I had to choose only one model to start with, which one?**
**Answer: Start with RBAC (Role-Based Access Control).**
It aligns with how businesses work (teams, job titles). It is simple, easy to audit, and most identity providers (like Okta or Azure AD) already support it.

* **Rule:** Use RBAC for **broad access** (e.g., "Tellers can open the App").

**Q2: When does RBAC stop being sufficient?**
**Answer: When "Context" matters.**
RBAC fails when access depends on *conditions*, not just your job title. If you hear requirements like "Only during business hours," "Only transactions under $10k," or "Only from the office," RBAC is too rigid. Using roles for this leads to "Role Explosion" (too many specific roles to manage).

**Q3: When should I introduce ABAC?**
**Answer: When you need fine-grained control.**
Use ABAC (Attribute-Based Access Control) when decisions depend on **runtime details**:

* **User:** Region, Department.
* **Resource:** Data sensitivity, Owner.
* **Environment:** Time of day, Device, IP address.
* **Rule:** Use ABAC for **specific rules** (e.g., "Tellers can only see accounts *in their own branch*").

**Q4: Can RBAC and ABAC be used together?**
**Answer: Yes, and they should be.**
They work best as a team in a **layered architecture**:

1. **RBAC acts as the Gatekeeper:** Checks broad access at the entry point (API Gateway). *("Is this user a Teller? Yes. Let them in.")*
2. **ABAC acts as the Guard:** Checks specific data rules inside the application. *("Is this specific account in the Teller's region? No. Block access.")*

**Q5: Why not just use ABAC for everything and skip RBAC?**
**Answer: Because ABAC is expensive and complex.**
Checking attributes for every single request creates a heavy performance load and makes the system harder to audit. RBAC is a cheap, fast way to filter out unauthorized users before they even reach the expensive ABAC logic.

* **Analogy:** RBAC is the badge reader at the building entrance; ABAC is the security guard inside a specific secure room. You need both.

**Q6: Where does PBAC fit in?**
**Answer: PBAC governs the rules.**
While ABAC is the *logic*, PBAC (Policy-Based Access Control) is the *management*. It moves authorization rules out of the code and into a centralized **Policy Engine** (like OPA). This ensures rules are consistent across all microservices and makes compliance easy.

---

## ⚙️ Part 2: Operational & Runtime Logic

**Q7: What happens if the Central Policy Engine goes down?**
**Answer: You need a fallback strategy.**
If the application cannot reach the PBAC engine to get a decision, it must choose one of these paths:

* **Fail-Closed (Safest):** Deny all access. Use this for high-risk actions (e.g., money transfers).
* **Fail-Open (Risky):** Allow access if the risk is low (e.g., reading a public blog post).
* **Cached Decision:** Use the last known good decision (if it's recent).

**Q8: Where do OAuth2/OIDC Tokens fit in?**
**Answer: Tokens are for Identity, not Permissions.**
Tokens should tell you **"Who the user is"** (User ID, Group, Auth Context), not **"What they can do."**

* **Don't put permissions in tokens:** Tokens are static. If you change a permission, the user still holds the old token until it expires.
* **Do put stable attributes in tokens:** User ID, MFA status, Roles.

**Q9: How do we handle changes (like firing an employee) if they still have a valid token?**
**Answer: By checking authorization at runtime.**
Since tokens cannot be changed once issued, you cannot rely on them alone.

* **Critical Actions:** Always check the database or policy engine in real-time for critical changes (like account suspension) before allowing sensitive actions.
* **Token Revocation:** Shorten token lifespans so unauthorized access doesn't last long.

**Q10: Is MFA part of Authorization?**
**Answer: No, MFA is Authentication.**
MFA proves *who you are*. However, Authorization can **check** if MFA was used.

* **Example:** "You can log in with a password, but to transfer over $10,000 (Authorization Rule), we require that you used MFA during login (Authentication Context)."

---

## 🚨 Part 3: Summary & Common Mistakes

**Q11: What is the most common IAM design mistake?**
**Answer: Trying to put authorization complexity inside the Token.**
Developers often stuff permissions, limits, and rules into the JWT. This creates a "stale data" problem where you can't revoke access instantly.

* **Golden Rule:** Identity goes in the Token; Authorization happens at Runtime.

**Q12: How would you explain this entire model to an auditor?**
**Answer: The "Layered Defense" approach.**

1. **RBAC** limits *who* enters the system (Least Privilege).
2. **ABAC** limits *what* data they touch based on context (Data Security).
3. **PBAC** ensures all rules are central, versioned, and auditable (Compliance).

---

## 💼 System Design Interview FAQs (Do Not Skip)

**Q1: Can RBAC and ABAC be used together?**
**A:** Yes, and they almost always are in enterprise systems. RBAC is used for coarse-grained access such as application entry points, while ABAC is used for fine-grained decisions such as record-level or field-level access.

**Q2: Why not use ABAC everywhere and drop RBAC?**
**A:** RBAC is simpler, easier to audit, and aligns well with organizational models. Using ABAC for everything increases complexity unnecessarily and makes high-level access decisions harder to reason about.

**Q3: Where does PBAC fit in a microservices architecture?**
**A:** PBAC is typically implemented as a shared authorization service or policy engine that multiple services query, ensuring consistent authorization decisions across the system.

**Q4: How does this relate to MFA — is MFA AuthN or AuthZ?**
**A:** MFA is part of Authentication because it proves identity. However, Authorization policies can require that MFA was used, for example by denying access to sensitive operations unless the authentication context indicates MFA was performed.

**Q5: How would you explain this model to an auditor?**
**A:** RBAC determines who can access the system, ABAC determines what specific data they can access, and PBAC ensures that complex regulatory rules are enforced consistently and centrally.
