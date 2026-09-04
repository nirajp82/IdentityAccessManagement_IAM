# JIT Provisioning vs SCIM

## Overview

Module 01 established the **Joiner–Mover–Leaver (JML)** lifecycle: people are hired, they change roles, and they leave, and access must follow each event. This chapter assumes you know that story — see [Module 01, Chapter 5](../01_Core_IAM_Concepts/05_IAM_Lifecycle_JML.md) if you need it — and asks the engineering question that follows: **what mechanism actually carries a JML event from the HR system into every downstream application?**

There are two answers. **JIT (Just-In-Time) provisioning** creates the user account the first time they log in. **SCIM (System for Cross-domain Identity Management)** pushes the change from the identity provider to the application in the background, whether the user logs in or not. They fail in completely different ways, and the correct production answer uses both.

🏦 **The MoneyGuard scenario for this chapter:** MoneyGuard has just signed **Acme Corp** onto the **Business Banking SaaS Portal**. Acme's IT admin needs to onboard 500 finance staff and pre-assign them to payroll workspaces **before their first day** — and, far more importantly, needs terminated Acme employees to lose portal access the moment HR marks them a leaver, without anyone at MoneyGuard being told. JIT can do neither.

## Why This Matters

- **Deprovisioning is where breaches live.** A fired employee who never logs in again will never trigger a JIT update, so a JIT-only application never learns they left. That account stays active forever.
- **"Zombie accounts" are an audit failure.** Dana Cole cannot sign off SOX controls if terminated staff still hold active records in MoneyGuard systems.
- **Enterprise buyers demand SCIM.** For a B2B product like the Business Banking SaaS Portal, "do you support SCIM?" is a procurement gate, not a nice-to-have.
- **Day-one experience is a product feature.** A new hire who logs in and finds nothing configured is a bad first impression; one whose workspace is already set up is a good one.

## Key Concepts

| Term | Expansion | Meaning |
|---|---|---|
| **Provisioning** | — | Creating and configuring an account in a downstream application |
| **Deprovisioning** | — | Disabling or deleting that account when access should end |
| **JIT** | Just-In-Time provisioning | Creating the account on first successful SSO login |
| **SCIM** | System for Cross-domain Identity Management | A REST + JSON standard for pushing identity changes between systems |
| **IdP** | Identity Provider | The system that authenticates users, e.g. MoneyGuard Entra ID |
| **System of record** | — | The authoritative source for a fact; Workday for employment |
| **Push vs pull** | — | Whether the IdP sends changes, or the app discovers them on login |
| **Pre-provisioning** | — | Creating an account before the user has ever logged in |
| **Ghost user** | — | Someone who exists in the IdP but not yet in the application |
| **Zombie account** | — | An account still active in the application after the person has left |
| **`externalId`** | — | The immutable IdP identifier stored by the application to anchor the link |

## Detailed Notes

### Concept 1 — Where provisioning came from: the CSV era

Before either mechanism existed, onboarding worked like this: Acme Corp's IT admin exported a spreadsheet from HR and uploaded it to MoneyGuard's portal.

⚠️ It fails in exactly one predictable way. When five Acme employees resign the following week, the admin forgets to remove them. Those accounts remain fully active. There is no event, no reminder, and nothing in MoneyGuard's systems that knows anything changed. Multiply by every customer and every month, and the portal accumulates **zombie accounts** indefinitely.

That is the failure both JIT and SCIM exist to solve — and only one of them actually solves it.

### Concept 2 — JIT provisioning: create on first login

**JIT** rides the SSO login itself. When an Acme employee clicks "Sign in with Microsoft" for the first time:

1. Acme's identity provider authenticates them and returns a signed SAML assertion or OIDC ID token.
2. That token carries the identity attributes MoneyGuard needs: `sub`, `email`, `given_name`, `family_name`, and often `groups`.
3. MoneyGuard's portal looks the user up, finds nothing, and reasons: *"I've never seen this person, but Acme's IdP cryptographically vouches for them."*
4. It creates the database record **on the fly** and logs them in.

```json
// The OIDC ID token that triggers JIT
{
  "iss": "https://login.microsoftonline.com/acme-corp/v2.0",
  "sub": "aad-user-object-12345",
  "email": "j.rivera@acmecorp.com",
  "given_name": "Jordan",
  "family_name": "Rivera",
  "groups": ["Acme-Finance", "Acme-Payroll-Approvers"],
  "exp": 1772539200
}
```

JIT is genuinely useful. It requires no configuration from the customer, works with any SAML or OIDC provider, and gets a user productive in one click. For a self-serve product it may be all you need.

### Concept 3 — Why JIT breaks in the enterprise

JIT is a **pull** mechanism: it only ever fires when the user actively logs in. That single property causes three failures.

#### Failure 1 — The ghost user (onboarding)

Acme's admin wants to assign Jordan Rivera to the Payroll workspace on Friday so everything is ready for Monday. She searches MoneyGuard's admin panel and Jordan is not there — because Jordan has never logged in, so no record exists. Jordan is a **ghost user**. Pre-configuration is impossible; the admin must wait for Jordan to log in first, then go back and configure things afterwards, which inverts the natural workflow.

#### Failure 2 — The deprovisioning gap (offboarding) ⚠️

This is the serious one. **JIT knows how to create users. It has no mechanism to remove them.**

Jordan is terminated on Friday and removed from Acme's Entra ID. But a terminated employee will never log in again — so the event that JIT depends on **never happens**. MoneyGuard's portal remains completely blind to the termination. Jordan's account stays `active: true` forever.

#### Failure 3 — Zombie accounts and back doors

"But they can't log in — SSO is blocked," is the usual objection. It is not sufficient:

- **API keys.** If Jordan generated a personal access token for a payroll integration, that key keeps working. It does not go through SSO.
- **Long-lived sessions.** A 30-day session cookie on a personal phone survives long after the SSO front door is closed.
- **Direct credentials.** If the portal ever allowed a local password as a fallback, that path is untouched by the IdP.
- **Audit.** Even with every back door closed, Dana still fails the control: the record says the person is an active user of a banking system, and MoneyGuard cannot prove otherwise.

The SSO front door being locked does not mean the building is secure.

### Concept 4 — SCIM: push the change, no login required

**SCIM (System for Cross-domain Identity Management)** inverts the direction. Instead of the application discovering changes when a user shows up, the **identity provider pushes them** the moment they happen.

Once Acme's admin connects their Entra ID to MoneyGuard's SCIM endpoint, Acme's IdP becomes the puppet master:

- **Hire** → `POST /scim/v2/Users` → MoneyGuard creates a dormant, pre-configured account.
- **Role change** → `PATCH /scim/v2/Users/{id}` or `PATCH /scim/v2/Groups/{id}` → entitlements update.
- **Termination** → `PATCH /scim/v2/Users/{id}` with `active: false` → MoneyGuard disables the account **and kills live sessions**.

Crucially, **none of this requires the user to do anything.** That is the entire point, and it is what fixes both the ghost user and the deprovisioning gap.

### Concept 5 — SCIM is a universal translator

Every identity provider and every SaaS application models users differently. If Acme's Okta calls a field `department_name` and MoneyGuard's portal calls it `business_unit`, they cannot talk.

SCIM solves this by dictating a **strict, non-negotiable REST route structure and JSON schema**. By implementing that exact standard once, MoneyGuard's portal can be driven by Entra ID, Okta, Ping, OneLogin or anything else — with no per-customer integration work. That is why enterprise buyers ask for it by name: it is the difference between one integration and one integration *per customer*.

### Concept 6 — Setting it up: the handshake

When Acme Corp buys a licence, MoneyGuard hands their IT admin exactly two things:

1. **The SCIM base URL** — a dedicated endpoint, e.g. `https://api.moneyguard.com/scim/v2`.
2. **A long-lived bearer token** — a cryptographically strong secret generated *specifically for Acme Corp*.

That token does double duty. It authenticates the caller, and it identifies the **tenant** — so every user created through it lands in Acme Corp's database partition and can never touch Globex Inc's. ⚠️ A shared SCIM token across tenants is a cross-tenant data leak waiting to happen.

The Acme admin pastes those two values into their Entra ID provisioning configuration, and the backend-to-backend pipeline is live.

### Concept 7 — The head-to-head comparison

| | **JIT Provisioning** | **SCIM Provisioning** |
|---|---|---|
| **Direction** | Pull — triggered by the user logging in | Push — triggered by an HR/IdP event |
| **Trigger** | First successful SSO login | Hire, role change, or termination in the IdP |
| **Onboarding** | Account exists only after first login | **Pre-provisioned** before day one |
| **Pre-assign licences/workspaces?** | ❌ No — the user is a ghost until they log in | ✅ Yes |
| **Deprovisioning** | ❌ **None.** No login means no signal | ✅ `PATCH active: false` pushed automatically |
| **Group / entitlement sync** | Only what happens to be in the login token | ✅ Full `/Groups` lifecycle, incremental |
| **Customer setup effort** | None | Base URL + bearer token + IdP configuration |
| **Latency** | Instant (at login) | Batched — Entra ID typically syncs every ~40 minutes |
| **Works if the app is down?** | No | ✅ Yes — the IdP retries with backoff |
| **Zombie account risk** | ⚠️ High | Low |
| **Right for** | Self-serve products, a safety net | Enterprise B2B, regulated industries |

💡 Read the latency row carefully: SCIM is *more reliable* but *slower* than JIT. That asymmetry is precisely what creates the race condition in [Chapter 3](./03_Handling_Sync_Race_Conditions.md).

### Concept 8 — The right answer is both

The two mechanisms have complementary failure modes, so production systems run them together:

- **SCIM is the primary channel.** It handles pre-provisioning, group sync, and — most importantly — deprovisioning.
- **JIT is the safety net.** If a user logs in before the SCIM batch has arrived (or before the customer has finished configuring SCIM at all), JIT creates them on the spot so they are not locked out.
- **Upsert logic reconciles them.** When SCIM later arrives for a user JIT already created, the application links the records via `externalId` rather than throwing a duplicate error.

🏦 At MoneyGuard, this means Jordan Rivera can start work at 09:02 even though the SCIM sync isn't scheduled until 09:40 — and Jordan's access still disappears automatically the day they leave. Neither mechanism alone delivers both.

### Concept 9 — Deprovisioning must do more than flip a flag

When the `active: false` arrives, setting a database column is only step one. A terminated user may hold live credentials that a database flag does not touch:

1. **Disable the account** in the application database.
2. **Trigger the kill switch** — revoke live sessions and tokens ([Module 04, Chapter 3](../04_Edge_Security_Tokens_And_Sessions/03_Token_Revocation_And_Session_Management.md)).
3. **Revoke API keys** and personal access tokens they generated.
4. **Delete relationship tuples** so the ReBAC graph no longer grants anything ([Module 03, Chapter 2](../03_Authorization_And_Policy_Engines/02_ReBAC_And_Google_Zanzibar.md)).
5. **Evict cached authorization decisions** via the Pub/Sub kill switch ([Module 03, Chapter 3](../03_Authorization_And_Policy_Engines/03_Policy_Engines_OPA_And_PBAC.md)).
6. **Cancel running jobs** they own — long-running exports, scheduled reports.
7. **Write the audit record** so Dana can prove exactly when access ended.

⚠️ Skipping step 2 is the single most common deprovisioning bug: the account is disabled but the person's existing access token remains valid for another 55 minutes.

## Workflow

```mermaid
flowchart TD
    HR["Acme Corp HR (Workday):<br/>Jordan Rivera terminated"] --> IDP["Acme Entra ID:<br/>account disabled"]

    IDP --> Q{"Does MoneyGuard's portal<br/>have SCIM configured?"}

    Q -- "No — JIT only" --> J1["Nothing happens.<br/>No login means no signal."]
    J1 --> J2["⚠️ Account stays active:true forever"]
    J2 --> J3["API keys still work<br/>Session cookies still valid<br/>SOX control fails"]

    Q -- "Yes — SCIM" --> S1["IdP pushes PATCH /scim/v2/Users/{id}<br/>{ active: false }"]
    S1 --> S2["Portal sets IsActive = false"]
    S2 --> S3["Trigger kill switch:<br/>revoke sessions + tokens"]
    S3 --> S4["Revoke API keys<br/>Delete ReBAC tuples<br/>Cancel running jobs"]
    S4 --> S5["Write audit record"]
    S5 --> S6["200 OK back to the IdP"]
```

## Architecture Diagram

```mermaid
flowchart LR
    subgraph AcmeCorp["Acme Corp (MoneyGuard's customer)"]
        WD["Workday<br/>HR system of record"]
        AAD["Acme Entra ID<br/>Identity Provider"]
        WD -->|"polls HR changes (~40 min)"| AAD
    end

    subgraph MG["MoneyGuard Business Banking SaaS Portal"]
        SCIMAPI["SCIM API<br/>/scim/v2/Users, /Groups"]
        SSO["SSO endpoint<br/>(SAML / OIDC)"]
        DB[("Acme Corp tenant partition")]
        KILL["Revocation service<br/>sessions, tokens, API keys"]
        SPICE[("SpiceDB relationship graph")]
        AUDIT["Audit log → SIEM"]
    end

    USER["Jordan Rivera<br/>(Acme employee)"]

    AAD -->|"PUSH: SCIM over HTTPS<br/>Bearer Acme_Scim_Key"| SCIMAPI
    USER -->|"PULL: SSO login (JIT fallback)"| SSO
    SCIMAPI --> DB
    SSO --> DB
    SCIMAPI --> KILL
    SCIMAPI --> SPICE
    SCIMAPI --> AUDIT
    KILL --> AUDIT
```

## Step-by-Step Process

1. **Confirm the system of record.** For employment facts it is Workday, not the IdP and certainly not MoneyGuard's database.
2. **Implement SSO first.** SCIM synchronises accounts; it does not authenticate anyone. You need SAML or OIDC working before SCIM is useful.
3. **Build the SCIM `/Users` endpoints** — `POST`, `GET`, `PUT`, `PATCH` — to the exact standard schema ([Chapter 2](./02_SCIM_Protocol_Deep_Dive.md)).
4. **Build the `/Groups` endpoints**, because group membership is how customers actually assign entitlements.
5. **Generate a per-tenant bearer token.** One secret per customer, stored hashed, revocable independently.
6. **Store `externalId` on every user record** and treat it as the permanent link — never the email address.
7. **Make every handler idempotent.** Identity providers retry, and duplicate requests must not create duplicate rows.
8. **Keep JIT enabled as a fallback**, with upsert logic so a later SCIM `POST` links to the existing record instead of failing.
9. **Wire `active: false` into the full revocation chain** — database, sessions, tokens, API keys, ReBAC tuples, cached decisions, running jobs.
10. **Return correct SCIM status codes.** `201` on create, `200`/`204` on update, `409` on conflict, `404` on unknown. Providers change behaviour based on these.
11. **Log every SCIM operation** with tenant, operation, target and outcome — this is Dana's offboarding evidence.
12. **Test with a real IdP.** Entra ID, Okta and Ping each have quirks; a mock will not find them.

## Commands and Examples

**1. The SCIM user creation payload Entra ID sends (a joiner):**

```json
POST /scim/v2/Users HTTP/1.1
Host: api.moneyguard.com
Authorization: Bearer AcmeCorp_Scim_Key_9f2b7c41...
Content-Type: application/scim+json

{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "userName": "j.rivera@acmecorp.com",
  "name": {
    "givenName": "Jordan",
    "familyName": "Rivera"
  },
  "emails": [
    { "value": "j.rivera@acmecorp.com", "type": "work", "primary": true }
  ],
  "active": true,
  "externalId": "aad-user-object-12345"
}
```

⚠️ `externalId` is the anchor. It is Acme's immutable object ID for Jordan. Email addresses change — people marry, companies rebrand domains — but this identifier never does. Match on it, always.

**2. The termination payload (a leaver):**

```json
PATCH /scim/v2/Users/mg-usr-77f21 HTTP/1.1
Authorization: Bearer AcmeCorp_Scim_Key_9f2b7c41...
Content-Type: application/scim+json

{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [
    { "op": "replace", "path": "active", "value": false }
  ]
}
```

**3. Handling a joiner, idempotently:**

```csharp
[Authorize(Policy = "ScimSyncPolicy")]   // validates the per-tenant bearer token
[ApiController]
[Route("scim/v2/Users")]
public class ScimUsersController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> CreateUser([FromBody] ScimUserDto scimUser)
    {
        var tenantId = User.FindFirst("tenant_id")!.Value;   // from the SCIM token

        // 1. IDEMPOTENCY: the IdP may send this twice. Don't crash, don't duplicate.
        var existing = await _users.FindByExternalIdAsync(tenantId, scimUser.ExternalId);
        if (existing is not null)
            return Conflict(new ScimError
            {
                Schemas = new[] { "urn:ietf:params:scim:api:messages:2.0:Error" },
                Detail  = "User already exists with this externalId",
                Status  = "409"
            });

        // 2. Map the SCIM schema to MoneyGuard's internal model
        var user = new PortalUser
        {
            Id         = Guid.NewGuid().ToString(),
            Email      = scimUser.UserName,
            FirstName  = scimUser.Name.GivenName,
            LastName   = scimUser.Name.FamilyName,
            IsActive   = scimUser.Active,
            ExternalId = scimUser.ExternalId,   // the permanent anchor
            TenantId   = tenantId               // ⚠️ never from the request body
        };

        await _users.InsertAsync(user);

        // 3. SCIM requires the created object back, carrying YOUR id
        scimUser.Id = user.Id;
        return Created($"/scim/v2/Users/{user.Id}", scimUser);
    }
}
```

💡 `TenantId` comes from the authenticated SCIM token, never from the request body. If a caller could specify the tenant, Acme's token could write into Globex's partition.

**4. Handling a leaver — the full kill chain:**

```csharp
[HttpPatch("{id}")]
public async Task<IActionResult> PatchUser(string id, [FromBody] ScimPatchDto patch)
{
    var tenantId = User.FindFirst("tenant_id")!.Value;
    var user = await _users.FindByIdAsync(tenantId, id);
    if (user is null) return NotFound();

    foreach (var op in patch.Operations)
    {
        if (op.Op.Equals("replace", StringComparison.OrdinalIgnoreCase) &&
            op.Path?.Equals("active", StringComparison.OrdinalIgnoreCase) == true)
        {
            var isActive = op.Value.GetBoolean();
            user.IsActive = isActive;

            if (!isActive)
            {
                // Disabling the row is NOT enough. Kill everything they hold.
                await _revocation.RevokeUserAsync(user.Id, "scim_deprovision",
                                                  TimeSpan.FromMinutes(15));
                await _apiKeys.RevokeAllForUserAsync(user.Id);
                await _spiceDb.DeleteAllRelationshipsForSubjectAsync("user", user.Id);
                await _jobs.CancelActiveJobsAsync(user.Id);
                _audit.Write("scim.deprovision", tenantId, user.Id);
            }
        }
    }

    await _users.UpdateAsync(user);
    return Ok(ToScim(user));   // SCIM expects 200 with the updated resource
}
```

**5. JIT as a fallback, with upsert:**

```csharp
public async Task<PortalUser> ResolveOnLoginAsync(ClaimsPrincipal idToken, string tenantId)
{
    var externalId = idToken.FindFirst("sub")!.Value;

    var user = await _users.FindByExternalIdAsync(tenantId, externalId);
    if (user is not null) return user;                    // normal path: SCIM got here first

    // JIT FALLBACK: SCIM hasn't arrived yet. Create from the token so the user isn't blocked.
    return await _users.InsertAsync(new PortalUser
    {
        Id         = Guid.NewGuid().ToString(),
        Email      = idToken.FindFirst("email")!.Value,
        FirstName  = idToken.FindFirst("given_name")?.Value ?? "",
        LastName   = idToken.FindFirst("family_name")?.Value ?? "",
        ExternalId = externalId,      // ⚠️ set it now so SCIM can link later, not duplicate
        TenantId   = tenantId,
        IsActive   = true,
        Source     = "JIT"            // so you can measure how often SCIM loses the race
    });
}
```

## Best Practices

- **Run SCIM as primary and JIT as a fallback.** SCIM alone locks out eager users; JIT alone never deprovisions.
- **Anchor on `externalId`, never on email.** Email changes; the IdP object ID does not.
- **Derive the tenant from the SCIM token**, never from the payload.
- **Make every operation idempotent.** Providers retry aggressively; a duplicate `POST` must be a no-op, not a duplicate row.
- **Treat `active: false` as a security event**, not a database update — trigger the whole revocation chain.
- **Prefer soft-disable over hard delete.** `active: false` preserves the audit trail; `DELETE` destroys the evidence Dana needs.
- **Issue one SCIM token per customer**, stored hashed, individually revocable.
- **Return standards-compliant status codes and error bodies** — identity providers branch on them.
- **Log every SCIM call** with tenant, operation, target, result and correlation ID.
- **Record the provisioning source** (`SCIM` or `JIT`) on each record, so you can measure how often the race is being lost.

## Common Mistakes

- ⚠️ **Shipping JIT and calling provisioning "done."** You have onboarding with no offboarding, which is the more dangerous half.
- ⚠️ **Matching users by email address.** When Jordan Rivera's surname changes and IT updates the address, you create a duplicate and orphan the original.
- ⚠️ **Reading `tenantId` from the request body.** A cross-tenant write primitive handed to any customer who reads your API docs.
- ⚠️ **Setting `IsActive = false` and stopping there.** The session, API keys and cached authorization decisions all survive.
- ⚠️ **Hard-deleting on deprovision.** You lose the record that proves the person ever had access — and any audit trail attached to it.
- ⚠️ **Returning `500` on a duplicate `POST`.** Entra ID will retry it forever. Return `409` with a proper SCIM error body.
- ⚠️ **Assuming SCIM is real-time.** Entra ID batches on roughly a 40-minute cycle. Designing as though it is instant creates the race condition in Chapter 3.
- ⚠️ **Sharing one SCIM bearer token across customers.** One leak compromises every tenant simultaneously.
- ⚠️ **Ignoring `/Groups`.** Customers assign entitlements via groups; without it, every licence change is a manual ticket.
- ⚠️ **Trusting the `groups` claim in the login token as authoritative.** It reflects the moment of login only, and large group lists get truncated by most providers.

## Pro Tips

- 💡 **The one-sentence distinction:** *"JIT is a pull triggered by login; SCIM is a push triggered by an HR event — which is why only SCIM can offboard."*
- 💡 **Measure the JIT fallback rate.** If a large share of users are created by JIT rather than SCIM, either customers haven't configured SCIM or your sync is too slow — both are worth knowing.
- 💡 **Make deprovisioning a single function** called by SCIM, the admin console and the SOC kill switch alike. Divergent implementations are where gaps appear.
- 💡 **Sell SCIM as a security feature, not a convenience.** "Your terminated employees lose access automatically" closes enterprise deals.
- 💡 **Keep a reconciliation sweep.** A nightly job comparing the IdP's user list to yours catches anything the push channel dropped.
- 💡 **Instrument time-to-deprovision** from IdP event to full revocation. It is the number an auditor will ask for.
- 💡 **Interview framing:** name the two failure modes by their names — *ghost user* for onboarding, *deprovisioning gap* for offboarding. It shows you have shipped this.

## Real-World Use Cases

1. **Acme Corp bulk onboarding.** 500 `POST /Users` calls arrive the moment SCIM is connected, creating dormant accounts. Acme's admin pre-assigns payroll workspaces on Friday; everyone is productive on Monday.
2. **Automatic offboarding.** Jordan Rivera is terminated in Workday at 16:30. By 17:10 Entra ID has pushed `active: false`, MoneyGuard has disabled the account, killed the session, revoked two API keys and deleted six ReBAC tuples.
3. **Contractor expiry.** Sam Torres's 90-day engagement ends. Workday drives the change; nobody at MoneyGuard has to remember anything.
4. **Licence right-sizing.** Acme has 50 finance staff but 10 premium licences. The admin creates an Entra ID group, drops in 10 people, and `PATCH /Groups` maps exactly those 10 to the premium role — no HR data manipulation needed.
5. **Merger cleanup.** Acme acquires a subsidiary and re-domains 200 email addresses. Because MoneyGuard matched on `externalId`, all 200 records update cleanly instead of duplicating.
6. **SOX evidence.** Dana pulls the SCIM audit log and shows, per user, the exact timestamp between HR termination and full access revocation across every MoneyGuard system.

## Key Takeaways

- **JIT is pull, SCIM is push.** That one difference explains every advantage and every failure of each.
- JIT's two failures: the **ghost user** (cannot pre-configure) and — far worse — the **deprovisioning gap** (a leaver never logs in, so the app never learns).
- **Zombie accounts survive SSO being blocked** via API keys, long-lived sessions and local credentials. Blocking the front door is not deprovisioning.
- **SCIM is a universal translator**: one standard implementation works with Entra ID, Okta, Ping and the rest, which is why enterprise buyers require it.
- **Production runs both**: SCIM as the primary channel, JIT as a safety net, joined by **upsert logic keyed on `externalId`**.
- **`externalId` is the anchor.** Never match on email.
- **The tenant comes from the SCIM token**, never from the payload.
- **`active: false` is a security event**, triggering sessions, tokens, API keys, ReBAC tuples, cached decisions and running jobs — not just a database column.
- **SCIM is batched, not instant** (~40 minutes for Entra ID), which is exactly what creates the race condition in Chapter 3.

## Glossary

| Term | Definition |
|---|---|
| **Deprovisioning** | Disabling or removing an account when access should end. |
| **`externalId`** | The IdP's immutable identifier for a user, stored by the application to anchor the link. |
| **Ghost user** | Someone who exists in the IdP but has no record in the application because they have never logged in. |
| **IdP (Identity Provider)** | The system that authenticates users and, with SCIM, pushes lifecycle changes. |
| **Idempotency** | The property that repeating an operation produces the same result as performing it once. |
| **JIT provisioning** | Creating an application account on the user's first successful SSO login. |
| **JML** | Joiner–Mover–Leaver, the identity lifecycle (see Module 01, Chapter 5). |
| **Pre-provisioning** | Creating and configuring an account before the user first logs in. |
| **SCIM** | System for Cross-domain Identity Management — a REST/JSON standard for identity synchronisation. |
| **Soft delete** | Marking a record inactive rather than removing it, preserving the audit trail. |
| **System of record** | The authoritative source for a class of facts; Workday for employment status. |
| **Upsert** | Update-or-insert: link to an existing record if present, otherwise create one. |
| **Zombie account** | An account still active in an application after the person has left the organisation. |

## Revision Notes

- **JIT = pull, on login. SCIM = push, on HR event.**
- JIT failures: **ghost user** (onboarding), **deprovisioning gap** (offboarding — the dangerous one).
- Zombie accounts persist via **API keys, long-lived sessions, local passwords** even when SSO is blocked.
- SCIM is the **universal translator** — one implementation, every IdP.
- Setup handshake = **SCIM base URL + per-tenant bearer token**.
- **`externalId` is the anchor.** Emails change; object IDs do not.
- **Tenant from the token, never the body.**
- `active: false` ⇒ DB flag **+ kill sessions + revoke API keys + delete ReBAC tuples + evict authz cache + cancel jobs + audit**.
- **Entra ID syncs in ~40-minute batches** → SCIM is reliable but not instant.
- **Run both**: SCIM primary, JIT fallback, reconciled by **upsert on `externalId`**.

## Interview Questions

**Q1. Why isn't JIT provisioning enough for an enterprise SaaS product?**
Because JIT is a pull mechanism triggered by login, and the two things enterprises care most about don't involve a login. Onboarding: an admin wants to assign a new hire to a workspace before day one, but under JIT that person doesn't exist in the application until they first sign in — they're a ghost user, so pre-configuration is impossible. Offboarding, which is the serious one: a terminated employee will never log in again, so the trigger JIT depends on never fires and the application stays permanently blind to the termination. JIT can create users but has no mechanism whatsoever to remove them.

**Q2. If SSO is disabled for a terminated employee, why does the zombie account still matter?**
Because SSO is only one of several doors. If that person generated a personal API key for an integration, that key doesn't go through SSO and keeps working. A long-lived session cookie on a personal device survives the IdP change. If the application ever supported a local password fallback, that path is untouched. And even with every door genuinely closed, the audit control still fails: the application's records assert that a terminated person is an active user of a banking system, and you can't prove otherwise. Locking the front door is not the same as deprovisioning.

**Q3. Why should you match users on `externalId` rather than email?**
Because email is mutable and `externalId` isn't. The `externalId` is the IdP's immutable object identifier — Entra ID's object ID, for instance — and it survives every change to the person's name, address or department. Email changes routinely: marriages, rebrands, domain migrations after an acquisition. If you key on email, the moment IT updates an address you get a duplicate record and an orphaned original that still holds entitlements, which is both a data-integrity bug and a security one. I'd store `externalId` on first contact — including on the JIT path — precisely so the later SCIM message can link rather than duplicate.

**Q4. What must happen when a SCIM `active: false` arrives?**
Setting the database flag is step one of about seven, and stopping there is the most common deprovisioning bug. You also have to kill live sessions and access tokens — otherwise the person's existing token stays valid until it expires, which could be another hour. Revoke any API keys or personal access tokens they generated, since those bypass SSO entirely. Delete their relationship tuples from the authorization graph so nothing grants them access by inheritance. Publish a revocation event so every service evicts its cached authorization decisions. Cancel long-running jobs they own. And write the audit record with a timestamp. The right implementation is a single revocation function that the SCIM handler, the admin console and the SOC kill switch all call, so the paths can't diverge.

**Q5. How do you handle a user who logs in before the SCIM sync has created them?**
Keep JIT enabled as an explicit fallback with upsert logic. Entra ID batches provisioning on roughly a forty-minute cycle while SSO is instant, so a keen new hire will regularly beat the sync. Rather than rejecting them, the SSO handler creates the record from the ID token's claims and — crucially — writes the `sub` into `externalId` immediately. When the SCIM `POST` arrives later, the handler looks up by `externalId`, finds the existing record, and updates it instead of failing with a duplicate error. The user is never blocked, and the two systems converge. I'd also stamp the record with its source so I can measure how often SCIM is losing the race.

**Q6. How do you secure a SCIM endpoint?**
A long-lived, cryptographically strong bearer token issued per customer tenant — never shared across customers, because one leak would then compromise everyone. Store it hashed, make it independently revocable, and support rotation without downtime. The token does double duty: middleware validates it and extracts the tenant ID, and that tenant ID is what scopes every read and write. The critical rule is that the tenant must come from the authenticated token and never from the request body, because a caller who can name the tenant has a cross-tenant write primitive. Beyond that: TLS only, rate limiting, IP allow-listing where the customer's IdP has stable egress, and full logging of every operation.

**Q7. What happens if your API is down when the IdP tries to sync?**
Nothing catastrophic, because SCIM is designed around eventual consistency. If the endpoint returns a 5xx or is unreachable, the identity provider queues the event and retries with exponential backoff — Entra ID and Okta will keep retrying for hours or days and surface a provisioning error to the customer's admin. The directory converges once you're back. The two things you must get right for this to work are honest status codes — don't return 200 when you've actually failed, or the event is lost forever — and idempotent handlers, because those retries mean you will receive the same operation more than once. I'd also run a nightly reconciliation sweep comparing the IdP's roster to mine, as a backstop for anything the push channel silently dropped.

**Q8. Would you ever ship SCIM without JIT, or JIT without SCIM?**
JIT without SCIM is defensible only for a self-serve consumer-grade product with no enterprise customers and no compliance obligations, and even then you're accepting that accounts are never cleaned up. For anything B2B or regulated it's not acceptable, because you've built onboarding with no offboarding. SCIM without JIT is safer but produces a bad failure mode: any user who logs in before their sync batch lands gets locked out with a confusing error on their first day, and the same happens during the window while a customer is still configuring provisioning. So in practice I'd always ship both — SCIM as the authoritative channel, JIT as the safety net — with upsert reconciliation between them.

## Practice Exercises

1. **Trace the gap.** MoneyGuard's Business Banking SaaS Portal currently supports JIT only. Jordan Rivera is terminated by Acme Corp at 16:30 on a Friday. Write a timeline of what MoneyGuard knows and when, listing every credential Jordan still holds at 09:00 Monday and exactly why each one survives. Then rewrite the timeline assuming SCIM is enabled.

2. **Design the handshake.** Write the onboarding instructions MoneyGuard gives Acme Corp's IT admin to connect SCIM. Include what MoneyGuard generates, what Acme configures, how the token is rotated, and what happens if the token leaks. Then list three things that must *not* appear in those instructions and say why.

3. **Kill the zombie.** Write the complete `active: false` handler for MoneyGuard's portal — every downstream system it must touch, in order, with the correct behaviour if any one of them fails. Decide whether a partial failure should return `200` or `500` to the IdP, and justify your choice in terms of what the provider will do next.

4. **Break the tenant boundary.** An engineer implements `POST /scim/v2/Users` reading `tenantId` from the JSON body "to support multi-tenant admins." Describe the exact request Acme Corp could send to create a user inside Globex Inc's partition, then rewrite the handler correctly and name two other places in the SCIM implementation the same rule must apply.

5. **Choose the mechanism.** For each MoneyGuard situation, decide JIT, SCIM, both, or neither, and justify it in one line: (a) an internal MoneyGuard employee tool used only by staff already in MoneyGuard Entra ID, (b) the Business Banking SaaS Portal for Acme Corp, (c) a public customer-facing mobile banking signup, (d) PartnerBank staff federating in for a joint mortgage product, (e) the Fraud Detection Pipeline's service account.

---

⬅ Back to [Module 05 README](./README.md) · [Main README](../README.md) · [MoneyGuard Scenario](../SCENARIO.md)
➡ Next: [SCIM Protocol Deep Dive](./02_SCIM_Protocol_Deep_Dive.md)
