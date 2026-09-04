# Ideation — odooPilot 🧭

> **Start point & memory** for the SaaS built on top of `odoo-mcp` with Claude Haiku 4.5 as the driving model.
> This file is the single entry point: read this first in any new session, then follow the links.
> Status: **DRAFT v0.4 · 2026-08-23** — owner: Product (this repo's product track).
> Doc set **approved** by product owner 2026-08-23. Name locked: **odooPilot**.

---

## 1. The idea in one paragraph

A hosted, multi-tenant B2B SaaS — **"chat with your Odoo"** — where customers connect their existing Odoo 16–19 instance in minutes (zero server-side install, credentials they already have), and business users get a conversational workspace powered by **Claude Haiku 4.5** driving the [odoo-mcp](../../README.md) tool surface: natural-language queries, self-service finance reports, business playbooks, and ERP writes that are always human-approved through the MCP's gated write workflow. Two wedges competitors can't cheaply copy: **safety-first AI writes** (approval tokens + audit trail, already built into the MCP) and an **agency fleet console** (cross-instance fan-out across many client DBs). Monetization: subscription tiers + usage credits; Haiku 4.5 keeps marginal cost per query at cents.

## 2. Document map

| Doc | What it contains | Status |
| --- | --- | --- |
| [hld.md](./hld.md) | Product definition, system context diagram, components, key flows (read/write/onboarding/fleet), tenancy & security posture, cost model, assumptions | DRAFT v0.3 |
| [lld.md](./lld.md) | Repo layout, service decomposition, DB schema, API surface, MCP session manager design, agent turn-loop state machine, testing strategy, deployment | DRAFT v0.3 |
| [roadmap.md](./roadmap.md) | Phase 0–4 step-by-step execution with gates G0–G3, team/budget, risks, immediate next actions | DRAFT v0.2 |
| [interviews.md](./interviews.md) | Stage B interview script: hypotheses H1–H6 (incl. MCP-only D11 and Holded-style intake), screener, 30-min guide, synthesis template | v1 2026-09-01 |
| [competitive.md](./competitive.md) | Competitive notes: Cledoo, mn_mcp_server (in-instance MCP modules vs MCP Connect), Holded (AI document intake idea) | v1 2026-09-02 |

Related upstream docs (this repo): [docs/architecture.md](../architecture.md) · [docs/field-acl.md](../field-acl.md) · [docs/partner-playbook.md](../partner-playbook.md).

## 3. Grounding facts (from the codebase, verified)

- odoo-mcp exposes **41 tools**: 11 read/discover, 5 write/operate (`preview_write → validate_write → execute_approved_write`, `execute_method`, `chatter_post`), 3 diagnose, 3 migrate, 3 audit/plan, 3 knowledge search (local BM25), 2 accounting pack, 4 background tasks, 3 cross-instance fan-out, 2 utility.
- Writes are gated by default: same-session approval token + live `fields_get` validation + `confirm=true` + `ODOO_MCP_ENABLE_WRITES=1`. Direct create/write/unlink are blocked.
- One server process supports **multiple named instances**; instance-bound tokens; per-instance schema caches; cross-instance fan-out is read-only and partial-failure-tolerant.
- Field-level ACL policy file enforced on every read path; rate limiting (`warn|block`); JSONL audit log; HITL elicitation mode for writes.
- Transports: stdio + Streamable HTTP (local-bind hardened); Odoo XML-RPC 16–18, JSON-2 for 19+ (XML-RPC dies in Odoo 22 / fall 2028).
- Published to PyPI as `odoo-mcp`; MIT. SaaS consumes a **pinned version** and upstreams fixes here.

## 4. Key decisions made (decision log)

| # | Decision | Rationale | Where |
| --- | --- | --- | --- |
| D1 | Model = Claude Haiku 4.5 primary; router abstraction kept for later Sonnet escalation | Cost/latency economics; strong tool-calling; caching | HLD §3 |
| D2 | One odoo-mcp stdio subprocess per org (multi-instance config file rendered from DB) | Tenant isolation for free; reuses hardened defaults; avoids exposing our own HTTP auth surface | LLD §5.1 |
| D3 | Writes disabled by default per org; exact side-effect allowlist; UI approval maps to HITL gate; no broad mode ever | Compliance story is the moat | HLD §5 |
| D4 | SaaS lives in a sibling repo consuming pinned PyPI pkg; mcp-odoo stays pure OSS upstream | Clean layering per CLAUDE.md contract | LLD §1 |
| D5 | EU region first (ES home market), Spanish UI second | BitClick base + local advantage | HLD §8, roadmap P3 |
| D6 | **All LLM inference via AWS Bedrock, EU geography only** (EU in-region / EU cross-region inference profile); zero non-EU data paths anywhere in the stack — mandatory customer requirement | Data residency is non-negotiable; Bedrock geo-profiles keep requests inside the EU while giving multi-AZ resilience | HLD §3/§5, LLD §5.0 |
| D7 | Official Haiku 4.5 pricing locked into cost model ($1.10/$5.50 per MTok in/out; cache read $0.11; batch at 50%); availability SLO relaxed **99.9% → 99.5%** | Budget certainty from real prices; 99.9% was unrealistic for this team size/stage | HLD §6/§7, roadmap G3 |
| D8 | Bedrock profile pinned: **`eu.anthropic.claude-haiku-4-5-20251001-v1:0`**, `AWS_DEFAULT_REGION=eu-north-1` (Stockholm); AWS-side infra co-located there | Exact profile confirmed; one home region keeps latency and the residency audit simple | HLD §3, LLD §5.0/§8 |
| D9 | Name approved: **odooPilot** (2026-08-23) | Owner decision; run a trademark search before paid marketing/GA | All docs |
| D10 | Phase 0 reordered: tech spike (sub-gate G0a) runs **before** interviews and landing page | Don't spend GTM effort before the core Haiku×MCP loop is proven; spike failure would invalidate positioning anyway | Roadmap Phase 0 Stage A/B |
| D11 | Second product line: **MCP-only tier** (2026-09-01) — customer subscribes without the Bedrock chat; wizard takes Odoo URL/db + user + API key and returns connection instructions for the customer's **own MCP client** (Claude Desktop, Cursor, …); we bill the subscription and serve basic usage metrics (tool calls, instances, health) | Monetizes the hosted MCP data plane with zero LLM COGS (customer brings their own tokens); widens the funnel below the Copilot tiers. **Amends D2's scope**: MCP-only requires exposing per-org workers over Streamable HTTP with per-customer auth + gateway/metering — HLD/LLD ripple pending | This table; AGENTS.md (odoo-copilot-saas) |

## 5. Open questions

- OIDC provider choice (Clerk recommended default) — LLD §9.
- Exact credit/overage pricing after Phase 2 telemetry.
- ~~Phase 0 spike still verifies tool-use + prompt-caching parity/latency on the pinned Bedrock profile~~ **Resolved 2026-09-01**: tool use works; prompt caching **works and is verified** on the EU profile (probe: cache_write 16,802 → cache_read 16,802 on the second call, 1h TTL). The earlier zeros were the **Haiku 4.5 minimum of 4,096 tokens per cache checkpoint** — below it Bedrock silently skips caching. The spike's trimmed 12-tool prefix (~3.1K tokens) sits under the minimum; production presets (~18–20 tools + fuller system prompt) should clear it. Two design notes for the runtime: use `ttl: "1h"` (done in spike agent, matches D7) and remember the **cache is per-region** — the EU cross-region profile balances across 6 EU regions, so real cache-hit rates will be below 100% of turns; HLD §6's 80% cache-read assumption stays plausible but must be re-measured with production-size prompts in Phase 1.
- Bedrock on-demand quota in eu-north-1 throttles bursts (429s): spike now paces requests + retries with backoff. Request a quota raise before any multi-user phase.
- Trademark search for "odooPilot" before paid marketing / GA (name itself approved, D9).
- D11 ripple: revise HLD (product lines, pricing tiers) and LLD (HTTP gateway, per-customer auth, metering for MCP-only) at the next doc revision.

## 6. Status ledger (update me every session)

| Date | Milestone | State |
| --- | --- | --- |
| 2026-08-23 | ideation/HLD/LLD/roadmap authored (v0.1) | ✅ done |
| 2026-08-23 | v0.2 revisions: AWS Bedrock EU-only orchestration (mandatory), official Haiku pricing locked, SLO 99.5% | ✅ done |
| 2026-08-23 | v0.3: Bedrock profile pinned `eu.anthropic.claude-haiku-4-5-20251001-v1:0` @ eu-north-1; infra co-location aligned | ✅ done |
| 2026-08-23 | **Doc set approved** (G0 entry) · name locked: odooPilot · Phase 0 reordered (spike first, D10) | ✅ done |
| 2026-09-01 | D11 recorded: MCP-only subscription tier (BYO MCP client, basic metrics); HLD/LLD ripple pending | ✅ done |
| 2026-09-01 | Phase 0 Stage A: Haiku×MCP spike + golden set (G0a gate) | ✅ **G0a PASS** — accuracy 100% (30/30), $0.023/query, EU-only verified, Odoo **19** demo (JSON-2 + API key), latency p50 5.9s / p95 37.4s. Report: `odoo-copilot-saas/reports/spike_20260901_152352.*` |
| 2026-09-01 | Phase 0 Stage B assets: interview script ([interviews.md](./interviews.md), H1–H6) + landing ES/EN with demo tabs (`odoo-copilot-saas/landing/`) | 🔶 in progress — interviews under way (owner) |
| 2026-09-04 | Waitlist backend live: n8n workflow "odooPilot - Waitlist to CRM" (`b1nEBvXwmKXMSsLx` @ agentic.bitclick.solutions) → dedupe + `crm.lead` in BitClick Odoo; landing form wired and E2E-tested (test leads #71/#72, deletable) | ✅ done — pending: domain/hosting for the landing, trademark search |

**Next session should start at:** Phase 0 Stage B (interviews + landing page + waitlist, roadmap 0.3/0.4) · in parallel: resolve the prompt-caching-on-Bedrock caveat and refresh HLD §6 cost model with real spike telemetry ($0.023/query uncached).
