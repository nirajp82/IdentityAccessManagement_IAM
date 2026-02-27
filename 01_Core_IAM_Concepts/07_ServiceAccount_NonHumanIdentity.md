# 📘 Section: Service Accounts & Non-Human Identities (NHI)

### Context: When Code Needs an Identity at *MoneyGuard*

Not every identity in an enterprise is a human. In today’s microservices-driven architectures, machines often **interact with other machines far more than humans do**. Examples include:

* Microservices calling APIs to fetch transaction data.
* CI/CD pipelines deploying applications across multiple environments.
* Batch jobs aggregating customer data overnight.
* Background workers processing high-volume payments.

If humans are managed via standard IAM, Non-Human Identities (NHIs) are the **nervous system of the platform**—they are the unsung entities keeping workflows running. Each of these actors still requires authentication, authorization, rotation, auditing, and proper deprovisioning. Mistakes here can be catastrophic.

---

## 1️⃣ What Is a Service Account?

A **service account** is a digital identity **used exclusively by software or automation**, not a person. It could represent a backend service, an API client, a scheduled batch job, or a CI/CD pipeline.

**Key properties of Service Accounts:**

* **No Interactive Login:** No human types a password or uses MFA.
* **Fully Automated:** Operations happen without user intervention.
* **Strict Least Privilege:** They should have *only* the rights necessary for their specific operation.
* **Ephemeral:** Short-lived or dynamically managed credentials are heavily preferred over static passwords.

> **MoneyGuard Example:** The `PaymentProcessorService` account can initiate daily batch payments, but it cannot read customer PII or delete accounts.

---

## 2️⃣ Why Service Accounts Are Dangerous (The #1 Breach Vector)

Service accounts are a favorite target for attackers because they are often:

* **Long-lived:** Created years ago and never rotated.
* **Over-privileged:** Given generic "Admin" rights to simplify automation.
* **Poorly Monitored:** Lacking real-time visibility into their usage.
* **Shared:** Multiple systems use the exact same credentials.
* **Orphaned:** Left active long after projects or jobs are decommissioned.

**The Threat:** If an attacker steals a service account credential, there is **no MFA prompt and no behavioral anomaly alert** (like a login from a new country). It just silently works.

---

## 3️⃣ Common Anti-Patterns to Avoid

### ❌ Pattern 1: Shared Credentials Across Systems

* **The Flaw:** `username: payment_service` shared by three different apps.
* **The Impact:** You cannot track which system actually used the credential. You cannot rotate the password without breaking multiple systems at once.

### ❌ Pattern 2: Long-Lived Static Secrets

* **The Flaw:** API keys or client secrets valid for 10 years, embedded in source code, Docker images, or `config.json` files.
* **The Impact:** If leaked, attackers gain permanent access with no expiration.

### ❌ Pattern 3: Using Human Accounts for Automation

* **The Flaw:** *"Let’s just use Bob’s account for the nightly ETL jobs."*
* **The Impact:** When Bob leaves the company, the jobs either break, or his account is forced to remain active (violating HR termination policies).

---

## 4️⃣ Proper Types of Non-Human Identities

### A. Application Identity (Service Principals)

Used for **service-to-service communication** across external boundaries (e.g., API calls to SaaS).

* **Protocol:** OAuth 2.0 Client Credentials flow.
* **Features:** Short-lived tokens scoped to specific API operations.
* **Example:** `PaymentProcessorService` gets a token to `POST /payments` but cannot `GET /customers`.

### B. Managed Workload Identity (Cloud-Native)

Used when code runs **inside cloud infrastructure**, where static secrets should not exist at all. The identity is **injected automatically** by the platform and rotated behind the scenes.

* **AWS Virtual Machines (EC2):** Uses an **Instance Profile**. The AWS platform uses the Instance Metadata Service (IMDS) to automatically inject temporary credentials into the VM.
* **AWS Kubernetes (EKS):** Uses **IRSA** (IAM Roles for Service Accounts). AWS links a native K8s `ServiceAccount` directly to an AWS IAM Role via OIDC.
* **AWS Serverless/Containers:** Uses **Execution Roles** (Lambda) or **Task Roles** (ECS).
* **Azure Equivalent:** Managed Identities for Azure Resources.

### C. Pipeline Identity

Used by **CI/CD pipelines and automation frameworks** (GitHub Actions, Terraform).

* **Best Practices:** Each pipeline gets a unique identity. Access is scoped *only* to the environments or resources that specific pipeline touches.

---

## 5️⃣ Authentication & Authorization for NHIs

### 🔑 Authentication: The OAuth 2.0 Client Credentials Flow

This is the standard for machine-to-machine authentication. Machines do not use passwords.

1. The service authenticates using a certificate or federated token.
2. The Access Plane issues a **short-lived Access Token**.
3. The service calls downstream APIs using that token.

### 🛡️ Authorization: Strict Least Privilege

Service accounts should **never have admin by default**.

**Example Policy for `PaymentProcessorService`:**
| Target API | Action | Allowed? |
| :--- | :--- | :--- |
| `/payments` | `POST` | ✅ **Yes** |
| `/customers` | `GET` | ❌ **No** |
| `/transactions` | `DELETE` | ❌ **No** |

---

## 6️⃣ Secret Management & Lifecycle

### 🚫 Never Store Secrets In:

* Source code or Git repositories.
* Docker images or container layers.
* Plaintext environment variables.

### ✅ Proper Storage & Lifecycle:

* **The Vault:** Use a centralized secrets vault (e.g., HashiCorp Vault, AWS Secrets Manager) for automatic rotation and access logging.
* **JIT Injection:** Secrets are injected into the runtime environment temporarily and never stored on disk.
* **Deprovisioning:** When a pipeline or app is retired, its NHI must be immediately revoked. **Orphaned service accounts are silent attackers waiting to happen.**

---

## 🏦 Real-World *MoneyGuard* Incident (Case Study)

* **The Incident:** A batch job, long retired, still had an active service account with access to the transactions database.
* **The Breach:** The legacy credentials were leaked via an old, unsecured backup file. An attacker extracted them and replayed data extraction requests.
* **The Failure:** Because it was a "trusted" service account, reading sensitive data did not trigger any behavioral alerts.
* **The Remediation:** All service accounts are now tied to a strict **deployment lifecycle** with automatic revocation, enforcing short-lived tokens only.

---

## 🚦 In-Depth Architecture FAQ: Service Accounts

**Q1: Why can’t service accounts use MFA?**
**A:** MFA requires human interaction (biometrics, hardware keys). Machines rely on **cryptographic trust** (short-lived tokens, scoped access, and secure platform issuance) instead.

**Q2: Should service accounts ever have admin rights?**
**A:** Only temporarily via **Just-In-Time (JIT) elevation** for approved infrastructure-as-code deployments. Every action must be logged and auditable.

**Q3: How do we audit service account usage?**
**A:** All token requests and API calls must include the Identity ID, the exact scopes granted, a timestamp, and the target resource. This ensures full forensic traceability.

---

### 💡 Final Mental Model

> * Humans authenticate with **MFA**.
> * Machines authenticate with **short-lived, scoped, non-reusable tokens**.
> * Any long-lived static secret is a future breach waiting to happen. Treat service accounts as highly sensitive assets that require the exact same rigor as human identities.
> 
> 
