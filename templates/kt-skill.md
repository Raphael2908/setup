---
name: kt
description: >-
  Knowledge transfer for <PRODUCT>. Read this BEFORE starting any new task, feature, change,
  fix, or plan in this repository. Loads the design source of truth (architecture.md), the
  current build state (current_progress.md), and the backlog (todo.md) in the required order,
  and points to pricing.md, marketing.md, and the CLAUDE.md conventions when a task calls for
  them. Triggers on beginning work, implementing, debugging, refactoring, or planning changes
  to <PRODUCT>.
---

# kt — knowledge transfer for <PRODUCT>

> **Template note — delete this block when adapting.** This file becomes
> `.claude/skills/kt/SKILL.md` in the new project. Replace every `<PRODUCT>`; regenerate the §-index
> in section 3 from the *actual* `architecture.md` you wrote (drop sections that were deleted);
> trim the conditional table in section 2 to the docs the product actually has (no `pricing.md` if
> not metered). Keep the file short — it's a reading protocol, not a summary of the docs.

Run this at the **start of any new task** in this repo so you inherit the design intent and the
current build state before you touch code. It auto-loads from its `description`; you can also
invoke it as `/kt`.

This is a **context-priming skill, not an app-run skill** — there is no driver script, no
screenshot, and no app to launch from here. The "harness" is the reading protocol below.

All paths are relative to the repo root.

## 1. Read these first, in this order (non-negotiable)

1. **`architecture.md`** — the design source of truth. Everything (data model, pipeline, API,
   infra) derives from it.
2. **`current_progress.md`** — the running build log; where the code actually is versus the design.
3. **`todo.md`** — the prioritized product backlog and why each item is there.

**Governing rule (from `CLAUDE.md`):** every design decision in code traces back to a section of
`architecture.md`. **If code and that document disagree, the document wins — or the document gets
amended in the same change.** Never let them drift silently.

## 2. Then, when the task calls for it

| If the task touches… | Also read |
|---|---|
| credits, billing, vendor cost, margins, the per-step credit map | `pricing.md` *(if metered)* |
| landing page, positioning, funnel, ICP, launch channels | `marketing.md` |
| build / run / deploy, `make` targets, compose services | `README.md` + `Makefile` |
| "how do we do X here" conventions (provider abstraction, repo pattern, mock mode, no-network tests) | the conventions/gotchas section of `CLAUDE.md` |
| a specific subsystem | jump straight to its `architecture.md` §section (index below) |

## 3. architecture.md section index

Navigate to the relevant section instead of re-reading the whole document on every follow-up.
*(Regenerate this list from the product's actual `architecture.md`.)*

- **§1** Goal & Scope (phasing: MVP vs Phase 2)
- **§2** System Overview (topology diagram + design principles)
- **§3** Tech Stack (+ model/vendor defaults)
- **§4** Infrastructure & Deployment (Phase 0 single box → scale-up trigger)
- **§5** Data Model (tables, RLS, indexes)
- **§6** Generation / Work Pipeline (state machine, step contracts) *(if long-running work)*
- **§7** Async Orchestration (Celery + Redis) *(if async)*
- **§8** Storage & CDN *(if files)*
- **§9** Auth (Supabase JWT via JWKS)
- **§10** Billing & Credits (Stripe, ledger) *(if metered)*
- **§11** API Design (surface + conventions)
- **§12** Frontend (screens, apiClient)
- **§13** Observability, Security, Rate Limiting
- **§14** Cost Model *(if metered)*
- **§15** Open Questions

## 4. After reading

Proceed with the task. At the **end of any substantial change**, update `current_progress.md`
(and `todo.md` / `pricing.md` when applicable) — keeping the docs and the code in step is part of
the change, not a follow-up.
