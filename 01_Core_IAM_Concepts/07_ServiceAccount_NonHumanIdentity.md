# 📘 Section: Service Accounts & Non-Human Identities (NHI)

### Context: When Code Needs an Identity at *MoneyGuard*

Not all identities are humans.

In modern systems, **machines talk to machines far more often than humans do**:

* Microservices calling APIs
* CI/CD pipelines deploying code
* Batch jobs pulling data
* Background workers processing transactions

These actors still need **authentication, authorization, rotation, auditing, and revocation**.

This is where **Non-Human Identities (NHIs)** come in.

If human IAM controls *employees*, NHI controls *the nervous system of the platform*.

---

## 1️⃣ What Is a Service Account?

A **service account** is an identity **used by software**, not a person.

It represents:

* An application
* A workload
* A background process
* A pipeline

Key properties:

* No interactive login
* No password typing
* No MFA prompt
* Fully automated

At MoneyGuard, service accounts move **real money** — mistakes here are catastrophic.

---

## 2️⃣ Why Service Accounts Are Extremely Dangerous

Service accounts are often:

* Long-lived
* Over-privileged
* Poorly monitored
* Shared across systems
* Forgotten after creation

This makes them the **#1 breach vector in enterprises**.

> If an attacker steals a service account credential, there is **no user to phish, no MFA to bypass, and no behavior anomaly** — it just works.

---

## 3️⃣ Common Service Account Anti-Patterns (Do NOT Do This)

### ❌ Pattern 1: Shared Credentials

```text
username: payment_service
password: stored in config.json
```

Problems:

* No attribution
* No rotation
* No blast radius control

---

### ❌ Pattern 2: Long-Lived Secrets

* Client secrets valid for years
* API keys copied across environments
* Secrets embedded in source code

Once leaked, the attacker has **permanent access**.

---

### ❌ Pattern 3: Using Human Accounts for Automation

Example:

> “Let’s just use Bob’s account for the nightly job.”

When Bob leaves:

* Job breaks
* Or worse — Bob’s account is never disabled

---

## 4️⃣ Proper Types of Non-Human Identities

### A. Application (Service Principal) Identity

Used when:

* One service calls another
* APIs authenticate each other

Characteristics:

* OAuth 2.0 based
* Short-lived tokens
* Fine-grained scopes

---

### B. Managed Workload Identity

Used when:

* Code runs inside cloud infrastructure
* No secrets should exist at all

Examples:

* Kubernetes workloads
* Cloud VMs
* Serverless functions

The platform injects identity automatically.

---

### C. Pipeline Identity

Used by:

* CI/CD systems
* Infrastructure automation
* Terraform deployments

Highly sensitive — often needs **elevated but tightly scoped access**.

---

## 5️⃣ Authentication for Service Accounts (How They Prove Who They Are)

### OAuth 2.0 Client Credentials Flow

This is the **standard for machine-to-machine auth**.

Flow summary:

1. Service authenticates using its identity
2. No user involved
3. Receives an access token
4. Calls downstream APIs

Important properties:

* No refresh tokens
* Short token lifetimes
* Scoped access

---

### Why Passwords Are Not Used

Passwords imply:

* Human interaction
* Manual rotation
* High leakage risk

Machines should authenticate using:

* Certificates
* Federated identity
* Platform-issued tokens

---

## 6️⃣ Authorization for Service Accounts (What They Are Allowed to Do)

Service accounts **must never be “admin by default.”**

Authorization should be:

* Explicit
* Narrow
* Context-aware

### Good Example (MoneyGuard)

**PaymentProcessorService**

* Can: `POST /payments`
* Cannot: `GET /customers`
* Cannot: `DELETE /transactions`

Even if compromised, the attacker:

* Cannot dump databases
* Cannot escalate privileges
* Cannot pivot laterally

---

## 7️⃣ Secret Management (Where Credentials Live)

### Never Store Secrets In:

* Source code
* Git repos
* Docker images
* Environment variables in plaintext

---

### Proper Secret Storage

Use a centralized secrets system:

* Automatic rotation
* Access logging
* Versioning
* Revocation

At runtime:

* Secrets are injected
* Used briefly
* Never written to disk

---

## 8️⃣ Rotation & Lifecycle Management

Service accounts must follow **the same lifecycle discipline as humans**.

### Creation

* Automated
* Justified
* Scoped

### Rotation

* Regular (days, not years)
* Automatic
* Non-disruptive

### Decommissioning

* On service deletion
* On pipeline removal
* On environment teardown

Orphaned service accounts are **silent attackers waiting to happen**.

---

## 🏦 Real-World MoneyGuard Incident

**Incident:**
A deprecated batch job still had access to the transaction database.

**What Happened:**

* Job credentials leaked via old backup
* Attacker replayed requests
* Fraudulent reads went undetected

**Fix:**

* Service account tied to deployment lifecycle
* Automatic revocation on pipeline deletion
* Short-lived tokens only

---

## ❓ FAQ: Service Accounts & NHIs

### Q1: Why can’t service accounts use MFA?

**A:** MFA requires human interaction. Security for NHIs comes from **short-lived credentials and strict scoping**, not prompts.

---

### Q2: Should service accounts ever have admin rights?

**A:** Only temporarily, via Just-In-Time elevation, and only for automation workflows with full auditing.

---

### Q3: How do we audit service account usage?

**A:** Every token issuance and API call must be logged with:

* Identity ID
* Scope
* Timestamp
* Target resource

---

### Q4: What is the biggest mistake teams make with service accounts?

**A:** Treating them as “set and forget” identities.

---

## 🧠 Mental Model (Lock This In)

> Humans authenticate with MFA.
> Machines authenticate with **short-lived, scoped, non-reusable identities**.
> Any long-lived secret is a future breach.

---
