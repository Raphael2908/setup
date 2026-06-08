# <PRODUCT> — Implementation Progress

> The running build log and the second source of truth (with `architecture.md`). Future sessions read
> this first to learn where the project actually is. Keep it updated after each batch of substantive
> work — write a real narrative from the diff, not a changelog stub. Convert relative dates to absolute.

## Where we are

<One short paragraph: the current state. On first scaffold: "Scaffold stood up — the full stack boots in
mock mode with no secrets; backend tests green offline; <what's wired, what's still mock/stub>.">

## What's built

- **Backend core** — config (`app/config.py`, all keys mock-safe), repo pattern (`SupabaseRepo` +
  `InMemoryRepo`), JWKS auth, FastAPI factory + routers for <resources>, `/healthz`.
- **Providers** *(if any)* — interfaces + factory + mock impls for <vendors>; real impls <stubbed/wired>.
- **Pipeline** *(if async)* — Celery app, per-stage queues, <steps> tasks, beat watchdog.
- **Billing** *(if metered)* — credit ledger, Stripe checkout + webhook (idempotent), guards.
- **Frontend** — Next App Router, Supabase auth, `apiClient`, authed `(app)/` routes, <screens>.
- **Infra** — docker-compose (redis/api/worker/beat/frontend/caddy), Caddyfile, migration `0001_init`,
  graceful-deploy script.

## Verification

- `make test` → <N passed> offline (no network, mock providers, in-memory repo, eager Celery).
- `make up` → all services healthy; landing loads; `/api/healthz` 200.
- Mock end-to-end: <the path you exercised> completes.

## Conventions / decisions locked

- <Anything non-obvious decided during scaffolding: vendor access path, billing model, naming, etc.>
- Mock mode is total — every external call is behind a provider/repo, so the stack runs keyless.

## What's next

1. <Provision Supabase/Stripe/vendors; fill `.env`; apply migrations.>
2. <Write the first real provider behind its interface.>
3. <The first real product feature beyond CRUD.>

## Open questions
<Mirror `architecture.md` §15; note which are now resolved.>
