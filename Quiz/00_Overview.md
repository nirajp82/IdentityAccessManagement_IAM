
# 🏗️ The Thumbnail Maker: Grand Security Architecture

This diagram illustrates how a request flows from a human at Acme Corp all the way to your high-availability database, protected by every layer we've built.

```mermaid
graph TD
    subgraph "The Enterprise (Acme Corp)"
        HR[Workday HR] -->|Sync| AAD[Azure AD / IdP]
        User((Engineer)) -->|1. SSO Login| AAD
    end

    subgraph "Edge & Protection (Day 5/6)"
        GW[API Gateway / WAF]
        RL[Rate Limiter]
        GW --- RL
    end

    subgraph "Identity & Auth (Day 1/2/4)"
        SCIM[SCIM Endpoint]
        AuthN[OIDC/JWT Handler]
        AAD -->|2. Push Changes| SCIM
        User -->|3. Access Request| GW
    end

    subgraph "Authorization & Logic (Day 3/8)"
        Spice[SpiceDB / ReBAC]
        Redis[(Redis L2 Cache)]
        DB[(Postgres RLS DB)]
        Sidecar[Identity Sidecar]
    end

    subgraph "Monitoring (Day 7)"
        Logs[Structured Logs]
        SIEM[Azure Sentinel]
    end

    %% Connections
    AAD -.->|JWT w/ acr claim| AuthN
    GW --> AuthN
    AuthN --> Sidecar
    Sidecar -->|Check Cache| Redis
    Redis -.->|Miss| Spice
    Spice --> DB
    SCIM -->|Update| Spice
    SCIM -->|Invalidate| Redis
    
    %% Observability
    TM_API[Thumbnail API] --> Logs
    Logs --> SIEM

```

---

## 🏛️ Final Architect’s Summary

| Component | Responsibility | Day |
| --- | --- | --- |
| **OIDC / JWT** | Proving *who* the user is via Acme Corp’s trusted IdP. | 1 & 2 |
| **ReBAC (SpiceDB)** | The "Zanzibar" graph that handles complex, fine-grained permissions. | 3 |
| **SCIM** | The background pipe that automates onboarding and the "Kill Switch." | 4 |
| **Workload Identity** | Removing static secrets so code talks to code via "DNA" (mTLS/SPIFFE). | 5 |
| **WebAuthn / Step-Up** | Requiring a YubiKey tap for $300k destructive GPU actions. | 6 |
| **SIEM / Otel** | The "Security Cameras" that detect botnets and credential stuffing. | 7 |
| **Postgres RLS** | The "Hard Shield" that prevents Customer A from seeing Customer B's data. | 8 |
| **Global Resilience** | Failing-closed and using Redis to keep the system fast and local. | 8 |

---

### 📝 The Final Cheat Sheet: "The Architect's Creed"

1. **Never Trust the Client:** Always re-validate JWTs and permissions on the backend.
2. **ExternalId is the Anchor:** Never link users by email; emails change, but the `externalId` from the IdP is forever.
3. **Fail-Closed is the Only Way:** If the security check is uncertain, the answer is "No."
4. **No Static Secrets:** Use Managed Identities and SPIFFE to kill the "Secret Zero" problem.
5. **Log for the Machine:** Use structured JSON logs so your SIEM can defend you while you sleep.
