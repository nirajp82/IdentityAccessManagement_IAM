# Secrets Management

## Overview

A **secret** is any credential that grants access: a database password, an API token, a private signing key, a TLS certificate key, an OAuth client secret. Chapter 1 moved MoneyGuard's identity configuration into Git — but the credentials that let the pipeline *apply* that configuration must never go there. This chapter is about where secrets actually live, how applications get them at runtime, and how you rotate them without an outage.

The governing rule is simple and absolute: **secrets never live in code, in a container image, in a Git repository, in a CI configuration file, or in a log.** They live in a purpose-built secrets manager and are injected at runtime, ideally as short-lived credentials that expire on their own.

🏦 **MoneyGuard scenario:** Alex Kim is deploying the wire-transfer microservice. It needs a database password, the Okta API token for the CI pipeline, a private key for `private_key_jwt` client authentication, and an encryption key for stored payment data. In 2015 all four would have been in a `web.config` checked into source control. Today, all four come from **HashiCorp Vault** at container start, three of them are dynamically generated with a one-hour lifetime, and none of them exists anywhere on disk.

## Why This Matters

- **Leaked credentials are the top breach vector.** Hardcoded secrets in repositories and CI logs are found by automated scanners within minutes of being pushed to a public host.
- **Git history is forever.** Deleting a secret in a later commit does not remove it. Anyone who cloned the repository still has it, and it remains in the reflog and in every fork.
- **Static long-lived secrets are unmanageable at scale.** MoneyGuard has hundreds of services; a database password shared by twenty of them can never be rotated without a coordinated outage, so it never gets rotated.
- **Compliance requires rotation and access records.** PCI-DSS and SOX both expect credential rotation and an auditable record of who accessed what.
- **A short-lived credential is worth far less to an attacker.** A one-hour dynamic database credential leaked at 09:00 is useless at 10:00. That single property does more for your posture than most other controls.

## Key Concepts

- **Secret** — any credential that grants access: passwords, API tokens, private keys, certificates, encryption keys.
- **Secrets manager** — a purpose-built system that stores secrets encrypted, controls access, and audits every read. Examples: HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, Google Secret Manager.
- **Static secret** — a long-lived credential stored and handed out unchanged.
- **Dynamic secret** — a credential generated on demand, unique to the requester, with a short lifetime (a **lease**), automatically revoked on expiry. Vault's signature capability.
- **Lease / TTL** — how long a dynamic secret remains valid.
- **Rotation** — replacing a credential with a new one on a schedule or after exposure.
- **Envelope encryption** — encrypting data with a **data key**, then encrypting that data key with a **key encryption key (KEK)** held in a KMS/HSM.
- **KMS / HSM** — Key Management Service / Hardware Security Module: systems that hold master keys and perform cryptographic operations without ever exporting the key.
- **Workload identity** — a service proving *what it is* (via a platform-issued identity) rather than presenting a credential it stores. See [Module 06](../06_Machine_Identity_And_Workload_Auth/README.md).
- **OIDC federation for CI** — the pipeline exchanges a short-lived, platform-signed identity token for cloud credentials, eliminating static keys entirely.
- **Secret zero / bootstrap problem** — to fetch secrets you need a credential; where does *that* one come from?
- **Sealed Secrets / SOPS** — tools that let encrypted secrets live safely in Git, decryptable only by the target environment.
- **Secret scanning** — automated detection of credentials in code, history, and logs (Gitleaks, TruffleHog, GitHub push protection).

## Detailed Notes

### Concept 1 — Where secrets must never be

| Location | Why it's wrong |
|---|---|
| **Source code / config files in Git** | Permanent in history; visible to everyone with repo access; scanned by attackers within minutes on public hosts |
| **Container images** | Every layer is extractable with `docker history`; images get pushed to registries and shared |
| **CI configuration files** | Usually in the same repository, and often printed by verbose build output |
| **Environment variables set at build time** | Baked into the image; visible in process listings and crash dumps |
| **Terraform `.tf` files or committed `.tfvars`** | Same as source code, plus they end up in the state file |
| **Logs** | See [Module 09 Chapter 3](../09_Resilience_Observability_And_SIEM/03_Audit_Logging_And_Observability.md) — a log platform is a searchable index with weaker controls than a vault |
| **Wikis, tickets, chat** | No encryption, no rotation, no audit, indefinite retention |
| **Kubernetes `Secret` objects, unconfigured** | ⚠️ Base64 is **encoding, not encryption**. Without encryption-at-rest enabled on etcd, they're effectively plaintext |

⚠️ The Kubernetes point catches people constantly. `kubectl get secret -o yaml` returns base64, which looks scrambled and is trivially reversible. Kubernetes Secrets are acceptable as a *delivery mechanism* when etcd encryption-at-rest and RBAC are properly configured, but they are not a secrets *manager* — no rotation, no dynamic generation, weak audit.

### Concept 2 — What a secrets manager provides

Beyond "an encrypted place to put things," a real secrets manager provides:

- **Encryption at rest**, with the master key held in a KMS or HSM and never exported.
- **Fine-grained access control** — this service may read this path and nothing else.
- **Full audit logging** — every read, by whom, from where, when. This is often the single biggest gain, because it makes "who could have seen this credential?" answerable.
- **Versioning** — so a rotation can be rolled back.
- **Dynamic secret generation** — the capability that changes the security model entirely.
- **Automatic rotation** for supported backends.
- **Encryption as a service** — encrypt/decrypt data without the application ever handling the key.

### Concept 3 — Dynamic secrets: the idea that changes everything

With a **static** secret, one database password exists, twenty services share it, it's been unchanged for three years, and rotating it requires coordinating twenty deployments. So nobody rotates it. If it leaks, you don't know which service leaked it, and revoking it takes everything down.

With a **dynamic** secret, the wire microservice asks Vault for database credentials. Vault connects to PostgreSQL, creates a brand-new user with a random password and exactly the needed grants, and returns it with a **one-hour lease**. When the lease expires, Vault drops the user. Every instance of every service gets its own unique credential.

The consequences are large:

- **Leaked credentials expire on their own.** No emergency rotation.
- **Attribution is automatic.** A credential traced back through Vault's audit log identifies exactly which service and which instance requested it.
- **Revocation is surgical.** Revoke one lease and one instance is affected, not the entire estate.
- **Rotation stops being an event.** It happens continuously, invisibly, by design.
- **Least privilege becomes practical.** The read-only reporting job gets a credential with only `SELECT`, because generating a bespoke role costs nothing.

💡 If you take one thing from this chapter: **a short-lived credential is dramatically less valuable to an attacker than a long-lived one.** Where you can't have dynamic secrets, shorten lifetimes and rotate aggressively.

### Concept 4 — The bootstrap problem ("secret zero")

To read secrets from Vault, a service must authenticate to Vault. That requires a credential. If that credential is a static token in a config file, you've moved the problem, not solved it.

The answer is **workload identity**: the service proves *what it is* using something the platform issues and attests, rather than something the service stores.

| Platform | How the workload proves identity |
|---|---|
| **Kubernetes** | A projected ServiceAccount token, which Vault validates against the cluster's TokenReview API |
| **AWS** | The EC2/ECS/EKS instance identity or an IRSA-projected OIDC token |
| **Azure** | Managed Identity — the platform issues a token; nothing is stored |
| **GitHub Actions** | An OIDC token signed by GitHub, exchanged for a scoped, short-lived cloud role |
| **Bare metal** | A SPIFFE/SPIRE agent performing node and workload attestation |

In each case there is **no stored credential**. The platform vouches for the workload's identity, Vault or the cloud IAM system verifies that attestation, and issues a short-lived token. This is the same principle as [Module 06](../06_Machine_Identity_And_Workload_Auth/README.md)'s workload identity, applied to secret retrieval.

⚠️ Under no circumstances should a CI pipeline hold a long-lived cloud access key. GitHub Actions, GitLab CI, and Azure DevOps all support OIDC federation. Using it eliminates the single most commonly leaked class of credential in the industry.

### Concept 5 — Getting secrets into a running application

Four delivery mechanisms, from worst to best:

| Method | How | Rotation | Exposure risk | Verdict |
|---|---|---|---|---|
| **Baked into the image** | `ENV DB_PASS=...` in a Dockerfile | None | Extractable from any layer | ❌ Never |
| **Environment variables at runtime** | Injected by the orchestrator at container start | Requires a restart | Visible in `/proc`, crash dumps, and child processes | ⚠️ Acceptable, common, but leaky |
| **Mounted file (tmpfs)** | Written to an in-memory volume the container reads | Can be updated in place without a restart | Only readable inside the container; never on disk | ✅ Good |
| **Fetched by the application via SDK** | The app authenticates with workload identity and calls the secrets manager | Full control; can renew a lease mid-flight | Never leaves memory | ✅ Best |

The **Vault Agent sidecar** pattern is the common middle ground: a small companion process authenticates using workload identity, fetches secrets, writes them to a tmpfs file, and renews the lease continuously. The application just reads a file and knows nothing about Vault — which means no SDK dependency and no Vault-specific code.

### Concept 6 — Rotation without an outage

Rotating a shared static credential naively causes downtime: you change it, and every service still using the old value fails until it's redeployed.

The standard solution is **two active credentials at all times**, sometimes called the two-slot or dual-credential pattern:

1. Both credential A and credential B are valid. Services use A.
2. Rotate B to a new value. Nothing is using B, so nothing breaks.
3. Switch services to B — through a config reload or rolling restart.
4. Verify nothing is still using A (the audit log tells you).
5. Rotate A. It's now the idle slot, ready for the next cycle.

AWS Secrets Manager implements this natively with `AWSCURRENT` and `AWSPENDING` staging labels. Signing keys use the same idea via **JWKS key rotation**, where the identity provider publishes both the old and new public keys so tokens signed with either verify successfully during the overlap — covered in [Module 04](../04_Edge_Security_Tokens_And_Sessions/README.md).

The rotation triggers that matter:
- **Scheduled** — every 30 or 90 days for static secrets.
- **On personnel change** — anyone who left had access to whatever they could read.
- **On suspected exposure** — immediately, and assume exposure whenever a secret appears in a log or a repository.
- **Continuously** — which is what dynamic secrets give you for free.

### Concept 7 — When secrets must live in Git: SOPS and Sealed Secrets

Sometimes a GitOps workflow genuinely needs configuration containing secrets in the repository. Two tools make this safe by ensuring what's committed is **already encrypted**, decryptable only by the target environment:

- **SOPS (Secrets OPerationS)** encrypts only the *values* in a YAML or JSON file, leaving keys readable. Diffs stay meaningful — you can see that `db_password` changed without seeing either value. Encryption keys come from a cloud KMS, so access is controlled by cloud IAM and every decryption is audited.
- **Sealed Secrets** (Bitnami) encrypts a Kubernetes Secret with a public key whose private counterpart exists only in the target cluster. The encrypted `SealedSecret` is safe to commit; only that cluster's controller can decrypt it.

Both are genuinely safe. Both are still second-best to a real secrets manager, because neither gives you dynamic generation or automatic rotation. They're the right tool when GitOps requires the value to be in the repository.

### Concept 8 — Detecting leaks, and responding when they happen

**Prevention:**
- **Pre-commit hooks** (Gitleaks, TruffleHog) that block a commit containing a credential pattern.
- **Push protection** at the hosting provider, which rejects the push outright.
- **CI scanning** of the full history, not just the diff.
- **Log redaction** with an allowlist (Module 09, Chapter 3).

**Response, when a secret does leak — and the order matters:**
1. **Rotate first.** Not "investigate first." The credential is compromised the moment it's exposed; treat it as used.
2. **Revoke the old value** at the source system so it cannot be used even if held.
3. **Search the audit logs** for use of that credential from unexpected sources.
4. **Then** clean the repository history — but understand that this is cleanup, not remediation. Anyone who cloned it still has the secret.
5. **Add a detection** so this class of leak is caught at commit time next time.

⚠️ The most common mistake is inverting steps 1 and 4 — spending an hour rewriting Git history while the live credential remains valid.

## Workflow

A service starting up and obtaining a dynamic database credential:

```mermaid
sequenceDiagram
    autonumber
    participant K8S as Kubernetes
    participant POD as Wire Microservice pod
    participant AGENT as Vault Agent sidecar
    participant VAULT as HashiCorp Vault
    participant PG as PostgreSQL

    K8S->>POD: Start pod with projected<br/>ServiceAccount token
    Note over POD: No credential is stored<br/>in the image or in config
    POD->>AGENT: Sidecar starts
    AGENT->>VAULT: Login with the ServiceAccount JWT
    VAULT->>K8S: TokenReview: is this token genuine,<br/>and for which service account?
    K8S-->>VAULT: Valid: sa=wire-service, ns=payments
    VAULT-->>AGENT: Vault token, TTL 1h,<br/>policy: wire-service-policy

    AGENT->>VAULT: Read database/creds/wire-service
    VAULT->>PG: CREATE ROLE v-k8s-wire-a8f3 WITH PASSWORD ...<br/>GRANT SELECT, INSERT ON wires
    PG-->>VAULT: role created
    VAULT-->>AGENT: {username, password}, lease 1h

    AGENT->>POD: Write to /vault/secrets/db (tmpfs, never on disk)
    POD->>PG: Connect using the unique, short-lived credential

    loop every 30 minutes
        AGENT->>VAULT: Renew lease
        VAULT-->>AGENT: extended
    end

    Note over VAULT,PG: If the pod dies or the lease expires,<br/>Vault DROPs the role automatically.<br/>A leaked credential self-destructs.
    VAULT-)SIEM: Audit: who read which path, when, from where
```

## Architecture Diagram

```mermaid
graph TB
    subgraph "Identity sources - no stored credentials"
        K8SID[Kubernetes<br/>ServiceAccount tokens]
        GHID[GitHub Actions<br/>OIDC token]
        AZID[Azure Managed Identity]
    end

    K8SID -->|attested identity| VAULT
    GHID -->|federated token exchange| AWSIAM[AWS IAM<br/>role assumption]
    AZID --> AKV[Azure Key Vault]

    VAULT[HashiCorp Vault<br/>central secrets manager]

    VAULT --> KMS[(Cloud KMS / HSM<br/>master key never exported)]

    VAULT --> STATIC[Static secrets<br/>3rd-party API keys,<br/>legacy credentials]
    VAULT --> DYN[Dynamic secrets engine]
    VAULT --> TRANSIT[Transit engine<br/>encryption as a service]
    VAULT --> PKI[PKI engine<br/>short-lived mTLS certs]

    DYN -->|creates ephemeral roles| PG[(PostgreSQL<br/>Core Banking / Business Banking)]
    DYN -->|short-lived creds| AWSDYN[AWS STS credentials]

    subgraph "Consumers"
        WIRE[Wire Microservice<br/>Vault Agent sidecar]
        BBSP[Business Banking SaaS Portal]
        FRAUD[Fraud Detection Pipeline]
        ATL[Atlantis / CI pipeline<br/>Chapter 01]
    end

    VAULT --> WIRE
    VAULT --> BBSP
    VAULT --> FRAUD
    VAULT --> ATL
    AWSIAM --> ATL

    VAULT --> AUDIT[Vault audit device]
    AUDIT --> SIEM[SOC / SIEM "Sentinel"<br/>every secret read is an event]
    SIEM --> DETECT[Detections:<br/>unusual path reads,<br/>reads from new IPs,<br/>bulk enumeration]
```

## Step-by-Step Process

Building MoneyGuard's secrets practice:

1. **Inventory what exists.** Scan every repository's full history, every CI configuration, and every container image for credentials. Assume you'll find more than expected.
2. **Rotate everything you find, immediately**, before any cleanup work.
3. **Stand up the secrets manager** — Vault, or the cloud-native equivalent — with the master key in a KMS or HSM.
4. **Enable the audit device first**, before storing anything. An unaudited secrets manager is a single point of failure with no forensics.
5. **Define access policies per workload**, not per team. The wire service reads `database/creds/wire-service` and nothing else.
6. **Solve secret zero with workload identity.** Kubernetes ServiceAccount auth, IRSA, Managed Identity, or OIDC federation for CI. No static bootstrap tokens.
7. **Convert static secrets to dynamic** wherever a backend supports it — databases first, since they're the highest-value and best-supported.
8. **Deliver via tmpfs file or SDK**, not build-time environment variables.
9. **Set short lifetimes.** One hour for dynamic database credentials, fifteen minutes for cloud credentials.
10. **Implement two-slot rotation** for whatever must remain static.
11. **Add secret scanning** as a pre-commit hook, a push-protection rule, and a CI check over full history.
12. **Enforce log redaction** so secrets can't leak through telemetry.
13. **Forward the Vault audit log to the SIEM** and write detections: reads from unexpected paths, from new source IPs, or bulk enumeration.
14. **Write and rehearse the leak-response runbook**, with rotation as step one.
15. **Review access quarterly.** Policies accumulate; prune them.

## Commands and Examples

**⚠️ The anti-pattern this chapter exists to eliminate.**

```hcl
# NEVER. This is in Git forever, even if you delete it in the next commit.
provider "okta" {
  org_name  = "moneyguard"
  api_token = "00aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890abcd"   # ❌
}
```

```csharp
// NEVER. Baked into the image, extractable from any layer.
const string ConnectionString =
    "Host=core-ledger.moneyguard.internal;Password=Wint3r2026!";   // ❌
```

**Vault policy: least privilege for one workload.**

```hcl
# policies/wire-service.hcl
# This service may generate database credentials and nothing else.

path "database/creds/wire-service" {
  capabilities = ["read"]
}

# Encryption as a service - the app encrypts payment data WITHOUT
# ever holding the key. Vault does the crypto; the key never leaves.
path "transit/encrypt/payment-data" { capabilities = ["update"] }
path "transit/decrypt/payment-data" { capabilities = ["update"] }

# Explicitly NOT granted: other services' paths, the Okta admin token,
# any wildcard, and anything under sys/.
```

**Configuring dynamic database credentials.**

```bash
# 1. Vault connects to PostgreSQL as an admin that can create roles.
vault write database/config/core-banking \
    plugin_name="postgresql-database-plugin" \
    allowed_roles="wire-service,reporting-readonly" \
    connection_url="postgresql://{{username}}:{{password}}@core-ledger.moneyguard.internal:5432/banking" \
    username="vault_admin" \
    password="$VAULT_DB_BOOTSTRAP_PW"

# 2. Define what a wire-service credential looks like: least privilege,
#    one-hour lease, hard cap at 24 hours.
vault write database/roles/wire-service \
    db_name="core-banking" \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
                         GRANT SELECT, INSERT ON wires, wire_audit TO \"{{name}}\";" \
    revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"

# 3. A service requests a credential - unique per instance, self-destructing.
vault read database/creds/wire-service
# Key                Value
# ---                -----
# lease_id           database/creds/wire-service/pQ8xY2mN4kL7
# lease_duration     1h
# lease_renewable    true
# password           A1a-9Xk2LmQ7vB3nR8sT
# username           v-k8s-wire-service-pQ8xY2mN-1756900000
#
# In one hour Vault runs DROP ROLE automatically.
# A credential leaked at 09:00 is worthless at 10:00.
```

**Kubernetes auth: solving secret zero with workload identity.**

```bash
# Vault trusts the cluster's TokenReview API - no stored bootstrap credential.
vault auth enable kubernetes

vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Bind ONE service account in ONE namespace to ONE policy.
vault write auth/kubernetes/role/wire-service \
    bound_service_account_names="wire-service" \
    bound_service_account_namespaces="payments" \
    policies="wire-service" \
    ttl="1h"
```

**Vault Agent sidecar injection — the application reads a file and knows nothing about Vault.**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wire-service
  namespace: payments
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "wire-service"
        vault.hashicorp.com/agent-inject-secret-db: "database/creds/wire-service"
        # Render straight into the connection-string format the app expects.
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "database/creds/wire-service" -}}
          Host=core-ledger.moneyguard.internal;Database=banking;Username={{ .Data.username }};Password={{ .Data.password }}
          {{- end }}
        # The agent renews the lease continuously - no restart to rotate.
        vault.hashicorp.com/agent-pre-populate-only: "false"
    spec:
      serviceAccountName: wire-service   # this IS the identity
      containers:
        - name: wire-service
          image: moneyguard/wire-service:4.2.1
          env:
            # A PATH, not a secret. Written to tmpfs, never to disk.
            - name: DB_CONNECTION_FILE
              value: /vault/secrets/db
```

**CI with OIDC federation — no static cloud keys anywhere.**

```yaml
name: deploy-identity-infra
on:
  push: { branches: [main] }

permissions:
  id-token: write     # GitHub mints a short-lived, signed OIDC token
  contents: read

jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Exchange the OIDC token for a ~15-minute AWS role.
      # There is NO AWS_SECRET_ACCESS_KEY stored anywhere.
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/moneyguard-identity-apply
          aws-region: us-east-1

      # Same principle for Vault: authenticate with the GitHub OIDC token.
      - uses: hashicorp/vault-action@v3
        with:
          url: https://vault.moneyguard.internal
          method: jwt
          role: github-identity-pipeline
          secrets: |
            okta/data/api-token token | OKTA_API_TOKEN ;

      - run: terraform apply -auto-approve
        env:
          OKTA_API_TOKEN: ${{ env.OKTA_API_TOKEN }}   # in memory only
```

**Encryption as a service — the app never holds the key.**

```csharp
// The wire service must store an encrypted payee account number.
// It never sees, holds, or stores the encryption key.
public async Task<string> EncryptPayeeAccountAsync(string accountNumber)
{
    var response = await _vault.PostAsync("/v1/transit/encrypt/payment-data",
        new { plaintext = Convert.ToBase64String(Encoding.UTF8.GetBytes(accountNumber)) });

    // -> "vault:v3:8SDd3WHDOjf7mq69CyCqYjBXAiQQAVZRkFM13ok481zoCmHnSeDX9vyf7w=="
    // The "v3" is the key version: rotating the key does NOT require
    // re-encrypting existing data - old versions still decrypt.
    return response.Data.Ciphertext;
}
```

**Two-slot rotation for a static secret (AWS Secrets Manager).**

```bash
# Slot model: AWSCURRENT is live, AWSPENDING is the new value being staged.
# Nothing ever loses a valid credential mid-rotation.

aws secretsmanager rotate-secret \
  --secret-id moneyguard/partnerbank/api-key \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:111122223333:function:rotate-partner-key \
  --rotation-rules AutomaticallyAfterDays=30

# The rotation Lambda's four steps:
#   createSecret  -> generate a new value, store as AWSPENDING
#   setSecret     -> configure the new value at PartnerBank
#   testSecret    -> verify AWSPENDING actually works
#   finishSecret  -> promote AWSPENDING to AWSCURRENT
# If testSecret fails, rotation aborts and AWSCURRENT is untouched.
```

**Secret scanning as a pre-commit hook.**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks          # blocks the commit before the secret exists

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

**Detecting anomalous secret access in the SIEM.**

```kusto
// Vault audit logs are forwarded to Sentinel (Module 09).
// A workload reading a path it has never read before is a strong signal.
let Baseline = VaultAudit
  | where TimeGenerated between (ago(30d) .. ago(1d))
  | where Type == "response" and Error == ""
  | summarize KnownPaths = make_set(RequestPath) by Identity = Auth.DisplayName;
VaultAudit
| where TimeGenerated > ago(15m)
| where Type == "response" and Error == ""
| join kind=inner Baseline on $left.Auth_DisplayName == $right.Identity
| where RequestPath !in (KnownPaths)
| project TimeGenerated, Identity, RequestPath, RemoteAddress
| extend Severity = "High",
         Note = "Workload reading a secret path outside its 30-day baseline"
```

## Best Practices

- **Never commit a secret.** Enforce with pre-commit hooks, push protection, and CI scanning over full history.
- **Prefer dynamic over static** wherever the backend supports it. Databases first.
- **Solve secret zero with workload identity** — Kubernetes ServiceAccount auth, IRSA, Managed Identity, SPIFFE. Never a static bootstrap token.
- **Use OIDC federation in CI.** A pipeline holding a long-lived cloud key is the industry's most-leaked credential.
- **Deliver via tmpfs file or SDK**, not build-time environment variables.
- **Keep lifetimes short** — one hour for database credentials, fifteen minutes for cloud credentials.
- **Enable the audit device before storing anything**, and forward it to the SIEM.
- **Scope policies per workload, not per team**, and grant exactly the paths needed with no wildcards.
- **Two-slot rotation** for anything that must stay static, so rotation never causes an outage.
- **Use encryption as a service** (Vault Transit, KMS) so applications never hold key material.
- **Rotate first, investigate second** when a secret leaks.
- **Rotate on personnel change**, not just on a schedule.
- **Use SOPS or Sealed Secrets** when GitOps genuinely requires a value in the repository — encrypted before commit, never after.

## Common Mistakes

- ⚠️ **Committing a secret and "fixing" it with a later delete commit.** It remains in history, in every clone, and in every fork.
- ⚠️ **Treating base64 as encryption.** Kubernetes Secrets are encoded, not encrypted, unless etcd encryption-at-rest is explicitly enabled.
- ⚠️ **A static Vault token in a config file** to bootstrap secret access — this moves the problem rather than solving it.
- ⚠️ **Long-lived cloud access keys in CI.** The single most commonly leaked credential class in the industry.
- ⚠️ **Build-time environment variables**, which bake the secret into the image layer permanently.
- ⚠️ **One shared database password across twenty services**, which can never be rotated and gives you no attribution when it leaks.
- ⚠️ **No audit device on the secrets manager**, leaving you unable to answer "who could have read this?"
- ⚠️ **Wildcard Vault policies** like `path "secret/*"`, which quietly grant everything.
- ⚠️ **Rotating without the two-slot pattern**, causing an outage and teaching the organization that rotation is dangerous.
- ⚠️ **Cleaning Git history before rotating.** An hour of history rewriting while the live credential is still valid.
- ⚠️ **Not rotating after someone leaves.** They had access to whatever they could read, and probably still have a copy.
- ⚠️ **Logging the secret.** Covered in Module 09, and it turns your log platform into an unmanaged credential store.
- ⚠️ **Storing secrets in Terraform variables** — they end up in the state file in plaintext.

## Pro Tips

- 💡 **Dynamic secrets change the threat model, not just the hygiene.** Once a credential expires in an hour, most credential-leak incidents downgrade from "emergency rotation across twenty services" to "already handled."
- 💡 **Turn on the audit device before you store the first secret.** Retrofitting it means a permanent gap in exactly the record an investigator will ask for.
- 💡 **Vault's Transit engine solves problems people don't realize it can.** Encryption as a service means the application never holds key material at all, and key rotation doesn't require re-encrypting existing data because the ciphertext carries its key version.
- 💡 **Scope policies to the workload identity, not the team.** Team-scoped policies grow until they grant everything.
- 💡 **Feed Vault's audit log into the SIEM and baseline it.** A workload reading a secret path it has never read before is one of the highest-fidelity detections available — machine identities have very tight baselines (see [Module 09 Chapter 4](../09_Resilience_Observability_And_SIEM/04_SIEM_And_Threat_Detection.md)).
- 💡 **Make short TTLs the default and require justification for longer ones**, rather than the reverse.
- 💡 **Interview move:** say *"secrets never touch Git; workloads authenticate with platform-attested identity and receive short-lived dynamic credentials, so a leak expires on its own and every read is attributable in the audit log."*

## Real-World Use Cases

- **The wire microservice deploy.** Alex Kim's service starts with no stored credentials at all. The Vault Agent sidecar authenticates using the pod's ServiceAccount token, receives a one-hour PostgreSQL credential unique to that pod, writes it to tmpfs, and renews it continuously. If the pod dies, Vault drops the database role automatically.
- **The leaked CI token.** An Okta API token appears in a verbose build log. Because it's a static third-party secret, it can't be dynamic — so the response is: rotate immediately, revoke the old token at Okta, search Okta's admin logs for use from unexpected IPs, then fix the log redaction and add a scanning rule. Total exposure: eleven minutes.
- **The departed contractor.** Sam Torres's contract ends. Beyond revoking his access, MoneyGuard rotates every static secret he could read. Because most credentials are dynamic, the list is short — a handful of third-party API keys rather than every database password in the estate.
- **Encrypting payee account numbers.** The Business Banking SaaS Portal encrypts stored payee account numbers through Vault's Transit engine. The application never holds the key, and when the key is rotated, existing ciphertext still decrypts because each value carries its key version.
- **The 03:41 Vault detection.** Continuing the incident from Module 09 Chapter 4: the Fraud Detection Pipeline's identity reads a Vault path it has never touched in thirty days. The baseline detection fires at High severity, and the audit log shows the exact path, source IP, and time — which is what let the SOC scope the incident in minutes.
- **Passing the PCI audit.** The assessor asks how credentials are rotated and who has accessed them. Raj demonstrates one-hour dynamic leases, thirty-day automated rotation for static secrets, per-workload policies, and a complete audit log of every read. That is a substantially stronger answer than a rotation spreadsheet.

## Key Takeaways

1. **Secrets never go in code, images, CI config, Terraform files, logs, wikis, or chat.** Git history is permanent.
2. **A secrets manager gives you** encryption at rest, per-workload access control, full audit logging, versioning, and dynamic generation.
3. **Dynamic secrets are the single biggest win:** unique per requester, short-lived, self-revoking, automatically attributable.
4. **A short-lived credential is worth far less to an attacker.** Where you can't be dynamic, shorten and rotate.
5. **Solve secret zero with workload identity** — Kubernetes ServiceAccount auth, IRSA, Managed Identity, SPIFFE — never a stored bootstrap token.
6. **CI must use OIDC federation**, never a long-lived cloud access key.
7. **Deliver secrets via tmpfs file or SDK**, not build-time environment variables. Base64 in Kubernetes is encoding, not encryption.
8. **Two-slot rotation** keeps rotation from causing outages; the same idea underlies JWKS key rotation.
9. **Enable the audit device first**, forward it to the SIEM, and baseline it — a workload reading an unfamiliar path is a high-fidelity detection.
10. **When a secret leaks: rotate first, revoke, investigate, then clean history.** History cleanup is not remediation.
11. **Encryption as a service** lets applications protect data without ever holding key material.

## Glossary

| Term | Meaning |
|---|---|
| **Secret** | Any credential that grants access: password, API token, private key, certificate. |
| **Secrets manager** | A purpose-built system that stores secrets encrypted, controls access, and audits reads. |
| **Static secret** | A long-lived credential handed out unchanged. |
| **Dynamic secret** | A credential generated on demand, unique per requester, with a short lease and automatic revocation. |
| **Lease / TTL** | How long a dynamic secret stays valid before automatic revocation. |
| **Rotation** | Replacing a credential with a new value on a schedule or after exposure. |
| **Two-slot rotation** | Keeping two valid credentials so one can always be rotated without an outage. |
| **KMS / HSM** | Key Management Service / Hardware Security Module — hold master keys and never export them. |
| **Envelope encryption** | Encrypting data with a data key, then encrypting that key with a master key. |
| **Encryption as a service** | The secrets manager performs encryption so the application never holds key material. |
| **Workload identity** | A service proving what it is via platform attestation rather than a stored credential. |
| **Secret zero / bootstrap problem** | The credential needed to obtain other credentials. |
| **OIDC federation** | Exchanging a platform-signed identity token for short-lived cloud credentials. |
| **IRSA** | IAM Roles for Service Accounts — AWS workload identity for Kubernetes. |
| **SOPS** | A tool that encrypts values inside YAML/JSON so files can live safely in Git. |
| **Sealed Secrets** | Encrypts a Kubernetes Secret so only the target cluster can decrypt it. |
| **tmpfs** | An in-memory filesystem; contents never touch disk. |
| **Secret scanning** | Automated detection of credentials in code, history, and logs. |
| **Vault Agent** | A sidecar that authenticates, fetches secrets, renders them to a file, and renews leases. |

## Revision Notes

- **Never in:** code, Git, images, CI config, `.tf` files, logs, wikis, chat. **Git history is forever.**
- **Kubernetes Secrets are base64 = encoding, not encryption.** Needs etcd encryption-at-rest.
- **Dynamic > static.** Unique per requester, short lease, auto-revoked, attributable.
- **Secret zero solved by workload identity:** K8s ServiceAccount + TokenReview, IRSA, Azure Managed Identity, GitHub OIDC, SPIFFE. **Never a stored bootstrap token.**
- **CI: OIDC federation**, never a long-lived access key.
- **Delivery, worst → best:** baked into image ❌ → runtime env var ⚠️ → tmpfs file ✅ → SDK fetch ✅.
- **Vault Agent sidecar** = app reads a file, knows nothing about Vault.
- **Two-slot rotation:** rotate the idle slot, switch, verify, rotate the other. Same idea as JWKS overlap.
- **Rotation triggers:** schedule, personnel change, suspected exposure, continuously (dynamic).
- **Transit engine** = encryption as a service; ciphertext carries the key version, so rotation needs no re-encryption.
- **Leak response order: ROTATE → revoke → investigate → clean history.** History cleanup is not remediation.
- **Audit device on before the first secret**, forwarded to the SIEM, baselined per workload.
- **SOPS / Sealed Secrets** when GitOps needs the value in the repo — encrypted before commit.

## Interview Questions

**Q1. Where do secrets live, and how does an application get them at runtime?**
A: In a purpose-built secrets manager — Vault, AWS Secrets Manager, Azure Key Vault — with the master key held in a KMS or HSM, per-workload access policies, and full audit logging of every read. They never live in code, images, CI configuration, Terraform files, or logs. At runtime the application authenticates using workload identity — a Kubernetes ServiceAccount token, IRSA, or a Managed Identity — rather than presenting a stored credential, and receives a short-lived token. Delivery is either an SDK fetch into memory or a Vault Agent sidecar that writes the value to a tmpfs file the application reads. I'd avoid build-time environment variables, since those bake into the image layer, and be cautious with runtime environment variables, which are visible in process listings and crash dumps.

**Q2. Explain dynamic secrets and why they matter more than good hygiene.**
A: With a static secret, one database password is shared by twenty services, unchanged for years, and rotating it requires a coordinated outage — so nobody does. With dynamic secrets, each service instance asks Vault for credentials, and Vault creates a brand-new database user with exactly the needed grants, a random password, and a one-hour lease, dropping the user when the lease expires. That changes the threat model rather than just the hygiene: a credential leaked at 09:00 is worthless at 10:00, attribution is automatic because the audit log identifies which instance requested it, revocation is surgical rather than estate-wide, and rotation stops being an event because it happens continuously by design.

**Q3. What's "secret zero" and how do you solve it?**
A: To read secrets from a vault, a service must authenticate to the vault — which requires a credential. If that's a static token in a config file, you've relocated the problem rather than solved it. The answer is workload identity: the service proves *what it is* using something the platform issues and attests, not something the service stores. On Kubernetes, Vault validates a projected ServiceAccount token against the cluster's TokenReview API. On AWS it's the instance identity or an IRSA-projected OIDC token; on Azure, Managed Identity; in GitHub Actions, an OIDC token exchanged for a short-lived role; on bare metal, SPIFFE/SPIRE attestation. In every case nothing is stored — the platform vouches, the vault verifies, and a short-lived token is issued.

**Q4. A developer commits an API key and pushes. Walk me through your response.**
A: Rotate first — not investigate first. The moment a credential is exposed, treat it as used; generate a new value and deploy it. Then revoke the old value at the source system so it can't be used even by someone holding it. Then search the audit logs for any use of that credential from unexpected source IPs or at unexpected times, which tells you whether this is a hygiene issue or an incident. Only then clean the Git history — and I'd be explicit that this is cleanup, not remediation, because anyone who cloned the repository or any fork still has the value. Finally, add the detection: a pre-commit hook and push protection so the same class of leak is blocked before the commit exists. The most common mistake is inverting the order and spending an hour rewriting history while the live credential is still valid.

**Q5. How do you rotate a credential shared by many services without downtime?**
A: The two-slot pattern. Keep two valid credentials at all times; services use slot A. Rotate slot B — nothing is using it, so nothing breaks. Switch services to B via config reload or a rolling restart. Verify from the audit log that nothing is still using A. Then rotate A, which is now the idle slot ready for the next cycle. AWS Secrets Manager implements this natively with `AWSCURRENT` and `AWSPENDING` staging labels, and its rotation Lambda has an explicit test step that aborts the rotation if the new value doesn't work. The same principle underlies JWKS key rotation for token signing, where the identity provider publishes both old and new public keys during an overlap window so tokens signed with either still verify.

**Q6. Are Kubernetes Secrets good enough?**
A: They're a reasonable *delivery* mechanism and not a secrets *manager*. The trap people fall into is that `kubectl get secret -o yaml` returns base64, which looks scrambled but is encoding, not encryption — so without etcd encryption-at-rest explicitly enabled, they're effectively plaintext to anyone who can read etcd or take a backup. Even properly configured, they give you no dynamic generation, no rotation, no versioning, and only coarse audit through the Kubernetes API. So I'd use them as the surface a pod reads from, populated by an external secrets operator or a Vault Agent sidecar from a real secrets manager, and I'd enable etcd encryption and tight RBAC regardless.

**Q7. How do you keep static cloud credentials out of CI entirely?**
A: OIDC federation. GitHub Actions, GitLab CI, and Azure DevOps can all mint a short-lived, platform-signed identity token describing the repository, branch, and workflow. The cloud provider is configured to trust that issuer with a condition on those claims, and the pipeline exchanges the token for a role with roughly a fifteen-minute lifetime. There is no `AWS_SECRET_ACCESS_KEY` stored anywhere — which matters because long-lived CI keys are the single most commonly leaked credential class in the industry. The claim conditions are the part people get wrong: the trust policy must pin the specific repository and ideally the branch or environment, or any repository in the organization can assume the role.

**Q8. Your secrets manager is now the most valuable target in the estate. How do you protect and monitor it?**
A: Assume it will be attacked and make the attack visible. Enable the audit device before storing the first secret, because retrofitting leaves a permanent gap in exactly the record an investigator wants. Forward that audit log to the SIEM and baseline it per workload identity — machine identities have extremely tight behavioural baselines, so a workload reading a secret path it has never read in thirty days is a high-fidelity detection. Scope policies to individual workloads with explicit paths and no wildcards, so a compromised service can reach only its own secrets. Keep the master key in a KMS or HSM so it's never exported. Require multiple operators to unseal. And review policies quarterly, because they accumulate and drift toward over-permission.

## Practice Exercises

1. **Convert to dynamic.** MoneyGuard's Core Banking Ledger currently uses one shared PostgreSQL password across six services. Write the Vault database configuration and role definitions to give each service its own dynamic credential with least-privilege grants. Then write the migration plan that gets from the shared password to dynamic credentials with zero downtime.

2. **Design the leak runbook.** An Okta admin API token has appeared in a public CI log. Write the incident runbook: every step in order, who does it, the target time for each, and what evidence is captured. Explicitly state why rotation precedes history cleanup, and what you'd tell Dana Cole about the exposure window.

3. **Write the Vault policies.** For the Business Banking SaaS Portal, the Fraud Detection Pipeline, and the Atlantis CI runner, write three Vault policies. Each must grant exactly what's needed and nothing more. Then write three requests that each policy must *deny*, and explain what would go wrong if a wildcard path were used instead.

4. **Eliminate static CI credentials.** Take the GitHub Actions workflow from Chapter 1 and specify the complete OIDC federation setup: the AWS trust policy with claim conditions, the Vault JWT role, and the permissions block. Then describe the misconfiguration that would let *any* repository in the organization assume the role, and how you'd detect it.

5. **Detect secret abuse.** Write three SIEM detection rules over Vault audit logs: a workload reading a path outside its baseline, bulk enumeration of secret paths, and a read from an unexpected source IP. For each, give the threshold, the exclusions, the severity, and the response action — and say which one you'd trust enough to automate.

---

⬅ Back to [Module 10 Index](./README.md) | 📖 [Main README](../README.md) | 🏦 [The MoneyGuard Story](../SCENARIO.md)

⬅ Previous: [01_Infrastructure_As_Code_For_Identity.md](./01_Infrastructure_As_Code_For_Identity.md)
