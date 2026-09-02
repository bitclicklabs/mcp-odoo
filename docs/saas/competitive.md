# Competitive notes — odooPilot

> Living document. Companion docs: [ideation.md](./ideation.md) · [HLD](./hld.md) · [roadmap](./roadmap.md).
> v1 2026-09-02 — Cledoo, mn_mcp_server, Holded.

## 1. In-instance MCP modules (compete with MCP Connect / D11)

Both are **modules installed inside the customer's Odoo** — the customer brings
their own AI client (Claude Desktop, ChatGPT, Cursor) and their own LLM tokens.
Neither is a chat product, neither is multi-instance, neither manages fleets.

### Cledoo (cledoo.com)

- Native Odoo module, "no external server". 17 MCP tools, native Odoo
  permissions as the security model.
- Pricing: Free tier (OAuth 2.1, API keys, basic tools) · **Pro €189
  one-time per Odoo version** — governance pack: read-only kill-switch,
  per-user policies, privacy masking, human approval for risky writes,
  outbound-communication blocks, audit + alerts.

### mn_mcp_server (Odoo Apps, Moaz Nabil, LGPL-3, free)

Source reviewed 2026-09-02 (~3.9K lines, well built). Odoo 17–19, `POST /mcp`
endpoint, OAuth 2.1 + PKCE, scoped API keys, confirmation gates, audit with
SHA-256 arg hashing, playground + live monitor. 27 tools including direct
`odoo_create/write/unlink`, `odoo_send_email`, `odoo_run_server_action`
(simple confirm gate — more permissive than our preview→validate→approve).

**Best idea in the package — the semantic layer** (adopt in Phase 2):

- *Glossary*: business terms defined once ("ARR = sum of `recurring_total` on
  active subscriptions").
- *Metrics*: named, reviewed calculations — the AI asks for `arr` and gets the
  CFO-approved number, not an improvised read_group.
- *Model hints*: "for revenue use `account.move`, not `sale.order`" — stops
  the wrong query before it is made.

This directly attacks our top roadmap risk ("Haiku mis-selects tools on messy
Odoo schemas") and is exactly what an agency would configure per client.

### The GDPR hole in their story — our wedge

"Data processed inside your Odoo" is true **only for the MCP server**. Every
chat turn still ships ERP data to the user's LLM provider — Claude Desktop and
ChatGPT default to US infrastructure. A module cannot control that.
odooPilot's Copilot line is **EU end-to-end** (Bedrock EU geography, verified
in the G0a spike): for data-protection-strict customers, "native module" ≠
"GDPR-compliant AI". Structural advantage — copying it means becoming a SaaS.

### Positioning against them

| Front | In-instance modules | odooPilot |
| --- | --- | --- |
| Install | Module + admin rights + re-buy/reinstall per Odoo version | Zero install; version migrations don't touch you |
| Buyer | Technical user with own Claude/ChatGPT subscription | Business user who wants answers |
| Compliance | Data exits to the user's LLM (typically US) | EU end-to-end, verified |
| Fleet | One DB per install | Cross-instance fan-out for agencies |
| Writes | Simple confirm | preview→validate→approve token + audit |

Implications:

- **MCP Connect cannot be sold as bare "MCP access"** (reserve price: free /
  €189 one-time). Sell it as *hosted* (no module, for those who can't or won't
  install), multi-instance, metered, and EU-guaranteed when paired with Copilot.
- Steal Cledoo Pro's marketing vocabulary — *kill-switch*, *masking*, *human
  approval* — we already have the features upstream (field ACL, write gate,
  audit JSONL); name them this way on the landing.
- Their playground/live monitor is a good pattern for the MCP Connect wizard.
- Interviews (H4): ask *"would you install a module in your production Odoo
  for this?"* — install friction is our wedge or our threat; measure it.
- Risk: these modules commoditize the MCP layer. Our moat shifts to what a
  module cannot be: hosted convenience, fleet, EU inference, business-user chat.

## 2. Holded (compete with the *outcome*, not the tool)

Holded is the ES-market SMB management suite growing at Odoo's expense. Its
killer convenience: **a dedicated inbox e-mail per company — forward purchase
invoices, OCR extracts everything, the document lands booked and linked to the
right supplier**. Zero data entry.

Our answer (candidate feature, validate in interviews): **AI document intake
for Odoo** —

1. Each org (or each client of an agency) gets an intake address
   (`facturas@<org>.odoopilot.eu`).
2. Inbound mail → attachment parsed by the LLM (invoice fields, supplier
   matching against `res.partner`, duplicates check).
3. Proposed `account.move` goes through the **same gated write flow** — an
   approval card ("supplier bill F/2026/0455, Deco Addict, 1.210,00 €
   — approve?"), then posted to Odoo with full audit.
4. EU end-to-end, works on **Community** (Odoo's own OCR digitization is an
   Enterprise/IAP feature — Community users have nothing native).

Why it fits us: it reuses the moat (gated writes + audit + EU) on the highest
volume pain an SMB has, and it gives agencies a per-client service to resell.
Positioning line: "lo que te gusta de Holded, sin dejar Odoo".

Status: **idea — not committed**. Validate demand in Stage B interviews before
design (H6 below), then slot as a Phase 2/3 playbook if signal is strong.
