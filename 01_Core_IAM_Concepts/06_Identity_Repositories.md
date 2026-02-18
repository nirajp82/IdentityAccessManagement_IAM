# 📘 Section: Identity Repositories (AD · LDAP · Cloud Directory)

### Context: Where Identity Actually Lives

An identity is only as secure as **where it is stored and how it is accessed**.

Understanding identity repositories is mandatory for designing authentication and authorization correctly.

---

## 1️⃣ Active Directory (AD DS)

### What It Is

* On-premise identity authority
* Directory + authentication services
* Kerberos + LDAP based

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

* AD speaks LDAP
* OpenLDAP speaks LDAP
* LDAP is the query language

### Mental Model

| Concept   | Analogy     |
| --------- | ----------- |
| Directory | Database    |
| LDAP      | SQL         |
| DN        | Primary key |

---

### Why LDAP Still Exists

* Linux servers
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

## ❓ Identity Repository FAQ

**Q: Should apps talk directly to AD?**
A: Modern apps should not. Use cloud identity providers.

**Q: Why keep AD at all?**
A: Legacy systems, device auth, and Windows security model.

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
