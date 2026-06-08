---
name: setup
description: Scaffold a fresh product from scratch reusing the HexUGC tech stack and conventions — a documentation-driven FastAPI + Celery + Redis + Supabase + Next.js + Docker stack with a provider abstraction, a repo pattern, a credit ledger, phased single-box→scale-up infra, and no-network tests. Use when starting a NEW project and the user wants to reuse "that stack" / "the HexUGC setup" / build "like hexugc". Produces the five source-of-truth docs first, then the code scaffold that boots in mock mode with no secrets.
---

# Set up a new project on the HexUGC stack

This skill reproduces a proven, documentation-driven web-product stack for a **brand-new product**.
It is not a copy of HexUGC's domain (avatars/videos) — it is the *reusable skeleton* underneath it:
the architecture, the conventions, and the infrastructure patterns. You bring the product idea; this
skill turns it into a bootable scaffold with the same bones.

**The core idea: docs drive the code.** The original project's strength is that `architecture.md` is
the single source of truth and everything else (data model, pipeline, API, infra) is derived from it.
So you write the docs *first*, then scaffold code that matches them. Do the phases in order.

## What you are reproducing

The stack, at a glance (full detail in `reference/stack.md`):

| Layer | Choice |
|---|---|
| Frontend | Next.js (App Router) + Tailwind v4 + Supabase JS |
| API | FastAPI (Python 3.12, `uv`, Pydantic v2), thin — validates, authorizes, enqueues |
| Async jobs | Celery + Redis (one named queue per long-running stage) — *only if the product has long work* |
| DB / Auth | Supabase (Postgres + GoTrue + RLS); API verifies user JWTs via JWKS |
| Object storage | S3 (+ CloudFront later); presigned upload/download |
| Payments | Stripe (Checkout + webhooks → an append-only credit ledger) — *if metered/paid* |
| Email | Resend (transactional + inbound support webhook) |
| Containers | One backend image, role chosen by `ROLE=api\|worker\|beat`; docker-compose for the whole stack |

And the **conventions that make it work** (full detail in `reference/patterns.md`):

1. **Documentation-driven.** Five markdown docs are the source of truth: `architecture.md` (design),
   `current_progress.md` (running build log), `todo.md` (product backlog), `pricing.md` (cost/margin
   model, if metered), `marketing.md` (GTM). A `CLAUDE.md` points future agents at them.
2. **Thin API, heavy workers.** The API never blocks on slow/external work; it enqueues. Workers own
   the long-running calls. Postgres is the source of truth for job state; Redis is transport + ephemeral.
3. **Provider abstraction.** Every external vendor sits behind an interface; a `factory` returns a
   **mock** (default, no keys, fully bootable) or a **real** impl based on `PROVIDER_MODE`.
4. **Repo pattern.** Data access goes through one resource-typed `Repo`; prod uses `SupabaseRepo`,
   tests use an `InMemoryRepo` dict-double. Never touch the DB SDK directly in handlers/tasks.
5. **Centralized typed config.** One `Settings` (pydantic-settings); every vendor key has a mock-safe
   blank default so the whole stack boots with no secrets. Production fails loudly if core creds are blank.
6. **No-network tests.** `conftest.py` autouse fixtures install the in-memory repo, force mock
   providers, and run Celery inline — the whole backend is exercisable deterministically, offline.
7. **Phased infra.** Launch on one cheap box (compose runs every role); the one-image/ROLE design
   makes the split into an autoscaled topology mechanical later. Don't build the scaled topology up front.

## Process

Work through these phases in order. Don't jump to code before the docs exist — the docs are what keep
the scaffold coherent, and they're half the value the user is asking for.

### Phase 0 — Locate the canonical reference (optional but recommended)

If the HexUGC repo is reachable on this machine (commonly a sibling dir, e.g. `../hexugc`), read its
real files as ground truth when a pattern is unclear — they are the most faithful copy of any snippet.
Ask the user for the path if you're unsure. If it's not reachable, this skill's `reference/` files carry
enough to reproduce the stack on their own.

### Phase 1 — Define the product (interview)

You cannot template the docs without knowing the product. Ask the user the questions below (batch them
with `AskUserQuestion` where it helps). Keep it tight — infer sensible defaults and confirm rather than
interrogating.

- **What is the product?** One-paragraph description: who it's for, what it produces/does.
- **Core domain objects.** The nouns (e.g. for HexUGC: avatars, products, projects, jobs, assets).
  These become the data model (§5) and the resources the API + repo expose.
- **Is there long-running / external work?** (AI calls, video/image processing, third-party APIs that
  take seconds–minutes.) If **yes** → keep Celery + the provider abstraction + the pipeline state
  machine. If **no** (plain CRUD/SaaS) → drop Celery/workers/beat and the provider layer; keep the
  thin-API + repo + Supabase + Next + Docker bones. Decide this explicitly; it changes the scaffold.
- **External vendors** (if any) and what each does → these become provider interfaces + real impls.
- **Billing model.** Metered credits? Flat subscription? Free? If metered → keep `pricing.md` + the
  credit ledger; calibrate the credit map to real vendor cost at a target margin. If flat/free → keep
  a simpler Stripe subscription sync and skip the ledger/pricing.md.
- **GTM / ICP.** Who are the first users and how are they reached? Drives `marketing.md`.

Capture the answers; they are the inputs to every template below.

### Phase 2 — Write the source-of-truth docs

Create these at the new project root, adapting the templates in `templates/` to the product from Phase
1. Write real content, not placeholders left in — the templates show the *structure and depth* expected.

1. **`architecture.md`** (from `templates/architecture.md`) — the spine. Fill all sections: goal/scope,
   system overview + diagram, tech-stack table, infra phasing, **data model** (the Phase-1 nouns as
   tables), the **pipeline state machine** (only if there's long work), async orchestration, storage,
   auth, billing, the **API surface**, frontend screens, observability, cost model, and open questions.
   Everything downstream derives from this — get it right first.
2. **`current_progress.md`** (from `templates/current_progress.md`) — start it as "scaffold stood up":
   where the project is, what's built, conventions locked, what's next. It's the running log future
   sessions read; seed it now and keep it updated as you scaffold.
3. **`todo.md`** (from `templates/todo.md`) — the prioritized product backlog, newest first.
4. **`pricing.md`** (from `templates/pricing.md`) — only if metered. Derive the credit map from real
   vendor cost at a stated target margin; the numbers live in code (`services/credits.py`), this is the
   rationale. Skip if the product isn't usage-metered.
5. **`marketing.md`** (from `templates/marketing.md`) — the GTM plan to the first N users.
6. **`CLAUDE.md`** — a short pointer file (model it on HexUGC's): what the project is, the commands
   (`make` targets), the architecture in brief, and the conventions/gotchas. Tell future agents to read
   `architecture.md` + `current_progress.md` first.

### Phase 3 — Scaffold the code

Build the skeleton to match the docs. Follow `reference/scaffold-checklist.md` for the exact file list
and build order, and `reference/patterns.md` for the concrete, faithful code patterns (config, repo,
providers, factory, celery app, auth, fixtures, entrypoint, compose, API client). Key rules:

- **Boot in mock mode with no secrets.** Every vendor key defaults blank; `PROVIDER_MODE=mock` returns
  canned data. The whole stack must `make up` and `make test` green before any real key exists.
- **Reproduce the bones, not HexUGC's domain.** Use the product's own nouns, providers, pipeline steps,
  and routers. The *patterns* (provider/factory/repo/config/celery/auth/fixtures) transfer verbatim;
  the *contents* are the new product's.
- **One image, ROLE dispatch.** `entrypoint.sh` switches `api|worker|beat`; compose runs each role.
  Drop the worker/beat services if Phase 1 said there's no long work.
- **Match the style.** Backend: ruff (line length 100, `E/F/I/UP/B`), `from __future__ import
  annotations`, Python 3.12. Frontend: App Router, all API calls through `lib/apiClient.ts`.

### Phase 4 — Verify

- `cd backend && uv venv && uv pip install -e ".[dev]" && uv run pytest -q` → tests green offline.
- `make up` → the full compose stack boots; the API serves `/healthz`; the frontend loads.
- Exercise one mock end-to-end path (e.g. create a resource → enqueue → see it complete in mock mode).
- Update `current_progress.md` with what was built and what's next.

Then hand back: summarize what was scaffolded, what's stubbed (mock providers, blank keys), and the
first real steps (provision Supabase/Stripe/vendors, fill `.env`, write the first real provider).

## Reference files (read as needed)

- `reference/stack.md` — the full annotated tech stack, exact dependency versions, and the directory tree.
- `reference/patterns.md` — the reusable code patterns with faithful snippets (the heart of "the stack").
- `reference/scaffold-checklist.md` — ordered file-by-file build list with what each file does.
- `templates/*.md` — the five doc templates to adapt in Phase 2.

Stay matched to these patterns; the user chose this stack deliberately. Don't invent new structure when
an existing convention fits, and don't carry HexUGC's domain into the new product.
