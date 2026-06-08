# <PRODUCT> — Technical Architecture

> This is the **design source of truth**. Everything else (data model, pipeline, API, infra) derives
> from it. Fill every section with real content for <PRODUCT>; delete sections that genuinely don't
> apply (e.g. the pipeline state machine for a product with no long-running work) and say why.

## 1. Goal & Scope

<One paragraph: what <PRODUCT> does, for whom, and what it produces/delivers. State what's in scope for
the MVP and what's explicitly deferred.>

### Phasing
- **Phase 1 (MVP):** <the end-to-end thing that must work to launch>.
- **Phase 2:** <the next module — designed for now so it slots in without rework>.

## 2. System Overview

> This diagram is the **scale-up target**. At launch we collapse all roles onto **one box** (§4 Phase 0)
> and split them only once we hit the scale trigger.

```
Browser ─► Next.js ─(HTTPS/JWT)─► FastAPI API tier ─enqueue─► Redis ─consume─► Celery workers ─► external APIs
                                        │ read/write                                   │
                                        ▼                                               ▼
                                  Supabase (Postgres + Auth + RLS)                   S3 + CDN
```

**Design principles**
- API tier is **stateless and thin** — validates, authorizes, (debits), enqueues. Never blocks on slow work.
- Workers own all long-running work — external calls, polling, CPU-heavy jobs. Scale independently.
- Postgres is the source of truth for state; Redis is transport + ephemeral state.

## 3. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend | Next.js (App Router) + Tailwind v4 | |
| API | FastAPI (Python 3.12) | thin, async |
| Async jobs | Celery + Redis | one queue per long stage *(drop if no async work)* |
| DB / Auth | Supabase (Postgres + GoTrue + RLS) | JWT verified via JWKS |
| Object storage | S3 (+ CloudFront later) | presigned upload/download |
| Payments | Stripe | webhooks → credit ledger *(drop if not metered)* |
| Email | Resend | transactional + inbound support |
| Containers | Docker | one image, `ROLE=api\|worker\|beat` |
| <external vendor> | <product/model> | <what it does + config var + default> |

### Model / vendor defaults
<Table of each external model id / mode, its config var (env-overridable), default, and a note. The
exact ids live in `backend/app/config.py`.>

## 4. Infrastructure & Deployment

Phased to match demand. Launch on one cheap box; split into the §2 topology only at the scale trigger.

### Phase 0 — MVP (launch) · ~$40/mo
One instance runs everything via docker-compose (api + worker + beat). Redis in a local container with
AOF persistence. No ALB/NAT/autoscaling. S3 + presigned URLs. Supabase free/Pro.

### Scale-up — trigger: <~first 50 paying users>
API tier behind an ALB in an ASG; worker tier on a separate ASG scaled on queue depth; Redis →
ElastiCache; CloudFront in front of S3; Secrets Manager. The one-image/ROLE design makes this mechanical.

## 5. Data Model (Supabase / Postgres)

Tables (RLS keyed on `user_id = auth.uid()` unless noted):

- **profiles** — mirrors `auth.users`: `id`, `email`, `stripe_customer_id`, `plan`, `created_at`.
  Auto-provisioned by an `AFTER INSERT ON auth.users` trigger (`handle_new_user`).
- **<resource>** — `id`, `user_id`, <fields>, `created_at`. <one line: what it is>.
- **jobs** *(if async)* — one row per pipeline step: `id`, `<parent>_id`, `step`, `status`
  (`queued|running|succeeded|failed|cancelled`), `provider`, `provider_job_id`, `attempts`,
  `input jsonb`, `output jsonb`, `error`, `cost_credits`, `idempotency_key`, timestamps.
- **credit_ledger** *(if metered)* — append-only: `user_id`, `delta`, `reason`, `ref_id`, `balance_after`.
- **subscriptions**, **stripe_events** *(if billed)*.

Indexes: <the hot query paths>.

## 6. Generation / Work Pipeline (state machine)  *(if long-running work)*

Each <parent resource> advances through ordered steps; each step is a **job** row + a Celery task,
chained so one step's output is the next's input.

```
<created> ─► [step1] ─► [step2] ─► … ─► [final] ─► <succeeded>
```

**Step contracts** — for each step: input, the provider call, output, and failure mode (retryable vs fatal).

**Provider abstraction.** Each external dependency sits behind an interface so a vendor swaps without
touching orchestration. Selected provider + `provider_job_id` persisted on the job for traceability.

## 7. Async Orchestration (Celery + Redis)  *(if async)*

- **Queues:** one per stage so each scales/limits independently.
- **Chaining:** a Celery chain; each task reads inputs from the DB by parent id (not big broker payloads).
- **Long calls:** submit → store `provider_job_id` → self-reschedule with backoff to poll (don't block a slot).
- **Reliability:** idempotency key per job; exponential backoff with jitter, capped; retryable vs fatal;
  `soft_time_limit` + a beat watchdog that fails jobs stuck `running` past SLA; cancel → refund.

## 8. Storage & CDN  *(if files)*
Private S3 bucket; presigned PUT for uploads; signed GET for downloads; lifecycle expiry for intermediates.

## 9. Auth
Supabase Auth (GoTrue) issues JWTs. FastAPI verifies the access token against the project JWKS
(asymmetric keys, no shared secret); `sub` → `user_id`. New opaque keys: publishable (client) / secret
(server). Direct client reads guarded by RLS; credit/job-touching writes go through the API.

## 10. Billing & Credits (Stripe)  *(if metered)*
Subscription tiers grant monthly credits; one-off packs top up. Steps debit credits per type. Reserve on
enqueue, refund on failure/cancel. Webhooks (signature-verified, idempotent via `stripe_events`) are the
only path that grants credits. Balance checked before enqueue; insufficient → 402 with shortfall.

## 11. API Design (FastAPI)
Auth required except webhooks/health. JSON; cursor pagination on lists.

```
POST/GET/...  /<resource>        CRUD per resource
POST          /<parent>          create → (quote +) enqueue        (if async)
GET           /<parent>/{id}     detail incl. per-step job status
POST          /uploads/presign   presigned S3 PUT                  (if files)
POST          /billing/checkout  Stripe Checkout                   (if billed)
POST          /webhooks/stripe   signature-verified, idempotent    (if billed)
GET           /healthz
```

## 12. Frontend (Next.js + Tailwind)
Auth via Supabase JS; RLS reads for lists. Key screens: <list them>. Authed routes under `app/(app)/`.
All API calls through `lib/apiClient.ts`. Large uploads go direct to S3 via presigned URLs.

## 13. Observability, Security, Rate Limiting
Structured JSON logs correlated by `<parent>_id`/`job_id`; queue-depth + step success/latency metrics;
request_id propagated API → worker → job rows; per-user Redis token-bucket limits + global per-vendor
concurrency caps; secrets in Secrets Manager; S3 private + signed URLs; RLS; input/content validation
before paid steps; abuse/cost guards (max in-flight, daily spend cap, upload limits).

## 14. Cost Model  *(if metered)*
Marginal cost ≈ sum of vendor calls + storage/egress + worker compute. Credit pricing covers this with
margin; the per-step credit map (§10) is the lever. See `pricing.md`. Track realized cost on `jobs.cost_credits`.

## 15. Open Questions / To Confirm Later
1. <the real unknowns — launch target, vendor access path, thresholds — with what each would change>.
