# setup — scaffold a fresh product on a proven stack

A reusable [Claude Code](https://claude.com/claude-code) **skill** that scaffolds a brand-new web
product on a documentation-driven, batteries-included stack:

**Next.js + Tailwind · FastAPI (Python 3.12) · Celery + Redis · Supabase (Postgres + Auth + RLS) ·
S3 · Stripe · Resend · Docker.**

It reproduces the *bones and conventions* of a working product — not any one product's domain — so a
fresh project inherits a coherent architecture instead of a blank page.

## What you get

Run the skill against a new product idea and it walks through four phases:

1. **Interview** — captures the product: its domain nouns, whether it has long-running/async work, its
   external vendors, billing model, and ICP.
2. **Write the docs first** — generates the five source-of-truth docs from the templates.
   `architecture.md` is the spine; everything else derives from it.
3. **Scaffold the code** — builds the backend, frontend, and infra to match, following an ordered
   checklist and faithful code patterns.
4. **Verify** — the stack boots in **mock mode with no secrets** and backend tests pass **offline**.

## The conventions it reproduces

- **Documentation-driven** — five markdown docs are the source of truth (`architecture.md`,
  `current_progress.md`, `todo.md`, `pricing.md`, `marketing.md`) plus a `CLAUDE.md` pointer and a
  `/kt` knowledge-transfer skill that loads them in order at the start of every task.
- **Thin API, heavy workers** — the API validates/authorizes/enqueues; workers own slow external work.
- **Provider abstraction** — every vendor sits behind an interface; a factory returns a **mock**
  (default, keyless, bootable) or a **real** impl by `PROVIDER_MODE`.
- **Repo pattern** — one resource-typed `Repo`; `SupabaseRepo` in prod, `InMemoryRepo` in tests.
- **Centralized typed config** — one `Settings`; every key has a mock-safe default; prod fails loudly.
- **No-network tests** — autouse fixtures install the in-memory repo, force mock providers, run Celery
  inline; the whole backend is exercisable deterministically, offline.
- **Phased infra** — launch on one cheap box (one image, `ROLE=api|worker|beat`); the design makes the
  split into an autoscaled topology mechanical later.

The skill is **adaptive**: it drops Celery/workers for plain CRUD products, and the credit ledger /
`pricing.md` for non-metered ones, so the skeleton fits each new product.

## Layout

```
SKILL.md                       # the orchestrator — start here
reference/
  stack.md                     # annotated stack + exact dependency versions + directory tree
  patterns.md                  # the 10 reusable code patterns with faithful snippets
  scaffold-checklist.md        # ordered, file-by-file build list
templates/
  architecture.md              # 15-section design template (the spine)
  current_progress.md          # running build-log template
  todo.md                      # product backlog template
  pricing.md                   # cost → credit margin model template
  marketing.md                 # go-to-market (first-N users) template
  kt-skill.md                  # /kt knowledge-transfer skill template (→ .claude/skills/kt/SKILL.md)
```

## Using it

Make the skill discoverable to Claude Code, then invoke it with your product idea:

```bash
# clone it where Claude Code looks for skills
git clone git@github.com:Raphael2908/setup.git ~/.claude/skills/setup
# or add it to a single project
git submodule add git@github.com:Raphael2908/setup.git .claude/skills/setup
```

Then, in a fresh project directory:

```
/setup a marketplace for <…>   # or just describe the product and reference "the setup skill"
```

Claude reads `SKILL.md`, interviews you, writes the docs, and scaffolds the code.
