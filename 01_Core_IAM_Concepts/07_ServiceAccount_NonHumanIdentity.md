# 📘 Section : Service Accounts & Non-Human Identities (NHI)

### Context: When Code Needs an Identity at *MoneyGuard*

Not every identity in an enterprise is a human. In today’s microservices-driven architectures, machines often **interact with other machines far more than humans do**. Examples include:

* Microservices calling APIs to fetch transaction data.
* CI/CD pipelines deploying applications across multiple environments.
* Batch jobs aggregating customer data overnight.
* Background workers processing high-volume payments.

Each of these actors still requires **authentication (proving who they are), authorization (what they can do), rotation (updating credentials), auditing, and proper deprovisioning**. If humans are managed via IAM, Non-Human Identities (NHIs) are the **nervous system of the platform** — they are the unsung entities keeping workflows running, and mistakes here can be catastrophic.

---

## 1️⃣ What Is a Service Account?

A **service account** is a digital identity **used exclusively by software or automation**, not a person. It could represent:

* A backend service (e.g., PaymentProcessorService)
* An API client calling external SaaS
* A scheduled batch job
* CI/CD pipeline automation

**Key properties of service accounts:**

* They do **not require interactive login**, meaning no human types a password or uses MFA.
* Fully automated operations; tasks happen without user intervention.
* Scoped permissions: they should have **only the rights necessary** for their operation.
* Short-lived or dynamically managed credentials are preferred.

**MoneyGuard Example:** The “PaymentProcessorService” service account can initiate daily batch payments but cannot read customer PII or delete accounts. If mismanaged, a single leaked service account could allow attackers to move millions or compromise critical data undetected.

---

## 2️⃣ Why Service Accounts Are Dangerous

Service accounts are a favorite target for attackers because they are often:

* **Long-lived**: Many were created years ago and never rotated.
* **Over-privileged**: Often given “Admin” rights to simplify automation.
* **Poorly monitored**: No real-time visibility into their usage.
* **Shared**: Multiple systems use the same credentials.
* **Forgotten**: Orphaned accounts remain after projects or jobs are decommissioned.

> Imagine a stolen service account credential: there is **no MFA prompt, no user behavior anomaly**, nothing to raise alarms — it just works. This makes service accounts the #1 breach vector in most enterprises.

---

## 3️⃣ Common Anti-Patterns to Avoid

### ❌ Pattern 1: Shared Credentials Across Systems

```text
username: payment_service
password: stored in config.json
```

**Problems:**

* No way to track which system used the credential.
* No way to rotate it safely without breaking multiple systems.
* Any breach exposes multiple systems.

---

### ❌ Pattern 2: Long-Lived Secrets

* API keys or client secrets valid for years.
* Credentials embedded in source code or Docker images.
* Manual secrets copied across environments.

**Result:** If leaked, attackers gain permanent access with no expiration.

---

### ❌ Pattern 3: Using Human Accounts for Automation

> “Let’s just use Bob’s account for nightly ETL jobs.”

**Problems:**

* When Bob leaves, jobs either break or his account remains active.
* Creates uncontrolled privilege creep.
* Impossible to audit reliably.

---

## 4️⃣ Proper Types of Non-Human Identities

### A. Application (Service Principal) Identity

Used for **service-to-service communication**, such as API calls or microservices integration.
**Characteristics:**

* OAuth 2.0 Client Credentials flow.
* Short-lived tokens.
* Scoped permissions per API or operation.
* Can be integrated with Azure AD, Okta, or custom IAM.

**Example:** PaymentProcessorService gets a token to POST `/payments` but cannot GET `/customers`.

---

### B. Managed Workload Identity

Used when code runs **inside cloud infrastructure**, where secrets should not exist at all.
**Examples:**

* Kubernetes workloads (using K8s ServiceAccount + OIDC)
* Cloud VMs (Azure Managed Identity, AWS IAM Role)
* Serverless functions

**Key Feature:** Identity is **injected automatically** by the platform and rotated behind the scenes.

---

### C. Pipeline Identity

Used by **CI/CD pipelines and automation frameworks** (Jenkins, GitHub Actions, Terraform).
**Best Practices:**

* Each pipeline gets a unique identity.
* Scoped access: only to environments/resources the pipeline touches.
* Auditable: every deployment action is logged against the identity.

---

## 5️⃣ Authentication for Service Accounts

### OAuth 2.0 Client Credentials Flow

This is the **standard for machine-to-machine authentication**.

**Flow:**

1. Service account authenticates using its identity credentials.
2. No human is involved.
3. Receives a short-lived access token.
4. Calls downstream APIs using the token.

**Key Points:**

* No refresh tokens — short-lived access tokens minimize exposure.
* Scopes define **exact permissions**.
* Credentials can be certificates, federated identity tokens, or signed JWTs — never plain passwords.

---

### Why Passwords Are Not Used

Passwords imply:

* Human interaction.
* Manual rotation.
* Risk of leakage (hardcoded in config, scripts, images).

Instead, machines authenticate with:

* **Certificates** (signed key pairs)
* **Federated identity tokens**
* **Platform-issued short-lived tokens** (e.g., Azure Managed Identity)

---

## 6️⃣ Authorization for Service Accounts

Even more critical than authentication is **what the service can do**.

**Guiding Principle:** Service accounts should **never have admin by default**. They must follow the **least privilege model**, just like humans.

**Example:** `PaymentProcessorService`

| Action               | Allowed? |
| -------------------- | -------- |
| POST /payments       | ✅ Yes    |
| GET /customers       | ❌ No     |
| DELETE /transactions | ❌ No     |

Even if this account is compromised, the attacker cannot:

* Dump customer databases.
* Escalate privileges to other services.
* Pivot laterally across systems.

---

## 7️⃣ Secret Management

### Never Store Secrets In:

* Source code or Git repositories.
* Docker images or container layers.
* Environment variables in plaintext.

### Proper Storage

Use a **centralized secrets vault**:

* Automatic rotation.
* Access logging and versioning.
* Revocation when no longer needed.

At runtime:

* Secrets are injected into the environment temporarily.
* Used for a single operation.
* Never stored on disk in plaintext.

---

## 8️⃣ Lifecycle Management (Creation → Rotation → Decommissioning)

Service accounts require **full lifecycle discipline**:

### Creation

* Automated provisioning through the IAM system.
* Justified with purpose and scope.
* Tightly scoped access per job, pipeline, or service.

### Rotation

* Frequent, automatic, non-disruptive.
* Avoid long-lived static secrets.
* Align with enterprise security policies.

### Decommissioning

* Remove identity when the pipeline or service is retired.
* Orphaned accounts must be discovered and revoked.
* Ensures attackers cannot exploit forgotten identities.

> Orphaned service accounts are **silent attackers waiting to happen** — every idle account is a potential breach vector.

---

## 🏦 Real-World MoneyGuard Incident

**Incident:** A batch job, long retired, still had access to the transactions database.
**Impact:**

* Credentials were leaked via an old backup.
* Attacker replayed requests.
* Sensitive data was read without triggering alerts.

**Remediation:**

* Tie each service account to a **deployment lifecycle**.
* Automatic revocation when a pipeline or service is deleted.
* Enforce **short-lived, scoped tokens only**.

---

## ❓ FAQ: Service Accounts & NHIs

### Q1: Why can’t service accounts use MFA?

**A:** MFA is designed for humans. Machines rely on **short-lived credentials, scoped access, and secure token issuance** instead. Proper lifecycle and auditing provide security equivalent to MFA for NHIs.

---

### Q2: Should service accounts ever have admin rights?

**A:** Only temporarily via **Just-In-Time elevation** for approved workflows. Every action must be logged and auditable.

---

### Q3: How do we audit service account usage?

**A:** All token requests and API calls must include:

* Identity ID
* Scope and permissions granted
* Timestamp
* Target resource

This allows tracing any operation back to the specific account.

---

### Q4: What is the biggest mistake teams make with service accounts?

**A:** Treating them as “set and forget” entities. They are often created once and left unmanaged, leading to silent vulnerabilities.

---

### Q5: How do we prevent privilege creep for NHIs?

**A:** Combine **least privilege**, short-lived tokens, automatic revocation, and scoped access. Regular audits of service account roles and permissions are critical.

---

## 🧠 Mental Model

> Humans authenticate with MFA.
> Machines authenticate with **short-lived, scoped, non-reusable identities**.
> Any long-lived secret is a future breach waiting to happen.
> Treat service accounts as highly sensitive assets that require **the same rigor as human identities**.

---
