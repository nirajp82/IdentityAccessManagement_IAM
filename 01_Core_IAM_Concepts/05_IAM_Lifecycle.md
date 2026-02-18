# 📘 The IAM Lifecycle (Joiner · Mover · Leaver)

### Context: HR-Driven Identity at *MoneyGuard*

The **Joiner–Mover–Leaver (JML)** lifecycle defines how identities are **created, changed, and destroyed** inside an enterprise.
In a mature IAM program, **HR is the source of truth**, and IAM systems react automatically—without tickets, emails, or manual scripts.

Your objective as an IAM architect is **zero-touch identity lifecycle automation**, ensuring:

* Users get access **on time**
* Access changes **when roles change**
* Access is removed **instantly when employment ends**

Failures in JML are responsible for **most real-world breaches**.

---

## 1️⃣ Joiner — Identity Birth (Provisioning)

### Scenario: Alice Joins MoneyGuard as a Senior Developer

**Trigger (Source of Truth):**
HR creates Alice’s employee record in Workday with:

* Start Date: Feb 1
* Department: Engineering
* Location: US
* Job Code: Senior Developer

HR does **not** open an IT ticket.

---

### IAM Automation Flow

1. **Detection**

   * IAM platform (e.g., SailPoint or Okta) polls or receives an event from HRIS

2. **Identity Creation**

   * Active Directory user created
   * Synced to cloud directory (Azure AD / Entra ID)
   * Immutable employee ID assigned

3. **Birthright Access (Non-Negotiable)**
   Based on *department + employment type*:

   * Corporate email
   * Slack access
   * VPN access
   * Engineering wiki
   * MFA enrollment enforced

4. **Application Provisioning**

   * GitHub org membership
   * CI/CD access
   * Dev environment access
   * All via **SCIM provisioning**

---

### Day-1 Experience (Why This Matters)

Alice logs in on Day 1 and everything works.

* No tickets
* No waiting
* No insecure temporary passwords

This is not convenience — this is **security through automation**.

---

### Joiner Failure Mode (What Goes Wrong)

* Manual user creation
* Missed MFA enrollment
* Over-privileged default access
* Shared onboarding spreadsheets

This leads directly to **orphaned accounts and audit failures**.

---

### Joiner FAQ

**Q: What is “Birthright Access”?**
A: A minimal, non-optional access bundle granted to every employee based on verified HR attributes.

**Q: Why not give broad access on Day 1?**
A: Because joiners are the **highest-risk identities** (new, untrained, phishable).

---

## 2️⃣ Mover — Identity Mutation (Change Management)

### Scenario: Alice Becomes an Engineering Manager

HR updates Alice’s job profile:

* Title: Engineering Manager
* Reports: 8 developers
* Cost Center: Engineering Management

This change is **not cosmetic** — it is a security event.

---

### IAM Automation Flow

1. **Change Detection**

   * IAM detects attribute deltas from HR

2. **Access Grants**

   * Management dashboards
   * Budget tools
   * Performance review systems

3. **Access Revocation (Most Important Step)**

   * Removes write access to restricted codebases
   * Removes break-glass admin eligibility
   * Removes individual-contributor entitlements

---

### Why Movers Are Dangerous

Mover failures cause **privilege creep**:

> “Alice used to need that access… so we never removed it.”

Over time:

* Users accumulate **toxic combinations**
* No one understands who has what
* Audits fail
* Breaches become inevitable

---

### Mover Best Practice

> **Every role change must include access removal — not just access addition**

If nothing is removed, your IAM system is broken.

---

### Mover FAQ

**Q: Why is revocation more important than granting?**
A: Because excessive access is invisible until exploited.

**Q: Should access changes be instant?**
A: Yes. Delayed revocation is equivalent to no revocation.

---

## 3️⃣ Leaver — Identity Death (Deprovisioning)

### Scenario: Alice Leaves MoneyGuard

HR marks Alice as:

* Status: Terminated
* Effective Time: 5:00 PM Today

This is a **security kill-switch event**.

---

### IAM Automation Flow

1. **Immediate Disable**

   * AD account disabled
   * Cloud identity blocked

2. **Session Termination**

   * OAuth refresh tokens revoked
   * Active web/mobile sessions killed

3. **Device Enforcement**

   * Corporate laptop locked
   * BYOD data wiped via MDM

4. **Downstream Deprovisioning**

   * SaaS accounts disabled
   * API tokens revoked
   * SSH keys invalidated

---

### Why Timing Matters

Most insider breaches happen:

* **Within hours of termination**
* Before manual IT actions occur

Automated deprovisioning removes that window completely.

---

### Leaver FAQ

**Q: Why revoke tokens instead of waiting for expiration?**
A: Because valid tokens are equivalent to valid credentials.

**Q: Should access be removed before exit interview?**
A: Yes. Access removal should precede or coincide with termination notification.

---

## 🏦 Enterprise Use Case: Failed Leaver Incident

**Incident:**
A contractor retained VPN access for 3 weeks after exit and copied customer data.

**Root Cause:**
HR termination was not integrated with IAM.

**Fix:**
HR → IAM → SCIM deprovisioning pipeline enforced with real-time triggers.
