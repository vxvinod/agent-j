# 07 — Build order

The spec's seven phases, resequenced for one person.

## The two structural changes

**1. The pricing engine moves from Phase 4 to Phase 1.** It is pure functions with provably correct
answers, so it is fully testable before any AI or UI exists; it is the highest-severity failure in the
system; and a correct itemised quote PDF is the most credible thing you can put in front of a shop owner.
The AI layer's `calculate_quote` tool needs it to exist anyway — building AI first means building against
a stub and rewriting the integration. **Building the AI first and the money math last inverts the risk.**

**2. Channels and CRM ship before AI**, so you have a usable, sellable product with **zero AI** by week 9.
That de-risks the pilot, and the paste box doubles as your training-data collection instrument.

Week counts assume solo work with AI assistance and are deliberately not aggressive. An efficient run
reaches the pilot around week 16; plan for 22.

---

## Phase 0 — Validation · weeks 1–4, in parallel, no code

Interview 10 jewellery shops. What you need out of it:

- how enquiries actually arrive
- who currently follows up, and on what
- how a quote gets made today
- what happens to an enquiry that goes cold
- what they would pay to fix it

**Run this alongside Phase 1, not before it.** Do not gate four weeks of undifferentiated foundation work —
schema, auth, tenancy — behind ten interviews you can conduct in the evenings. Nothing in Phase 1 depends
on what the interviews say.

**Do gate Phase 3 on it.** The AI layer is the first place the findings change what you build.

**Milestone:** one sentence per shop — the pain they named and what they said they would pay. If the
answers do not converge, stop before Phase 3. That finding is worth the entire budget.

---

## Phase 1 — Foundation and pricing engine · weeks 3–6

Build:

- Repo, full Docker Compose stack, `Makefile`, CI (ruff, mypy, pytest, vitest)
- Alembic, the two Postgres roles, RLS policies + **the forced-RLS test**
- Organisations, users, JWT + Argon2 auth, permission matrix
- **Design tokens and app shell** — the white-and-gold palette, type scale, both themes, defined once
  before there are twenty components to retrofit. Half a day now, a week later.
- Product categories, products, images → MinIO, CSV import
- Gold rate entry, pricing rules + precedence resolver
- **Pricing engine** (`Decimal`, versioned), quotations + items + snapshot
- PDF via WeasyPrint

**Milestone:** `make up` on a clean machine brings the whole stack live. Upload a 50-product CSV, set
today's 22K rate, build a two-line quote, download the PDF. **Then change the gold rate and reopen
yesterday's quote — the total must be unchanged.** Zero AI involved.

---

## Phase 2 — Enquiry intake and CRM · weeks 7–9

Build:

- `inbound_events`, the adapter protocol, `NormalizedInbound`
- Public web enquiry form (Turnstile + rate limit)
- Paste-a-conversation box
- Customers, leads, conversations, messages
- Pipeline board, lead detail, assignment
- **Deterministic rule-table lead scoring** with stored, displayed reasons
- Manual reply drafting + the `wa.me` dispatch card

**Milestone:** paste a real WhatsApp thread → the customer and a scored lead appear on the board → draft a
reply → dispatch it through the deep link. **This is a working, sellable product with no AI in it**, which
is the real de-risking move of the whole plan.

The paste box is also your training-data instrument. Every real conversation pasted here during weeks 7–9
becomes an eval case in Phase 3 — which is exactly why it ships before the AI rather than after.

---

## Phase 3 — AI extraction layer · weeks 10–13

**Gated on Phase 0 findings.**

Build:

- Provider abstraction with **two working implementations plus the fake** — one implementation is a
  wrapper, not an abstraction
- Triage schema and system prompt, with the Tamil/Tanglish glossary and few-shot block
- The validation ladder: retry → repair → escalate model → regex fallback → human
- Confidence thresholds and the `needs_review` path
- Deterministic scorer wired to the model's signals; reasons UI
- Response generator + the numeric guard
- `ai_interactions` and the cost view
- **The eval harness, built before prompt tuning**

All of this runs on the free tier at ₹0, using synthetic and anonymised conversations.

**Milestone:** `"Bro 5 pavun necklace venum 3 lakh kulla"` → lead created, slots extracted correctly,
score 60/Warm with the reason list visible. The eval suite reports per-slot accuracy, and the injection and
fake-pricing cases score a hard zero.

---

## Phase 4 — Recommendation and quote drafting · weeks 14–16

Build:

- Local embedding provider + `embed_product` jobs
- Hybrid retriever (filter CTE → pgvector → exact pricing → blend)
- Deterministic "why it matches"
- Quote drafted automatically from extracted requirements
- **The human approval gate**
- PDF and a shareable quote link

**Milestone:** enquiry → three matched products with match reasons and engine-computed prices → draft quote
→ you approve → PDF. This is the complete core loop and your first real sales demo.

---

## Phase 5 — Follow-up automation and handoff · weeks 17–19

Build:

- `followup_rules`, ladder scheduling, `tick_followups` + `send_followup`
- All nine suppression rules, quiet hours, caps
- Appointments
- Human takeover / return-to-AI, org-level automation kill switch
- Audit trail and the "not sent, and why" report

**Milestone:** a quote sent yesterday surfaces a follow-up draft today. Taking over the conversation flips
the remaining two steps to `cancelled` / `HUMAN_TAKEOVER`, visible in the audit log. Running the worker
twice sends exactly once.

---

## Phase 6 — Deploy and pilot · weeks 20–22

Build:

- VPS, Caddy, production compose overlay
- Nightly `pg_dump` to R2 **plus one tested restore**
- Sentry, `/health/worker`, uptime monitor → push notification to your phone
- Dashboard: the 8 metrics as SQL views, today's priorities, AI cost card, **automation health strip**
- Self-serve onboarding wizard
- Rate limits, audit log viewer

**Milestone:** **a brand-new shop is onboarded end to end by someone who is not you, in under 30 minutes,
using only the UI** — and then uses it for a full week without you touching anything.

That is the spec's actual success criterion: *a real user can use the system without developer assistance.*

---

## Phase 7 onward — only if the pilot works

- WhatsApp Cloud API: Meta verification, **per-shop numbers via Embedded Signup**, webhook adapter,
  templates, delivery status
- Analytics: conversion, pipeline value, salesperson and follow-up performance
- Subscription billing via Razorpay

**Milestone:** the real webhook replaces the paste box with no pipeline changes — proving the adapter
abstraction was real.

Start the Meta business-verification paperwork during Phase 6, not when you need it. It takes 1–3 weeks.

---

## Cut from the MVP entirely

| Cut | Why |
|---|---|
| Inventory, orders, manufacturing workflow | `WON` plus a note is enough |
| After-sales, exchange and repair flows | Classify the intent, then always hand off to a human. Never automate. |
| AI business assistant | The one genuinely agentic use case — defer past the pilot |
| Salesperson performance analytics | One SQL view; only if there is slack in week 22 |
| Inbound media analysis | Store the images; the salesperson looks at them. No vision calls. |
| Delivery status tracking | Meaningless without the WhatsApp API |
| LLM-generated "AI insights" cards | Three SQL aggregates through a sentence template. Making them model calls is pure cost for zero insight. |
| Multi-currency / non-India | Hardcode INR, IST, Indian GST |
| **Multi-language UI** | The AI must understand Tamil and Tanglish **input** from day one. The dashboard stays English-only — staff who use software are comfortable in English, and localising the interface is weeks of work that validates nothing. |

---

## Verification

None of these require a deployed environment until Phase 6.

**Pricing (Phase 1).** `make test` over the pricing module: each making-charge type, both GST modes, rule
precedence across all four levels, unit conversion including `UnknownUnitError`. Plus the golden-file test
and the property test that `sum(line_total) == total_amount` **exactly**. And the reproducibility test —
create a quote, change the rate, assert the total is unchanged.

**CRM (Phase 2).** Submit the public form from a phone browser against your laptop over a Cloudflare
Tunnel. Confirm the customer and lead appear, the score matches a hand-computed value from the rule table,
and **submitting the same enquiry twice does not create two leads.**

**AI (Phase 3).** `make eval` outputs per-field accuracy. Do not eyeball it. Injection cases pass only if
the model produces no price and no discount regardless of what it says.

**Core loop (Phase 4).** End to end by hand: paste a Tanglish enquiry, verify the returned products are
actually within the stated budget and weight (compute it yourself), approve, open the PDF, check every line
against a manual calculation.

**Automation (Phase 5).** Insert a quotation dated yesterday, run the worker, confirm exactly one follow-up.
Then verify each suppression rule individually. Run the worker twice against the same due work and confirm
no double-send.

**Pilot readiness (Phase 6).** Restore the nightly backup into a fresh container and query it. Then
**authenticate as shop A and attempt to read shop B's leads, products and quotations directly via the API** —
an explicit test, not an assumption.

**Provider swap (any time).** Run the full eval suite against both providers and compare per-slot accuracy
before switching production traffic. Assert the production config refuses to boot on the free tier.

**Ongoing.** Track LLM cost per shop per month from Phase 3. If it exceeds ₹500 at pilot volume, the prompt
is too large or caching is not working — verify cached-token counts are non-zero before changing anything
else.
