# 📘 Section: Access Control Models (RBAC, ABAC, PBAC)

### Context: The *MoneyGuard* Banking Platform

In a small .NET application, authorization is often implemented using attributes like `[Authorize(Roles = "Admin")]`, which tightly couples authorization logic to static roles embedded directly in application code. While this approach works for small teams and simple products, it breaks down quickly in enterprise systems like MoneyGuard, where authorization decisions must account for regulatory constraints, geographic boundaries, data sensitivity, and evolving business rules.

In IAM system design interviews, this topic is not about syntax or frameworks, but about **how authorization models evolve as scale and complexity increase**, and how to combine them correctly instead of treating them as mutually exclusive choices.

This section explains the evolution from **RBAC** to **ABAC** to **PBAC**, using realistic enterprise constraints and .NET-based implementation patterns.

---

## 1️⃣ RBAC (Role-Based Access Control)

### “The Baseline Authorization Model”

RBAC is the foundational authorization model used in most enterprise systems because it aligns naturally with organizational structure. Instead of granting permissions directly to users, permissions are assigned to roles, and users inherit those permissions by being members of those roles.

The key idea behind RBAC is that **authorization is determined by job function**, not by situational context.

---

### Conceptual Model

In RBAC, authorization decisions follow a simple, deterministic chain:

* Alice is assigned the role **Teller**
* The **Teller** role has permission to view account balances
* Therefore, Alice is authorized to view account balances

This simplicity is why RBAC is often the **first layer** of authorization in an enterprise IAM system.

---

### MoneyGuard Example

MoneyGuard employs 1,000 users across multiple departments. Instead of managing permissions per user, the IAM team creates roles such as **Teller**, **BranchManager**, and **Auditor** in an identity provider like Azure AD or Okta. These roles are then mapped to application-level permissions, such as access to `AccountService` or `LoanApprovalService`.

When Alice joins MoneyGuard as a Teller, onboarding simply consists of adding her to the *Teller* role, instantly granting her all Teller-related permissions without requiring any code changes or redeployments.

---

### Typical .NET Mapping

```csharp
[Authorize(Roles = "Teller")]
public IActionResult ViewAccount(Guid accountId)
{
    ...
}
```

This works well for **coarse-grained authorization**, such as determining whether a user can access an application, API, or high-level feature.

---

### Limitation: Role Explosion

RBAC begins to fail when access decisions depend on **contextual differences** rather than job title alone.

For example:

* Tellers in the New York branch cannot view VIP accounts
* Tellers in London can approve loans up to $5,000
* Some Tellers can work remotely, others must be on the corporate network

Attempting to model these rules purely with RBAC leads to an explosion of narrowly defined roles such as `NY_Teller`, `London_Loan_Teller`, or `WFH_Teller`. Over time, this results in thousands of roles, unclear semantics, and authorization logic that is difficult to audit or reason about.

---

## 2️⃣ ABAC (Attribute-Based Access Control)

### “Authorization Based on Context and Data”

ABAC addresses the shortcomings of RBAC by making authorization decisions based on **runtime attributes**, rather than static role membership. Instead of asking only *who the user is*, ABAC asks *under what conditions access should be allowed*.

In ABAC, access is granted when a policy expression evaluating **user attributes**, **resource attributes**, and **environment attributes** evaluates to true.

---

### Attribute Categories (Interview-Critical)

* **Subject (User) Attributes:** Department, region, clearance level, employment type
* **Resource Attributes:** Data classification, owning region, sensitivity, monetary value
* **Environment Attributes:** Time of day, IP range, device posture, risk score

---

### MoneyGuard Example

Alice is a Teller attempting to view a VIP customer account while working from home. While RBAC alone would permit this action, MoneyGuard enforces stricter controls for sensitive data.

An ABAC rule might state:

> Allow access only if the user is a Teller, the resource is an Account, and the user is connected from the corporate office network.

At runtime, the system evaluates Alice’s role, the account type, and her network location. Because she is accessing the system from home, the authorization decision fails, and access is denied.

This rule eliminates the need for specialized roles like *WFH_Teller* and instead captures intent directly in policy logic.

---

### .NET Implementation Pattern

In enterprise .NET systems, ABAC logic is usually implemented via:

* Custom authorization handlers
* Middleware
* Database row-level filters

User attributes are typically stored as **claims in the ID token**, while resource attributes are retrieved from the database at request time.

---

## 3️⃣ PBAC (Policy-Based Access Control)

### “Authorization Logic as a Managed System”

PBAC builds on ABAC by externalizing authorization logic into a **centralized policy engine**, instead of embedding complex decision logic directly in application code.

While ABAC defines *what* attributes are evaluated, PBAC defines *where and how* authorization logic is authored, versioned, tested, and enforced.

---

### MoneyGuard Example

MoneyGuard has a regulatory rule stating that transactions above $10,000 require approval from two managers, but only if the transaction risk score is below a certain threshold and the request occurs during business hours.

Encoding this logic directly in C# would result in duplicated, brittle logic across services. Instead, MoneyGuard defines this rule in a centralized policy engine using policy-as-code and has services query the engine for authorization decisions.

---

### PBAC Flow

1. The MoneyGuard API sends user, resource, and environment attributes as a structured request.
2. The policy engine evaluates the request against stored policies.
3. The engine returns an allow/deny decision.
4. The application enforces the decision without embedding business logic.

---

## ⚔️ Comparison: Interview Framing

| Dimension       | RBAC             | ABAC               | PBAC                          |
| --------------- | ---------------- | ------------------ | ----------------------------- |
| Decision Driver | Static roles     | Runtime attributes | Centralized policy logic      |
| Strength        | Simplicity       | Flexibility        | Governance & auditability     |
| Weakness        | Role explosion   | Logic sprawl       | Operational complexity        |
| Best Use        | App-level access | Data-level access  | Regulatory & enterprise rules |

---

## ❓ System Design Interview FAQs (Do Not Skip)

### Q1: Can RBAC and ABAC be used together?

**A:** Yes, and they almost always are in enterprise systems. RBAC is used for coarse-grained access such as application entry points, while ABAC is used for fine-grained decisions such as record-level or field-level access.

---

### Q2: Why not use ABAC everywhere and drop RBAC?

**A:** RBAC is simpler, easier to audit, and aligns well with organizational models. Using ABAC for everything increases complexity unnecessarily and makes high-level access decisions harder to reason about.

---

### Q3: Where does PBAC fit in a microservices architecture?

**A:** PBAC is typically implemented as a shared authorization service or policy engine that multiple services query, ensuring consistent authorization decisions across the system.

---

### Q4: How does this relate to MFA — is MFA AuthN or AuthZ?

**A:** MFA is part of Authentication because it proves identity. However, Authorization policies can require that MFA was used, for example by denying access to sensitive operations unless the authentication context indicates MFA was performed.

---

### Q5: How would you explain this model to an auditor?

**A:** RBAC determines who can access the system, ABAC determines what specific data they can access, and PBAC ensures that complex regulatory rules are enforced consistently and centrally.

---

## 📘 Mapping RBAC, ABAC, and PBAC to OAuth2 / OpenID Connect (OIDC)

In modern distributed systems, OAuth2 and OpenID Connect (OIDC) are not only authentication protocols but also the primary mechanism by which **identity and authorization context is propagated across services**. Understanding how access control models such as RBAC, ABAC, and PBAC map onto OAuth2/OIDC tokens is critical to designing scalable and secure IAM systems.

A common source of confusion is attempting to encode *all* authorization logic into tokens. In practice, tokens should carry **stable identity context**, while authorization decisions that depend on data or environment must be evaluated at runtime.

This section explains what each model contributes, what belongs in tokens, and what must remain external.

---

### 1️⃣ RBAC and OAuth2 / OIDC Claims

#### Conceptual Mapping

RBAC maps most naturally to OAuth2 and OIDC because roles are typically **stable, low-cardinality attributes** that represent organizational intent rather than request-specific conditions. As a result, RBAC information is commonly embedded directly into tokens as claims.

RBAC answers the question:

> “What high-level capabilities does this identity have within the system?”

---

#### Common Claims Used for RBAC

RBAC is usually represented using claims such as:

* `roles`
* `groups`
* `scope` (particularly for service-to-service access)

Example token payload:

```json
{
  "sub": "user-123",
  "email": "alice@moneyguard.com",
  "roles": ["Teller"],
  "groups": ["Branch-NY"]
}
```

These claims can be evaluated locally by services without additional network calls, which makes RBAC efficient and easy to reason about.

---

#### Practical Usage

When a user authenticates, the Identity Provider issues a token containing role information. Each downstream service inspects the token and determines whether the role permits access to a given API or feature.

In a .NET application, this often results in role-based authorization checks such as:

```csharp
[Authorize(Roles = "Manager")]
public IActionResult ApproveLoan()
{
    ...
}
```

This approach works well for **coarse-grained authorization**, such as application access, feature toggling, or administrative capabilities.

---

#### Key Constraint

RBAC works best when roles remain few and semantically clear. As soon as roles are used to encode situational logic, the model begins to break down.

---

### 2️⃣ ABAC and OAuth2 / OIDC Claims

#### Why ABAC Cannot Be Fully Token-Based

ABAC relies on attributes that are often **dynamic, resource-specific, or environment-dependent**, which makes it impractical and unsafe to encode the full authorization decision into a token.

Instead, ABAC is implemented using a **split-responsibility model**:

* Tokens carry **user attributes**
* Applications fetch **resource attributes**
* Environment context is evaluated at request time

ABAC answers the question:

> “Given who the user is, what they are accessing, and under what conditions, should access be allowed?”

---

#### User Attributes in Tokens

OIDC tokens commonly include subject attributes such as:

```json
{
  "sub": "user-123",
  "role": "Teller",
  "department": "RetailBanking",
  "region": "US",
  "acr": "urn:mfa"
}
```

These claims describe the identity but do not by themselves determine authorization.

---

#### Runtime Attribute Evaluation

When a request is made, the application retrieves additional information such as:

* Resource ownership or region from the database
* Data classification or sensitivity
* Request context (network location, time, device)

The authorization decision is made by combining token claims with runtime attributes.

For example, even if a user’s role allows access in general, a rule may deny access when the user’s region does not match the data’s region.

---

#### Implementation Pattern

In practice, ABAC logic is implemented in:

* Authorization handlers
* Middleware layers
* Database query filters

rather than in controllers or tokens.

This ensures that access decisions reflect **current data and conditions**, not stale token contents.

---

### 3️⃣ PBAC and OAuth2 / OIDC

#### Separation of Identity and Policy Logic

PBAC extends ABAC by treating authorization logic as a **managed system component** rather than application logic. While OAuth2 and OIDC establish identity and authentication context, PBAC systems perform authorization decisions externally using centralized policies.

PBAC answers the question:

> “Is this action allowed right now, given all relevant context and business rules?”

---

#### Role of Tokens in PBAC

Tokens in a PBAC-based system typically include:

* The subject identifier (`sub`)
* High-level role or group information
* Authentication context (`acr`, `amr`)

Example:

```json
{
  "sub": "manager-456",
  "roles": ["Manager"],
  "acr": "urn:mfa"
}
```

Tokens identify *who* is calling the system, but they do not encode *whether* a particular action is permitted.

---

#### Runtime Policy Evaluation

When a protected operation is requested, the application sends a structured request to a policy engine containing:

* User attributes from the token
* Resource attributes from the data layer
* Environment attributes from the request context

The policy engine evaluates the applicable rules and returns an allow or deny decision, which the application then enforces.

This allows authorization rules to evolve independently of application code.

---

### ⚠️ Common Design Pitfalls

* Encoding fine-grained permissions directly into tokens
* Treating OAuth2 scopes as a replacement for ABAC
* Duplicating authorization logic across services
* Relying on long-lived tokens for sensitive access decisions

---

### ❓ Frequently Asked Questions

#### Q1: Can RBAC and ABAC be used together?

Yes. RBAC is typically used for coarse-grained authorization, such as determining which applications or APIs a user can access, while ABAC is used to enforce fine-grained, data-level rules within those boundaries.

---

#### Q2: Why not use ABAC for all authorization decisions?

While ABAC is more expressive, it introduces additional complexity and runtime dependency on data retrieval. RBAC remains valuable for simple, stable access decisions that do not require contextual evaluation.

---

#### Q3: How is MFA reflected in authorization decisions?

MFA is part of authentication, not authorization. However, the fact that MFA was performed is conveyed through authentication context claims such as `acr` or `amr`, which authorization logic can require for sensitive operations.

---

#### Q4: How are attribute changes handled after a token is issued?

Critical attributes are evaluated at runtime rather than relying solely on token claims. Tokens are typically short-lived, and changes to sensitive attributes may trigger token revocation or session invalidation.

---

#### Q5: What happens if a centralized policy engine is unavailable?

Systems must explicitly define fail-open or fail-closed behavior based on the sensitivity of the operation. In many designs, critical authorization decisions fail closed, while lower-risk decisions may rely on cached results.

---

#### Core Mental Model (Summary)

> OAuth2 and OIDC establish identity and authentication context, RBAC governs high-level access, ABAC enforces data- and context-aware rules at runtime, and PBAC centralizes complex authorization logic so that it can evolve independently of application code.
