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
