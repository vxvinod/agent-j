# 01 — Architecture

**One datastore, three application processes, no message broker, no agent loop.**

Dev and prod run the identical compose file plus a small production overlay.

---

## Container topology

| Container | Image / entrypoint | Role |
|---|---|---|
| `db` | `pgvector/pgvector:pg17` | Business data, vectors, job queue and audit — one Postgres |
| `api` | backend image · `uvicorn app.main:app` | REST API, public enquiry endpoint, future WhatsApp webhook. Stateless. |
| `worker` | **same backend image** · `python -m app.worker` | AI pipeline, follow-ups, embeddings, PDF generation |
| `web` | node · `next start` | Next.js UI |
| `minio` | `minio/minio` | S3-compatible object storage locally; Cloudflare R2 in prod via three env vars |
| `caddy` | `caddy:alpine` | TLS + reverse proxy — **production overlay only** |

`api` and `worker` share one image with different entrypoints. This halves the build and deploy surface
and means a pricing bug is fixed once.

**They never call each other.** The `jobs` table is the only channel between them, which is what makes
the transactional outbox property work: a stored message and the job that will process it commit together
or not at all.

---

## Docker setup

Nothing installed on the host except Docker. Dev tooling runs in containers, invoked through a `Makefile`
so there is one command per task and no environment drift.

### `docker-compose.yml`

```yaml
services:
  db:
    image: pgvector/pgvector:pg17
    environment:
      POSTGRES_USER: app_owner
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-devpassword}
      POSTGRES_DB: jewel
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./backend/scripts/init-roles.sql:/docker-entrypoint-initdb.d/10-roles.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_owner -d jewel"]
      interval: 5s
      retries: 10

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: [miniodata:/data]

  api:
    build: ./backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    env_file: .env
    depends_on:
      db: {condition: service_healthy}
    ports: ["8000:8000"]
    volumes: ["./backend:/srv"]        # dev only; drop in prod overlay

  worker:
    build: ./backend
    command: python -m app.worker
    env_file: .env
    depends_on:
      db: {condition: service_healthy}
    volumes: ["./backend:/srv"]

  web:
    build: ./frontend
    command: npm run dev
    env_file: .env
    depends_on: [api]
    ports: ["3000:3000"]
    volumes: ["./frontend:/srv", "/srv/node_modules"]

volumes:
  pgdata:
  miniodata:
```

### `backend/scripts/init-roles.sql`

This is load-bearing. **The app must connect as a non-owner or RLS silently does nothing.**

```sql
CREATE ROLE app_user LOGIN PASSWORD 'devpassword';
GRANT CONNECT ON DATABASE jewel TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT USAGE, SELECT ON SEQUENCES TO app_user;
```

Two roles, two purposes:

- **`app_user`** — what `api` and `worker` connect as. RLS policies apply.
- **`app_owner`** — what Alembic migrations connect as. Bypasses policies, which is intended.

Getting this backwards disables tenant isolation with no visible symptom, which is exactly why it lives
in the compose file rather than a README.

### `Makefile`

```make
.PHONY: up down logs migrate revision test eval seed typegen fmt lint backup restore shell

up:       ; docker compose up -d --build
down:     ; docker compose down
logs:     ; docker compose logs -f api worker
shell:    ; docker compose exec api bash

migrate:  ; docker compose exec api alembic upgrade head
revision: ; docker compose exec api alembic revision --autogenerate -m "$(m)"

test:     ; docker compose exec api pytest -q
eval:     ; docker compose exec api python -m tests.evals.run_eval
seed:     ; docker compose exec api python -m app.scripts.seed_demo

typegen:  ; docker compose exec api python -m app.scripts.export_openapi && \
            docker compose exec web npx openapi-typescript /shared/openapi.json -o src/lib/types.gen.ts

fmt:      ; docker compose exec api ruff format . && docker compose exec api ruff check --fix .
lint:     ; docker compose exec api ruff check . && docker compose exec api mypy app

backup:   ; docker compose exec db pg_dump -U app_owner jewel | gzip > backups/$(shell date +%F).sql.gz
restore:  ; gunzip -c $(f) | docker compose exec -T db psql -U app_owner jewel
```

Alembic connects with the **owner** URL; the app connects with the **app_user** URL. Keep both in `.env`:

```
DATABASE_URL=postgresql+asyncpg://app_user:devpassword@db:5432/jewel
MIGRATION_DATABASE_URL=postgresql+psycopg://app_owner:devpassword@db:5432/jewel
```

### Production

`docker-compose.prod.yml` overlay adds Caddy, drops MinIO in favour of R2, removes the source bind mounts
and `--reload`, and sets `restart: unless-stopped`. Deployment is:

```bash
git pull && docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

Nothing here needs Kubernetes, a managed queue, or a managed vector database.

---

## Background jobs: a Postgres queue

Build a ~150-line queue on a `jobs` table using `FOR UPDATE SKIP LOCKED`. Not Celery, not pgmq, not
APScheduler.

**Why not Celery + Redis.** The decisive argument is not the extra container — it is that job state would
live outside the database, so you could not answer *"why didn't Priya get her follow-up?"* with a SQL join.
The spec makes auditability a hard requirement. You also lose the transactional outbox: with a Postgres
queue, `INSERT INTO messages` and `INSERT INTO jobs` commit atomically.

**Why not pgmq.** It is good, but it is a non-default extension and you already require `pgvector`. Adding
a second extension dependency means maintaining a custom image or narrowing your future hosting options.
`SKIP LOCKED` is core Postgres and runs on Neon, Supabase, RDS, Railway or a bare install. That preserved
optionality is worth more than the ~100 lines saved.

**Why not APScheduler in-process.** It couples job execution to the API process: a six-second model call
competes with request handlers, a deploy kills in-flight work with no durable record, and there is no
retry history to inspect.

### The claim query

```sql
UPDATE jobs SET status='running', attempts=attempts+1, locked_at=now(), locked_by=:worker_id
WHERE id IN (
  SELECT id FROM jobs
  WHERE status='queued' AND run_after <= now() AND kind = ANY(:kinds)
  ORDER BY priority, run_after
  FOR UPDATE SKIP LOCKED
  LIMIT :batch
)
RETURNING *;
```

### Worker loop

- asyncio, `WORKER_CONCURRENCY=4` tasks — the work is I/O-bound on the model API
- 500ms poll when idle, immediate re-loop when a batch came back
- backoff `run_after = now() + interval '30 seconds' * 2^attempts`, capped at 1 hour
- a **reaper** in each tick returns rows `running` with `locked_at < now() - '5 minutes'` to `queued`,
  or to `dead` past `max_attempts`
- every handler is wrapped so an exception writes `jobs.last_error` and a Sentry event **without killing
  the loop** — one poison job must never stop the queue

### Job kinds

`process_inbound_message` · `send_followup` · `tick_followups` · `embed_product` · `summarize_lead` ·
`generate_pdf`

### Scaling path

Each step requires no rewrite:

1. Run more `worker` containers — `SKIP LOCKED` is already correct for N consumers. Zero code change.
2. Split queues by kind (`--kinds=process_inbound_message` vs `--kinds=embed_product`) so an embedding
   backlog cannot starve a customer waiting on a reply. The `kind` filter is already in the claim query.
3. Add `LISTEN/NOTIFY` on enqueue to drop poll latency to ~0. About 20 lines.
4. Only past ~50 sustained jobs/sec consider a broker. **You will hit model-provider rate limits an order
   of magnitude before you hit this queue's limits.**

No Redis anywhere. Rate limiting is `slowapi` in-process (correct enough at one API container); caching is
unnecessary since gold rates are four rows.

---

## Tenant isolation: Postgres RLS

Use RLS, with application-layer scoping kept as well.

The argument is specifically about who is writing the code. A solo founder generating a lot of code with
AI assistance will eventually ship a query missing `.where(Lead.org_id == ctx.org_id)`. Under
application-only scoping that is a cross-tenant data breach. Under RLS it returns zero rows — a support
ticket, not an incident. That asymmetry justifies the setup cost.

```sql
ALTER TABLE leads ENABLE ROW LEVEL SECURITY;
ALTER TABLE leads FORCE  ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON leads
  USING      (org_id = current_setting('app.org_id', true)::uuid)
  WITH CHECK (org_id = current_setting('app.org_id', true)::uuid);
```

`current_setting('app.org_id', true)` returns NULL when unset, so the predicate is false and the table is
empty by default. **Fail-closed.**

### Four things that must be right, or RLS does nothing

1. The app connects as a **non-owner** role and every tenant table has **`FORCE ROW LEVEL SECURITY`**.
   Without `FORCE`, the table owner bypasses policies and RLS is decorative.
2. A FastAPI dependency runs `SET LOCAL app.org_id = :org` **inside the request transaction**. `SET LOCAL`
   is transaction-scoped, so it is safe under PgBouncer transaction pooling and cannot leak across pooled
   connections.
3. The worker does the same from `jobs.payload.org_id` before touching business tables. `jobs`,
   `organizations` and `users` are the exceptions it reads on a privileged path.
4. **`WITH CHECK` is present**, or a tenant can *insert* rows belonging to another org.

### The one test that keeps this correct

```python
def test_every_tenant_table_has_forced_rls(sync_conn):
    rows = sync_conn.execute(text("""
        SELECT c.relname
        FROM pg_class c
        JOIN pg_namespace n ON n.oid = c.relnamespace
        JOIN information_schema.columns col
          ON col.table_name = c.relname AND col.column_name = 'org_id'
        WHERE n.nspname = 'public' AND c.relkind = 'r'
          AND NOT (c.relrowsecurity AND c.relforcerowsecurity)
    """)).scalars().all()
    assert rows == [], f"tables with org_id but no forced RLS: {rows}"
```

Write this in week 1. It is what keeps the invariant true across four months of generated code.

Keep `org_id` on every table and keep passing it in application queries anyway — it enables composite
indexes like `(org_id, status, created_at)`, and it means a test can prove isolation twice.

---

## Channel adapters

The abstraction lives in two columns — `channel` and `external_id` — with a partial unique index. Every
adapter mints an `external_id`:

| Channel | `external_id` |
|---|---|
| `web_form` | `wf_{client_idempotency_key}` — UUID minted by the widget on mount, so a double-submit is one message |
| `paste` | `paste_{sha256(org_id ‖ conversation_id ‖ normalised_body)[:32]}` — pasting the same text twice is a provable no-op |
| `whatsapp` | the provider's message id, verbatim |

Idempotency is therefore identical across all three, and the spec's *"a duplicate webhook must not create
duplicate leads or messages"* becomes a database constraint rather than application care.

### Inbound splits in two

This is what makes WhatsApp a new file rather than a refactor.

**1. `api` endpoint** — verify signature → `INSERT INTO inbound_events` + `INSERT INTO jobs` in one
transaction → return 200. Under 20 lines per adapter. `ON CONFLICT (channel, external_id) DO NOTHING`
makes replay free.

**2. `worker`** — `adapters[channel].normalize(raw_payload) -> NormalizedInbound` → the single shared
pipeline.

```python
class NormalizedInbound(BaseModel):
    phone_e164: str
    name: str | None
    body: str
    media: list[MediaRef] = []
    occurred_at: datetime
    thread_id: str | None
    source_metadata: dict = {}
```

Everything downstream sees only this model.

### Outbound

`OutboundAdapter.dispatch()`. For the MVP the implementation is **`ManualDispatchAdapter`**: it writes the
message with `status='draft'` and surfaces it in the UI as a *Ready to send* card with a copy button and a
`https://wa.me/{phone}?text={urlencoded}` deep link. The salesperson taps it, WhatsApp opens pre-filled,
they send, the card marks itself sent.

When the WhatsApp adapter lands, `dispatch()` becomes an API call and the card disappears. Nothing else
changes.

---

## Repo structure

Plain monorepo. A `Makefile` and compose files, no Turborepo or Nx.

```
agent-j/
  Makefile   docker-compose.yml   docker-compose.prod.yml   .env.example
  plan/                          # these documents
  docs/adr/
  backend/
    Dockerfile  pyproject.toml  alembic.ini  alembic/versions/
    scripts/init-roles.sql
    app/
      main.py  worker.py  settings.py
      db/         session.py  base.py  rls.py
      models/     org.py user.py customer.py lead.py conversation.py message.py
                  product.py pricing.py quotation.py followup.py appointment.py
                  ai.py job.py audit.py
      schemas/                              # Pydantic request/response DTOs
      api/
        deps.py                             # get_tenant_ctx (SET LOCAL), require_role
        v1/       auth.py orgs.py users.py customers.py leads.py conversations.py
                  products.py pricing.py quotations.py followups.py appointments.py
                  dashboard.py admin.py
        public/   enquiry.py  webhooks_whatsapp.py
        health.py                           # /health, /health/worker
      channels/   base.py web_form.py paste.py manual_dispatch.py whatsapp_cloud.py normalizer.py
      pricing/    engine.py rules.py units.py rounding.py versions.py
      ai/         provider.py gemini_provider.py anthropic_provider.py openai_provider.py
                  fake_provider.py runner.py schemas.py guards.py fallback_extractor.py
                  prompts/ triage_v1.md respond_v1.md glossary_ta.md examples_tanglish.md
      scoring/    rules.py scorer.py
      recommend/  embedder.py document.py retriever.py ranker.py explain.py
      pipelines/  inbound.py quotation.py followup.py
      tools/      registry.py products.py pricing.py crm.py handoff.py
      jobs/       queue.py reaper.py handlers/
      services/   customer.py lead.py quotation.py followup.py onboarding.py
      core/       security.py crypto.py permissions.py ratelimit.py audit.py storage.py errors.py
    tests/
      unit/         test_pricing.py test_scoring.py test_suppression.py test_units.py
      integration/  test_rls.py test_queue.py test_inbound_pipeline.py
      evals/        cases/*.yaml  run_eval.py  report.py
      fixtures/     golden_quotes/*.json  llm_responses/*.json
  frontend/
    Dockerfile  package.json  next.config.ts  tailwind.config.ts
    src/app/  src/components/  src/lib/
  packages/contracts/            # openapi.json  generate.sh
```

**Shared types:** Pydantic is the single source of truth. FastAPI emits `openapi.json`,
`openapi-typescript` generates `frontend/src/lib/types.gen.ts` in CI, and **CI fails if the generated file
is stale**. No hand-maintained duplicates, no tRPC.

---

## Critical files

These carry the load. Everything else is CRUD.

| File | Why it matters |
|---|---|
| `app/pricing/engine.py` | The deterministic formula, `Decimal` discipline, snapshot construction, engine versioning. Everything financial routes through here. |
| `app/pipelines/inbound.py` | The fixed orchestrator replacing an agent loop; where AI, CRM, retrieval, pricing and dispatch meet. |
| `app/ai/runner.py` | The single implementation of the retry → repair → escalate → fallback → human ladder, plus cost accounting. |
| `app/db/rls.py` + `app/api/deps.py` | `SET LOCAL app.org_id` on every request and job. The tenant isolation boundary. |
| `app/jobs/queue.py` | The `SKIP LOCKED` claim, backoff, reaper, transactional enqueue. |
| `app/channels/base.py` | The adapter protocol and `NormalizedInbound`. The seam that makes WhatsApp a new file. |

---

## ADRs worth writing

Write these in `docs/adr/` as you make them, one page each:

1. `0001-postgres-job-queue.md` — why not Celery/Redis
2. `0002-rls-tenant-isolation.md` — why RLS over application scoping
3. `0003-no-llm-tool-calling-in-mvp.md` — fixed pipeline over agent loop
4. `0004-quotation-snapshot-columns.md` — typed columns over JSON
5. `0005-manual-dispatch-adapter.md` — how the pilot sends messages
6. `0006-provider-abstraction.md` — tiers not model names; free tier is dev-only
