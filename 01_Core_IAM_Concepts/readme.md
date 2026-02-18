# 📘 Chapter 1: Core IAM Concepts & Architecture

**Focus:** Building the architectural foundation of Identity and Access Management (IAM). Moving beyond simple "login pages" to understanding the enterprise security perimeter.

> **Scenario Context:** Throughout this chapter, we use **"MoneyGuard"**, a hypothetical Global FinBank enterprise, to illustrate how these concepts apply in a regulated, distributed environment (On-Premise + Cloud).

---

## 📂 Topic Index

### [01_Intro.md](./01_Intro.md) | What is IAM?
* **Concept:** Understanding IAM as the "New Security Perimeter."
* **Key Takeaway:** Identity is the control plane. In a distributed world (Cloud/Remote), firewalls aren't enough; we must secure the *who* (Identity) rather than just the *where* (Network).
* **Enterprise Shift:** Moving from Siloed Identities (separate user DBs per app) to Centralized Identity Management.

### [02_AAA_Framework.md](./02_AAA_Framework.md) | The Holy Trinity of Security
* **Authentication (AuthN):** "Who are you?" (Verifying the Identity/Principal).
* **Authorization (AuthZ):** "What can you do?" (Verifying Permissions/Scopes).
* **Accounting (Auditing):** "What did you do?" (Non-repudiation & Compliance).
* **Critical Distinction:** AuthN creates the token; AuthZ evaluates the token.

### [03_AccessControlModels.md](./03_AccessControlModels.md) | RBAC, ABAC, & PBAC
* **RBAC (Role-Based):** Assigning permissions to static Roles (e.g., "Teller"). Good for coarse-grained access.
* **ABAC (Attribute-Based):** Policies based on User, Resource, and Environment attributes (e.g., "Teller" + "Time=9am" + "Location=Office").
* **PBAC (Policy-Based):** Logic-driven authorization using centralized policy engines (e.g., OPA).
* **Evolution:** Start with RBAC; layer ABAC/PBAC for complex "MoneyGuard" scenarios.

### [04_Principle_of_Least_Privilege_(PoLP).md](./04_Principle_of_Least_Privilege_(PoLP).md) | Security Hardening
* **The Rule:** Users/Services should have the *exact* minimum access required, and only for the time needed.
* **Implementation:** Moving from "Standing Access" (Permanent Admin) to "Just-In-Time (JIT) Access" (Temporary Admin).
* **Outcome:** Minimizes the "Blast Radius" if an identity is compromised.

### [05_IAM_Lifecycle.md](./05_IAM_Lifecycle.md) | Joiner, Mover, Leaver (JML)
* **The Pipeline:** Automating the flow of users through the organization.
* **Joiner:** Automated provisioning (Birthright access) triggered by HR.
* **Mover:** Detecting role changes to prevent "Privilege Creep" (accumulating access).
* **Leaver:** The "Kill Switch"—automated, immediate revocation of all access upon termination.

### [06_Identity_Repositories.md](./06_Identity_Repositories.md) | The Source of Truth
* **Active Directory (AD DS):** The legacy on-prem standard. Speaks LDAP/Kerberos. Hierarchical (OUs).
* **Azure AD (Entra ID):** The modern cloud standard. Speaks OIDC/OAuth/SAML. Flat structure (Graph API).
* **The Architecture:** Using specific repositories for specific needs (AD for legacy apps, Entra ID for SaaS/Cloud).

---

## 🧠 Chapter 1 Capstone: The "MoneyGuard" Architecture

By the end of Day 1, we have designed the high-level architecture for **MoneyGuard Bank**:

1.  **Centralized Identity:** We use a Federation Hub (Azure AD/Okta) so apps don't manage their own users.
2.  **Lifecycle Automation:** HR (Workday) triggers the creation of the AD account (Joiner).
3.  **Access Model:** We use **RBAC** (Teller Role) for basic access, enhanced by **ABAC** (Location/Device checks) for sensitive transactions.
4.  **Audit Trail:** Every access request is logged (Accounting) to satisfy banking regulations.

---

## 💡 Practice Questions (Self-Check)

1.  **How does IAM improve enterprise security?**
    * *Short Answer:* It reduces the attack surface by enforcing Least Privilege and centralizing control, ensuring a compromised user doesn't compromise the whole system.
2.  **What is the difference between IAM and PAM?**
    * *Short Answer:* IAM manages standard users (Email, Portal). PAM (Privileged Access Management) manages "Super Users" (Admins, DBAs) with vaulting and session recording.
3.  **Why separate Authentication from Authorization?**
    * *Short Answer:* Scalability. AuthN (IdP) is done once to establish identity. AuthZ (Policy) is done on every request to ensure the user still has permission *right now*.

---

