# The stack — annotated, with versions and layout

This is the tech stack the `setup` skill reproduces. It is the HexUGC stack with the domain stripped
out. Use it to fill `architecture.md` §3 and to pin dependency versions when scaffolding.

## Layers

| Layer | Choice | Notes |
|---|---|---|
| Frontend | Next.js (App Router) + Tailwind v4 | Authed routes under `app/(app)/`, marketing under `app/`. All API calls through `lib/apiClient.ts`. |
| API | FastAPI (Python 3.12, `uvicorn`/`gunicorn`) | Pydantic v2, async handlers. Thin: validate → authorize → (debit) → enqueue. Never blocks on slow work. |
| Async jobs | Celery + Redis | Redis = broker + result backend + ephemeral state (rate limits, semaphores, maintenance flag). One named queue per long stage. **Include only if the product has long-running work.** |
| DB / Auth | Supabase (Postgres 15 + GoTrue Auth + RLS) | API verifies user access tokens against the project JWKS (asymmetric keys, no shared secret). RLS guards direct client reads. |
| Object storage | AWS S3 (+ CloudFront later) | Presigned PUT for uploads (large bodies skip the API); presigned/signed GET for downloads. |
| Payments | Stripe | Checkout Sessions + webhooks. Webhooks are the **only** path that grants credits; idempotent via a `stripe_events` table. **Include only if metered/paid.** |
| Email | Resend | Transactional send + inbound support webhook (Svix-verified). |
| Containers | Docker | One backend image; `ENTRYPOINT` switches role via `ROLE=api\|worker\|beat`. `docker-compose.yml` runs all roles (this is both local dev and Phase-0 prod). |
| Reverse proxy | Caddy | TLS (Let's Encrypt) + routes `/api/*` → api, everything else → frontend. |

## Design principles

- **API tier is stateless and thin.** Validates, authorizes, debits, enqueues. The heavy/long work is
  external I/O handled off-box by workers, so a small API box serves high concurrency.
- **Workers own all long-running work.** External calls (seconds–minutes), polling, CPU-heavy jobs.
  They scale independently. For multi-minute vendor calls, use the submit→store-id→self-reschedule-poll
  pattern so a worker slot isn't blocked for the whole call.
- **Postgres is the source of truth** for job/resource state; Redis is transport + ephemeral state.
- **Phased infra.** Phase 0 = one `t3.medium` running every role via compose (~$40/mo). Scale-up
  (trigger: ~first 50 paying users) splits API tier (ALB + ASG), worker tier (separate ASG, scaled on
  queue depth), Redis → ElastiCache, S3 → CloudFront, secrets → Secrets Manager. The one-image/ROLE
  design makes the split mechanical. **Do not build the scaled topology up front.**

## Backend dependencies (`pyproject.toml`)

Python `>=3.12`. Trim to what the product uses (drop `celery`/`redis` if no async work; drop
`stripe`/`anthropic`/`yt-dlp` if not needed).

```toml
[project]
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "gunicorn>=23.0",
    "pydantic>=2.9",
    "pydantic-settings>=2.6",
    "celery>=5.4",          # if async work
    "redis>=5.2",           # if async work / rate limiting
    "httpx>=0.28",
    "supabase>=2.10",
    "pyjwt[crypto]>=2.10",  # JWKS verification
    "boto3>=1.35",          # S3
    "stripe>=11.0",         # if billing
    "resend>=2.0",          # if email
    "python-json-logger>=2.0",
    # + vendor SDKs the product needs, e.g. "anthropic>=0.49"
]

[project.optional-dependencies]
dev = ["pytest>=8.3", "ruff>=0.8"]

[tool.ruff]
line-length = 100
target-version = "py312"
[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]

[tool.pytest.ini_options]
pythonpath = ["."]
testpaths = ["app/tests"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
[tool.hatch.build.targets.wheel]
packages = ["app"]
```

## Frontend dependencies (`package.json`)

Next 15 + React 19 + Tailwind v4. `gray-matter`/`react-markdown`/`remark-gfm` only if you want a
file-based blog. `@radix-ui/*`, `motion`, `lucide-react`, `clsx`, `tailwind-merge` are the UI kit.

```json
{
  "dependencies": {
    "next": "^15.1.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@supabase/supabase-js": "^2.47.0",
    "@radix-ui/react-tooltip": "^1.1.6",
    "motion": "^11.15.0",
    "clsx": "^2.1.1",
    "tailwind-merge": "^2.6.0",
    "lucide-react": "^0.469.0",
    "gray-matter": "^4.0.3",
    "react-markdown": "^9.0.1",
    "remark-gfm": "^4.0.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "@types/node": "^22.10.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "tailwindcss": "^4.0.0",
    "@tailwindcss/postcss": "^4.0.0",
    "@tailwindcss/typography": "^0.5.16"
  }
}
```

## Directory layout

```
<project>/
├── architecture.md current_progress.md todo.md pricing.md marketing.md CLAUDE.md README.md
├── Makefile  docker-compose.yml  .env.example  .gitignore
├── backend/
│   ├── pyproject.toml  Dockerfile  entrypoint.sh
│   └── app/
│       ├── main.py            # FastAPI factory: middleware (request-id), mounts routers under /api
│       ├── config.py          # pydantic-settings Settings; all keys mock-safe; prod fail-fast
│       ├── celery_app.py      # Celery app, per-stage queues, timeouts, beat watchdog  (if async)
│       ├── api/
│       │   ├── routers/       # one router per resource (+ health, webhooks)
│       │   └── ownership.py   # owner-check helpers (RLS bypassed server-side, ownership in handlers)
│       ├── core/
│       │   ├── auth.py        # Supabase JWT verification via JWKS → CurrentUser
│       │   ├── context.py     # ContextVar correlation fields (request_id/job_id/…)
│       │   ├── logging.py     # structured JSON logging + correlation filter
│       │   └── redis.py       # Redis client singleton (if async)
│       ├── db/
│       │   ├── repo.py        # Repo ABC + SupabaseRepo + InMemoryRepo; get_repo()/set_repo()
│       │   └── supabase.py    # Supabase service-role client singleton (lazy)
│       ├── providers/         # (if external vendors)
│       │   ├── base.py        # interfaces + Result dataclasses + ProviderError/RetryableError/FatalError
│       │   ├── factory.py     # get_*_provider() → mock vs real by PROVIDER_MODE (real imported lazily)
│       │   ├── mock.py        # canned, deterministic impls — no network
│       │   └── real/          # one module per vendor; raises loudly if key missing
│       ├── services/          # pipeline.py, credits.py, billing.py, guards.py, storage.py, ratelimit.py …
│       ├── tasks/             # one Celery task module per stage + watchdog.py, signals.py, common.py
│       ├── schemas/           # pydantic request/response + domain models
│       └── tests/
│           ├── conftest.py    # autouse: InMemoryRepo, eager Celery, mock mode; client + seed fixtures
│           └── test_*.py
├── frontend/
│   ├── package.json  Dockerfile  next.config.js  tsconfig.json  postcss.config.mjs
│   ├── app/
│   │   ├── layout.tsx  page.tsx  globals.css  sitemap.ts
│   │   ├── (app)/             # authed routes (layout gates on session) + per-resource pages
│   │   ├── login/page.tsx
│   │   └── blog/[slug]/page.tsx   # optional file-based blog
│   ├── components/  (auth/, landing/, ui/, magicui/ — vendored animated primitives)
│   ├── lib/
│   │   ├── apiClient.ts       # apiFetch<T>(): attaches Supabase JWT, surfaces 503 as friendly message
│   │   ├── api.ts             # typed endpoint wrappers
│   │   ├── supabaseClient.ts  useSession.ts  types.ts  site.ts  utils.ts
│   │   └── blog.ts            # optional
│   └── content/blog/          # optional MDX posts
└── infra/
    ├── caddy/Caddyfile
    ├── redis/redis.conf       # appendonly yes; appendfsync everysec  (durable queue across restarts)
    ├── supabase/migrations/   # numbered 0001_init.sql, 0002_…  (+ rollbacks/ outside migrations/)
    ├── aws/                   # iam_policy.json, s3_lifecycle.json
    └── ops/graceful_deploy.sh # maintenance-on → drain in-flight → rebuild → health-gate → maintenance-off
```
