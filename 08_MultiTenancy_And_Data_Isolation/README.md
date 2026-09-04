# Module 08: Multi-Tenancy & Data Isolation

One application, many customers, zero data crossover — that is the multi-tenancy problem. This module covers the three standard ways to organize tenant data (siloed, pool, and bridge), the relational schema that maps users to roles to tenants, and PostgreSQL **Row-Level Security (RLS)**, which pushes tenant isolation down into the database engine so a forgotten `WHERE` clause becomes a bug instead of a breach.

🏦 The running example throughout is MoneyGuard's **Business Banking SaaS Portal**, where corporate clients such as **Acme Corp** and **Globex Inc** share the same code and the same database while remaining completely invisible to each other.

## 📂 Chapters

| # | Chapter | What It Covers |
|---|---------|----------------|
| 01 | [Tenancy Models](./01_Tenancy_Models.md) | Siloed vs. pool vs. bridge tenancy compared on isolation, cost, density, and migration effort; where the `tenant_id` must come from; tiered/hybrid tenancy; users who belong to more than one tenant. |
| 02 | [Database Schema & Row-Level Security](./02_Database_Schema_And_Row_Level_Security.md) | The tenants/users/roles/memberships schema; putting `tenant_id` on every table; PostgreSQL RLS policies (`USING` vs `WITH CHECK`); the `BYPASSRLS` trap; pooling-safe session variables; indexing and partitioning by tenant. |

## 🎯 Prerequisites

- **[Module 01 — Core IAM Concepts](../01_Core_IAM_Concepts/readme.md)** for authentication vs. authorization and least privilege.
- **[Module 03 — Authorization & Policy Engines](../03_Authorization_And_Policy_Engines/README.md)** helps, because RLS handles the *tenant boundary* while a policy engine handles *what a user may do inside their tenant*.
- **[Module 04 — Edge Security, Tokens & Sessions](../04_Edge_Security_Tokens_And_Sessions/README.md)** matters because the `tenant_id` that drives every isolation decision comes from a validated token claim.

New to the whole repo? Read [`SCENARIO.md`](../SCENARIO.md) first to meet MoneyGuard Bank, Acme Corp, and Globex Inc.

## 🔑 The One-Paragraph Summary

Pick **Pool** (shared tables plus a `tenant_id` column) as the default because it is cheapest and easiest to migrate, offer **Siloed** (a dedicated database) as a paid tier for clients with contractual or data-residency demands, and keep the application code identical across both so promotion is a data move rather than a rewrite. Then enforce the boundary in the database with Row-Level Security, driven by a session variable set from the validated JWT — with `FORCE ROW LEVEL SECURITY`, a `NOBYPASSRLS` application role, and both `USING` and `WITH CHECK` clauses, so that reads and writes are equally protected and an unset tenant context yields zero rows.

---

📖 [Main README](../README.md) | 🏦 [The MoneyGuard Story](../SCENARIO.md)

➡ Next module: [09 — Resilience, Observability & SIEM](../09_Resilience_Observability_And_SIEM/README.md)
