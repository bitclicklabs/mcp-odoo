# Roadmap — odooPilot

> Step-by-step execution plan. Companion docs: [ideation.md](./ideation.md) · [HLD](./hld.md) · [LLD](./lld.md).
> Status: **DRAFT v0.3** — dates are working estimates (start = approval of this doc, ✅ 2026-08-23).
> v0.3 changes: Phase 0 reordered — tech spike (G0a) strictly before interviews/landing; name locked: odooPilot.

---

## Phase 0 — Spike first, then validation (Weeks 1–3)

Goal: prove the Claude Haiku 4.5 × odoo-mcp loop on Bedrock EU **before** any demand-validation effort. Interviews and landing start only after the tech spike passes its sub-gate (decision D10).

### Stage A — Tech spike (goes first)

| # | Step | Output |
| --- | --- | --- |
| 0.1 | **Tech spike**: CLI harness — Claude Haiku 4.5 (`eu.anthropic.claude-haiku-4-5-20251001-v1:0` @ `eu-north-1`) driving a local odoo-mcp against Dockerized Odoo 17 demo | Golden-set v0 (~30 questions); validates tool use + prompt caching + EU residency on Bedrock; measures tool-selection accuracy & latency & cost/query with official prices ($1.10/$5.50 MTok) |
| 0.2 | Pricing sanity from spike telemetry | Cost model confirmed into HLD §6 |

**Sub-gate G0a (blocks Stage B):** tool-selection accuracy ≥90% on golden set · marginal cost/query <$0.10 · EU-only traffic verified.

### Stage B — Demand validation (starts only after G0a passes)

| # | Step | Output |
| --- | --- | --- |
| 0.3 | 15 customer interviews (SMB Odoo users + 5 Odoo agencies) | Validated problem ranking; pricing signal; pilot commitments |
| 0.4 | Landing page + waitlist (ES/EN) as **odooPilot** | ≥100 signups target |

**Gate G0 (exit Phase 0):** G0a passed · ≥3 interviewees pre-committed to a pilot · waitlist live.

## Phase 1 — MVP internal alpha (Weeks 3–8)

Goal: one org, real Odoo, chat that works end-to-end. Read-only.

Scope: monorepo scaffold (LLD §1) · OIDC auth + RBAC skeleton · connection wizard w/ live verification · chat UI with SSE streaming · agent runtime turn loop (read tools only) · metering to usage_events · basic audit view.

| # | Step | Notes |
| --- | --- | --- |
| 1.1 | Repo scaffold + CI gates (ruff/mypy/pytest) | Week 3 |
| 1.2 | MCP session manager + config renderer (contract tests vs pinned `odoo-mcp`) | Week 3–4 · the riskiest piece, do first |
| 1.3 | Turn loop + prompt presets (chat mode) + eval harness wired to CI | Week 4–6 |
| 1.4 | Web: wizard, chat, history | Week 5–7 |
| 1.5 | Internal dogfood on BitClick's own Odoo instance(s) | Week 7–8 |

**Gate G1:** ≥20 internal users · p95 simple-question <15 s · zero credential/log leaks in redaction audit · eval accuracy holds ≥90% · isolation test suite green.

## Phase 2 — Closed beta: writes, billing, playbooks (Weeks 9–14)

Goal: first external orgs paying (or committed to pay), safe writes live.

Scope: multi-tenant hardening + per-org workers at scale · **approval flow** (pending_actions, approvals inbox, admin policy, same-session resume) · Stripe subscriptions + credits · playbook v1: AR/AP aging report, invoice triage (`invoice_approval_chain` mapped), month-end checklist · agency multi-connection console (fleet read-only) · audit ingestion pipeline.

| # | Step | Notes |
| --- | --- | --- |
| 2.1 | Write gate E2E against demo Odoo incl. rejection/expiry paths | Week 9–10 · security review before any pilot sees it |
| 2.2 | Stripe integration + plan enforcement + overage credits | Week 10–11 |
| 2.3 | Playbooks v1 (3 workflows) + report rendering (tables/charts) | Week 11–13 |
| 2.4 | Fleet console MVP using cross-instance tools | Week 12–13 (agency differentiator) |
| 2.5 | Onboard 25 pilot orgs (from G0 waitlist), weekly feedback loop | Week 12–14 |

**Gate G2:** activation (first successful answer) >40% of signups · W4 retention >35% · gross margin >70% · zero unapproved-write incidents · NPS ≥40 from pilots.

## Phase 3 — GA readiness (Weeks 15–22)

Goal: public launch, sellable at scale.

Scope: EU data residency finalization + DPA/GDPR workflows · observability SLOs + on-call runbooks · scheduled reports (cron → email digest) · Slack share-out of reports · Spanish UI/locale pass (`ODOO_LOCALE` plumbing per tenant) · template gallery for playbooks · SOC2-ready practice checklist executed · pricing page + self-serve signup.

**Gate G3:** 99.5% SLO met for 30 consecutive days · 100 paying orgs or $10k MRR equivalent · monthly churn <5% · support load <5 tickets/org/month.

## Phase 4 — Scale & moat (Months 6+)

Backlog in priority order:

1. Model router: escalate complex turns to Sonnet (router abstraction already in runtime).
2. Custom playbook builder (org-authored workflows on top of the gated write path).
3. Partner/white-label tier for agencies (their brand, their clients, our engine).
4. Public API + webhooks (agent-as-API), n8n/Zapier connectors.
5. Conversation memory + embeddings layer beyond BM25 knowledge search.
6. Self-hosted enterprise option (leverage MIT odoo-mcp + Helm chart).

---

## Team & budget (steady state Phases 1–3)

- 1 Product (owner of these docs) · 2 Backend Python · 1 Frontend · 0.5 Design
- Variable costs: AWS Bedrock (Anthropic Haiku 4.5, EU geography) spend — metered per query with alerting at 80%/100% of margin guardrail; official prices in HLD §6 · Fly.io + managed PG/Redis (EU) · Stripe fees · domain/misc.

## Risks & mitigations

| Risk | Mitigation |
| --- | --- |
| Haiku mis-selects tools on messy Odoo schemas | Mode presets trim surface; eval suite as CI gate; schema_catalog hints in prompt |
| Prompt injection via record content | Untrusted-data framing, approval diffs shown raw, adversarial evals blocking release |
| Approval-token same-session breakage across worker restarts | LLD §5.3 re-validate-and-replay fallback; expiry ≤ idle TTL |
| Bedrock/Haiku price or capacity incident | Router abstraction; degraded queue mode (EU-only); EU cross-region profile spreads load across EU regions; caching keeps COGS low |
| Competitor copies UX but not safety | Lean into audit trail + HITL + fleet fan-out as the compliance story agencies require |
| mcp-odoo upstream drift | Pin exact version; contract tests fail loudly on bump; contribute fixes upstream (this repo) |

## Immediate next actions (this week)

1. ~~Review/approve this doc set~~ — ✅ **Approved 2026-08-23**; name approved: **odooPilot**.
2. Tech spike (Stage A, 0.1) — **scaffolded 2026-08-23**: `odoo-copilot-saas` @ `spike/haiku-loop`; harness (Bedrock EU + odoo-mcp stdio), golden-set v0 (30 q), runner with G0a scoring; ruff/mypy/pytest green. Remaining: boot `compose-odoo-demo.yaml`, configure AWS creds for eu-north-1, then run `check_bedrock` → `run_spike`.
3. After G0a passes: kick off interviews + landing page (Stage B).
