# 🔄 Day 4: Directory Sync & Identity Lifecycle (SCIM)

**Topic:** Automating onboarding and offboarding for enterprise customers at scale.

If Authentication (AuthN) is checking the ID badge at the door, and Authorization (AuthZ) is the card reader on the internal doors, **Directory Sync** is the automated HR process that prints the badge when you are hired and instantly burns it the second you are fired.

Managing users in a B2B SaaS application is notoriously difficult. If you rely on manual processes or basic login flows, you will inevitably create security vulnerabilities (like terminated employees retaining access) and administrative nightmares for your customers.

---

### Phase 1: The Evolution of Provisioning

To understand *why* we build SCIM, we have to look at how enterprise identity synchronization evolved, and exactly where early methods fail.

#### 1. The Manual Era (The "CSV Upload")

In the early days, if an enterprise bought your software for 500 employees, their IT admin downloaded a CSV from HR and uploaded it to your dashboard.

* **The Failure:** When 5 people quit the next week, the IT admin forgets to delete them from your app. You now have active "Zombie Accounts" that fail SOC2 compliance.

#### 2. Just-In-Time (JIT) Provisioning (The SSO Fallback)

To fix manual uploads, the industry adopted JIT. When Alice is hired, she is added to her company's Azure AD. She clicks "Login" on your app, Azure AD sends a SAML/OIDC token, and your `.NET API` creates her database record *on the fly*.

**The Breaking Point of JIT:**
JIT is a "pull" mechanism. It only works when the user acts. It fails in Enterprise SaaS for two massive reasons:

1. **The Ghost User:** If an Admin wants to assign Alice to a specialized "GPU Workspace" today, but Alice hasn't logged in yet, the Admin can't find her in your system. She is a ghost.
2. **The Deprovisioning Gap:** If Alice is fired on Friday, Azure AD deletes her. But your application *doesn't know that*. If Alice has a persistent 30-day session token or a hardcoded API key, she can continue downloading sensitive data until that session naturally expires.

---

### Phase 2: The SCIM Architecture (System for Cross-domain Identity Management)

SCIM (RFC 7644) solves the JIT problem by moving from a "Pull" to a **"Push"** model.

SCIM is a standardized REST API that you build into your `.NET` backend. You give the enterprise customer a secret Bearer Token and your API URL (`https://api.yoursaas.com/scim/v2`).

Now, Azure AD or Okta acts as the puppet master. The millisecond an HR event happens, the Identity Provider (IdP) pushes an HTTP request directly to your API.

#### The Core SCIM Endpoints

To be SCIM compliant, your API must implement specific routes with exact JSON schemas:

* `POST /scim/v2/Users` (Create a user)
* `PUT /scim/v2/Users/{id}` (Completely replace a user)
* `PATCH /scim/v2/Users/{id}` (Update specific fields or disable a user)
* `GET /scim/v2/Users` (Search and list users)
* *(Repeat the same for `/Groups`)*

---

### Phase 3: The C# Implementation & Pre-Provisioning

Let's look at the "Hyperscaler" use case. A massive tech company signs a contract and needs to provision 500 AI engineers into Lambda Workspaces before they even log in.

Azure AD wakes up and fires 500 `POST` requests to your API.

#### 1. The Standard SCIM Payload

Azure AD sends a strictly formatted JSON payload. Notice the `externalId`—this is the user's immutable ID in Azure AD (like their Object ID). **This is critical.** If Alice changes her name or email, this ID never changes, allowing you to link the records permanently.

```json
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "userName": "alice@hyperscaler.com",
  "name": {
    "familyName": "Smith",
    "givenName": "Alice"
  },
  "active": true,
  "externalId": "aad-user-object-12345"
}

```

#### 2. The .NET SCIM Controller

Here is how you handle that request, mapping the enterprise data to your internal SaaS database.

```csharp
[Authorize(Policy = "ScimSyncPolicy")] // Must authenticate via the customer's secret SCIM token
[ApiController]
[Route("scim/v2/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserRepository _userRepo;

    [HttpPost]
    public async Task<IActionResult> CreateUser([FromBody] ScimUserDto scimUser)
    {
        // 1. BEST PRACTICE: Idempotency Check using ExternalId
        var existingUser = await _userRepo.FindByExternalIdAsync(scimUser.ExternalId);
        if (existingUser != null)
        {
            return Conflict(new { detail = "User already exists with this ExternalId" });
        }

        // 2. Map SCIM schema to your internal SaaS Database Model
        var internalUser = new InternalSaaSUser
        {
            Id = Guid.NewGuid().ToString(),
            Email = scimUser.UserName,
            FirstName = scimUser.Name.GivenName,
            LastName = scimUser.Name.FamilyName,
            IsActive = scimUser.Active,
            ExternalId = scimUser.ExternalId, // Anchor them together
            TenantId = User.FindFirst("tenant_id")?.Value // Extracted from the SCIM Auth Token
        };

        await _userRepo.InsertAsync(internalUser);

        // 3. SCIM requires you to return the created object with your internal ID
        scimUser.Id = internalUser.Id; 
        return Created(new Uri($"/scim/v2/Users/{internalUser.Id}", UriKind.Relative), scimUser);
    }
}

```

**The Result:** Alice is now in your database. The Admin can assign her to the GPU workspace. When Alice logs in on Monday, everything is waiting for her.

---

### Phase 4: The "Kill Switch" (Deprovisioning via `PATCH`)

This is where SCIM pays for itself in security. Alice is terminated. Azure AD instantly fires a `PATCH` request to your API to disable her account.

#### 1. The SCIM Patch Payload

SCIM uses a specific "PatchOp" schema. It tells your API exactly which field to change.

```json
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [
    {
      "op": "replace",
      "path": "active",
      "value": false
    }
  ]
}

```

#### 2. The .NET Implementation (Integrating with Day 3)

When we receive this `active: false` command, we don't just update the database. We must trigger the **Redis Pub/Sub Kill Switch** we built in Day 3 to destroy her active sessions immediately.

```csharp
[HttpPatch("{id}")]
public async Task<IActionResult> PatchUser(string id, [FromBody] ScimPatchDto patchRequest)
{
    var internalUser = await _userRepo.FindByIdAsync(id);
    if (internalUser == null) return NotFound();

    // 1. Parse the SCIM Operations
    foreach (var operation in patchRequest.Operations)
    {
        if (operation.Op.ToLower() == "replace" && operation.Path.ToLower() == "active")
        {
            bool isActive = bool.Parse(operation.Value.ToString());
            internalUser.IsActive = isActive;

            // 2. REAL-WORLD SECURITY: The Kill Switch
            if (!isActive)
            {
                // Instantly broadcast to all sidecars to drop her tokens!
                await _redisRevocationService.RevokeUserAccessAsync(internalUser.Id);
                
                // Optional: Fire event to spin down her expensive AWS GPU instances
                await _infrastructureQueue.QueueDownscaleAsync(internalUser.WorkspaceId);
            }
        }
    }

    await _userRepo.UpdateAsync(internalUser);
    return Ok(internalUser); // SCIM expects a 200 OK with the updated user
}

```

---

### Phase 5: Solving the Core SCIM Race Condition

Because SCIM and SSO are two separate internet systems, architects must solve the **Race Condition**. Azure AD typically runs its SCIM sync cycle in batches every 40 minutes.

**The Problem:**
An IT Admin adds Bob to Azure AD, and Bob logs into your app 30 seconds later via SSO.
The SCIM `POST /Users` hasn't happened yet. Bob arrives via SSO, and your system says "Who is this?"

**The Architectural Fix (Upsert Logic):**
Your application must gracefully handle both flows colliding.

1. **If SSO comes first (The JIT Fallback):** When Bob logs in, your SSO handler executes a JIT creation. It creates Bob in the database, sets his `ExternalId` from the SAML/OIDC token claims, and logs him in.
2. **If SCIM comes later (The Linking):** 39 minutes later, the SCIM sync finally arrives with a `POST /Users` for Bob. Your `.NET API` looks up the `ExternalId`. Instead of throwing a 500 Error ("Email already taken"), it executes an **Upsert**. It realizes the user exists, updates any missing fields (like Title or Department), and returns a `200 OK` to Azure AD.

---

### 🏛️ Whiteboard FAQ: Defending the Identity Lifecycle

**Q: Why do we need the `/Groups` endpoint if the `/Users` endpoint already sends their department?**

> **A:** A user's department is a static string. A Group is a dynamic security entity. If the customer creates an Azure AD group called `Beta_Testers`, they want to drop 50 users into it, and then instantly remove 20 users next week. The `/Groups` endpoint sends explicit `members: [{id: 123}]` arrays, allowing your SaaS app to instantly sync bulk access to specific internal roles (which ties perfectly into our ReBAC SpiceDB engine from Day 3).

**Q: How do we secure the SCIM endpoint itself?**

> **A:** You generate a long-lived, cryptographically secure Bearer Token unique to that specific enterprise tenant (e.g., `Tenant_A_Scim_Key`). When Azure AD makes requests, it passes this token. Your API middleware validates the token, extracts `Tenant_A`, and ensures the SCIM sync only creates/modifies users within Tenant A's database shard.

**Q: What happens if Azure AD sends a SCIM request, but our API is down?**

> **A:** SCIM is designed for eventual consistency. If your `.NET API` returns a 500 error or is unreachable, Azure AD places that SCIM event into a retry queue. It will back off and try again later, ensuring that the directory eventually reaches total synchronization once your servers are back online.

---

### 📝 Day 4 Cheat Sheet: Directory Sync & SCIM

1. **JIT Provisioning:** Creates users on the fly during SSO login. Great for simple onboarding, but terrible for advanced pre-configuration.
2. **The Ghost User Problem:** JIT cannot assign resources to a user who hasn't logged in yet because they don't exist in your database.
3. **The Deprovisioning Gap:** JIT cannot revoke active sessions when an employee is fired because it relies on the user initiating a login.
4. **SCIM (RFC 7644):** An open REST API standard that allows Identity Providers (Azure AD/Okta) to **push** changes to your app in real-time.
5. **Pre-Provisioning (`POST /Users`):** Creates the user profile instantly, allowing admins to map them to projects before Day 1.
6. **The Kill Switch (`PATCH /Users`):** Replaces `active: true` with `active: false`. Your system must react by terminating live sessions (e.g., via Redis Pub/Sub).
7. **Group Sync (`POST /Groups`):** Syncs massive internal security groups directly to your SaaS application's internal ReBAC/PBAC roles.
8. **The `externalId`:** The immutable anchor that ties your database record to the customer's Azure AD record. Never rely solely on email addresses.
9. **The Race Condition:** Always implement "Upsert" logic. If SSO beats SCIM, create the user. When SCIM arrives later, map the `externalId` to the existing user.
10. **Idempotency:** SCIM endpoints will often receive duplicate requests. Your `.NET` controllers must handle duplicates gracefully without crashing or creating duplicate database rows.

---

