# 📘 Section: Identity Repositories (AD · LDAP · Cloud Directory)

### Context: Where Identity Actually Lives

An identity is only as secure as **where it is stored and how it is accessed**.

Understanding identity repositories is mandatory for designing authentication and authorization correctly.

---

## 1️⃣ Active Directory (AD DS)

### What It Is

* **What it is:** The on-premise "King" of identity. It is a database + a set of services (Kerberos, DNS, LDAP).
* **Structure:** Hierarchical (Tree). Users sit in **OUs** (Organizational Units) like `Corp > US > Users`.
* **Protocol:** **LDAP** (Lightweight Directory Access Protocol) and **Kerberos**.
* **Dev Context:** You access this via `System.DirectoryServices`.
* **MoneyGuard Use Case:** Used for logging into physical laptops, Wi-Fi, and legacy SQL Servers.

### Structure

* Forests → Domains → OUs
* Hierarchical and policy-driven

### What It’s Used For at MoneyGuard

* Laptop login
* Wi-Fi authentication
* Legacy Windows apps
* On-prem SQL Server auth

### Developer Reality

* Accessed via LDAP queries
* Kerberos handles authentication
* Group membership drives authorization

---

## 2️⃣ LDAP (Protocol, Not a Product)

### What LDAP Is

LDAP is **how you talk to directories**.

* **What it is:** LDAP is not a product; it is the **language** used to talk to AD (and other directories like OpenLDAP).
* **The Analogy:** AD is the *Database* (SQL Server); LDAP is the *SQL language* (`SELECT * FROM...`).
* **Dev Context:** You write "Connection Strings" like `LDAP://DC=MoneyGuard,DC=com`.

### Mental Model

| Concept   | Analogy     |
| --------- | ----------- |
| Directory | Database    |
| LDAP      | SQL         |
| DN        | Primary key |

---

### Why LDAP Still Exists

* **MoneyGuard Use Case:** Old Linux servers or Java apps often speak raw LDAP to authenticate users against AD.
* Network appliances
* Legacy Java applications

---

## 3️⃣ Cloud Directory (Azure AD / Entra ID)

### What It Is (And Is NOT)
* **What it is:** This is **NOT** just "AD in the cloud." It is a completely different beast.
* **No Kerberos/NTLM:** It speaks **HTTP/REST** (OIDC, OAuth, SAML).
* **Flat Structure:** No OUs. Just Users and Groups.
* **Dev Context:** You access this via **Microsoft Graph API** (REST calls).

### What It Uses

* OAuth 2.0
* OpenID Connect
* SAML
* REST APIs

### What It’s Used For at MoneyGuard

* Office 365
* SaaS apps
* Cloud workloads
* Modern APIs


* **MoneyGuard Use Case:** Used for Office 365, logging into the Azure Portal, and authenticating your modern .NET Core Microservices.
  
**Key Architect Takeaway:** In a Hybrid Enterprise (MoneyGuard), you sync on-prem AD to Azure AD using **Azure AD Connect**. AD remains the "Source of Truth" for users; Azure AD is the "Projection" for the cloud.

---

### Hybrid Reality (Critical)

Most enterprises operate **hybrid identity**:

* AD = Source of truth
* Cloud directory = Access plane
* Sync bridges them (one-way authority)

Breaking this model causes:

* Identity drift
* Duplicate users
* Security failures

---

**Q1: Should apps talk directly to AD?**
**A:** Modern applications should avoid talking directly to Active Directory (AD). Instead, use cloud identity providers (like Azure AD, Okta) to centralize authentication and reduce direct dependency on legacy systems.

**Q2: Why keep AD at all?**
**A:** AD remains important for legacy systems, device authentication, and the Windows security model, which many enterprise applications still rely on.

**Q3: How does IAM improve enterprise security?

**Answer:** IAM (Identity and Access Management) shifts security from **perimeter-based** (firewalls, network defenses) to **identity-based** security, which aligns with the **Zero Trust** model—trust nothing, verify everything.

1. **Attack Surface Reduction:**
   By enforcing **Least Privilege**, a compromised user (e.g., Alice) cannot affect the entire enterprise. The attacker is limited to only Alice’s assigned rights.

2. **Credential Protection:**
   Centralizing authentication with **SSO (Single Sign-On)** reduces weak or reused passwords across many apps. Global **MFA (Multi-Factor Authentication)** further strengthens account security.

3. **Visibility and Auditability:**
   IAM provides a single pane of glass for monitoring access. You can quickly answer *"Who accessed sensitive data?"* instead of manually auditing logs across multiple systems.


**Q4: What is the difference between IAM and PAM?

**Answer:** IAM and PAM are both critical for enterprise security, but they serve different purposes and protect different types of accounts.

* **IAM (Identity & Access Management):**
  Focuses on **general user access**. It ensures that regular employees can access necessary tools and applications for daily work—email, intranet, SaaS apps, etc.
  *(Example tools: Okta, Azure AD)*

* **PAM (Privileged Access Management):**
  Focuses on **high-risk accounts**—those with the power to control critical systems. These include admin accounts, root/system accounts, and service accounts. Compromise of these accounts could threaten the entire organization.
  *(Example tools: CyberArk, Delinea)*

**Key differences:**

| Feature               | IAM                                | PAM                                             |
| --------------------- | ---------------------------------- | ----------------------------------------------- |
| **Purpose**           | Productivity for general employees | Security for high-privilege accounts            |
| **Accounts Managed**  | Standard user accounts             | Admin, root, and service accounts               |
| **Security Controls** | SSO, MFA, role-based access        | Vaulting (password rotation), session recording |
| **Analogy**           | Front door key to the office       | Combination to the bank vault                   |

**In short:** IAM controls everyday access and keeps users productive, while PAM protects the “keys to the kingdom” and secures critical systems from misuse or compromise.

---

**Q5: What is provisioning & deprovisioning?

**Answer:**

* **Provisioning:**
  The **automated creation and management** of user accounts and attributes in target systems. For example, when someone joins the Sales team, IAM automatically creates a Salesforce account and assigns the correct permissions. This ensures employees have the right tools on **Day 1**.

* **Deprovisioning:**
  The **automated removal or disabling** of accounts when employees leave or change roles. This is a critical security control—ensuring access is revoked **immediately** to prevent data breaches or unauthorized access. Modern provisioning uses the **SCIM standard** to enforce this across all SaaS applications in real time.

---

# 🧠 Day 1 Consolidated Use Cases

## Use Case 1: Auditor Requests Access Review

IAM provides:

* Who has access
* Why they have it
* When it was granted
* Who approved it

Without IAM → weeks of spreadsheets
With IAM → minutes

---

## Use Case 2: Insider Threat Mitigation

* Least privilege limits damage
* JML automation removes lingering access
* Logs provide forensic traceability

---

## Use Case 3: SaaS Explosion Control

Without IAM:

* Manual onboarding
* Shadow IT
* No visibility

With IAM:

* SCIM provisioning
* Central policy enforcement
* Unified offboarding

---

## Final Mental Model (Memorize This)

> IAM is not login screens — it is **automated control of identity creation, change, and destruction**, enforced consistently across humans, systems, and applications.
