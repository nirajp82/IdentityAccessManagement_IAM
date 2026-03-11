Below are two ready‑to‑rehearse scripts:

1) a **general IAM system‑design answer** (for any “high‑level IAM problem”) and  
2) a **service‑to‑service / AI‑agent IAM answer**,  

both explicitly calling out **least privilege, separation of duties, auditing, scalability, and reliability**. [designgurus](https://www.designgurus.io/answers/detail/how-do-you-enforce-leastprivilege-iam-at-scale-policy-generation-review)

Use them as templates; swap in your own examples.

***

## Script 1 – Generic IAM system design

You can use this for almost any “Design an IAM system for our multi‑tenant platform” prompt.

### 1. Clarify and frame

“Let me clarify the problem and constraints, then I’ll propose a model and architecture, and finally dig into security, scalability, and reliability.”

Ask briefly:

- Who are the identities: humans (customers, internal staff), services, possibly AI agents? [bigid](https://bigid.com/blog/iam-for-ai-agents/)
- What are the main resources: orgs, projects, datasets, models, jobs, secrets, billing? [bigid](https://bigid.com/blog/iam-for-ai-agents/)
- Any explicit compliance (SOC2/SOX/PCI/HIPAA) or regional constraints?
- Any latency/availability expectations for auth and policy checks? [educative](https://www.educative.io/courses/grokking-the-system-design-interview/non-functional-requirements-for-system-design-interviews)

Then say:

> “I’ll assume a multi‑tenant SaaS with an org → project hierarchy, human and service identities, and a need for fine‑grained permissions. I’ll start with the IAM model, then the architecture, then security/least‑privilege, scalability, and reliability.”

### 2. IAM model: identities, resources, permissions

“First, I want a clear **authorization model**.”

- Identities
  - Humans: end‑users, org admins, support. [interviews](https://www.interviews.chat/questions/identity-and-access-management-iam-architect)
  - Non‑humans: service accounts, CI/CD, background jobs, possibly AI agents. [bigid](https://bigid.com/blog/iam-for-ai-agents/)
  - Each identity has a stable ID and attributes (tenant, type, risk level, status). [bigid](https://bigid.com/blog/iam-for-ai-agents/)

- Resource hierarchy
  - Org: tenant boundary.
  - Project: logical grouping under org.
  - Resources: datasets, models, jobs, secrets, billing profiles.
  - Every resource has type, ID, owner (project/org), sensitivity, region. [bigid](https://bigid.com/blog/iam-for-ai-agents/)

- Permissions and roles (RBAC + optional ABAC)
  - Fine‑grained permissions: `dataset.read`, `dataset.write`, `job.submit`, `job.cancel`, `model.deploy`, `member.manage`, `billing.view`, `billing.manage`. [pomerium](https://www.pomerium.com/blog/iam-interview-questions-and-answers)
  - Roles:
    - Org‑level: `org_owner`, `org_admin`, `billing_admin`, `support_viewer`.
    - Project‑level: `project_owner`, `developer`, `viewer`, `mlops`.
  - RBAC baseline, with optional ABAC:
    - Conditions on attributes like environment, region, time, sensitivity. [carreersupport](https://carreersupport.com/identity-and-access-management-engineer-interview-questions/)

Call out principle:

> “This model makes least privilege practical: we expose small, composable permissions and assign only what’s required via roles, instead of broad wildcards.” [identitymanagementinstitute](https://identitymanagementinstitute.org/the-principle-of-least-privilege/)

### 3. Architecture: authN, authZ, enforcement

“Next, the architecture.”

- Authentication (IdP)
  - Central IdP supporting:
    - Username/password + MFA for smaller customers.
    - SSO via OIDC/SAML for enterprises. [pomerium](https://www.pomerium.com/blog/iam-interview-questions-and-answers)
  - Flow:
    - User authenticates → gets short‑lived access token (JWT) + refresh token.
    - JWT contains subject, org, possibly groups/roles. [carreersupport](https://carreersupport.com/identity-and-access-management-engineer-interview-questions/)

- Authorization service
  - Stateless **authZ service** exposing `IsAllowed(principal, action, resource)`. [hellointerview](https://www.hellointerview.com/community/questions/ai-agent-iam/cmgsd52f200gi07ad5f7knnsk)
  - Backed by:
    - Policy store: relational DB for identities, roles, bindings, policies. [interviews](https://www.interviews.chat/questions/identity-and-access-management-iam-architect)
    - Cache for permissions and policies.
  - Request flow:
    - API gateway or service validates JWT (sig, expiry, audience).
    - Extracts principal, org/project, action, resource ID.
    - Calls authZ service; it loads roles/policies (from cache/DB), evaluates RBAC/ABAC, returns allow/deny. [hellointerview](https://www.hellointerview.com/community/questions/ai-agent-iam/cmgsd52f200gi07ad5f7knnsk)

- Data model (policy store)
  - Tables:
    - `identities` (user/service), `organizations`, `projects`, `resources`.
    - `roles`, `permissions`, `role_permissions`.
    - `identity_role_bindings` (who has which role at which scope).
    - `policies` for any ABAC/custom rules. [interviews](https://www.interviews.chat/questions/identity-and-access-management-iam-architect)

- Enforcement pattern
  - Gateway: coarse checks (is user authenticated? allowed API family?).
  - Service middleware: fine‑grained resource‑level checks via authZ service.
  - Goal: no ad‑hoc checks scattered in business logic. [hellointerview](https://www.hellointerview.com/community/questions/ai-agent-iam/cmgsd52f200gi07ad5f7knnsk)

### 4. Security: least privilege, SoD, auditing

“Now I’ll focus on security, especially least privilege and separation of duties.”

- Least privilege at scale
  - Default‑deny: no implicit access; missing policy means deny. [deployu](https://deployu.ai/interviews/aws-interview/secure-iam-policies/)
  - Specific resources and actions, avoid `*` in policies. [aws.amazon](https://aws.amazon.com/blogs/security/techniques-for-writing-least-privilege-iam-policies/)
  - Permission boundaries (if we add delegation) to prevent admins from over‑granting. [designgurus](https://www.designgurus.io/answers/detail/how-do-you-enforce-leastprivilege-iam-at-scale-policy-generation-review)
  - Continuous loop:
    - Start with conservative templates.
    - Use access logs to refine policies (remove unused permissions). [aws.amazon](https://aws.amazon.com/blogs/security/strategies-for-achieving-least-privilege-at-scale-part-1/)
    - Periodic access reviews and recertifications. [identitymanagementinstitute](https://identitymanagementinstitute.org/the-principle-of-least-privilege/)

- Separation of duties
  - Distinct roles:
    - Org owners vs project owners vs billing admins.
    - Security admins (policy definition) vs platform operators (infra) vs developers. [deployu](https://deployu.ai/interviews/aws-interview/secure-iam-policies/)
  - Policies to prevent one identity from both creating and approving high‑risk changes (SoD). [designgurus](https://www.designgurus.io/answers/detail/how-do-you-enforce-leastprivilege-iam-at-scale-policy-generation-review)

- Credential & token security
  - Short‑lived access tokens; narrow‑scoped refresh tokens. [aiforiam.substack](https://aiforiam.substack.com/p/architecting-secure-agentic-ai-apps)
  - Service accounts:
    - Use short‑lived, issued credentials; minimize static keys.
    - Store secrets in a central secret manager; access via least‑privilege policies. [pomerium](https://www.pomerium.com/blog/iam-interview-questions-and-answers)
  - Enforce MFA for sensitive roles and actions. [deployu](https://deployu.ai/interviews/aws-interview/secure-iam-policies/)

- Auditing & logs (keep it concise unless they dive in)
  - What we log:
    - Identity lifecycle events (create/disable, role changes).
    - AuthN events (login success/failure, MFA).
    - AuthZ decisions on sensitive resources.
    - Admin/config & policy changes. [hoop](https://hoop.dev/blog/audit-logs-in-cloud-iam-best-practices-for-complete-secure-and-searchable-identity-audit-trails/)
  - Properties:
    - Structured JSON logs (actor, action, resource, result, IP, device, trace ID).
    - Central, immutable/tamper‑evident storage; encrypted, RBAC‑protected. [hoop](https://hoop.dev/blog/audit-logs-identity-and-access-management-iam/)
    - Dashboards + alerts for anomalies (failed logins, privilege escalations). [tryexponent](https://www.tryexponent.com/courses/security-analyst-technical-questions/cloud-audit-logs)

One line to emphasize:

> “Least privilege isn’t a one‑time config; it’s continuous: model → generate policies → observe via logs → tighten → periodically review.” [aws.amazon](https://aws.amazon.com/blogs/security/strategies-for-achieving-least-privilege-at-scale-part-1/)

### 5. Scalability

“On scalability, I focus on throughput of checks and growth in policies/resources.”

- Stateless authZ service
  - Horizontally scalable behind a load balancer. [interviews](https://www.interviews.chat/questions/identity-and-access-management-iam-architect)
- Caching
  - Cache user‑to‑role bindings and per‑resource ACLs.
  - In‑memory per instance, optional distributed cache.
  - Invalidation on role/policy updates (versioning or pub/sub), short TTLs as backup. [umatechnology](https://umatechnology.org/remote-logging-architecture-in-iam-policy-structures-validated-in-staging-environments/)
- Data partitioning
  - Partition policy store by tenant/org for very large scale.
  - This improves locality and limits blast radius. [hoop](https://hoop.dev/blog/audit-logs-in-cloud-iam-best-practices-for-complete-secure-and-searchable-identity-audit-trails/)
- Complexity control
  - Keep the policy language constrained for predictable, fast evaluation.
  - Add richer conditions only when justified. [dev](https://dev.to/fahimulhaq/guide-to-nonfunctional-requirements-for-system-design-interviews-4eje)

### 6. Reliability / availability

“Finally, reliability and failure modes.”

- Multi‑AZ, backups, and failover
  - AuthZ service and DB deployed across zones; automatic failover.
  - Regular backup/restore tests. [educative](https://www.educative.io/courses/grokking-the-system-design-interview/non-functional-requirements-for-system-design-interviews)
- IdP and token validation
  - Services cache IdP JWKS keys to validate tokens locally even if IdP is down; only new logins depend on IdP availability. [dev](https://dev.to/fahimulhaq/guide-to-nonfunctional-requirements-for-system-design-interviews-4eje)
- Failure modes for authZ
  - For sensitive operations, fail closed on authZ error/timeouts.
  - For low‑risk reads, return explicit errors so clients know it’s system‑level, not a permission denial.
  - Rate limiting and back‑pressure to protect the authZ service. [educative](https://www.educative.io/courses/grokking-the-system-design-interview/non-functional-requirements-for-system-design-interviews)
- Observability
  - Metrics: auth allow/deny, latency, error rate, cache hit ratio.
  - Traces that include auth checks.
  - SLOs and alerts around p95/p99 latency and availability. [dev](https://dev.to/fahimulhaq/guide-to-nonfunctional-requirements-for-system-design-interviews-4eje)

***

## Script 2 – Service‑to‑service / AI‑agent IAM design

Use this if they give you something like: “We have AI agents / services calling our APIs. Design IAM.”

### 1. Frame the problem

> “I’ll treat AI agents and services as first‑class identities, but with stricter guardrails than humans, focusing on scoped delegation, runtime controls, and full traceability.” [aiforiam.substack](https://aiforiam.substack.com/p/architecting-secure-agentic-ai-apps)

Clarify:

- Are agents long‑lived or ephemeral?
- Do they act on their own or strictly on behalf of users (delegation)? [aiforiam.substack](https://aiforiam.substack.com/p/architecting-secure-agentic-ai-apps)
- What data sensitivity levels are involved?

### 2. Identity model for agents/services

- Unique agent/service identities
  - Each agent/service has:
    - Stable ID, owning team/tenant, purpose, environment, risk tier. [bigid](https://bigid.com/blog/iam-for-ai-agents/)
  - Lifecycle:
    - Provisioned at deployment.
    - Attributes updated as context changes.
    - Deprovisioned automatically when job/app is retired. [bigid](https://bigid.com/blog/iam-for-ai-agents/)

- Entitlements and guardrails
  - Small, purpose‑built roles:
    - E.g., “test‑data‑only‑reader”, “metrics‑writer”, “deployment‑trigger”.
  - No broad “admin” for agents; no standing ability to grant roles. [aiforiam.substack](https://aiforiam.substack.com/p/architecting-secure-agentic-ai-apps)

### 3. Delegation vs. direct authority (critical point)

> “For safety, I’d prefer **delegation** over raw authority: agents request actions on behalf of a user, with limited scoped tokens, rather than owning broad credentials.” [aiforiam.substack](https://aiforiam.substack.com/p/architecting-secure-agentic-ai-apps)

- Delegated model (OAuth2‑style)
  - User authorizes the agent to operate within narrow scopes via an OAuth2‑like consent flow. [aiforiam.substack](https://aiforiam.substack.com/p/architecting-secure-agentic-ai-apps)
  - Agent gets a short‑lived token with explicit scopes (e.g., “read project X issues”, not “manage roles”). [aiforiam.substack](https://aiforiam.substack.com/p/architecting-secure-agentic-ai-apps)
  - Agent calls APIs with that token; IAM system enforces scopes. [aiforiam.substack](https://aiforiam.substack.com/p/architecting-secure-agentic-ai-apps)

- Governance workflow
  - For high‑risk operations (role changes, production data):
    - Agent can only **request** access or changes.
    - A human approver and the IAM back‑end perform actual provisioning, with SoD checks. [aiforiam.substack](https://aiforiam.substack.com/p/architecting-secure-agentic-ai-apps)

### 4. Runtime access controls and least privilege

“Runtime is where we enforce **least privilege + zero standing privilege**.”

- Zero standing privilege where possible
  - Agents don’t hold permanent powerful roles.
  - Tokens are short‑lived, scoped, and revocable. [bigid](https://bigid.com/blog/iam-for-ai-agents/)
- Context‑aware ABAC
  - Include environment, time, risk score, and data sensitivity in decisions.
  - Example: test agents can never access prod PII datasets regardless of their nominal role. [bigid](https://bigid.com/blog/iam-for-ai-agents/)
- Continuous policy refinement
  - Analyze logs of agent usage; reduce unnecessary permissions.
  - Automated drift detection when behavior diverges from expected patterns. [designgurus](https://www.designgurus.io/answers/detail/how-do-you-enforce-leastprivilege-iam-at-scale-policy-generation-review)

### 5. Auditing, detection, and forensics

“For agents, **traceability** is key: we must be able to answer ‘who did what, why, and under whose authority’.”

- Log content
  - Every agent action logged with:
    - Agent ID, owning user/team (if delegated), resource, action, result, context. [bigid](https://bigid.com/blog/iam-for-ai-agents/)
- Immutable, centralized logs
  - Same pattern as before: structured, centralized, immutable, and RBAC‑protected. [hoop](https://hoop.dev/blog/audit-logs-identity-and-access-management-iam/)
- Behavior analytics
  - Baseline expected agent behavior.
  - Alerts when an agent:
    - Touches unexpected data classes.
    - Spikes access volume.
    - Acts in unusual time or region. [ateam-oracle](https://www.ateam-oracle.com/aidriven-iam-audit-analysis-for-oracle-saas-from-raw-logs-to-actionable-security-insights-with-oci-log-analytics?SC=%3Aso%3Ach%3Aor%3Aawr%3A%3A%3A%3A&pcode=)

### 6. Scalability & reliability

- Large number of ephemeral identities
  - Automate provisioning/deprovisioning tied to infrastructure (e.g., per‑job or per‑pod identities). [bigid](https://bigid.com/blog/iam-for-ai-agents/)
- Token issuance at scale
  - Stateless token service, horizontally scalable.
  - Rate limiting and quotas per tenant and per agent type. [educative](https://www.educative.io/courses/grokking-the-system-design-interview/non-functional-requirements-for-system-design-interviews)
- Failure handling
  - If IAM/Token service is down:
    - Existing tokens continue to work until expiry.
    - New agents may fail to obtain credentials; systems degrade gracefully with clear errors. [dev](https://dev.to/fahimulhaq/guide-to-nonfunctional-requirements-for-system-design-interviews-4eje)

***

## Key “Staff‑level” bullet points to remember

You can sprinkle these phrases during the interview:

- “I’d enforce **least privilege** as a continuous process: conservative defaults, data‑driven policy tightening from logs, and scheduled access reviews.” [datadoghq](https://www.datadoghq.com/blog/iam-least-privilege/)
- “We’d design roles and permission boundaries to avoid wildcards and limit blast radius, especially for internal admins and agents.” [aws.amazon](https://aws.amazon.com/blogs/security/techniques-for-writing-least-privilege-iam-policies/)
- “Separation of duties ensures no single identity can both grant and approve high‑risk access, which is crucial for passing audits and reducing insider risk.” [deployu](https://deployu.ai/interviews/aws-interview/secure-iam-policies/)
- “For non‑functional requirements, I think explicitly about performance, availability, scalability, and operational complexity, and I’m happy to trade some flexibility to keep the system understandable and safe.” [educative](https://www.educative.io/courses/grokking-the-system-design-interview/non-functional-requirements-for-system-design-interviews)

If you paste your actual resume bullets or a specific Lambda job description, I can tailor these scripts with 1–2 concrete war stories that fit your background and sound natural.
