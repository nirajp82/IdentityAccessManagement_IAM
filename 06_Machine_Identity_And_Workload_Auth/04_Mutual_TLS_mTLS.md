# Mutual TLS (mTLS)

## Overview

**Mutual TLS (mTLS)** is TLS where *both* sides prove who they are. In ordinary HTTPS only the server presents a certificate — your browser checks that `moneyguard.com` really is MoneyGuard, but the server has no idea who you are until you send a password or a token *inside* the encrypted tunnel.

With mTLS the server also demands an **X.509 client certificate** during the handshake. If the client cannot present one the server trusts, the connection is dropped at the **transport layer** — before a single byte of HTTP is read.

That single property is what makes mTLS the strongest machine-to-machine authentication available: the credential is a private key that never leaves the machine, so a stolen token or copied config file is useless without it.

🏦 **MoneyGuard scenario:** The **Fraud Detection Pipeline** pods call the **Core Banking Ledger** — the crown jewels — thousands of times a minute. Even with the short-lived tokens of [Chapter 02](./02_OAuth_Client_Credentials_Grant.md), an attacker who dumps a pod's memory for 15 minutes can replay that token from anywhere. With mTLS, the Ledger refuses the TCP connection unless the caller presents the SVID certificate SPIRE issued to `spiffe://moneyguard.internal/ns/fraud-prod/sa/svc-fraud-detection` — a certificate whose private key lives only in that pod's memory and rotates hourly.

## Why This Matters

- **It kills token replay.** The classic breach — Eve steals a valid token and uses it from her laptop — simply fails, because her laptop has no client certificate.
- **It fails fast and cheap.** Rejection happens in the TLS handshake. Your application code never executes, so a flood of unauthenticated attackers costs almost nothing to repel.
- **It is Zero Trust made real.** "Never trust, always verify" means being on the internal network grants exactly zero access. mTLS enforces that at every hop.
- **It is legally required in places.** Open Banking / PSD2 obliges financial APIs to use mTLS to identify third-party providers. MoneyGuard cannot opt out.
- **It pairs perfectly with SPIFFE.** [Chapter 03](./03_Workload_Identity_Federation.md) produces SVID certificates; mTLS is what you *do* with them.

## Key Concepts

| Term | Plain-English meaning |
|---|---|
| **TLS** | Transport Layer Security — the encryption behind HTTPS. |
| **mTLS** | TLS where the client also presents a certificate. |
| **X.509 certificate** | A signed document binding an identity to a public key. |
| **Private key** | The secret half of the pair. It must never leave the machine. |
| **CA (Certificate Authority)** | The trusted issuer whose signature makes a certificate believable. |
| **Trust bundle / root store** | The set of CA certificates a party is willing to trust. |
| **Handshake** | The negotiation at the start of a connection where certificates are exchanged and verified. |
| **SAN (Subject Alternative Name)** | The certificate field holding names/URIs — where a SPIFFE ID lives. |
| **Thumbprint / fingerprint** | A hash of a certificate, used to pin or identify it exactly. |
| **CRL / OCSP** | Mechanisms for checking whether a certificate has been revoked. |
| **Service mesh** | Infrastructure (Istio, Linkerd) that transparently adds mTLS between services via sidecar proxies. |
| **Termination point** | Where TLS is decrypted — the gateway, the sidecar, or the app itself. |

## Detailed Notes

### Concept 1 — Standard TLS versus mutual TLS

| Feature | TLS (standard HTTPS) | Mutual TLS |
|---|---|---|
| Server authenticates to client | ✅ Yes | ✅ Yes |
| Client authenticates to server | ❌ No | ✅ Yes |
| Certificates involved | Server only | Server **and** client |
| Where the client is identified | Application layer (password/token, after the tunnel opens) | **Transport layer**, before any HTTP |
| Rejection cost | Full request parsed, then `401` | TCP reset during handshake |
| Typical use | Browsers reaching websites | APIs, microservices, IoT, partner banking |

The row that matters is the fourth. In standard TLS the server accepts the connection from anybody and then decides. In mTLS the server never accepts the connection at all.

### Concept 2 — The handshake, step by step

1. **Client Hello** — "I want to connect securely."
2. **Server Hello + server certificate** — the server proves it is the Core Banking Ledger.
3. **Client verifies** the server certificate against its trust bundle. (This half is ordinary TLS.)
4. **Certificate Request** — "Now show me your ID." *(This is the mutual part.)*
5. **Client sends its X.509 certificate** and proves possession of the matching private key by signing handshake data.
6. **Server verifies** the client certificate: signed by a trusted CA, not expired, not revoked, and the identity inside it is one this API accepts.
7. **Either** the connection is dropped (TCP reset), **or** the encrypted tunnel opens and HTTP finally begins.

⚠️ Step 5 is why a certificate file alone is not enough: without the private key you cannot complete the signature. This is exactly why mTLS resists copying.

### Concept 3 — Why mTLS beats bearer tokens

| Attack | Bearer token only | mTLS (or token bound to mTLS) |
|---|---|---|
| Eve steals a token from a log/memory dump | ✅ Works from her laptop | ❌ No client certificate → connection refused |
| Eve is on the internal VPN | ✅ Reaches the API and tries tokens | ❌ Handshake fails; app never sees her |
| Eve replays an old request | ✅ Works until token expiry | ❌ Needs the live private key |
| Eve MITMs the connection | Possible if certificate validation is sloppy | ❌ Both ends verify certificates |
| Eve phishes a credential | Possible | ❌ Nothing phishable — the key never moves |

💡 The one-line version for an interview: **"a bearer token proves you know a secret; a client certificate proves you *are* a specific machine."**

The strongest production pattern combines both: an OAuth access token whose `cnf` claim is bound to the client certificate (RFC 8705, covered in [Chapter 02](./02_OAuth_Client_Credentials_Grant.md)). The token carries the scopes; the certificate carries the machine identity; neither works without the other.

### Concept 4 — Where the identity actually lives in the certificate

Two conventions:

- **Traditional:** the Common Name or a DNS SAN, e.g. `CN=fraud-scorer.fraud-prod.svc.moneyguard.internal`. Workable, but tied to network naming.
- **SPIFFE (preferred):** a URI SAN, e.g. `URI:spiffe://moneyguard.internal/ns/fraud-prod/sa/svc-fraud-detection`. Network-independent, human-readable, and directly usable as an authorization subject.

The Ledger's rule becomes a readable policy statement rather than an IP allow-list:

```
allow  spiffe://moneyguard.internal/ns/fraud-prod/sa/svc-fraud-detection  →  POST /v1/fraud-scores
deny   everything else
```

### Concept 5 — Authentication is not authorization

⚠️ The most common design error: "the certificate is valid, therefore let them in." A valid certificate only proves *which* workload is calling. You still must check that **this specific identity** is allowed to call **this specific endpoint**.

A compromised low-value service inside the mesh has a perfectly valid certificate. If the Ledger accepts any certificate signed by the internal CA, that compromised service can drain it. Always match the SPIFFE ID (or thumbprint) against an allow-list per route.

### Concept 6 — Certificate lifecycle: the hard part

mTLS fails in production for operational reasons, not cryptographic ones.

- **Issuance.** Manual certificate handling does not scale to 50 pods that restart constantly. Use SPIRE, cert-manager, or a service mesh so issuance is automatic.
- **Rotation.** Short lifetimes are a feature: SPIRE SVIDs typically live about an hour and rotate automatically. Long-lived certificates recreate the static-secret problem.
- **Revocation.** CRL and OCSP are slow and operationally painful. 💡 **Short lifetimes are the practical revocation strategy** — a one-hour certificate that is not renewed is dead within an hour, no distributed revocation list required.
- **Trust bundle distribution.** Every service needs the CA bundle, and it must be updated *before* the CA rotates. Getting this wrong causes a total, simultaneous outage.
- **Clock skew.** Certificates are time-bound. A node with a drifting clock rejects perfectly valid certificates. Run NTP everywhere.

### Concept 7 — Who terminates TLS? Three deployment models

| Model | How it works | Pros | Cons |
|---|---|---|---|
| **In the application** | Kestrel/NGINX in the app process demands the client certificate | Full control; identity visible to app code | Every team must implement it correctly |
| **Service mesh sidecar** | Istio/Linkerd proxy handles mTLS transparently; app speaks plain HTTP to localhost | Zero app changes; consistent everywhere; automatic rotation | Extra moving parts; app must trust the sidecar's forwarded identity header |
| **At the edge / API Gateway** | The gateway terminates client mTLS, then uses its own identity internally | Simple for external partners | Internal hops are unprotected unless a mesh covers them |

MoneyGuard uses all three: mesh mTLS inside the cluster, gateway mTLS for PartnerBank, and explicit in-app mTLS for the Core Banking Ledger because it wants the identity check in its own code, not delegated.

⚠️ In the sidecar model, the app must **only** trust the identity header from the sidecar on the loopback interface. If an attacker can set `X-Forwarded-Client-Cert` from outside, the whole scheme collapses.

### Concept 8 — Where mTLS is a poor fit

- **Human users.** Installing and managing client certificates on 50,000 employees' laptops and phones is an operational nightmare. Humans get WebAuthn instead ([Module 07](../07_Advanced_Authentication_And_MFA/README.md)).
- **Public browser-facing APIs.** Browsers handle client certificates badly, and the UX is terrible.
- **Very short-lived serverless with cold starts**, where the handshake cost matters — though session resumption largely mitigates this.

## Workflow

```mermaid
sequenceDiagram
    autonumber
    participant FD as Fraud Detection Pod
    participant Agent as SPIRE Agent (same node)
    participant LED as Core Banking Ledger API

    Note over FD,Agent: Startup - no secrets in the image
    FD->>Agent: Fetch X.509 SVID (over local socket)
    Agent->>Agent: Attest caller via kernel (pod, namespace, service account)
    Agent-->>FD: Short-lived cert + private key (in memory, ~1h TTL)

    Note over FD,LED: Phase 1 - standard TLS
    FD->>LED: Client Hello
    LED-->>FD: Server Hello + Ledger server certificate
    FD->>FD: Verify server cert against MoneyGuard trust bundle

    Note over FD,LED: Phase 2 - the mutual part
    LED->>FD: Certificate Request
    FD-->>LED: X.509 SVID + proof of private key possession
    LED->>LED: Verify chain, expiry, revocation, then read SAN URI

    alt SPIFFE ID on the allow-list for this route
        Note over FD,LED: Phase 3 - tunnel open
        FD->>LED: POST /v1/fraud-scores (Bearer token also bound via cnf)
        LED-->>FD: 201 Created
    else No certificate, untrusted CA, or unknown SPIFFE ID
        LED-->>FD: TCP reset - handshake failed, app code never ran
    end
```

## Architecture Diagram

```mermaid
flowchart TB
    subgraph EXT["External"]
        PB["PartnerBank<br/>client certificate"]
        EVE["Eve's laptop<br/>stolen token, no certificate"]
    end

    GW["MoneyGuard API Gateway<br/>terminates partner mTLS"]

    subgraph MESH["MoneyGuard Kubernetes - service mesh, mTLS everywhere"]
        FD["Fraud Detection Pipeline<br/>spiffe://.../sa/svc-fraud-detection"]
        LED["Core Banking Ledger<br/>spiffe://.../ledger/core-banking-api"]
        RPT["Reporting Service<br/>spiffe://.../sa/svc-reporting"]
    end

    subgraph PKI["Identity + PKI Plane"]
        SPIRE["SPIRE Server<br/>internal CA"]
        BUNDLE["Trust bundle<br/>distributed to all workloads"]
    end

    SIEM["SIEM - Sentinel"]

    PB -->|"mTLS"| GW
    EVE -.->|"no client cert<br/>TCP RESET"| GW
    GW -->|"mTLS"| FD
    FD -->|"mTLS + SPIFFE ID allow-list"| LED
    RPT -.->|"valid cert but ID not on<br/>Ledger allow-list - 403"| LED
    SPIRE -->|"issues short-lived SVIDs"| FD
    SPIRE -->|"issues short-lived SVIDs"| LED
    SPIRE --> BUNDLE
    LED -->|"peer SPIFFE ID + result"| SIEM
    GW -->|"handshake failures"| SIEM
```

## Step-by-Step Process

**Rolling out mTLS between the Fraud Detection Pipeline and the Core Banking Ledger:**

1. **Stand up the internal CA.** SPIRE (or cert-manager with an intermediate CA) becomes the single issuer for workload certificates. Never use a public CA for internal workload identity.
2. **Distribute the trust bundle** to every service *first*. Everyone must be able to verify before anyone starts requiring.
3. **Issue certificates automatically.** SPIRE agents mint SVIDs with the SPIFFE ID in the SAN URI and a ~1-hour TTL, rotating in the background.
4. **Enable permissive mode.** The Ledger accepts both plain TLS and mTLS, and *logs* which callers presented a certificate. This is the critical migration step — it tells you who would break.
5. **Watch the logs until every legitimate caller is presenting a certificate.** Chase the stragglers.
6. **Build the authorization allow-list.** Map each route to the SPIFFE IDs permitted to call it. `POST /v1/fraud-scores` → the fraud pipeline only.
7. **Flip to strict mode.** The Ledger now requires a client certificate; anything without one gets a TCP reset.
8. **Add sender-constrained tokens.** Bind the OAuth access token to the certificate via the `cnf` claim so authentication and authorization are cryptographically linked.
9. **Monitor the right signals**: handshake failure rate, certificate expiry distribution (nothing should be within an hour of expiry without renewing), CA rotation status, and clock skew per node.
10. **Rehearse CA rotation** in staging before you need it in production. This is the operation most likely to cause a total outage.

## Commands and Examples

**Generate a test client certificate (local lab):**

```bash
openssl req -x509 -newkey rsa:4096 -nodes -days 365 \
  -keyout fraud-client.key -out fraud-client.crt \
  -subj "/CN=fraud-scorer.moneyguard.internal"
```

**Scenario A — Eve, with a valid stolen token but no certificate:**

```bash
curl -k https://ledger.moneyguard.internal/v1/fraud-scores \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...stolen..."
```

```text
curl: (56) OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:
      tlsv13 alert certificate required, errno 0
```

⚠️ Read that error carefully: it is **not** `401 Unauthorized` or `403 Forbidden`. The request never reached HTTP. OpenSSL killed the connection at the transport layer, and the Ledger's application code was never invoked — it does not even know Eve tried.

**Scenario B — the genuine Fraud Detection Pipeline:**

```bash
curl https://ledger.moneyguard.internal/v1/fraud-scores \
  --cert fraud-client.crt --key fraud-client.key \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiJ9..." \
  -d '{"transaction_id":"TX-88213","risk_score":0.94}'
```

```text
{"status":"accepted","transaction_id":"TX-88213"}
```

**Server side — demand a certificate at the transport layer (C#/Kestrel):**

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1. TRANSPORT LAYER: refuse the connection without a client certificate.
builder.WebHost.ConfigureKestrel(o =>
{
    o.ConfigureHttpsDefaults(https =>
    {
        https.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
    });
});

// 2. APPLICATION LAYER: turn the certificate into an identity we authorize on.
builder.Services.AddAuthentication(CertificateAuthenticationDefaults.AuthenticationScheme)
    .AddCertificate(options =>
    {
        options.AllowedCertificateTypes = CertificateTypes.Chained;
        options.RevocationMode = X509RevocationMode.NoCheck;   // SVIDs are ~1h; TTL is the revocation

        options.Events = new CertificateAuthenticationEvents
        {
            OnCertificateValidated = ctx =>
            {
                // The SPIFFE ID lives in the SAN URI (OID 2.5.29.17).
                var san = ctx.ClientCertificate.Extensions["2.5.29.17"]?.Format(false) ?? "";

                const string FraudPipeline =
                    "URI=spiffe://moneyguard.internal/ns/fraud-prod/sa/svc-fraud-detection";

                if (!san.Contains(FraudPipeline, StringComparison.OrdinalIgnoreCase))
                {
                    ctx.Fail($"Untrusted workload identity: {san}");
                    return Task.CompletedTask;
                }

                // AuthN done. AuthZ still applies per route.
                ctx.Principal = new ClaimsPrincipal(new ClaimsIdentity(
                    new[] { new Claim("spiffe_id", FraudPipeline) },
                    CertificateAuthenticationDefaults.AuthenticationScheme));
                ctx.Success();
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization(o =>
    o.AddPolicy("FraudPipelineOnly", p =>
        p.RequireClaim("spiffe_id",
            "URI=spiffe://moneyguard.internal/ns/fraud-prod/sa/svc-fraud-detection")));

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapPost("/v1/fraud-scores", (FraudScore s) => Results.Created($"/v1/fraud-scores/{s.TransactionId}", s))
   .RequireAuthorization("FraudPipelineOnly");
app.Run();
```

**Client side — attach the SPIRE-issued SVID (C#):**

```csharp
using var workload = await WorkloadApiClient.CreateAsync();
var svid = (await workload.FetchX509ContextAsync()).DefaultSvid.Certificate;

var handler = new HttpClientHandler();
handler.ClientCertificates.Add(svid);       // private key stays in memory, never on disk
using var http = new HttpClient(handler);

var res = await http.PostAsJsonAsync(
    "https://ledger.moneyguard.internal/v1/fraud-scores",
    new { transaction_id = "TX-88213", risk_score = 0.94 });
```

**Service mesh alternative — mTLS with no application code at all (Istio):**

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ledger-prod
spec:
  mtls:
    mode: STRICT                # reject any plaintext connection
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: ledger-fraud-scores
  namespace: ledger-prod
spec:
  selector:
    matchLabels: { app: core-banking-ledger }
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["moneyguard.internal/ns/fraud-prod/sa/svc-fraud-detection"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/v1/fraud-scores"]
```

**Inspect what a peer actually presented:**

```bash
openssl s_client -connect ledger.moneyguard.internal:443 \
  -cert fraud-client.crt -key fraud-client.key 2>/dev/null \
  | openssl x509 -noout -text | grep -A2 "Subject Alternative Name"
```

```text
X509v3 Subject Alternative Name:
    URI:spiffe://moneyguard.internal/ns/fraud-prod/sa/svc-fraud-detection
```

## Best Practices

- **Automate issuance and rotation** (SPIRE, cert-manager, or a mesh). Manual certificates do not survive contact with 50 restarting pods.
- **Keep certificate lifetimes short** — hours, not years. Short TTLs *are* your revocation mechanism.
- **Put the identity in the SAN URI as a SPIFFE ID**, not in the Common Name, and authorize on it.
- **Always authorize after authenticating.** A trusted CA signature is not permission to call `/v1/fraud-scores`.
- **Roll out in permissive mode first**, log who is missing certificates, then flip to strict.
- **Distribute the trust bundle before requiring it**, and rehearse CA rotation in staging.
- **Combine mTLS with sender-constrained tokens** (`cnf` / RFC 8705) on high-value APIs like the Ledger.
- **Use a private internal CA** for workload identity — never a public CA, whose certificates are trusted far too broadly.
- **Run NTP everywhere.** Clock skew is a leading cause of mysterious mTLS failures.
- **Log the peer identity on every request** so Dana Cole can answer "which workload called the Ledger" without guessing.

## Common Mistakes

- ⚠️ **Accepting any certificate signed by the internal CA.** Every workload in the mesh then has Ledger access. Authenticate *and* authorize.
- ⚠️ **Long-lived client certificates.** A 2-year certificate with a key on disk is a static secret wearing a hat.
- ⚠️ **Relying on CRL/OCSP for revocation.** They're slow, often fail open, and add a hard dependency. Prefer short TTLs.
- ⚠️ **Forgetting to renew before expiry.** The classic 3am outage: every certificate in a namespace expires within the same minute because they were all issued together.
- ⚠️ **Rotating the CA without distributing the new bundle first.** Instant, total, cluster-wide outage.
- ⚠️ **Trusting a forwarded client-certificate header from outside the pod.** If `X-Forwarded-Client-Cert` is settable by a caller, mTLS is decorative.
- ⚠️ **Terminating mTLS at the edge and leaving internal hops in plaintext**, then calling it Zero Trust.
- ⚠️ **Trying to use mTLS for human users.** Certificate management on personal devices is a support catastrophe; use WebAuthn instead.
- ⚠️ **Disabling revocation checks and validation in a lab, then shipping that config** to production.

## Pro Tips

- 💡 **Run the local lab.** Start a Kestrel API in `RequireCertificate` mode, hit it with plain `curl`, and watch the TLS error rather than a `401`. Seeing the connection die before your code runs is the moment mTLS clicks.
- 💡 **"Short TTL is the revocation strategy."** Saying this in an interview shows you have actually operated PKI, because everyone else reaches for CRLs and then discovers they don't work.
- 💡 **Certificate expiry is your top alert.** Alert at 50% of remaining lifetime, not at expiry. And stagger issuance so certificates don't all expire simultaneously.
- 💡 **Use the mesh for breadth, in-app mTLS for the crown jewels.** Blanket coverage from Istio, plus an explicit in-code identity check on the Ledger so its security does not depend entirely on a sidecar being configured correctly.
- 💡 **mTLS plus token = defence in depth.** The certificate answers "which machine?", the token answers "allowed to do what?". Bind them with `cnf` so neither is useful alone.

## Real-World Use Cases

- 🏦 **Fraud Detection Pipeline → Core Banking Ledger.** Strict mTLS with SPIFFE SVIDs, per-route allow-list, and `cnf`-bound tokens. Eve's stolen token from a memory dump gets a TCP reset.
- 🏦 **PartnerBank → MoneyGuard API Gateway.** Regulated open-banking traffic where the partner's certificate is registered in advance and pinned by thumbprint.
- **Service mesh across MoneyGuard's cluster.** Istio in `STRICT` mode gives every service-to-service hop mTLS with no application changes, plus automatic rotation.
- **Legacy Windows reconciliation service.** A SPIRE agent attests the process via the Windows kernel and issues an SVID, so a 20-year-old service joins the zero-trust mesh without being rewritten.
- **IoT-style estate: MoneyGuard's ATM fleet.** Each ATM holds a device certificate with the key in a hardware security module, so a cloned ATM image without the hardware key cannot connect.
- **Blocking lateral movement.** A compromised reporting service holds a valid mesh certificate but its SPIFFE ID is not on the Ledger's allow-list, so it gets a `403` instead of the crown jewels — Concept 5 paying for itself.

## Key Takeaways

1. mTLS = TLS where **both** sides present certificates; the client is authenticated at the **transport layer**, before HTTP.
2. It defeats **token theft and replay**, because the private key never leaves the machine and cannot be copied out of a log or a config file.
3. A valid certificate proves *who*, not *what they may do* — **always authorize on the identity** (ideally the SPIFFE ID in the SAN URI).
4. The hard part is **lifecycle**: automated issuance, short TTLs as the revocation strategy, and careful trust-bundle/CA rotation.
5. Deploy it via a **service mesh** for breadth and **in-app** for crown-jewel services; terminate partner mTLS at the **API Gateway**.
6. Pair it with **sender-constrained tokens** (`cnf`) so machine identity and scoped authorization are cryptographically linked.
7. Use it for **machines**, not humans — humans get WebAuthn ([Module 07](../07_Advanced_Authentication_And_MFA/README.md)).

## Glossary

- **CA (Certificate Authority)** — The entity whose signature makes a certificate trustworthy.
- **CRL (Certificate Revocation List)** — A published list of revoked certificates.
- **`cnf` claim** — A token claim binding the token to a specific certificate or key.
- **Handshake** — The initial exchange establishing a TLS connection.
- **mTLS** — Mutual TLS; both parties authenticate with certificates.
- **OCSP** — Online Certificate Status Protocol; a live revocation check.
- **Permissive mode** — A migration setting where both plaintext and mTLS connections are accepted.
- **PSD2** — EU payment services regulation that mandates strong API identification, including mTLS.
- **SAN (Subject Alternative Name)** — Certificate field holding alternate identities, including SPIFFE URIs.
- **Service mesh** — Sidecar-proxy infrastructure that applies mTLS and policy transparently.
- **SVID** — SPIFFE Verifiable Identity Document; the certificate carrying a SPIFFE ID.
- **Thumbprint** — A hash uniquely identifying a certificate, used for pinning.
- **Trust bundle** — The set of CA certificates a workload will accept.
- **TTL (Time To Live)** — How long a certificate remains valid.
- **X.509** — The standard format for public-key certificates.

## Revision Notes

- Standard TLS = server proves identity. mTLS = **both** prove identity, at the transport layer.
- Handshake order: Client Hello → Server cert → **Certificate Request** → Client cert + key proof → verify → tunnel.
- Failure looks like a **TLS alert / TCP reset**, not `401` — the app never runs.
- Identity goes in the **SAN URI** as a SPIFFE ID; authorize per route against an allow-list.
- **Short TTL (~1h) is the revocation strategy.** CRL/OCSP are the fallback, not the plan.
- Deployment models: in-app (control), sidecar mesh (breadth), gateway (partners).
- Combine with `cnf`-bound tokens: certificate = *who*, token = *what*.
- Top operational risks: expiry, CA rotation, trust-bundle distribution, clock skew.
- Machines yes, humans no.

## Interview Questions

**Q1. What exactly does mTLS protect against that a bearer token does not?**
A: Credential portability. A bearer token is a string; anyone who obtains a copy — from a log, a memory dump, a proxy, a screenshot — can use it from anywhere for its whole lifetime. An mTLS client credential is a private key that must sign live handshake data, so possessing a copy of the *certificate* is useless without the key, and well-run deployments keep the key in memory or hardware and rotate it hourly. It also moves rejection to the transport layer, so an unauthenticated attacker never reaches your application code.

**Q2. Eve steals a valid 15-minute access token from a pod's memory. Walk me through what happens with and without mTLS.**
A: Without mTLS, she sends it from her laptop, the Ledger validates the signature, audience, expiry and scope, everything passes, and she reads or writes crown-jewel data for the remaining window. With mTLS, her TCP connection is closed during the handshake with a `certificate required` alert; the Ledger never parses her HTTP request, so the token is irrelevant. If the token is additionally `cnf`-bound to the pod's certificate, even a compromised in-mesh service that could complete a handshake would fail token validation because its certificate hash doesn't match.

**Q3. How do you handle certificate revocation at scale?**
A: Mostly by not needing it. Issue certificates with roughly one-hour TTLs and rotate automatically, so a compromised identity dies within an hour of you deleting its registration entry — no distributed revocation list to propagate. CRL and OCSP exist, but CRLs go stale and OCSP adds a live dependency that typically fails open, which is worse than useless. For the rare "revoke right now" case, delete the SPIRE registration entry (stopping renewal) and push a deny rule for that SPIFFE ID at the authorization layer, which takes effect immediately.

**Q4. The Ledger accepts any certificate signed by MoneyGuard's internal CA. What's wrong?**
A: It has conflated authentication with authorization. Every workload in the mesh — the reporting service, a batch job, a third-party sidecar — holds a validly signed certificate. So compromising any single low-value service yields full Ledger access, which is exactly the lateral movement Zero Trust is meant to prevent. The fix is a per-route allow-list keyed on the SPIFFE ID in the SAN URI, so `POST /v1/fraud-scores` is callable only by the fraud pipeline's identity.

**Q5. How would you migrate a live service to mTLS with no downtime?**
A: Distribute the trust bundle to every participant first, so everyone *can* verify. Automate issuance so all clients obtain certificates. Put the server in permissive mode, accepting both plaintext and mTLS, and log which callers present certificates. Watch that log until the plaintext count reaches zero, chasing stragglers by name. Then flip to strict mode during a low-traffic window with a tested rollback to permissive. Only after strict mode is stable do you add the per-route SPIFFE ID allow-list — one change at a time, so any breakage has an obvious cause.

**Q6. What are the operational failure modes of mTLS, and how do you mitigate each?**
A: (1) Expiry — automate renewal, alert at 50% of remaining lifetime, and stagger issuance so certificates don't expire in lockstep. (2) CA rotation — always distribute the new trust bundle before switching the signing CA, and support multiple trusted roots simultaneously. (3) Clock skew — enforce NTP, since a drifting node rejects valid certificates. (4) Trust bundle drift — treat it as configuration with a rollout pipeline, not a file someone copies. (5) Handshake cost — enable session resumption and keep-alives so short-lived callers don't pay a full handshake per request.

**Q7. Why is mTLS unsuitable for MoneyGuard's 50,000 employees but ideal for its services?**
A: Certificates need secure key storage, enrolment, renewal, and recovery on every device. For humans that means provisioning to laptops, phones, and replacement devices, plus a helpdesk process for every lost or reimaged machine — an enormous support burden, and browsers present client-certificate prompts badly. Services, by contrast, are created by automation, so a SPIRE agent or mesh sidecar can enrol and rotate them with zero human involvement. For humans the equivalent security property — origin-bound, unphishable, key-never-leaves-the-device — comes from WebAuthn instead.

**Q8. Where should MoneyGuard terminate TLS, and what are the trade-offs?**
A: For external partners, at the API Gateway: it centralises certificate pinning and partner onboarding, but internal hops after it are only protected if a mesh covers them. For general internal traffic, at the mesh sidecar: universal coverage and automatic rotation with no application changes, but the app must trust the sidecar's forwarded identity and only from loopback. For the Core Banking Ledger, in the application itself: the identity check lives in code that the Ledger team owns and reviews, so its security does not depend solely on a mesh configuration someone might weaken. Running all three is normal and deliberate.

## Practice Exercises

1. **Build the lab.** Create a minimal API in `RequireCertificate` mode, generate a client certificate with OpenSSL, then call it twice — once without `--cert` and once with. Record the exact error text in the failing case and explain, in your own words, why it is not a `401`.

2. **Write the authorization layer.** Extend the Ledger example so that `POST /v1/fraud-scores` accepts only the fraud pipeline's SPIFFE ID, `GET /v1/reports` accepts only the reporting service's, and everything else is denied. Then describe the test that proves a valid mesh certificate from the wrong service gets `403`, not `201`.

3. **Design the rollout.** Write the migration plan to put the Core Banking Ledger into strict mTLS, listing each stage, the signal that says you may proceed, and the rollback trigger. Include what you do about a legacy caller that cannot present a certificate.

4. **Plan a CA rotation.** MoneyGuard must rotate the internal CA that signs all workload certificates. Write the ordered steps, state which step would cause a cluster-wide outage if done first, and name the two metrics you watch throughout.

5. **Threat model the sidecar.** In the mesh model, the application trusts an identity header from its sidecar. List three ways an attacker could try to forge or bypass that header and the control that stops each one.

---

⬅ [Chapter 03 — Workload Identity Federation](./03_Workload_Identity_Federation.md) | [Module 06 Index](./README.md) | [Main README](../README.md) | [MoneyGuard Scenario](../SCENARIO.md)
Next module: [Module 07 — Advanced Authentication & MFA](../07_Advanced_Authentication_And_MFA/README.md) ➡
