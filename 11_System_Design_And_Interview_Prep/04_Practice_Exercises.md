# Chapter 04: Practice Exercises

> **Module 11 · Chapter 4 of 4** — 12 whiteboard and hands-on scenarios. Hints and grading checklists, deliberately no full solutions.

## 📋 Overview

Reading an architecture is not the same as producing one. This chapter is the **final exam** for the repository: twelve MoneyGuard Bank scenarios that you work through yourself.

Each exercise gives you:

- **The scenario** — as an interviewer or an incident channel would actually phrase it.
- **💡 Hint** — one nudge if you stall, no more.
- **✅ What a strong answer covers** — the grading checklist. Score yourself against it *after* you attempt the exercise.

⚠️ **There are no full solutions, and that is intentional.** The gap between "I could have said that" and "I did say that" is exactly what an interview measures. If you read the checklist first, you get recognition; if you attempt first, you get recall — and only recall survives interview pressure.

## 🎯 How To Use This Chapter

| | Method |
|---|---|
| **Design exercises** (1, 3, 5, 8, 10, 12) | Set a 30-minute timer. Draw on paper or a whiteboard. Narrate **out loud** — silent drawing hides the exact gaps you need to find. |
| **Diagnostic exercises** (2, 4, 6, 9) | Write your hypotheses *in priority order* before reading the hint. Ranking is the skill; listing possibilities is not. |
| **Hands-on exercises** (7, 11) | Actually build the thing. An afternoon of implementation beats a week of reading. |

Work through the **⭐ Foundational** tier first if you're less experienced, then **⭐⭐ Intermediate**, then **⭐⭐⭐ Advanced**.

| # | Exercise | Type | Level | Primary modules |
|---|---|---|---|---|
| 1 | [Onboard 500 business-banking clients](#exercise-1--onboard-500-business-banking-clients) | Design | ⭐⭐ | 05, 08 |
| 2 | [PartnerBank's SSO fails intermittently](#exercise-2--partnerbanks-sso-fails-intermittently) | Diagnostic | ⭐⭐ | 02, 09 |
| 3 | [The 60-second kill switch](#exercise-3--the-60-second-kill-switch) | Design | ⭐⭐ | 04, 05 |
| 4 | [Globex Inc sees Acme Corp's payroll](#exercise-4--globex-inc-sees-acme-corps-payroll) | Diagnostic | ⭐⭐⭐ | 08 |
| 5 | [Secrets out of the Fraud Detection Pipeline](#exercise-5--get-the-secrets-out-of-the-fraud-detection-pipeline) | Design | ⭐⭐⭐ | 06, 10 |
| 6 | [Eve's credential-stuffing campaign](#exercise-6--eves-credential-stuffing-campaign) | Diagnostic | ⭐⭐ | 07, 09 |
| 7 | [Implement Row-Level Security](#exercise-7--implement-row-level-security-hands-on) | Hands-on | ⭐ | 08 |
| 8 | [Step-up authentication for wire transfers](#exercise-8--step-up-authentication-for-wire-transfers) | Design | ⭐⭐ | 07, 03 |
| 9 | [Dana Cole's audit finds 1,200 orphans](#exercise-9--dana-coles-audit-finds-1200-orphaned-accounts) | Diagnostic | ⭐⭐ | 01, 05 |
| 10 | [The AuthZ service is at 40% error rate](#exercise-10--the-authz-service-is-at-40-error-rate) | Design | ⭐⭐⭐ | 09, 03 |
| 11 | [Decode and validate a JWT](#exercise-11--decode-and-validate-a-jwt-hands-on) | Hands-on | ⭐ | 04 |
| 12 | [The full mock: design MoneyGuard's IAM](#exercise-12--the-full-mock-design-moneyguards-iam) | Design | ⭐⭐⭐ | All |

---

## Exercise 1 · Onboard 500 business-banking clients

**Level:** ⭐⭐ Intermediate · **Type:** Whiteboard design · **Time:** 30 min

🏦 **Scenario.** MoneyGuard just signed a channel deal. Over the next quarter, **500 new corporate clients** will onboard onto the Business Banking SaaS Portal. Each brings between 5 and 2,000 of its own employees. Roughly 40 of them — the largest, including Acme Corp — insist on using their own corporate identity provider for SSO rather than MoneyGuard-managed passwords. Each client needs its own admin who can manage their own users without MoneyGuard's help desk in the loop.

Raj Patel cannot manually configure 500 tenants. Design the onboarding system.

> 💡 **Hint.** Separate the *tenant* lifecycle from the *user* lifecycle — they are two different state machines that happen to be created at the same time. Then ask what the client's admin can self-serve versus what requires MoneyGuard.

<details>
<summary>✅ What a strong answer covers</summary>

- **Tenant provisioning as its own flow:** create the tenant record, allocate a `tenant_id`, seed default roles, designate the first admin. Automated and idempotent — a retry must not create two Acme Corps.
- **Bootstrapping the first admin** without a chicken-and-egg problem: a signed, single-use, time-limited invitation link, not a shared password emailed in plaintext.
- **Delegated administration:** the client's admin manages their own users, scoped so they can *only ever* touch their own tenant. Permission boundaries so a tenant admin cannot grant themselves more than they hold.
- **Two identity paths, one model:** MoneyGuard-managed users versus BYO-SSO federation. Both must resolve to the same principal shape downstream so authorization doesn't fork.
- **Tenant-routed SSO:** resolve the tenant *before* deciding which IdP to redirect to — typically by email domain mapping, with a fallback for users whose domain is ambiguous or shared.
- **Self-service SSO configuration** for the 40 federating clients — upload metadata, test the connection — because 40 manual SAML integrations is a quarter of Raj's year.
- **SCIM offered to clients who want it**, so their own HR system drives their user lifecycle into MoneyGuard.
- **Isolation from day one:** `tenant_id` on every row, RLS enforced, and a test proving the new tenant cannot read an existing one.
- **Operational scale:** onboarding is a queue, not a synchronous request; progress is observable; failures are retriable and visible to whoever is running the migration.
- **Certificate/metadata expiry tracking** for all 40 federations, with alerting — otherwise you have created 40 future outages.
- ⚠️ **Names the pitfall:** the temptation to hand-configure the first few "just to get going," which produces snowflake tenants that break every subsequent automation.

</details>

---

## Exercise 2 · PartnerBank's SSO fails intermittently

**Level:** ⭐⭐ Intermediate · **Type:** Diagnostic · **Time:** 20 min

🏦 **Scenario.** PartnerBank staff federate into MoneyGuard over SAML for the joint mortgage product. For the past three days, roughly **1 in 10 login attempts fails** with a generic error. Retrying usually works. It affects some users far more than others. MoneyGuard's dashboards show no errors — the failures never reach the application logs. PartnerBank insists nothing changed on their side.

Diagnose it. What are your hypotheses, in priority order, and how would you confirm each?

> 💡 **Hint.** "Intermittent," "retry fixes it," and "affects some users more than others" is a very specific signature. Think about what differs between two attempts by the same user seconds apart — and what differs between two *servers* handling them.

<details>
<summary>✅ What a strong answer covers</summary>

- **Prioritised hypotheses**, not an unordered list. Strong candidates, roughly in order:
  - **Clock skew** between PartnerBank's IdP and one or more MoneyGuard nodes. SAML assertions have a tight validity window (`NotBefore` / `NotOnOrAfter`); a node drifted by 30 seconds rejects a fraction of assertions. Explains intermittency, load-balancer stickiness explaining why some users hit it more, and "nothing changed."
  - **One bad node in the pool.** A single instance with stale configuration, an old certificate, or a failed deploy. ~1 in 10 ≈ one node in ten.
  - **Certificate rollover in progress.** PartnerBank rotating their signing certificate, with both old and new in use; MoneyGuard only trusts one.
  - **Assertion replay cache behaving inconsistently** across nodes — an assertion ID accepted by one node and rejected as a replay by another.
  - **Assertion size** exceeding a proxy or header limit for users with many group memberships (explains "some users more than others" cleanly).
  - **Session/state affinity:** the flow starts on node A and the callback lands on node B, which has no record of the relevant request state.
- **"No errors in our logs" is itself the biggest clue** — it means the failure occurs *before* the application, at the load balancer, proxy, or SAML middleware. First action is to look at a layer you have not yet instrumented.
- **Confirmation method per hypothesis:** compare NTP offset across all nodes; correlate failures against the node ID serving them; capture and inspect a failing raw assertion; check certificate validity windows on both sides.
- **Correct escalation posture:** "nothing changed on our side" is not evidence — ask PartnerBank for their IdP-side logs for a specific failed correlation ID and the specific timestamps.
- **Immediate mitigation while diagnosing:** drain the suspect node, widen the clock-skew tolerance *temporarily*, or pin traffic to healthy nodes.
- **The fix that outlasts the incident:** log SAML validation failures with a reason code so this class of failure is never invisible again.
- ⚠️ **Avoids the trap** of jumping straight to "it's a network issue" or blaming PartnerBank without evidence.

</details>

---

## Exercise 3 · The 60-second kill switch

**Level:** ⭐⭐ Intermediate · **Type:** Whiteboard design · **Time:** 30 min

🏦 **Scenario.** Dana Cole's SOX audit produced a finding: when Sam Torres's contract ended, his access to three systems persisted for **over 20 minutes**. The remediation commitment to the regulator is that **all access must be revoked within 60 seconds** of a termination event in Workday — measurably, with evidence.

Design the system that achieves this, across MoneyGuard Entra ID, MoneyGuard AD, the API Gateway, and the Core Banking Ledger. Then explain how you would *prove* the 60 seconds to Dana.

> 💡 **Hint.** Disabling the account in the IdP is the step everyone thinks of, and on its own it cannot meet the requirement. Ask what is *already issued* and still in flight at the moment of termination.

<details>
<summary>✅ What a strong answer covers</summary>

- **Names the gap precisely:** disabling in the IdP stops *new* token issuance, but an already-issued access token remains cryptographically valid until `exp`. That window is the 20 minutes in the finding.
- **The multi-step fan-out**, ideally with rough timings: deactivate in Entra ID → disable in AD → **publish a revocation event to the edge** → **invalidate cached authorization decisions** → revoke refresh tokens → log the whole sequence.
- **The revocation list at the API Gateway** as the mechanism that kills in-flight tokens: an in-memory/distributed lookup on `jti` or subject, checked on every request, with entries self-expiring when the token would have expired anyway so the list stays small.
- **A `sessions_invalid_before` timestamp per subject** as the blunt instrument that kills every session at once.
- **Cache invalidation as an explicit step** — a stale `ALLOW` in the L2 cache defeats everything else. TTL alone is not sufficient.
- **Short access token lifetimes** (10–15 min) as the backstop, not the primary mechanism.
- **Legacy systems that cannot participate:** AD-only applications, systems with local user tables. Name them honestly and propose compensating controls rather than pretending they're covered.
- **Idempotency and retry**, with deprovisioning operations prioritised ahead of routine updates in the queue.
- **A failure path that pages a human.** A silently failed revocation is precisely the scenario the finding was about.
- **Proving it to Dana:** emit a timestamped event at each stage with a shared correlation ID; report the p50/p95/p99 distribution of Workday-event-to-fully-revoked; alert on any breach of 60 seconds; and run a **periodic synthetic termination** in production to demonstrate the control works continuously, not just on the day of the audit.
- ⚠️ **Names the pitfall:** claiming an SLA that is never measured. An unmeasured control is an assertion, and auditors do not accept assertions.

</details>

---

## Exercise 4 · Globex Inc sees Acme Corp's payroll

**Level:** ⭐⭐⭐ Advanced · **Type:** Diagnostic · **Time:** 25 min

🏦 **Scenario.** A Globex Inc administrator calls support: a payroll report they downloaded contains **rows belonging to Acme Corp**. The web application is known to filter correctly by tenant, and Row-Level Security is enabled on the payroll tables. This is a live data breach in a regulated environment.

What do you do, in what order, and where is the bug?

> 💡 **Hint.** The scenario says the *web application* filters correctly and RLS is enabled. So ask: what produced this report, and did it travel the same code path as a web request?

<details>
<summary>✅ What a strong answer covers</summary>

- **Containment before diagnosis.** Disable the report feature immediately. A breach that is still leaking outranks a breach you fully understand. Preserve logs before anything is redeployed.
- **Regulatory clock awareness:** this is a reportable data incident with a notification deadline. Engage legal/compliance in parallel with engineering — not afterwards.
- **The most likely root cause: a code path that bypasses RLS.** Report generation, batch jobs, and exports typically run without a user request context, so nothing sets the `app.current_tenant` session variable — and an RLS policy comparing against an unset variable either errors or, if the code sets a permissive default, returns everything.
- **Other candidates, ranked:** the reporting service connects with a database role holding `BYPASSRLS` or superuser; a cache or object-storage key without a tenant prefix serving one tenant's generated file to another; a connection-pool reuse bug where the session variable persists from a previous tenant's transaction; a report query using a service token whose `tenant_id` claim was never populated.
- **Connection pooling specifically** is worth naming — `SET` on a pooled connection that isn't reset between checkouts is a classic and subtle cross-tenant leak.
- **Determining blast radius:** query the audit log for every report generated by this path, for every tenant, over the full period the code has been live. Assume it is not the only affected report until proven otherwise.
- **The layered fix:** correct the immediate bug; ensure *every* non-request code path sets tenant scope explicitly; verify the application's DB role is subject to RLS; add tenant prefixes to all cache and storage keys.
- **Prevention that outlasts the fix:** an automated CI test that authenticates as one tenant and attempts to read a known row of another, run on every commit; a Sentinel detection for any query returning rows spanning multiple `tenant_id` values; and a review of every background job for tenant scoping.
- ⚠️ **Names the pitfall:** treating RLS as a guarantee rather than a layer. RLS only protects the paths that go through it.

</details>

---

## Exercise 5 · Get the secrets out of the Fraud Detection Pipeline

**Level:** ⭐⭐⭐ Advanced · **Type:** Whiteboard design · **Time:** 30 min

🏦 **Scenario.** The Fraud Detection Pipeline runs ~200 Kubernetes pods scoring transactions in near real-time. Today, each pod reads a **static API key** for the Core Banking Ledger from an environment variable, populated from a Kubernetes Secret that was created by hand 18 months ago. The key has never been rotated. Nobody is certain who else holds it, and a copy was found in a Confluence page.

Design the replacement. Include the migration — the pipeline cannot go down.

> 💡 **Hint.** The goal is not "store the key better." It is to arrive at a state where **there is no key to store**. Work out where the pod's identity can come from if it isn't something the pod carries.

<details>
<summary>✅ What a strong answer covers</summary>

- **Names the failure modes of the current state:** no expiry, no rotation, unknown copies, no attribution (the ledger cannot tell which pod called), and almost certainly excessive scope.
- **Platform-attested workload identity as the target:** the cluster projects a short-lived, auto-rotating signed token into the pod asserting its service account, namespace, and cluster. This is derived from *what the pod is*, not something it holds.
- **Token exchange:** the pod trades that platform token at a token service for a narrowly-scoped, short-lived access token (`transactions:read`, ~15 min).
- **mTLS to the ledger** so the caller is verified at the transport layer too, and a stolen token is useless from an unregistered machine.
- **"Secret zero" articulated:** the problem isn't secret storage, it's secret existence. Platform-derived identity eliminates the bootstrap credential entirely.
- **Least privilege applied while you're here:** scope to exactly the operations the pipeline performs. An 18-month-old hand-made key is almost certainly over-privileged; migration is the moment to fix it.
- **A safe migration sequence:** accept both credentials at the ledger → roll out the new identity to a canary subset → verify via logs that traffic is arriving on the new path → complete the rollout → **confirm zero traffic on the old key** → revoke the old key → remove it from Confluence and every other copy.
- **Rollback plan** at each stage, since the pipeline is production-critical.
- **Ongoing governance:** the workload identity gets a registered owner, an expiry, inclusion in reconciliation and recertification, and behavioural baselining (alert on new data classes, volume spikes, unusual regions).
- **Treat the exposed key as compromised** — it was in a wiki. Audit its historical use for anomalies rather than assuming no one abused it.
- ⚠️ **Names the pitfall:** migrating to a secrets manager and stopping there. Vault is a real improvement, but the pod still holds a credential to fetch the credential — the problem has moved, not gone.

</details>

---

## Exercise 6 · Eve's credential-stuffing campaign

**Level:** ⭐⭐ Intermediate · **Type:** Diagnostic + design · **Time:** 25 min

🏦 **Scenario.** Sentinel alerts at 02:40. Over the past hour, the MoneyGuard Mobile App's login endpoint has taken **1.2 million authentication attempts** from ~8,000 distinct IP addresses across 40 countries. The success rate is **0.4%** — low, but not zero, meaning roughly 4,800 customer accounts have been accessed. Each IP is making only a handful of attempts, so per-IP rate limits never triggered.

You are on call. What do you do now, and what do you change afterwards?

> 💡 **Hint.** The 0.4% success rate is the signature: these are *valid* credentials from a breach elsewhere, being replayed. Also note *why* the rate limiter missed — the attack is distributed precisely to defeat per-IP counting.

<details>
<summary>✅ What a strong answer covers</summary>

**Immediate (minutes):**
- **Identify and contain the ~4,800 successful accounts** first — they are the actual breach. Force session revocation via the edge revocation list, force a password reset, and require step-up on any money movement from those accounts.
- **Raise friction on the login path** globally: CAPTCHA or proof-of-work, and mandatory MFA challenge on any login showing a new device or new geography.
- **Rate limit on dimensions the attacker cannot cheaply rotate** — per *account*, per device fingerprint, per ASN, and globally on the endpoint — rather than per IP, which is exactly what the attack was built to evade.
- **Preserve evidence** and open an incident with a single correlation ID before mitigation changes the traffic pattern.

**Short term (days):**
- **Notify affected customers** and coordinate with legal/compliance on disclosure obligations.
- **Check for downstream fraud** on the 4,800 accounts — the login is the means, not the goal. Correlate with the Fraud Detection Pipeline.
- **Breached-password screening:** compare stored password hashes against known-breach corpora and force resets on matches, since these credentials came from someone else's breach.

**Structural:**
- **Move customers toward passkeys.** Credential stuffing is structurally impossible against a phishing-resistant, origin-bound credential with no shared secret — this is the only change that removes the attack class rather than raising its cost.
- **Adaptive/risk-based authentication:** device reputation, impossible travel, behavioural signals feeding a risk score that drives step-up.
- **Better detection:** alert on *aggregate* failure-rate anomalies and on distributed low-and-slow patterns, not just per-IP thresholds. The gap that let this run for an hour is a detection gap as much as a control gap.

- ⚠️ **Names the pitfall:** blocking the 8,000 IPs and declaring victory. The IPs are disposable, the credentials are not, and the 4,800 compromised accounts are still compromised.

</details>

---

## Exercise 7 · Implement Row-Level Security (hands-on)

**Level:** ⭐ Foundational · **Type:** Hands-on · **Time:** 45 min

🏦 **Scenario.** Build a miniature version of the Business Banking SaaS Portal's data isolation, locally, in PostgreSQL.

**Build:**
1. A `tenants` table with **Acme Corp** and **Globex Inc**.
2. A `payroll_batches` table with a `tenant_id` column and several rows for each tenant.
3. Row-Level Security enabled, with a policy scoping rows to a session variable.
4. A non-superuser application role that the policy actually applies to.
5. A small script that sets the tenant, runs `SELECT * FROM payroll_batches`, and prints the result — for each tenant in turn.

**Then deliberately break it and observe what happens:**
- Run a query with the session variable **unset**.
- Run the same query as a **superuser**, or as a role with `BYPASSRLS`.
- Write a query that *forgets* the `WHERE tenant_id = ...` clause and confirm RLS still protects you.
- Run two queries on the **same pooled connection** without resetting the variable between them.

> 💡 **Hint.** The interesting part is not making it work — it's the four break cases. Each one corresponds to a real production breach pattern, and the fourth is the subtlest.

<details>
<summary>✅ What a strong answer covers</summary>

- A working policy in the shape of `USING (tenant_id = current_setting('app.current_tenant')::uuid)`, with the variable set per transaction.
- **Understands `FORCE ROW LEVEL SECURITY`** and why table owners and superusers bypass policies by default — meaning your application's DB role must be an ordinary role, and you should verify that rather than assume it.
- **Observed and can explain each break case:**
  - *Unset variable* → error or empty result, depending on `current_setting`'s missing-ok argument. Notes that a permissive default here (returning all rows) would be catastrophic.
  - *Superuser / BYPASSRLS* → policy silently does nothing. This is the "reporting service connects as postgres" breach.
  - *Missing `WHERE` clause* → still safe. **This is the entire point of RLS** — it converts an application bug into a functional bug instead of a breach.
  - *Pooled connection reuse* → the previous tenant's value persists and leaks. Fix by setting the variable inside the transaction (`SET LOCAL`) or resetting on connection checkout.
- **Recognises the write side too:** a `USING` clause governs reads; a `WITH CHECK` clause is needed to stop a tenant *writing* a row belonging to another.
- **Considers indexing:** `tenant_id` must be indexed and generally leading in composite indexes, or the policy predicate degrades query plans across the board.
- **Turns it into a permanent control:** a test that authenticates as Acme Corp, attempts to read a known Globex Inc row, and asserts zero rows — wired into CI.
- ⚠️ **Names the pitfall:** enabling RLS but connecting as the table owner, so nothing is enforced and everything appears to work.

</details>

---

## Exercise 8 · Step-up authentication for wire transfers

**Level:** ⭐⭐ Intermediate · **Type:** Whiteboard design · **Time:** 25 min

🏦 **Scenario.** MoneyGuard is adding outbound wire transfers to the Mobile App and the Employee Portal. Risk has mandated: **transfers over $10,000 require a fresh strong authentication**, regardless of how long the user has been logged in. Additionally, any transfer to a payee added in the last 24 hours requires it too, at any amount.

Design it. Where is the check enforced, what does the token carry, and what does the user experience look like?

> 💡 **Hint.** Two independent properties matter about an authentication: how *strong* it was, and how *recent* it was. A four-hour-old passkey tap satisfies one and not the other. Then ask what stops Eve from simply not using the app.

<details>
<summary>✅ What a strong answer covers</summary>

- **The `acr` claim** (Authentication Context Class Reference) carries the assurance level of the current session, and **`auth_time`** carries recency. Both are needed — strength and freshness are different requirements.
- **Enforcement is server-side, at the API that executes the transfer.** UI prompting is a nicety; Eve calls the API directly. This is the single most important point in the exercise.
- **The challenge flow:** API returns a structured `insufficient_authentication` response naming the required `acr` and max age → client initiates re-authentication with the IdP requesting that level → IdP returns a token with elevated `acr` and fresh `auth_time` → request retried and succeeds.
- **Binding the elevated authentication to the specific action.** A generic "user stepped up" flag is weak — a step-up obtained for a $50 transfer should not authorise a $2M one. Bind to the transaction (transaction-bound confirmation showing amount and payee in the authentication prompt itself, so malware cannot swap the details behind the user's back).
- **Policy lives in the PDP, not hardcoded.** Thresholds and payee-age rules are policy, so they can change without a release, and so Dana can read them.
- **Both triggers implemented:** amount threshold *and* the 24-hour new-payee rule — the second is a strong anti-fraud control because new-payee-plus-large-amount is the classic scam signature.
- **Fallback and accessibility:** what happens when the user has no passkey enrolled, or their device is unavailable. There must be a path, and it must not be weaker than the control it replaces.
- **Replay and race safety:** the elevated token is single-use for the action or very short-lived; the request carries an idempotency key so a retried transfer doesn't execute twice.
- **Auditing:** log the step-up challenge, the factor used, the `acr` achieved, and the resulting action, so a disputed transfer can be reconstructed.
- **Names the UX trade-off:** step up too often and users habituate to approving prompts, which erodes the control's value. Reserve it for genuinely consequential actions.
- ⚠️ **Names the pitfall:** treating a valid session as blanket consent, or enforcing only in the mobile client.

</details>

---

## Exercise 9 · Dana Cole's audit finds 1,200 orphaned accounts

**Level:** ⭐⭐ Intermediate · **Type:** Diagnostic + design · **Time:** 25 min

🏦 **Scenario.** Dana Cole's annual access review reconciles the Workday roster against MoneyGuard Entra ID and MoneyGuard AD. She finds **1,200 active accounts with no corresponding active employee**. Spot checks show a mix: contractors whose engagements ended, employees who left over two years ago, several service accounts named after people who have left, and a handful nobody can identify at all. Two of them show login activity in the last 30 days.

Explain how this happened, what you do about the 1,200, and how you prevent it recurring.

> 💡 **Hint.** "A mix" is the important word — these are several distinct root causes wearing the same symptom. Treat them separately. And note that "two show recent activity" changes the priority of everything else.

<details>
<summary>✅ What a strong answer covers</summary>

**Triage first:**
- **The two with recent logins are an active incident**, not an audit finding. Investigate immediately: disable, preserve logs, determine what was accessed, and whether the credentials are being used by someone unauthorised.

**Root causes, treated distinctly:**
- **Contractors** were often never in Workday at all, so no Leaver event could ever fire for them. This is a *coverage* gap, not a sync failure.
- **Long-departed employees** point at a period when deprovisioning was manual, or at silently failed SCIM pushes with no reconciliation to catch them.
- **Service accounts named after people** are non-human identities with no lifecycle and no owner — the account survives because nothing connects it to an employment record.
- **Unidentifiable accounts** point at hand-created accounts outside the provisioning pipeline, which is the deeper governance problem: if accounts can be created by hand, no downstream control is complete.

**Remediating the 1,200:**
- Categorise, then **disable rather than delete** initially — preserving them keeps the audit trail and allows recovery from a misclassification. Delete after a defined quarantine.
- Route anything still needed (a service account doing real work) through proper re-provisioning with a registered owner, a scope, and an expiry — do not simply leave it.
- Communicate before disabling in bulk; a disabled account that something depended on becomes an outage.

**Prevention:**
- **Close the contractor gap:** every non-employee gets a record in the system of record with a mandatory, enforced end date. Non-employee identities should default to expiring.
- **Nightly automated reconciliation** comparing the roster against every directory, reporting orphans and gaps — so the next finding is 3 accounts, caught overnight, not 1,200 caught annually.
- **Remove the ability to hand-create accounts.** Provisioning happens through the pipeline or not at all; break-glass creation is alerted and reviewed.
- **Non-human identity governance:** registered owner, purpose-scoped naming, default expiry, quarterly recertification.
- **Alert on the leading indicator** — SCIM deprovisioning failures should page, not sit in an unwatched dead-letter queue.
- ⚠️ **Names the pitfall:** treating this as a one-time cleanup. Without reconciliation, the number regrows to 1,200 by the next audit.

</details>

---

## Exercise 10 · The AuthZ service is at 40% error rate

**Level:** ⭐⭐⭐ Advanced · **Type:** Whiteboard design + incident · **Time:** 30 min

🏦 **Scenario.** 09:15 on a Monday. The AuthZ service's policy-store primary has failed over to a replica. During failover, the service returned errors for 90 seconds; it is now at a **40% error rate** and climbing, with p99 latency at 4 seconds. Because MoneyGuard fails closed, customers are seeing permission errors on the Mobile App and tellers cannot open accounts in branches. The customer-facing impact is total.

You are incident commander. What do you do now, and what do you change so this cannot recur?

> 💡 **Hint.** A failover that lasted 90 seconds has caused an outage that is still growing ten minutes later. Something is amplifying it. Ask what every client did the moment their request failed.

<details>
<summary>✅ What a strong answer covers</summary>

**Recognising the amplification:**
- **A retry storm.** Every failed request retried, multiplying load onto a replica that is also cold. The original 90-second failover is over; the outage is now self-sustaining. This is the key insight — treating the symptom as "the database is still broken" leads to the wrong actions.
- **Cold caches** after failover mean the miss rate spikes, so the store takes far more load than steady state exactly when it can least handle it.
- Possibly **thundering herd** on cache repopulation — thousands of concurrent misses for the same hot keys.

**Immediate actions:**
- **Shed load** to break the amplification: aggressive rate limiting, or reject a fraction of traffic outright so the surviving fraction succeeds. A service at 40% errors serving everyone is worse than one serving 60% of traffic reliably.
- **Extend cache TTLs temporarily** and enable stale-while-revalidate for low-risk read decisions — an explicit, time-boxed, logged security trade-off, approved by someone accountable, and reverted afterwards.
- **Keep failing closed on money movement.** Degrade to read-only rather than opening writes. Partial service beats an unsafe service.
- **Communicate:** status page, branch staff, and the help desk — tellers seeing "permission denied" will otherwise raise 500 access tickets and drown the IAM team.

**Structural fixes:**
- **Exponential backoff with jitter, and circuit breakers** in every client. This is the single change that prevents recurrence.
- **Request coalescing / single-flight** so N concurrent misses for the same key become one backend call.
- **Cache warming** on the standby replica so failover doesn't start cold, and pre-warming as part of the deploy process.
- **Faster, rehearsed failover** — and regular game-day exercises so the first real failover isn't the first one anyone has seen.
- **Correct status codes:** `503`, not `403`, so clients and dashboards can distinguish an outage from a permissions change. In this incident, `403` would have sent everyone hunting a policy bug.
- **Risk-tiered failure behaviour designed in advance**, so the fail-open decision for low-risk reads is a pre-approved policy rather than an improvised call at 09:20.
- **Bulkheads:** isolate the ledger's authorization path from the mobile app's, so one cannot starve the other.
- ⚠️ **Names the pitfall:** globally switching to fail-open under pressure. That converts an availability incident into a security incident, and it will be the finding in the post-mortem.

</details>

---

## Exercise 11 · Decode and validate a JWT (hands-on)

**Level:** ⭐ Foundational · **Type:** Hands-on · **Time:** 30 min

🏦 **Scenario.** Get properly comfortable with the token that everything in this repository depends on.

**Do:**
1. Obtain a real JWT — from any IdP's developer tier, or mint one yourself with a signing library.
2. **Decode it by hand.** Split on `.`, Base64URL-decode the header and payload. Do not use a website; do it in code or a terminal so you internalise that **anyone holding the token can read every claim**.
3. Identify and explain: `iss`, `sub`, `aud`, `exp`, `iat`, `nbf`, `jti`, `kid`, `acr`, `scope`.
4. **Fetch the issuer's JWKS** (`/.well-known/openid-configuration` → `jwks_uri`) and find the key whose `kid` matches the token header.
5. **Verify the signature** in code, then validate `iss`, `aud`, and `exp` explicitly.
6. **Break it four ways** and confirm each is rejected:
   - Edit a claim in the payload and re-encode without re-signing.
   - Change the `alg` header to `none` and strip the signature.
   - Use a token whose `aud` is a different application.
   - Use an expired token.

> 💡 **Hint.** The `alg: none` case is the one to dwell on. Ask yourself why a naive library call might accept it, and what a validator must do to refuse.

<details>
<summary>✅ What a strong answer covers</summary>

- **Internalises that JWT payloads are encoded, not encrypted.** Direct consequence: never put PII, secrets, or sensitive internal identifiers in a token.
- **Explains each claim's validation role**, especially the three that get skipped: `aud` (was this token meant for *me*? — skipping it enables audience confusion, where a token for service A is replayed against service B), `iss` (did *my* IdP issue it?), and `exp` with a deliberate, small clock-skew tolerance.
- **Understands `kid` → JWKS lookup**, that public keys are cached, and that this is why validation needs no network call and survives an IdP outage.
- **The `alg: none` attack explained:** historically, libraries that read the algorithm *from the token* would accept an unsigned token as valid. The defence is that the validator specifies the expected algorithm and key — **never trusts the token's own header** to decide how to verify it. Also covers the RS256→HS256 confusion attack, where the public key gets used as an HMAC secret.
- **Confirms the tampering case fails**, and can explain why: the signature covers the exact encoded header and payload bytes, so a single changed character invalidates it.
- **Distinguishes signature validity from authorization.** A perfectly valid signature says the token is authentic; it says nothing about whether the bearer may perform the action. Signature check ≠ permission check.
- **Connects to revocation:** a valid signature does not mean the session is still live — hence the `jti`/subject revocation list at the edge.
- ⚠️ **Names the pitfall:** decoding without verifying. `decode()` and `verify()` are different functions in every library, and using the first where you meant the second is a real, shipped vulnerability class.

</details>

---

## Exercise 12 · The full mock: design MoneyGuard's IAM

**Level:** ⭐⭐⭐ Advanced · **Type:** Full whiteboard mock · **Time:** 45 min

🏦 **The prompt**, as an interviewer would give it:

> *"Design the IAM system for MoneyGuard Bank. 50,000 employees must log in with corporate SSO and be onboarded and offboarded automatically from Workday. Millions of retail customers use web and mobile banking. Corporate clients like Acme Corp and Globex Inc use a multi-tenant Business Banking SaaS Portal, and the largest want to bring their own identity provider. Internal services such as the Fraud Detection Pipeline call the Core Banking Ledger APIs continuously. It must be highly available, meet SOX audit requirements, and be secure."*

This is the capstone. Run the full 45 minutes: 5 minutes clarifying, 10 on components, 15 on data flows, 15 on deep dives. **Narrate out loud the entire time** — ideally record yourself and watch it back.

> 💡 **Hint.** Do not start drawing. Start by writing the Laws of the System from your clarifying questions, and keep them visible — every later decision should point back at one of them. If you stall at any point, walk the lifecycle: Joiner → Mover → Leaver.

<details>
<summary>✅ What a strong answer covers</summary>

**Step 1 — Clarify (5 min)**
- Asks about identity populations, resource hierarchy, latency and availability targets, the offboarding SLA, tenant BYO-SSO, and compliance drivers — then *writes down* 3–5 laws (HR is source of truth; Zero Trust; 60-second deprovisioning; fail closed; everything attributable).
- Does rough numbers and draws the right conclusion: **authentication is low volume, authorization is ~10k/sec.** Scaling effort goes to the authorization path.

**Step 2 — Components (10 min)**
- Zones, not a scatter of boxes: Enterprise/HR → Lifecycle → Identity Providers → Clients → Workloads → Edge → Authorization → Data → Observability → IaC.
- One sentence of responsibility per box, narrated while drawing.
- Draws Observability and the revocation list — the two boxes most candidates omit.

**Step 3 — Data flows (15 min)**
- **Human SSO login:** OIDC Authorization Code + PKCE, and names what `state`, `nonce`, and PKCE each defeat.
- **An authorized request:** gateway (signature, `iss`/`aud`/`exp`, revocation, rate limit) → PEP → PDP → cache → RLS at the database.
- **Leaver kill switch:** SCIM fan-out including the revocation event and cache invalidation, meeting the 60-second law.
- **Workload identity:** platform-attested token → exchange → narrow scope → mTLS, with no static secret.
- Mentions SAML brokering for PartnerBank and tenant-routed SSO for Acme Corp.

**Step 4 — Deep dives (15 min)**
- Handles at least three of: JWT vs opaque; why permissions don't live in the token; RBAC/ABAC/ReBAC hybrid; fail-open vs fail-closed with the 503/403 distinction; pooled tenancy with RLS; step-up via `acr`; caching with event-driven invalidation.
- Uses the trade-off pattern: **"X because [law]; the cost is Y; if [other law] then Z."**

**Throughout**
- Points back at the Laws when justifying decisions.
- Volunteers at least one seam where the design could break, and how it would be caught — the clearest seniority signal available.
- Manages the clock, checks in with the interviewer, and adapts rather than defends when pushed back on.
- ⚠️ **Avoids:** designing only the login flow; forgetting non-human identities; no audit story; silent whiteboarding.

</details>

---

## 📊 Self-Assessment

After working through these, score yourself honestly:

| Signal | You are ready when… |
|---|---|
| **Recall** | You can draw the [full architecture](./02_MoneyGuard_Full_Architecture_Walkthrough.md) from memory, with all zones labelled. |
| **Narration** | You can talk continuously for 45 minutes on Exercise 12 without long silences. |
| **Depth** | For any box you draw, you can go three questions deep without saying "I'd have to look that up." |
| **Trade-offs** | Every design choice comes with a stated cost and a named alternative. |
| **Seams** | You volunteer failure modes before being asked. |
| **Diagnosis** | On exercises 2, 4, 6, and 9 you produced a *prioritised* hypothesis list, not a brainstorm. |

⚠️ If you scored well by reading the checklists first, that's recognition, not recall. Redo the exercise a week later, cold.

## 💡 Pro Tips

- **Record yourself on Exercise 12 and watch it back.** It is uncomfortable and it is the single highest-leverage hour of preparation available. You will hear filler, hedging, and gaps you cannot detect from the inside.
- **Do the diagnostic exercises against a timer.** Incident response is a time-pressured skill, and interviewers ask these specifically to see how you behave when the answer isn't immediate.
- **Attempt exercise 12 twice** — once cold now, once after working through 1–11. The delta shows you exactly what the practice bought.
- **Build a war story.** Take one exercise, do it for real at work if you can, and you'll have a concrete example to reach for when an interviewer asks "have you dealt with this?"

## 🔑 Key Takeaways

1. Recognition is not recall. Attempt before reading the checklist, every time.
2. **Diagnosis is about prioritising hypotheses**, not listing them.
3. **Containment precedes investigation** in every security incident — exercises 4 and 6 both hinge on this.
4. The recurring root causes across these scenarios are the same seven seams from [chapter 02](./02_MoneyGuard_Full_Architecture_Walkthrough.md): missed revokes on Mover, deprovisioning that stops at the IdP, stale caches, code paths that bypass isolation, unowned machine identities, TTL-only invalidation, and logging failures but not successes.
5. Every strong answer names a **trade-off and a cost**, not just a choice.

---

⬅ [03 · Interview Questions & Answers](./03_Interview_Questions_And_Answers.md) | [Module 11 Index](./README.md)

⬅ Back to [Main README](../README.md) | 🏦 [The MoneyGuard Scenario](../SCENARIO.md)
