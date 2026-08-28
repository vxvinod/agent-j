# 02 — Database schema

Postgres 17 with `pgvector`. Every business entity carries `org_id` and is protected by forced RLS
(see [01](01-architecture.md#tenant-isolation-postgres-rls)).

Conventions used throughout: money is `numeric(14,2)`, weights are `numeric(10,3)`, percentages are
`numeric(5,2)`, primary keys are `uuid` except high-volume append-only tables which use `bigserial`.

---

## Tenancy and identity

```sql
CREATE TABLE organizations (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name               text NOT NULL,
  slug               text UNIQUE NOT NULL,
  country            text NOT NULL DEFAULT 'IN',
  currency           text NOT NULL DEFAULT 'INR',
  timezone           text NOT NULL DEFAULT 'Asia/Kolkata',
  automation_enabled boolean NOT NULL DEFAULT true,
  gst_mode           text NOT NULL DEFAULT 'composite'
                     CHECK (gst_mode IN ('composite','itemised')),
  plan               text NOT NULL DEFAULT 'pilot',
  created_at         timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE users (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  email         citext NOT NULL,
  password_hash text NOT NULL,
  full_name     text NOT NULL,
  phone_e164    text,
  role          text NOT NULL CHECK (role IN ('owner','manager','salesperson')),
  is_active     boolean NOT NULL DEFAULT true,
  last_login_at timestamptz,
  created_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE (org_id, email)
);

CREATE TABLE refresh_tokens (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash text NOT NULL,
  expires_at timestamptz NOT NULL,
  revoked_at timestamptz,
  user_agent text,
  ip         inet,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

Argon2id hashing, 15-minute JWT access tokens, rotating refresh tokens.

**Cut the spec's `roles` table.** A role enum plus a permission matrix in `app/core/permissions.py` is
fewer joins and fewer bugs. Add the table when a customer actually asks for custom roles.

---

## Catalogue

```sql
CREATE TABLE product_categories (
  id        uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id    uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name      text NOT NULL,
  slug      text NOT NULL,
  parent_id uuid REFERENCES product_categories(id),
  is_system boolean NOT NULL DEFAULT false,
  UNIQUE (org_id, slug)
);

CREATE TABLE products (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id                uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  sku                   text NOT NULL,
  name                  text NOT NULL,
  category_id           uuid REFERENCES product_categories(id),
  subcategory           text,
  purity_label          text NOT NULL,          -- '22K','18K','916' — must match pricing_rules
  purity_karat          numeric(4,1),
  gross_weight_g        numeric(10,3) NOT NULL,
  net_gold_weight_g     numeric(10,3) NOT NULL,
  stone_weight_g        numeric(10,3) NOT NULL DEFAULT 0,
  stone_charge          numeric(12,2) NOT NULL DEFAULT 0,
  making_charge_type    text CHECK (making_charge_type IN ('percentage','per_gram','fixed')),
  making_charge_value   numeric(12,2),          -- product-level override; NULL = use rule
  wastage_pct_override  numeric(5,2),
  style_tags            text[] NOT NULL DEFAULT '{}',
  occasion_tags         text[] NOT NULL DEFAULT '{}',
  gender                text,
  availability          text NOT NULL DEFAULT 'in_stock'
                        CHECK (availability IN ('in_stock','made_to_order','out_of_stock')),
  description           text,
  is_active             boolean NOT NULL DEFAULT true,
  created_at            timestamptz NOT NULL DEFAULT now(),
  updated_at            timestamptz NOT NULL DEFAULT now(),
  UNIQUE (org_id, sku)
);
CREATE INDEX ON products (org_id, category_id, purity_label, net_gold_weight_g)
  WHERE is_active;

CREATE TABLE product_images (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      uuid NOT NULL,
  product_id  uuid NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  storage_key text NOT NULL,
  sort_order  int NOT NULL DEFAULT 0,
  is_primary  boolean NOT NULL DEFAULT false,
  width int, height int
);

CREATE TABLE product_embeddings (
  product_id     uuid PRIMARY KEY REFERENCES products(id) ON DELETE CASCADE,
  org_id         uuid NOT NULL,
  embedding      vector(384) NOT NULL,
  embedding_text text NOT NULL,
  content_hash   text NOT NULL,
  model          text NOT NULL,
  updated_at     timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_pe_embedding ON product_embeddings
  USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
```

`product_embeddings` is a separate table so re-embedding never touches the hot `products` row, and so a
dimension change is one table's migration.

**HNSW, not IVFFlat.** IVFFlat requires a representative training set and retraining after bulk loads;
a pilot shop starts at zero products and grows to a few hundred, which is exactly the range where
IVFFlat's lists would be badly tuned. HNSW builds incrementally with better recall at low row counts.

---

## Gold rates and pricing rules

```sql
CREATE TABLE gold_rates (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id            uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  purity_label      text NOT NULL,
  rate_per_gram     numeric(12,2) NOT NULL,
  effective_from    timestamptz NOT NULL DEFAULT now(),
  effective_to      timestamptz,
  source            text NOT NULL DEFAULT 'manual' CHECK (source IN ('manual','api')),
  created_by_user_id uuid REFERENCES users(id),
  created_at        timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON gold_rates (org_id, purity_label, effective_from DESC);
```

**Append-only. Never `UPDATE`.** The current rate is the latest row where
`effective_from <= now() AND (effective_to IS NULL OR effective_to > now())`. Yesterday's rate stays
queryable, which is what makes quote reproduction auditable rather than merely stored.

```sql
CREATE TABLE pricing_rules (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  category_id         uuid REFERENCES product_categories(id),   -- NULL = any
  purity_label        text,                                     -- NULL = any
  making_charge_type  text NOT NULL CHECK (making_charge_type IN ('percentage','per_gram','fixed')),
  making_charge_value numeric(12,2) NOT NULL,
  min_making_charge   numeric(12,2),
  wastage_pct         numeric(5,2) NOT NULL DEFAULT 0,
  gst_metal_pct       numeric(5,2) NOT NULL DEFAULT 3.00,
  gst_making_pct      numeric(5,2) NOT NULL DEFAULT 5.00,
  specificity         smallint GENERATED ALWAYS AS (
                        (category_id IS NOT NULL)::int * 2 + (purity_label IS NOT NULL)::int
                      ) STORED,
  effective_from      timestamptz NOT NULL DEFAULT now(),
  effective_to        timestamptz,
  is_active           boolean NOT NULL DEFAULT true
);
```

**Resolution — one winning rule, whole-record, never field-merged:**

```sql
SELECT * FROM pricing_rules
WHERE org_id = :org AND is_active
  AND (category_id IS NULL OR category_id = :category_id)
  AND (purity_label IS NULL OR purity_label = :purity)
  AND effective_from <= :at AND (effective_to IS NULL OR effective_to > :at)
ORDER BY specificity DESC, effective_from DESC
LIMIT 1;
```

Precedence: **category+purity (3) > category (2) > purity (1) > org default (0)**, with a product-level
override beating all four. Category outranks purity because in Indian jewellery making charges vary far
more by product type (bangles vs chain) than by karat.

Whole-record resolution rather than per-field merging means you can show a shop owner exactly one row and
say *"this is the rule that priced your quote."*

**Every org gets a seeded specificity-0 default row at signup**, so resolution can never return NULL.

---

## Quotations — the snapshot

```sql
CREATE TABLE quotations (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id                uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  quote_number          text NOT NULL,
  lead_id               uuid REFERENCES leads(id),
  customer_id           uuid NOT NULL REFERENCES customers(id),
  created_by_user_id    uuid REFERENCES users(id),
  source                text NOT NULL CHECK (source IN ('ai_auto','salesperson')),
  status                text NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','sent','accepted','rejected','expired')),
  valid_until           date,
  currency              text NOT NULL DEFAULT 'INR',
  gst_mode              text NOT NULL,          -- frozen from org at creation
  subtotal              numeric(14,2) NOT NULL,
  discount_amount       numeric(14,2) NOT NULL DEFAULT 0,
  gst_amount            numeric(14,2) NOT NULL,
  total_amount          numeric(14,2) NOT NULL,
  pricing_engine_version text NOT NULL,
  snapshot              jsonb NOT NULL,
  pdf_storage_key       text,
  notes                 text,
  created_at            timestamptz NOT NULL DEFAULT now(),
  UNIQUE (org_id, quote_number)
);

CREATE TABLE quotation_items (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        uuid NOT NULL,
  quotation_id  uuid NOT NULL REFERENCES quotations(id) ON DELETE CASCADE,
  line_no       int NOT NULL,
  product_id    uuid REFERENCES products(id),
  description   text NOT NULL,

  -- FROZEN PRICING INPUTS — the snapshot. Copied at creation, never updated.
  category_id         uuid,
  purity_label        text NOT NULL,
  gross_weight_g      numeric(10,3) NOT NULL,
  net_gold_weight_g   numeric(10,3) NOT NULL,
  stone_weight_g      numeric(10,3) NOT NULL DEFAULT 0,
  gold_rate_per_gram  numeric(12,2) NOT NULL,
  wastage_pct         numeric(5,2)  NOT NULL,
  making_charge_type  text          NOT NULL,
  making_charge_value numeric(12,2) NOT NULL,
  stone_charge        numeric(12,2) NOT NULL DEFAULT 0,
  gst_metal_pct       numeric(5,2)  NOT NULL,
  gst_making_pct      numeric(5,2)  NOT NULL,

  -- FROZEN OUTPUTS
  metal_value          numeric(14,2) NOT NULL,
  wastage_amount       numeric(14,2) NOT NULL,
  making_charge_amount numeric(14,2) NOT NULL,
  line_subtotal        numeric(14,2) NOT NULL,
  gst_amount           numeric(14,2) NOT NULL,
  line_total           numeric(14,2) NOT NULL,

  -- PROVENANCE ONLY — never read when reproducing
  pricing_rule_id uuid REFERENCES pricing_rules(id),
  gold_rate_id    uuid REFERENCES gold_rates(id),
  computation     jsonb,

  UNIQUE (quotation_id, line_no)
);
```

**The key decision: frozen inputs are first-class typed numeric columns, not only JSON.**

Reproducing an old quote reads **only these columns**. It never dereferences `pricing_rule_id` or
`gold_rate_id` — those exist purely so an audit can answer *"which rule was in force"*. Columns rather
than JSON because they are typed, aggregatable, indexable and constrained.

`quotations.snapshot` additionally stores the resolved `pricing_rules` row, the `gold_rates` row and the
org tax config verbatim, as forensic backup for fields the columns did not anticipate.

`pricing_engine_version` plus a `PRICING_ENGINES = {"1.0.0": EngineV1, ...}` registry means a formula
change never retroactively alters an old quote's reprint.

---

## Channels, conversations, messages

```sql
CREATE TABLE inbound_events (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id          uuid,
  channel         text NOT NULL,
  external_id     text NOT NULL,
  raw_payload     jsonb NOT NULL,
  signature_valid boolean,
  received_at     timestamptz NOT NULL DEFAULT now(),
  processed_at    timestamptz,
  job_id          bigint,
  error           text,
  UNIQUE (channel, external_id)
);

CREATE TABLE conversations (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  customer_id         uuid NOT NULL REFERENCES customers(id),
  channel             text NOT NULL,
  channel_thread_id   text,
  state               text NOT NULL DEFAULT 'open' CHECK (state IN ('open','closed')),
  automation_mode     text NOT NULL DEFAULT 'ai' CHECK (automation_mode IN ('ai','human','paused')),
  taken_over_by_user_id uuid REFERENCES users(id),
  taken_over_at       timestamptz,
  returned_to_ai_at   timestamptz,
  last_message_at     timestamptz,
  created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON conversations (org_id, channel, channel_thread_id)
  WHERE channel_thread_id IS NOT NULL;

CREATE TABLE messages (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id            uuid NOT NULL,
  conversation_id   uuid NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  lead_id           uuid REFERENCES leads(id),
  direction         text NOT NULL CHECK (direction IN ('inbound','outbound')),
  sender            text NOT NULL CHECK (sender IN ('customer','ai','salesperson','system')),
  channel           text NOT NULL,
  external_id       text,
  body              text NOT NULL,
  detected_language text,
  media             jsonb NOT NULL DEFAULT '[]',
  status            text NOT NULL DEFAULT 'received'
                    CHECK (status IN ('received','draft','queued','sent','delivered','failed')),
  ai_interaction_id uuid,
  sent_by_user_id   uuid REFERENCES users(id),
  occurred_at       timestamptz NOT NULL,
  created_at        timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON messages (org_id, channel, external_id) WHERE external_id IS NOT NULL;
CREATE INDEX ON messages (org_id, conversation_id, occurred_at);
```

`messages.lead_id` is denormalised at processing time so "messages on this lead" is one index scan and
survives thread reuse.

---

## CRM

```sql
CREATE TABLE customers (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id             uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  phone_e164         text NOT NULL,
  name               text,
  email              citext,
  city               text,
  preferred_language text,
  opted_out          boolean NOT NULL DEFAULT false,
  opted_out_at       timestamptz,
  notes              text,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  UNIQUE (org_id, phone_e164)
);
```

**Phone is mandatory and is the identity key.** Both MVP channels can enforce it — the web form asks, and
the paste box requires staff to type the number, which they have because it is a WhatsApp thread. Making
it `NOT NULL` and unique per org removes an entire class of fuzzy identity-resolution bugs. Take the
constraint.

```sql
CREATE TABLE leads (
  id                       uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id                   uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  customer_id              uuid NOT NULL REFERENCES customers(id),
  conversation_id          uuid REFERENCES conversations(id),
  status                   text NOT NULL DEFAULT 'NEW' CHECK (status IN
                           ('NEW','CONTACTED','QUALIFIED','QUOTATION_SENT','FOLLOW_UP',
                            'APPOINTMENT','NEGOTIATION','WON','LOST')),
  source                   text,
  assigned_user_id         uuid REFERENCES users(id),
  score                    smallint,
  score_band               text CHECK (score_band IN ('cold','warm','hot','very_hot')),
  score_reasons            jsonb NOT NULL DEFAULT '[]',
  score_version            text,
  requirements             jsonb NOT NULL DEFAULT '{}',
  requirements_confidence  jsonb NOT NULL DEFAULT '{}',
  estimated_value          numeric(14,2),
  needs_review             boolean NOT NULL DEFAULT false,
  ai_summary               text,
  ai_summary_stale         boolean NOT NULL DEFAULT true,
  ai_summary_at            timestamptz,
  last_customer_message_at timestamptz,
  last_outbound_at         timestamptz,
  next_action_at           timestamptz,
  lost_reason              text,
  won_at timestamptz, lost_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON leads (conversation_id) WHERE status NOT IN ('WON','LOST');
CREATE INDEX ON leads (org_id, status, score DESC);
CREATE INDEX ON leads (org_id, assigned_user_id, next_action_at);

CREATE TABLE lead_status_history (
  id              bigserial PRIMARY KEY,
  org_id          uuid NOT NULL,
  lead_id         uuid NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
  from_status     text, to_status text NOT NULL,
  changed_by      text NOT NULL CHECK (changed_by IN ('ai','user','system')),
  changed_by_user_id uuid REFERENCES users(id),
  reason          text,
  created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE requirement_extractions (
  id                bigserial PRIMARY KEY,
  org_id            uuid NOT NULL,
  lead_id           uuid NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
  message_id        uuid NOT NULL REFERENCES messages(id),
  extracted         jsonb NOT NULL,
  confidence        jsonb NOT NULL,
  model             text NOT NULL,
  ai_interaction_id uuid,
  created_at        timestamptz NOT NULL DEFAULT now()
);
```

**One conversation has at most one open lead**, enforced by the partial unique index. When a WON customer
returns on the same thread, a new lead is created against the same conversation.

**Requirements live in two places on purpose.** `leads.requirements` is the current merged view used by
the UI and the retriever; `requirement_extractions` is the append-only per-message history. Merging is
deterministic Python (`merge_requirements()`): a later high-confidence value overwrites, and **a later
`null` never clears an existing value.** That single rule prevents the "the AI's third message wiped her
budget" bug class.

---

## Automation, AI, audit

```sql
CREATE TABLE followup_rules (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name          text NOT NULL,
  trigger_event text NOT NULL,                 -- 'quotation_sent', 'lead_created', ...
  steps         jsonb NOT NULL,                -- [{"delay_hours":24},{"delay_hours":48},{"delay_hours":96}]
  is_active     boolean NOT NULL DEFAULT true,
  quiet_start   time NOT NULL DEFAULT '09:00',
  quiet_end     time NOT NULL DEFAULT '21:00',
  max_per_lead  int NOT NULL DEFAULT 3,
  max_per_customer_per_day int NOT NULL DEFAULT 1
);

CREATE TABLE followups (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id          uuid NOT NULL,
  lead_id         uuid NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
  conversation_id uuid NOT NULL REFERENCES conversations(id),
  rule_id         uuid REFERENCES followup_rules(id),
  step_index      int NOT NULL,
  scheduled_for   timestamptz NOT NULL,
  status          text NOT NULL DEFAULT 'scheduled'
                  CHECK (status IN ('scheduled','claimed','sent','skipped','cancelled','failed')),
  channel         text NOT NULL,
  generated_body  text,
  sent_message_id uuid REFERENCES messages(id),
  skip_reason     text,
  attempts        int NOT NULL DEFAULT 0,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON followups (lead_id, step_index)
  WHERE status IN ('scheduled','claimed','sent');
CREATE INDEX ON followups (status, scheduled_for) WHERE status = 'scheduled';

CREATE TABLE appointments (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id           uuid NOT NULL,
  lead_id          uuid REFERENCES leads(id),
  customer_id      uuid NOT NULL REFERENCES customers(id),
  scheduled_at     timestamptz NOT NULL,
  duration_min     int NOT NULL DEFAULT 30,
  location         text,
  assigned_user_id uuid REFERENCES users(id),
  status           text NOT NULL DEFAULT 'scheduled'
                   CHECK (status IN ('scheduled','confirmed','completed','no_show','cancelled')),
  notes            text,
  created_at       timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE ai_interactions (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id                uuid,
  purpose               text NOT NULL,          -- triage | respond | summarize | followup
  provider              text NOT NULL,
  model                 text NOT NULL,
  prompt_version        text NOT NULL,
  input_tokens          int, output_tokens int,
  cached_input_tokens   int, cache_write_tokens int,
  cost_usd              numeric(10,6),
  latency_ms            int,
  attempt               smallint NOT NULL DEFAULT 1,
  stop_reason           text,
  status                text NOT NULL CHECK (status IN
                        ('ok','schema_invalid','repaired','escalated','failed','refused')),
  raw_response jsonb, parsed jsonb, error text,
  conversation_id uuid, message_id uuid, lead_id uuid,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON ai_interactions (org_id, created_at);
CREATE INDEX ON ai_interactions (org_id, purpose, created_at);

CREATE TABLE audit_logs (
  id            bigserial PRIMARY KEY,
  org_id        uuid,
  actor_type    text NOT NULL CHECK (actor_type IN ('user','ai','system')),
  actor_user_id uuid REFERENCES users(id),
  action        text NOT NULL,
  entity_type   text NOT NULL,
  entity_id     uuid,
  before jsonb, after jsonb,
  ip inet,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON audit_logs (org_id, entity_type, entity_id, created_at DESC);

CREATE TABLE integrations (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id            uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  kind              text NOT NULL,
  config            jsonb NOT NULL DEFAULT '{}',
  secret_ciphertext bytea,
  is_active         boolean NOT NULL DEFAULT true
);

CREATE TABLE worker_heartbeats (
  worker_id      text PRIMARY KEY,
  last_seen_at   timestamptz NOT NULL,
  jobs_processed bigint NOT NULL DEFAULT 0
);
```

Integration secrets are Fernet-encrypted app-side from `APP_ENCRYPTION_KEY`. They are never returned by
any API serializer — enforced by keeping them off the Pydantic response model entirely, not by filtering.

---

## Jobs

```sql
CREATE TABLE jobs (
  id           bigserial PRIMARY KEY,
  org_id       uuid,                            -- NULL for system jobs
  kind         text NOT NULL,
  payload      jsonb NOT NULL DEFAULT '{}',
  dedupe_key   text,
  run_after    timestamptz NOT NULL DEFAULT now(),
  priority     smallint NOT NULL DEFAULT 100,   -- lower runs sooner
  status       text NOT NULL DEFAULT 'queued'
               CHECK (status IN ('queued','running','done','failed','dead')),
  attempts     int NOT NULL DEFAULT 0,
  max_attempts int NOT NULL DEFAULT 5,
  locked_at    timestamptz, locked_by text,
  last_error   text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX jobs_dedupe ON jobs (dedupe_key)
  WHERE dedupe_key IS NOT NULL AND status IN ('queued','running');
CREATE INDEX jobs_claim ON jobs (priority, run_after) WHERE status = 'queued';
```

`jobs` is **not** an RLS table — the worker reads it on a privileged path before it knows which org it is
acting for.

---

## Cut from the spec's table list

| Table | Why |
|---|---|
| `orders` | `WON` status plus a note is enough for the pilot |
| `workflow_runs` / `workflow_steps` | `jobs` + `followups` + `audit_logs` cover every case in the spec. A generic workflow DSL is the classic solo-founder trap. |
| `roles` | Enum plus a permission matrix in code |

---

## RLS migration helper

Apply to every table with an `org_id`. Keep it as a helper so a new table cannot be forgotten:

```python
TENANT_TABLES = [
    "users", "customers", "leads", "lead_status_history", "requirement_extractions",
    "conversations", "messages", "products", "product_categories", "product_images",
    "product_embeddings", "gold_rates", "pricing_rules", "quotations", "quotation_items",
    "followups", "followup_rules", "appointments", "ai_interactions", "audit_logs",
    "integrations",
]

def apply_rls(op, table: str) -> None:
    op.execute(f"ALTER TABLE {table} ENABLE ROW LEVEL SECURITY")
    op.execute(f"ALTER TABLE {table} FORCE ROW LEVEL SECURITY")
    op.execute(f"""
        CREATE POLICY tenant_isolation ON {table}
          USING      (org_id = current_setting('app.org_id', true)::uuid)
          WITH CHECK (org_id = current_setting('app.org_id', true)::uuid)
    """)
```

The `test_every_tenant_table_has_forced_rls` test in [01](01-architecture.md) is what catches the table
you forget to add to this list.
