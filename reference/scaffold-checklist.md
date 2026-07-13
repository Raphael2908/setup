# Scaffold checklist — ordered build list

Build in this order. The goal at the end of each block is a stack that **boots in mock mode with no
secrets** and has **green tests offline**. Drop blocks that don't apply (no async work → skip Celery;
not metered → skip ledger/billing/PRICING). See `patterns.md` for the code of each starred file.

## Block 0 — Repo skeleton + tooling

- [ ] `git init`; create the directory tree from `stack.md`.
- [ ] Root `.gitignore` (Python: `__pycache__/ .venv/ .pytest_cache/ .ruff_cache/`; Node: `node_modules/
      .next/ *.tsbuildinfo`; env: `.env .env.*` except `.env.example`; Docker volumes: `data/ *.rdb *.aof`;
      OS: `.DS_Store`).
- [ ] `.env.example` — every key from `config.py`, grouped (Runtime, API/CORS, Redis, Supabase, AWS/S3,
      Stripe, Resend, vendors, guards, NEXT_PUBLIC_*). No real values.
- [ ] `Makefile` — `up down build logs ps test lint fmt migrate maintenance-on maintenance-off` (model
      on the reference project's; `up = docker compose up --build`, `test = cd backend && uv run pytest -q`).
- [ ] `README.md` — one screen: what it is, `make` commands, link to `architecture.md`.

## Block 1 — Backend core (boots + serves /healthz)

- [ ] `backend/pyproject.toml` (from `stack.md`, trimmed to what's used).
- [ ] `backend/Dockerfile` — Python 3.12 slim; install `uv`; `uv pip install -e .`; copy app; (add
      `ffmpeg`/system deps only if needed); `ENTRYPOINT ["./entrypoint.sh"]`.
- [ ] `backend/entrypoint.sh` ★ — ROLE dispatch (`chmod +x`).
- [ ] `app/config.py` ★ — `Settings`, mock-safe defaults, prod fail-fast validator.
- [ ] `app/core/logging.py` — structured JSON logging + correlation filter; `configure_logging()`.
- [ ] `app/core/context.py` — `ContextVar`s for request_id/job_id; getters/setters.
- [ ] `app/core/auth.py` ★ — JWKS verification → `CurrentUser`; `get_current_user` dep.
- [ ] `app/db/supabase.py` — lazy service-role client singleton (raises if unconfigured).
- [ ] `app/db/repo.py` ★ — `Repo` ABC + `SupabaseRepo` + `InMemoryRepo` + `get_repo`/`set_repo`. Add a
      method set per resource noun from the data model.
- [ ] `app/schemas/` — pydantic request/response + domain models for each resource.
- [ ] `app/api/routers/health.py` — `/healthz` (and `/api/healthz`).
- [ ] `app/api/routers/<resource>.py` — CRUD per resource; authorize via `get_current_user`, data via
      `get_repo()`, ownership-check, 404 on `None`.
- [ ] `app/main.py` — FastAPI factory: CORS from `settings.cors_origins_list`, request-id middleware,
      mount routers under `/api`, `configure_logging()` on import.
- [ ] **Checkpoint:** `cd backend && uv venv && uv pip install -e ".[dev]" && uv run uvicorn app.main:app`
      → `GET /healthz` 200.

## Block 2 — Providers (if external vendors)

- [ ] `app/providers/base.py` ★ — interfaces + Result dataclasses + error hierarchy.
- [ ] `app/providers/mock.py` ★ — canned impls of every interface.
- [ ] `app/providers/factory.py` ★ — mock vs real by `PROVIDER_MODE`, real imported lazily.
- [ ] `app/providers/real/` — one module per vendor (stub now; raises if key missing). Wire later.

## Block 3 — Async pipeline (if long-running work)

- [ ] `app/core/redis.py` — Redis client singleton.
- [ ] `app/celery_app.py` ★ — app, per-stage queues, routes, timeouts, beat watchdog, signal hooks.
- [ ] `app/tasks/<stage>.py` — one task per pipeline stage; flip `jobs.status`, read inputs from DB by
      parent id, call the provider, write outputs, chain next. Retryable vs fatal handling.
- [ ] `app/tasks/watchdog.py` — `watchdog_sweep` fails jobs stuck `running` past SLA.
- [ ] `app/tasks/signals.py` — bind correlation context on task start/end.
- [ ] `app/services/pipeline.py` — build + enqueue the chain; per-invocation timeout scaling if needed.

## Block 4 — Billing + guards (if metered/paid)

- [ ] `app/services/credits.py` — append-only ledger, credit map calibrated from `VENDOR_COST_USD`,
      reserve/refund, balance check.
- [ ] `app/services/billing.py` — Stripe checkout + webhook handler; grant on `pack`/`subscription`
      events, idempotent via `stripe_events`.
- [ ] `app/services/guards.py` — max in-flight, daily spend cap, rate limit (Redis token bucket).
- [ ] `app/api/routers/billing.py`, `app/api/routers/webhooks.py` — checkout/portal/credits; Stripe +
      Resend webhooks (signature-verified).

## Block 5 — Storage (if file uploads/outputs)

- [ ] `app/services/storage.py` — presigned PUT (uploads) + presigned/signed GET (downloads); apply S3
      lifecycle for intermediates on boot.
- [ ] `app/api/routers/uploads.py` — `/uploads/presign`; validate size + content-type before issuing.
- [ ] `infra/aws/iam_policy.json`, `infra/aws/s3_lifecycle.json`.

## Block 6 — Tests (green offline)

- [ ] `app/tests/conftest.py` ★ — autouse InMemoryRepo + eager Celery + mock mode; `client` + `seed`.
- [ ] `app/tests/test_<resource>.py` — CRUD + auth happy/sad paths.
- [ ] `app/tests/test_pipeline.py` — enqueue → inline chain → succeeded (if async).
- [ ] `app/tests/test_billing.py` — reserve/refund, webhook idempotency (if metered).
- [ ] **Checkpoint:** `make test` green with no network.

## Block 7 — Frontend

- [ ] `frontend/package.json` (from `stack.md`), `tsconfig.json`, `next.config.js`, `postcss.config.mjs`,
      Tailwind v4 setup, `app/globals.css`.
- [ ] `lib/supabaseClient.ts`, `lib/useSession.ts`, `lib/apiClient.ts` ★, `lib/api.ts` (typed wrappers),
      `lib/types.ts`.
- [ ] `app/layout.tsx`, `app/page.tsx` (landing), `app/login/page.tsx`.
- [ ] `app/(app)/layout.tsx` + `components/AuthGate.tsx` — session gate; per-resource pages under `(app)/`.
- [ ] `components/landing/`, `components/ui/`, `components/magicui/` — as much UI as the product needs.
- [ ] `frontend/Dockerfile` — Next build; `NEXT_PUBLIC_*` as build args.
- [ ] Optional: `content/blog/`, `lib/blog.ts`, `app/blog/[slug]/page.tsx`, `app/sitemap.ts`.

## Block 8 — Infra + compose (full stack boots)

- [ ] `docker-compose.yml` — services: `redis`, `api` (ROLE=api), `worker`+`beat` (if async), `frontend`,
      `caddy`. `frontend` gets `NEXT_PUBLIC_*` as build args. Worker `stop_grace_period` > longest hard
      timeout. Healthchecks on redis + api.
- [ ] `infra/redis/redis.conf` — `appendonly yes`, `appendfsync everysec`.
- [ ] `infra/caddy/Caddyfile` — `/api/*` → api:8000, else → frontend:3000; TLS for the prod domain.
- [ ] `infra/supabase/migrations/0001_init.sql` — the data-model tables + RLS + the `handle_new_user`
      profile trigger; later numbered migrations for added features. Manual rollbacks outside `migrations/`.
- [ ] `infra/ops/graceful_deploy.sh` — maintenance-on → drain → rebuild → health-gate → maintenance-off.
- [ ] `.github/workflows/deploy.yml` — SSH deploy on push to `master` running `graceful_deploy.sh` (later).
- [ ] **Checkpoint:** `make up` → all services healthy; landing loads; `/api/healthz` 200; one mock
      end-to-end path works.

## Block 9 — Docs + handback

- [ ] Confirm the five docs (Phase 2) match what was built; update `current_progress.md` to "scaffold
      stood up" with what's next.
- [ ] `CLAUDE.md` points future agents at `/kt` (and `architecture.md` + `current_progress.md`) and lists `make` cmds.
- [ ] `.claude/skills/kt/SKILL.md` (from `templates/kt-skill.md`) — the `/kt` knowledge-transfer skill;
      §-index regenerated from the actual `architecture.md`, conditional table trimmed to the docs
      that exist, template note deleted.
- [ ] Summarize for the user: what's scaffolded, what's mocked/blank, first real steps (provision
      Supabase/Stripe/vendors, fill `.env`, write the first real provider, apply migrations).
