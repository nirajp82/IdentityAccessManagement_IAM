# 📘 Chapter 1: Core IAM Concepts & Architecture

**Focus:** Understanding the foundations of Identity and Access Management (IAM). We go beyond “login pages” to see how IAM protects the enterprise in a distributed environment.

> **Scenario Context:** In this chapter, we use **"MoneyGuard"**, a fictional global bank, to show how IAM works in a real-world environment with both on-premise and cloud systems.

---

## 📂 Topic Index

### [01_Intro.md](./01_Intro.md) | What is IAM?

* **Concept:** IAM is the **new security perimeter**. Instead of just securing networks, we secure *who* can access what.
* **Key Takeaway:** Identity is the control plane. Firewalls alone aren’t enough in a cloud/remote world.
* **Enterprise Shift:** From siloed identities (separate accounts per app) to **centralized identity management**.

### [02_AAA_Framework.md](./02_AAA_Framework.md) | The Holy Trinity of Security

* **Authentication (AuthN):** *Who are you?* — Verifying the user’s identity.
* **Authorization (AuthZ):** *What can you do?* — Checking permissions or scopes.
* **Accounting (Auditing):** *What did you do?* — Tracking actions for compliance.
* **Critical Distinction:** AuthN issues the token; AuthZ decides what the token allows.

### [03_AccessControlModels.md](./03_AccessControlModels.md) | RBAC, ABAC, & PBAC

* **RBAC (Role-Based Access Control):** Permissions based on roles (e.g., “Teller”). Good for basic, broad access control.
* **ABAC (Attribute-Based Access Control):** Permissions based on attributes (e.g., role + location + time of day).
* **PBAC (Policy-Based Access Control):** Centralized, rule-based authorization (e.g., OPA engine).
* **Evolution:** Start with RBAC; add ABAC/PBAC for complex scenarios at **MoneyGuard**.

### [04_Principle_of_Least_Privilege_(PoLP).md](./04_Principle_of_Least_Privilege_%28PoLP%29.md) | Security Hardening

* **Rule:** Give users or services only the access they need—and only for the time they need it.
* **Implementation:** Move from permanent admin access to **Just-In-Time (JIT) access**.
* **Outcome:** Limits the damage if an account is compromised.

### [05_IAM_Lifecycle.md](./05_IAM_Lifecycle.md) | Joiner, Mover, Leaver (JML)

* **Pipeline:** Automates user access throughout the employee lifecycle.
* **Joiner:** HR triggers account creation with appropriate access (*Birthright Access*).
* **Mover:** Detects role changes to prevent **Privilege Creep** (access accumulating unnecessarily).
* **Leaver:** Automatically revokes all access when someone leaves—acts as a “kill switch.”

### [06_Identity_Repositories.md](./06_Identity_Repositories.md) | The Source of Truth

* **Active Directory (AD DS):** Legacy on-prem solution. Uses LDAP/Kerberos. Organizes users in a hierarchy (OUs).
* **Azure AD (Entra ID):** Modern cloud solution. Uses OIDC/OAuth/SAML. Flat structure via Graph API.
* **Architecture Tip:** Use AD for legacy apps, Entra ID for cloud/SaaS apps.

---

## 🧠 Chapter 1 Capstone: The "MoneyGuard" Architecture

By the end of this chapter, the high-level design of **MoneyGuard Bank** includes:

1. **Centralized Identity:** Federation Hub (Azure AD/Okta) ensures apps do not manage individual users.
2. **Lifecycle Automation:** HR (Workday) triggers AD account creation for new employees.
3. **Access Model:** RBAC for basic roles (e.g., Teller) + ABAC for sensitive actions (location/device checks).
4. **Audit Trail:** All access requests are logged for regulatory compliance.

---

## 💡 Practice Questions (Self-Check)

1. **How does IAM improve enterprise security?**
   *Short Answer:* It reduces the attack surface by enforcing **Least Privilege** and centralizing control, so a single compromised user cannot jeopardize the whole system.

2. **What is the difference between IAM and PAM?**
   *Short Answer:*

   * **IAM** manages normal users (Email, Portal, SaaS apps).
   * **PAM (Privileged Access Management)** protects **super users** (Admins, DBAs, service accounts) using **vaulting** (password rotation) and **session recording**.

3. **Why separate Authentication from Authorization?**
   *Short Answer:*

   * **Authentication (AuthN)** confirms identity once.
   * **Authorization (AuthZ)** checks permissions **every request**, ensuring the user is still allowed to perform the action at that moment.

---

## 1️ IAM vs PAM Diagram

```
          ┌───────────────────────┐
          │      IAM (Identity)   │
          └───────────────────────┘
                    │
          Manages regular users
       (Email, Portal, SaaS apps)
                    │
             Role-Based Access
                    │
         -------------------------
                    │
          ┌───────────────────────┐
          │      PAM (Privileged) │
          └───────────────────────┘
                    │
          Manages high-risk accounts
      (Admins, DBAs, Service Accounts)
                    │
         Vaulting + Session Recording
```

**Key Idea:**

* **IAM** = general users, productivity, day-to-day access
* **PAM** = privileged users, high-risk, super-sensitive controls

---

## 2️ MoneyGuard IAM Lifecycle (Joiner → Mover → Leaver)

```
           ┌───────────────┐
           │   Joiner      │
           │ (New Employee)│
           └──────┬────────┘
                  │
   HR triggers provisioning
                  │
           ┌──────┴────────┐
           │    Mover      │
           │ (Role Change) │
           └──────┬────────┘
                  │
   Update permissions / prevent
        privilege creep
                  │
           ┌──────┴────────┐
           │   Leaver      │
           │ (Exit/Leave)  │
           └───────────────┘
                  │
       Access revoked immediately
      (No lingering permissions)
```

**Key Idea:**

* Automation ensures **security and compliance** without relying on manual steps.
* **Joiner** = get access, **Mover** = adjust access, **Leaver** = remove access instantly.

---

💡 **Optional Enhancement:**
We can combine both into **one infographic** showing IAM, PAM, and JML flow together:

* IAM covers all regular users (Joiner → Mover → Leaver).
* PAM overlays the same flow for privileged accounts with vaulting/session logging.
