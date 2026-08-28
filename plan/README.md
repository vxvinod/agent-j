# AI Jewellery Sales & Workflow Automation SaaS — Implementation Plan

This folder is the complete build plan for the product described in
[Req-AI-Jewellery-Sales-Workflow-Automation-SaaS.md](../prompts/Req-AI-Jewellery-Sales-Workflow-Automation-SaaS.md).
It is written to be executed on a different machine with no further context from the planning session.

## Read in this order

| # | Document | What it settles |
|---|---|---|
| — | **This file** | Context, the four decisions everything rests on, how to start |
| 01 | [Architecture](01-architecture.md) | Container topology, Docker setup, job queue, tenant isolation, channel adapters |
| 02 | [Database schema](02-database-schema.md) | Full DDL, RLS policies, indexes |
| 03 | [Pricing engine](03-pricing-engine.md) | The formula, GST modes, unit conversion, the quotation snapshot |
| 04 | [AI layer](04-ai-layer.md) | Provider abstraction, the two-call pipeline, validation ladder, guardrails, evals |
| 05 | [Recommendation & follow-ups](05-recommendation-and-followups.md) | pgvector retrieval, the follow-up ladder, suppression rules |
| 06 | [UI design](06-ui-design.md) | White-and-gold token system, screen inventory |
| 07 | [Build order](07-build-order.md) | Phase-by-phase sequence, milestones, verification |
| 08 | [Capital, hosting & risk](08-capital-hosting-risk.md) | Costs, where to host, external APIs, risk register |

A published visual version of the plan, including the two architecture diagrams, is at
<https://claude.ai/code/artifact/867da382-d10c-4ece-9058-47bc391774e4>

> **On `plan.md` in this folder:** that is the single-file snapshot saved automatically when the plan was
> approved. The numbered documents above supersede it — they carry the full schema DDL, the compose files,
> the code interfaces and the eval design. Delete `plan.md` once you have skimmed it, or keep it as a
> summary; do not maintain both.

---

## Context

The spec describes a multi-tenant SaaS for Indian jewellery shops that turns customer enquiries into
leads, quotations, appointments and follow-ups.

The spec is written as if for a funded team building the full platform. **It is being built by one
person with AI coding assistance and near-zero capital.** That single fact changes almost every
decision. This plan is therefore not a restatement of the spec — it is the subset of the spec a solo
founder can actually ship, ordered so each phase is independently demoable, plus an explicit list of
what to cut.

**Intended outcome:** a working product in front of 3–5 real jewellery shops within about five months
for well under ₹1 lakh, with a validated answer to "will a shop owner pay for this" before any
serious money is spent.

---

## The four decisions everything rests on

### 1. Solo founder with AI coding assistance

Every architectural choice minimises the number of distinct systems one person must operate and debug.
Cleverness that costs operational surface area is a net negative here, regardless of how good it looks
on a diagram. This is why there is no Redis, no message broker, no Kubernetes and no generic workflow
engine anywhere in this plan.

### 2. Everything runs in Docker; development is entirely local

Nothing installed on the host except Docker. Zero cloud spend for the first four months. The identical
compose files run on the server later, so "the laptop and the server run the same thing" is literally
true rather than approximately true.

### 3. WhatsApp Business API is deferred out of the MVP

Meta business verification takes 1–3 weeks and can block a pilot outright. The MVP ingests enquiries
through a **web enquiry form** the shop links from their WhatsApp profile, plus a **paste-a-conversation
box** for staff. Outbound uses a **dispatch card** with a pre-filled `wa.me` link that the salesperson
taps to send.

Both intake routes are channel adapters feeding one normalised pipeline, so adding the official API
later is a new file, not a refactor. This decision is what makes the plan fundable out of pocket.

The pilot's honest proposition is **"automation drafts, a human dispatches"** — which is a better first
product anyway, because the shop keeps final say over every outgoing message while learning to trust it.

### 4. No single model provider

Call sites request a capability tier (`fast` / `smart`), never a model name. Provider and model are
environment variables with working adapters for Gemini, Claude and OpenAI.

Development and prompt tuning run on the **free Gemini tier at ₹0**. The pilot moves to a **paid** tier,
and the reason is not quota — see [08](08-capital-hosting-risk.md#the-free-tier-trains-on-what-you-send-it).

---

## The product principle, restated

This is not an AI chatbot for jewellery shops. It is an intelligent workflow system that converts
enquiries into sales while reducing repetitive work.

- **The AI reads.** It extracts intent and requirements from Tamil, English and Tanglish.
- **Deterministic code computes.** Every rupee, every weight, every tax.
- **The workflow engine chases.** Follow-ups on a schedule, with suppression rules.
- **The CRM is the system of record.**

The single most important invariant in the entire codebase:

> **The model fills structured slots. It never sees, emits, or influences a price.**

This is enforced structurally, not by prompt instruction — the response schema has no numeric fields,
and a post-generation guard rejects any amount that does not trace back to a computed fact.

---

## Two corrections to the spec

Following the spec literally would produce a wrong product in two places.

### GST is not one percentage on the subtotal

Spec §8 models GST as `subtotal × configured_percentage`. Indian gold jewellery does not work that way:
**3% on metal value, 5% on making charges**, and which applies depends on how the shop bills
(composite vs itemised). Both are legitimate and shops differ. The engine needs a per-business
`gst_mode` and must compute tax **per component**. Details in [03](03-pricing-engine.md).

### Unit conversion must never touch the model

*Pavun*, *sovereign*, *gram*, *tola* carry the largest error magnitude in the system. One sovereign is
**8 grams** — a mis-parse there is an 8× price error, not a rounding one. The model returns
`{value: 5, unit: "pavun"}`; a tested constant table in Python converts it.

---

## Getting started on the other machine

```bash
git init
mkdir -p backend/app frontend docs/adr
# then work through 07-build-order.md, Phase 1
```

Phase 1's milestone is the thing to aim at first: `make up` on a clean machine brings the stack live;
you add a product, set today's gold rate, and produce a correct itemised quote PDF — **then change the
gold rate and confirm yesterday's quote is unchanged.** No AI involved in any of that.

Write the ADRs in `docs/adr/` as you go. The decisions worth recording are already identified at the
end of [01](01-architecture.md).
