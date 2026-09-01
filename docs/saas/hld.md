# HLD — odooPilot

> High-Level Design for the SaaS built on top of `odoo-mcp`, driven by Claude Haiku 4.5.
> Status: **DRAFT v0.3** — owner: Product. Companion docs: [ideation.md](./ideation.md) · [LLD](./lld.md) · [Roadmap](./roadmap.md).
> v0.3 changes: Bedrock model pinned — `eu.anthropic.claude-haiku-4-5-20251001-v1:0`, `AWS_DEFAULT_REGION=eu-north-1`.
> v0.2 changes: official Haiku 4.5 pricing locked (§6); orchestration moved to **AWS Bedrock, EU geography only** (mandatory data residency); availability SLO relaxed to 99.5%.

---

## 1. Product summary

**odooPilot** is a hosted, multi-tenant AI assistant ("chat with your Odoo"). Customers connect their existing Odoo 16–19 instance (Community or Enterprise, zero server-side install), and business users get a conversational workspace where **Claude Haiku 4.5** is the reasoning agent that drives the [odoo-mcp](../../README.md) tool surface to answer questions, run reports, and — behind human approval — execute writes.

The SaaS does **not** fork odoo-mcp. It consumes the published PyPI package (pinned version) as the data plane, and upstreams any needed fixes into this repo.

### Why this wins

| Wedge | Evidence from the MCP |
| --- | --- |
| Zero-setup onboarding | MCP requires no App Store module, no admin access — a connection wizard can onboard a customer in minutes |
| Safe AI writes are already solved | `preview_write → validate_write → execute_approved_write` token gate + audit trail = the compliance story competitors lack |
| Agency fleet angle (unique) | Cross-instance fan-out tools (`search_across_instances`, `accounting_health_across_instances`) enable a multi-client console no single-DB competitor can copy cheaply |
| Finance self-service | Accounting pack (`receivable_payable_aging`, `accounting_health_summary`) answers the #1 SMB question with one call |
| Cost economics | Haiku 4.5 (~$1/$5 per MTok, prompt caching) keeps COGS per query at cents |

### Target customers (priority order)

1. **SMBs on Odoo Community** (16–18) priced out of Enterprise AI / dedicated admins.
2. **Odoo agencies & partners** managing many client DBs (fleet dashboards, cross-client reports).
3. **Finance/ops teams** wanting self-service AR/AP and reporting without building Odoo views.

---

## 2. System context

```
                       ┌────────────────────────────────────────────────┐
                       │                    USERS                       │
                       │   business users · agency admins · owners      │
                       └───────────────┬────────────────────────────────┘
                                       │ HTTPS
                        ┌──────────────▼───────────────┐
                        │        Web App (Next.js)     │  chat UI · connections ·
                        │        + API gateway (BFF)   │  playbooks · admin · billing
                        └───┬──────────────┬───────────┘
                            │ REST+SSE     │ OIDC
              ┌─────────────▼───────┐  ┌───▼────────────┐
              │   Agent Runtime     │  │  Identity/RBAC │
              │  Claude Haiku 4.5   │  │ (managed OIDC) │
              │  tool-use loop      │  └────────────────┘
              └───┬──────────┬──────┘
                  │          │ MCP (stdio subprocess per org)
    ┌─────────────▼──┐   ┌───▼──────────────────────────┐        ┌──────────────────┐
    │ Control plane  │   │  odoo-mcp worker pool         │ XML-RPC│ Customer Odoo    │
    │ Postgres·Redis │   │  (one process per org,        │ JSON-2 │ 16–19 instances  │
    │ Stripe·KMS·OTel│   │  multi-instance config file)  ├───────►│ (customer infra) │
    └────────────────┘   └──────────────────────────────┘        └──────────────────┘
```

**Trust boundary:** customer credentials live encrypted at rest; each org's Odoo traffic flows through its own spawned MCP worker; no cross-tenant cache sharing is possible because schema caches/approval tokens are per-process.

## 3. Components

| Component | Tech | Responsibility |
| --- | --- | --- |
| **Web app** | Next.js 14+ | Chat streaming UI, connection wizard, approvals inbox, playbook gallery, org admin, usage/billing pages |
| **API gateway (BFF)** | FastAPI | AuthN/Z enforcement, REST + SSE endpoints, quota checks, request tracing, fan-out to services |
| **Agent Runtime** | FastAPI workers + Bedrock client (`AnthropicBedrock`, EU only) | The core: conversation orchestration, Claude Haiku 4.5 tool-use loop, MCP session manager (subprocess pool), approval state machine, token accounting |
| **MCP worker pool** | `odoo-mcp` (PyPI pin) | Per-org stdio subprocess launched with generated `odoo_config.json`; owns Odoo connectivity, field ACLs, write gate, rate limits, audit JSONL |
| **Control plane DB** | Postgres | Orgs/users/connections/conversations/messages/tool invocations/usage events/audit mirror/subscriptions |
| **Cache/queues** | Redis | Session presence, platform rate limits, async job queue (reports, digests) |
| **Billing** | Stripe | Subscriptions + metered credits; webhooks → billing worker |
| **Secrets** | KMS + envelope encryption (AES-256-GCM) | Odoo credentials encrypted per record; decrypted only into the worker's config file at spawn |
| **Observability** | OpenTelemetry + Grafana/Langfuse | One trace spans HTTP → agent loop → LLM call → MCP tool call → Odoo RPC |

### Model strategy

- **Primary brain: `claude-haiku-4-5`, served exclusively via AWS Bedrock EU geography** — fast, cheap, strong tool-calling; system prompt + tool schemas are prompt-cached across turns. Runtime pins the EU inference profile **`eu.anthropic.claude-haiku-4-5-20251001-v1:0`** with `AWS_DEFAULT_REGION=eu-north-1`. Inference, storage, and queues stay inside EU regions — hard requirement, no non-EU fallback.
- Tool-surface trimming via `ODOO_MCP_TOOLS_INCLUDE`: expose a curated ~20-tool default set; mode presets (Finance, Ops, Migration) swap in specialized packs so Haiku isn't drowning in 41 schemas.
- **Model router abstraction** in the runtime so Phase 3+ can escalate hard turns to Sonnet without re-plumbing (kept out of MVP scope).
- Official pricing locked (table in §6). Scheduled reports/digests may run through the **Bedrock batch API** (50% cheaper per token) from Phase 3.

## 4. Key flows

### 4.1 Read flow (the 90% case)
1. User message → BFF → Agent Runtime (SSE stream opens).
2. Runtime loads org context (connection set, role, locale, write policy) and renders system prompt.
3. Haiku plans → calls read tools (`search_records`, `aggregate_records`, …) against the org's warm MCP worker.
4. Results stream back; final answer rendered with tables/charts; tokens metered per event.

### 4.2 Write flow (gated, human-in-the-loop)
1. User asks for a change ("confirm PO 123", "create customer X").
2. Runtime drives `preview_write → validate_write` → returns an **approval card** in chat; turn pauses; pending action persisted.
3. Approver confirms in the UI (maps to the MCP elicitation/HITL model). Org-level policy decides who may approve (Admin-only by default).
4. On approval, resume against the *same* warm worker (same-session token requirement) → `execute_approved_write`.
5. Every step mirrored from the worker's JSONL audit log into the control-plane DB; visible in the org's Audit page.

### 4.3 Connection onboarding
Wizard collects URL/db/credentials → runtime spawns ephemeral worker → calls `get_odoo_profile` + `health_check` → shows what the credential can see (`diagnose_access`) → stores encrypted connection → warm worker kept for the org.

### 4.4 Agency fleet flow
Agency org connects N client instances (multi-instance config) → fleet dashboard uses cross-instance fan-out tools → per-client attribution preserved in results (`_instance` tags).

## 5. Tenancy & security posture

| Concern | Decision |
| --- | --- |
| Isolation unit | Org → own MCP worker process(s), own config file, own schema caches/tokens |
| Credentials | Envelope-encrypted in Postgres; materialized only into tmpfs config file (0600) at spawn; never logged |
| Authorization | Platform RBAC (Owner/Admin/Member/Viewer) × Odoo-native permissions (the connected credential's roles/record rules still decide final results — defense in depth) |
| Field privacy | Map org policies to `ODOO_MCP_FIELD_POLICY_FILE` (e.g., deny salary fields for Member role) |
| Writes | Disabled by default per org; enabling is Admin action; side-effect methods via exact `ODOO_MCP_ALLOWED_SIDE_EFFECT_METHODS` allowlist (no broad mode); every write needs UI approval |
| Prompt injection | Odoo record content treated as untrusted data; approval cards show raw diffs; system prompt hardening + injection eval suite |
| Rate limiting | Two layers: MCP-native sliding window per instance/tool + platform credits/quota per plan |
| Audit | Worker JSONL audit → ingested to DB, immutable, exportable |
| Compliance path | **EU-only data residency is mandatory** — LLM inference via AWS Bedrock EU geography (in-region / EU cross-region inference profiles), storage/queues/KMS in EU regions; DPA templates; GDPR deletion workflow; SOC2-ready practices tracked in roadmap |

## 6. Cost model (estimate — verify before pricing)

Official Claude Haiku 4.5 prices (USD per 1M tokens):

| Input | Output | Batch in | Batch out | Cache write 5m | Cache write 1h | Cache read |
| --- | --- | --- | --- | --- | --- | --- |
| $1.10 | $5.50 | $0.55 | $2.75 | $1.375 | $2.20 | $0.11 |

Worked example per complex query (~12 tool round-trips, ~120K cumulative input tokens ≈80% cache-read, ~3K output):

| Component | Volume | Price | Cost |
| --- | --- | --- | --- |
| Fresh input | 24K | $1.10/M | $0.026 |
| Cache read | 96K | $0.11/M | $0.011 |
| Output | 3K | $5.50/M | $0.017 |
| **Total** | | | **≈ $0.054/query** |

Cache *writes* are paid once per cache miss (5m: $1.375/M, 1h: $2.20/M) — default to the **1h TTL** so warm conversations keep reusing the cached system prompt + tool schemas; simple queries land well below $0.03. Scheduled fleet reports route through the batch API at half price.

**Spike telemetry (2026-09-01, G0a run — 30-question golden set, Odoo 19, 12-tool preset):** avg **$0.023/query fully uncached** (range $0.008–$0.090), latency p50 5.9 s / p95 37.4 s (p95 dominated by multi-step schema exploration, 9–12 tool rounds). Caching did not engage because Haiku 4.5 requires **≥4,096 tokens per cache checkpoint** and the trimmed spike prefix is ~3.1K tokens; a separate probe over the threshold confirmed caching works on the EU profile (write 16,802 → read 16,802, 1h TTL). Production presets (~18–20 tools + full system prompt) clear the minimum, so the worked example above stands — with the caveat that the **cache is per-region** and the EU cross-region profile balances across 6 regions: re-measure the effective cache-read share (assumed 80%) with production prompts in Phase 1 before locking prices. Bedrock on-demand quota in `eu-north-1` throttles bursts (429): request a service-quota raise before multi-user phases.

Planned tiers (draft): Free trial (50 queries) · Pro $49/mo (500 queries + 25 approved writes) · Team $149/mo (2,000 + playbooks + 3 connections) · Business $499/mo (10,000 + fleet console + SLA). Credits for overage. Gross-margin target >70%.

## 7. Non-functional targets

| Metric | Target |
| --- | --- |
| First streamed token (p50 / p95) | < 2 s / < 4 s |
| Simple question end-to-end (p95) | < 15 s |
| Availability | 99.5% control plane (SLO); degraded-read mode if the LLM provider degrades (queued retries, EU only) |
| Tenant isolation failure tolerance | 0 cross-org reads tolerated (tests assert it) |
| Write safety | No unapproved write can ever execute; approval-token same-session invariant enforced by tests |

## 8. Explicit assumptions

1. BitClick wants a hosted product (not on-prem) initially — on-prem/self-host becomes a Phase 4 partner option since odoo-mcp is MIT anyway.
2. Customers supply their own Odoo credentials (BYO-instance model); we never host their Odoo.
3. English UI first, Spanish second (ES market is home turf).
4. mcp-odoo repo remains pure OSS upstream; the SaaS lives in a new sibling repo consuming the pinned package.

## 9. Open decisions (tracked in ideation.md)

- Product name/trademark search.
- OIDC provider choice (Clerk vs Auth0 vs self-hosted Keycloak) — see LLD §9.
- Exact credit definition & overage pricing after beta telemetry.
