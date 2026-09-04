# Module 10: IaC & Deployment Security

Identity configuration *is* security configuration — an MFA policy toggled in an admin console is a security control changed with no review, no diff, and no audit trail worth the name. This module covers managing identity infrastructure as code with Terraform and a GitOps workflow, and the discipline that makes it safe: never letting the credentials that drive that pipeline touch a repository, an image, or a log.

🏦 The running examples are MoneyGuard onboarding **Acme Corp**, **Globex Inc**, and **MegaFreight Ltd** onto the Business Banking SaaS Portal, and Alex Kim deploying the wire-transfer microservice with no stored credentials at all.

## 📂 Chapters

| # | Chapter | What It Covers |
|---|---------|----------------|
| 01 | [Infrastructure as Code for Identity](./01_Infrastructure_As_Code_For_Identity.md) | Declarative IaC with Terraform/OpenTofu; providers for Okta, Auth0, and Entra ID; what belongs in code and what belongs to SCIM; the GitOps workflow with pull-request review and Atlantis; reusable tenant modules; state-file security; policy as code; drift detection; Crossplane compared. |
| 02 | [Secrets Management](./02_Secrets_Management.md) | Where secrets must never live; HashiCorp Vault and cloud secrets managers; dynamic vs. static secrets; solving the "secret zero" bootstrap problem with workload identity; OIDC federation for CI pipelines; runtime injection patterns; two-slot rotation; leak detection and response. |

## 🎯 Prerequisites

- **[Module 06 — Machine Identity & Workload Auth](../06_Machine_Identity_And_Workload_Auth/README.md)** is the most relevant prerequisite: Chapter 02's answer to the bootstrap problem is workload identity, which that module covers in depth.
- **[Module 05 — Identity Lifecycle & SCIM](../05_Identity_Lifecycle_And_SCIM/README.md)** explains why individual user accounts belong to SCIM rather than to Terraform.
- **[Module 09 — Resilience, Observability & SIEM](../09_Resilience_Observability_And_SIEM/README.md)** is where drift alerts, apply events, and secrets-manager audit logs end up, and where the detections over them are written.

New to the repo? Read [`SCENARIO.md`](../SCENARIO.md) first to meet MoneyGuard Bank and its cast.

## 🔑 The One-Paragraph Summary

Put every identity object — applications, scopes, roles, groups and their membership rules, MFA policies, federation connections — in Git, and let a scoped automation identity be the only thing with production write access, so humans propose changes in pull requests where a `mfa_required: true → false` diff is impossible to miss and policy-as-code checks block bad configuration regardless of who approves. Individual users stay out of it; they belong to the SCIM-driven lifecycle. Then keep the credentials that drive all of this out of the repository entirely: workloads authenticate with platform-attested identity rather than stored secrets, receive short-lived dynamic credentials that expire on their own, and every read is recorded in an audit log that feeds the SIEM — so a leaked credential is already worthless and every access is attributable.

---

📖 [Main README](../README.md) | 🏦 [The MoneyGuard Story](../SCENARIO.md)

⬅ Previous module: [09 — Resilience, Observability & SIEM](../09_Resilience_Observability_And_SIEM/README.md) | ➡ Next module: [11 — System Design & Interview Prep](../11_System_Design_And_Interview_Prep/README.md)
