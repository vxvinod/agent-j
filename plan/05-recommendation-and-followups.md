# 05 — Recommendation and follow-up engine

Two subsystems that both sit in the worker, both deterministic, and neither of which is an LLM call.

---

# Part 1 — Product recommendation

Hybrid: structured filter first, semantic search second, exact pricing third.

## What gets embedded

Not the raw description — a **synthesised product document**, built deterministically so the style and
occasion vocabulary is guaranteed present even when a shop leaves `description` blank.

```python
# app/recommend/document.py
def build_document(p: Product, sovereigns: Decimal) -> str:
    return (
        f"{p.name}. {p.category_name} / {p.subcategory or ''}. {p.purity_label} gold. "
        f"{p.net_gold_weight_g}g ({sovereigns} sovereigns). "
        f"Style: {', '.join(p.style_tags)}. Occasion: {', '.join(p.occasion_tags)}. "
        f"For: {p.gender or 'any'}. Stones: {stone_summary(p)}. {p.description or ''}"
    )
```

Stored in `product_embeddings.embedding_text` with `content_hash = sha256(doc)`.

## When embeddings are generated

The product service enqueues an `embed_product` job **in the same transaction as the product write**. The
handler rebuilds the document, compares `content_hash`, and **returns early if unchanged**.

- CSV import enqueues in batches of 50.
- An admin `reembed_all` job exists for provider or dimension changes.

## Embedding provider

**Local, in the worker: `fastembed` with `BAAI/bge-small-en-v1.5`, 384 dimensions.** About 40ms on CPU,
zero cost, no API dependency — which is what satisfies the zero-cloud-spend constraint.

English-only is genuinely sufficient here because **both sides of the comparison are English**: the query
vector is built from the extractor's *normalised* slots (`style="modern"`, `occasion="wedding"`), not from
raw Tanglish.

`EmbeddingProvider` has a second implementation (`VoyageProvider`, `voyage-3.5-lite`) for later. Note the
cost of switching: a dimension change means an `ALTER COLUMN` plus a full re-embed job.

## Index

```sql
CREATE INDEX idx_pe_embedding ON product_embeddings
  USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
```

HNSW over IVFFlat because IVFFlat needs a representative training set and retraining after bulk loads —
and a pilot shop starts at zero products and grows to a few hundred, which is exactly the range where
IVFFlat's lists would be badly tuned the entire time. At these volumes HNSW's memory cost is irrelevant.

## Combining filter and vector search

The classic trap is filtered ANN: a selective filter (org + category + purity + weight) makes HNSW return
far fewer than *k* rows after post-filtering.

**At pilot scale, sidestep it.** Filter first into a CTE and let the planner do an exact scan over the small
candidate set — exact distance, no recall loss, and faster than fighting `ef_search`. The HNSW index stays
for when a tenant crosses ~5k products, and the same SQL transitions to using it.

```sql
WITH candidates AS (
  SELECT p.id, p.name, p.net_gold_weight_g, p.purity_label, p.availability,
         p.style_tags, p.occasion_tags, p.category_id, pe.embedding
  FROM products p
  JOIN product_embeddings pe ON pe.product_id = p.id
  WHERE p.org_id = current_setting('app.org_id')::uuid
    AND p.is_active
    AND (:category_id::uuid IS NULL OR p.category_id = :category_id)
    AND (:purity::text      IS NULL OR p.purity_label = :purity)
    AND (:min_w::numeric    IS NULL OR p.net_gold_weight_g >= :min_w)
    AND (:max_w::numeric    IS NULL OR p.net_gold_weight_g <= :max_w)
    AND (NOT :in_stock_only OR p.availability <> 'out_of_stock')
)
SELECT id, name, net_gold_weight_g, purity_label, availability, style_tags,
       1 - (embedding <=> :qvec) AS semantic_score
FROM candidates
ORDER BY embedding <=> :qvec
LIMIT 12;
```

### Budget filtering is a weight conversion, not a price computation

**Do not reimplement pricing in SQL.** Python resolves the pricing rule and current gold rate once, then
inverts the formula to a maximum affordable net weight (see
[03](03-pricing-engine.md#budget--weight-inversion)) and passes that as a plain weight bound.

There remains exactly one implementation of the price formula in the codebase.

## Ranking

Over the ~12 candidates, compute **exact** prices with the real pricing engine, then blend:

```python
final = 0.55 * semantic + 0.25 * budget_fit + 0.20 * tag_overlap
budget_fit = clip(1 - abs(price - target) / target, 0, 1)
```

Return the top 5.

## "Why it matches" is generated deterministically

Never by an LLM. Built from the score components:

```python
["22K as requested",
 "5.5 pavun (you asked for 5)",
 "₹2.94L — within your ₹3L budget",
 "modern style"]
```

Cheaper, faster, and structurally unable to hallucinate.

## Cold start

If a product has no embedding yet — shop just onboarded, jobs still running — the retriever falls back to
tag overlap plus weight/budget ordering. **It never returns an empty list because a background job has not
finished.**

---

# Part 2 — Follow-up engine

Two tables and two job kinds. **Not a generic workflow engine** — that is the classic solo-founder trap,
and `jobs` + `followups` + `audit_logs` already cover every case in the spec.

## Scheduling — the whole ladder up front

When a quotation reaches `sent`, the service calls `schedule_followup_ladder(lead)` **inside the same
transaction**, inserting all N rows at once:

```
followups: (lead, step 0, scheduled_for = now + 24h,  status='scheduled')
           (lead, step 1, scheduled_for = now + 72h,  status='scheduled')
           (lead, step 2, scheduled_for = now + 168h, status='scheduled')
```

Delays come from `followup_rules.steps`:
`[{"delay_hours":24},{"delay_hours":48},{"delay_hours":96}]` — **cumulative**, so the ladder is
1 → 2 → 4 days as the spec draws it.

All-up-front rather than chain-as-you-go, for two reasons: a crash cannot lose the tail of the ladder, and
the plan is visible in the UI (*"next follow-up: 3 Sep"*), which shop owners find reassuring and which makes
cancellation a single `UPDATE`.

`scheduled_for` is shifted into the org's quiet-hours window (09:00–21:00 IST) at insert time, and
re-checked at send time.

## Pickup — idempotent in three independent layers

A `tick_followups` job self-reschedules every 60 seconds, protected from fan-out by
`jobs.dedupe_key = 'tick_followups'` on the partial unique index.

```sql
UPDATE followups SET status='claimed', updated_at=now()
WHERE id IN (
  SELECT id FROM followups
  WHERE status='scheduled' AND scheduled_for <= now()
  ORDER BY scheduled_for
  FOR UPDATE SKIP LOCKED
  LIMIT 50
) RETURNING id, org_id;
```

It then enqueues one `send_followup` job per row with `dedupe_key = 'followup:' || id`.

**The three layers:**

1. The claim `UPDATE` means a row transitions out of `scheduled` exactly once.
2. The `jobs.dedupe_key` unique index makes a duplicate enqueue a no-op.
3. The send handler re-asserts `status='claimed'` and writes `status='sent'` + `sent_message_id` in the
   **same transaction** as the `messages` insert.

A crash anywhere leaves the row reclaimable by the reaper and never double-sends.

## Suppression — evaluated at *send* time, not schedule time

State changes over the three days between scheduling and sending, so `evaluate_suppression(followup)` runs
in the send handler, in this order:

| # | Condition | Outcome |
|---|---|---|
| 1 | `organizations.automation_enabled = false` | skip · `ORG_AUTOMATION_DISABLED` |
| 2 | `customers.opted_out` | skip · `CUSTOMER_OPTED_OUT` |
| 3 | `conversations.state = 'closed'` | skip · `CONVERSATION_CLOSED` |
| 4 | `leads.status IN ('LOST','WON')` | skip · `LEAD_CLOSED` |
| 5 | `conversations.automation_mode <> 'ai'` | skip · `HUMAN_TAKEOVER` |
| 6 | an inbound message exists with `created_at > followup.created_at` | skip · `CUSTOMER_RESPONDED` **and cancel remaining steps** |
| 7 | now outside quiet hours | **reschedule** to next window open |
| 8 | automated outbound to this customer in last 24h ≥ `max_per_customer_per_day` | **reschedule** +24h |
| 9 | sent count for lead ≥ `max_per_lead` | skip · `MAX_REACHED` |

**Rule 6 is the spec's "No response?" gate made concrete, and it is the anti-spam mechanism.**

Note the distinction between *skip* (terminal) and *reschedule* (try later). Rules 7 and 8 are reschedules —
getting these wrong means either spamming at 2am or silently dropping follow-ups.

### Also enforced eagerly, so the UI is truthful

The inbound pipeline and the takeover endpoint both run:

```sql
UPDATE followups SET status='cancelled', skip_reason=:reason
WHERE lead_id=:id AND status='scheduled';
```

Belt and braces — **eager for UX, lazy for correctness.** The lazy evaluation is what makes it right; the
eager one is what makes the screen match reality the instant a salesperson takes over.

### Audit

Every skip writes `followups.skip_reason` plus an `audit_logs` row. That turns the spec's "complete audit
trail" requirement into a queryable report:

> *every follow-up we did not send this week, and why*

which is genuinely useful to show a shop owner who asks whether the system is spamming their customers.

## Sending, when there is no WhatsApp API

The gap the spec does not address.

`send_followup` generates the draft (tier `fast`, `{message, confidence}` schema, then the numeric guard)
and calls `adapters[channel].dispatch()`. For both MVP channels that is **`ManualDispatchAdapter`**:

- writes the message with `status='draft'`
- surfaces it in the UI as a *Ready to send* card with a copy button and a
  `https://wa.me/{phone}?text={urlencoded}` deep link
- the salesperson taps it, WhatsApp opens pre-filled, they send
- the card marks itself sent, writing `sent_message_id` and `status='sent'`

When the WhatsApp adapter lands, `dispatch()` becomes an API call and the card disappears. Nothing else
changes.

## Human handoff

Mandatory per spec §14, and the mechanism is simple:

- `conversations.automation_mode` moves `ai → human`, recording `taken_over_by_user_id` and
  `taken_over_at`
- all `scheduled` follow-ups for the lead flip to `cancelled` / `HUMAN_TAKEOVER`
- the inbound pipeline still runs **triage** (to keep the CRM enriched) but skips response generation
- returning to AI sets `returned_to_ai_at` and re-arms the ladder from the current step

**The AI must never pretend to be human.** Outbound AI-generated messages are labelled as such in the UI,
and the shop decides how to present that to their customer.

## Tests

`tests/unit/test_suppression.py` — each of the nine rules verified individually by setting the condition
and asserting the outcome, including the skip/reschedule distinction.

`tests/integration/test_queue.py`:

```python
def test_worker_does_not_double_send(db, worker):
    insert_quotation(sent_at=yesterday())
    worker.run_tick(); worker.run_tick()          # run twice
    assert count_followups(status="sent") == 1
```

Fast-forward test: insert a quotation dated yesterday, run the worker, confirm **exactly one** follow-up is
generated.
