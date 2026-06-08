# The reusable patterns

These are the conventions that make the stack work. They transfer **verbatim** to a new product — only
the resource names, provider list, and pipeline steps change. Snippets below are generalized from the
real HexUGC files; reproduce them faithfully. (If the HexUGC repo is reachable, read the real files for
the exact current form.)

---

## 1. Centralized typed config (`app/config.py`)

One `Settings` class (pydantic-settings). **Every** external key lives here with a mock-safe blank
default so the whole stack boots with no secrets. A `model_validator` makes production fail loudly if
core creds are blank. Access via a cached `get_settings()` / module-level `settings`.

```python
from __future__ import annotations
from functools import lru_cache
from typing import Literal
from pydantic import model_validator
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")

    # --- Runtime ---
    ENV: Literal["development", "production"] = "development"
    ROLE: Literal["api", "worker", "beat"] = "api"
    PROVIDER_MODE: Literal["mock", "real"] = "mock"
    LOG_LEVEL: str = "info"

    # --- API / CORS ---
    API_HOST: str = "0.0.0.0"
    API_PORT: int = 8000
    CORS_ORIGINS: str = "http://localhost:3000,http://localhost"

    # --- Redis / Supabase / S3 / Stripe / vendors --- (all blank-default)
    REDIS_URL: str = "redis://redis:6379/0"
    SUPABASE_URL: str = ""
    SUPABASE_PUBLISHABLE_KEY: str = ""   # client (sb_publishable_…)
    SUPABASE_SECRET_KEY: str = ""        # server (sb_secret_…)
    # ... AWS_*, S3_BUCKET, STRIPE_*, RESEND_*, and each vendor key + model id ...
    # ... guards: MAX_INFLIGHT, DAILY_*_CAP, RATE_LIMIT_*, VENDOR_CONCURRENCY_* ...
    # ... reliability: TASK_MAX_RETRIES, *_TIMEOUT_S ...

    @property
    def cors_origins_list(self) -> list[str]:
        return [o.strip() for o in self.CORS_ORIGINS.split(",") if o.strip()]

    @model_validator(mode="after")
    def _require_core_in_production(self) -> "Settings":
        if self.ENV == "production":
            missing = [n for n in ("SUPABASE_URL", "SUPABASE_SECRET_KEY") if not getattr(self, n)]
            if missing:
                raise ValueError("Missing required production settings: " + ", ".join(missing))
        return self

@lru_cache
def get_settings() -> Settings:
    return Settings()

settings = get_settings()
```

Document each vendor key inline (what it's for, what a blank value does — usually "feature returns 501 /
mock mode"). Put model-id defaults here too so they're env-overridable.

---

## 2. Provider abstraction (`app/providers/`)

Only for products with external vendors. Orchestration depends **only** on the interfaces, so swapping
mock↔real is a drop-in. Three files:

**`base.py`** — interfaces + Result dataclasses + a 3-class error hierarchy (the retry policy hinges on
the distinction between retryable and fatal):

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

class ProviderError(Exception): ...
class RetryableError(ProviderError): ...   # 429 / 5xx / timeout → safe to retry
class FatalError(ProviderError): ...        # 4xx validation / policy → fail fast

@dataclass
class WidgetResult:                          # one per capability; carry provider_job_id for async vendors
    output_key: str
    provider_job_id: str | None = None

class WidgetProvider(ABC):
    @abstractmethod
    def do_thing(self, *, ...) -> WidgetResult: ...
```

**`mock.py`** — deterministic canned implementations of every interface. No network. This is what tests
and a keyless local stack run on.

**`factory.py`** — `get_<x>_provider()` returns the mock when `PROVIDER_MODE == "mock"`, else **lazily
imports** the real impl (so the API tier never pulls vendor SDKs it won't use). Each real impl raises
loudly at construction if its key is missing — no silent fallback.

```python
def get_widget_provider() -> WidgetProvider:
    if settings.PROVIDER_MODE == "mock":
        return mock.MockWidgetProvider()
    from app.providers.real.acme_widget import AcmeWidgetProvider
    return AcmeWidgetProvider()
```

Rule: **when adding a vendor capability, change the provider behind the interface — never the orchestration.**

---

## 3. Repo pattern (`app/db/repo.py`)

All data access goes through one resource-typed `Repo` ABC. Prod = `SupabaseRepo` (service-role client,
RLS bypassed, ownership checked in handlers). Tests = `InMemoryRepo` (dict-backed, no network).
`get_*` methods return `None` on a miss; handlers map that to 404.

```python
from abc import ABC, abstractmethod

class Repo(ABC):
    @abstractmethod
    def get_widget(self, widget_id: str) -> dict | None: ...
    @abstractmethod
    def list_widgets(self, user_id: str) -> list[dict]: ...
    @abstractmethod
    def create_widget(self, *, user_id: str, **fields) -> dict: ...
    # ... one set of methods per resource; + jobs/ledger/stripe_events if applicable ...

class SupabaseRepo(Repo):
    def __init__(self): self._sb = get_supabase()
    # ... implement against the Supabase client ...

class InMemoryRepo(Repo):
    def __init__(self): self._widgets: dict[str, dict] = {}
    # ... implement against dicts; generate uuids; stamp utcnow ...

_repo: Repo | None = None
def set_repo(r: Repo | None) -> None:
    global _repo; _repo = r
def get_repo() -> Repo:
    global _repo
    if _repo is None: _repo = SupabaseRepo()   # lazy in prod; raises if DB unconfigured
    return _repo
```

Handlers and tasks call `get_repo()`; they never import the Supabase SDK directly.

---

## 4. Auth via JWKS (`app/core/auth.py`)

Verify the user's Supabase access token against the project's JWKS endpoint (asymmetric keys, no shared
secret). Cache the JWKS client. Yield a `CurrentUser(id, email)`. This is **real logic** even in the
scaffold — the stub-ness lives in handlers, not auth.

```python
import jwt
from fastapi import Depends, Header, HTTPException, status

_ALGORITHMS = ["ES256", "RS256"]
def _issuer() -> str: return f"{settings.SUPABASE_URL.rstrip('/')}/auth/v1"

@lru_cache
def _jwk_client() -> jwt.PyJWKClient:
    return jwt.PyJWKClient(f"{_issuer()}/.well-known/jwks.json")

async def get_current_user(authorization: str | None = Header(default=None)) -> CurrentUser:
    if not authorization or not authorization.lower().startswith("bearer "):
        raise HTTPException(401, "Missing bearer token.")
    claims = jwt.decode(
        authorization.split(" ", 1)[1],
        _jwk_client().get_signing_key_from_jwt(...).key,
        algorithms=_ALGORITHMS, audience="authenticated", issuer=_issuer(),
    )
    return CurrentUser(id=claims["sub"], email=claims.get("email"))
```

(Handle missing `SUPABASE_URL` → 503, JWKS unreachable → 503, bad token → 401, as in the real file.)

---

## 5. One image, ROLE dispatch (`entrypoint.sh`)

The same backend image is api, worker, or beat depending on `ROLE`. This is what makes the Phase-0 →
scale-up split mechanical.

```bash
#!/usr/bin/env bash
set -euo pipefail
ROLE="${ROLE:-api}"
case "$ROLE" in
  api)    exec uvicorn app.main:app --host "${API_HOST:-0.0.0.0}" --port "${API_PORT:-8000}" ;;
  worker) exec celery -A app.celery_app.celery_app worker --loglevel="${LOG_LEVEL:-info}" \
            -Q q.stage1,q.stage2,q.stage3 ;;
  beat)   exec celery -A app.celery_app.celery_app beat --loglevel="${LOG_LEVEL:-info}" ;;
  *) echo "Unknown ROLE: $ROLE" >&2; exit 1 ;;
esac
```

---

## 6. Celery app — per-stage queues, timeouts, watchdog (`app/celery_app.py`)

Only if there's async work. One queue per long stage so each scales/rate-limits independently. `acks_late`
+ `prefetch_multiplier=1` for reliability. Per-task soft (raises inside task) / hard (SIGKILL backstop)
time limits. A beat watchdog fails jobs stuck `running` past their SLA.

```python
celery_app = Celery("<project>", broker=settings.REDIS_URL, backend=settings.REDIS_URL,
                    include=["app.tasks.stage1", "app.tasks.stage2", "app.tasks.watchdog"])
celery_app.conf.task_routes = {"tasks.stage1": {"queue": "q.stage1"}, ...}
celery_app.conf.update(task_acks_late=True, worker_prefetch_multiplier=1,
                       task_track_started=True, result_expires=3600, timezone="UTC")
celery_app.conf.task_annotations = {"tasks.stage1": {"soft_time_limit": 240, "time_limit": 300}, ...}
celery_app.conf.beat_schedule = {"watchdog-sweep": {"task": "tasks.watchdog_sweep", "schedule": 60.0}}
```

**The pipeline is a Celery chain** (`services/pipeline.py`): each task reads its inputs from the DB by
the parent id rather than passing big payloads through the broker, so any in-order *subset* of steps
chains cleanly. Each step is one `jobs` row driving a `queued→running→succeeded|failed` state machine.
For multi-minute vendor calls, submit → store `provider_job_id` → self-reschedule with backoff to poll
(don't block the worker slot). Distinguish retryable vs fatal; refund on fatal/cancel.

---

## 7. No-network tests (`app/tests/conftest.py`)

Autouse fixtures make the whole backend exercisable offline: install `InMemoryRepo`, force mock
providers (default), run Celery inline. `client` overrides auth to a fixed user; `seed` pre-populates a
ready resource + (if metered) a credit grant.

```python
@pytest.fixture(autouse=True)
def in_memory_repo():
    repo = InMemoryRepo(); repo_module.set_repo(repo); yield repo; repo_module.set_repo(None)

@pytest.fixture(autouse=True)
def eager_celery():
    celery_app.conf.task_always_eager = True
    celery_app.conf.task_eager_propagates = False
    celery_app.loader.import_default_modules(); yield
    celery_app.conf.task_always_eager = False

@pytest.fixture
def client():
    from app.main import app
    app.dependency_overrides[get_current_user] = lambda: CurrentUser(id="u1", email="u@x.test")
    yield TestClient(app); app.dependency_overrides.pop(get_current_user, None)
```

---

## 8. Append-only credit ledger (`app/services/credits.py`) — if metered

A `credit_ledger` table is the source of truth for balance: rows are `(user_id, delta, reason, ref_id,
balance_after)`. **Reserve** on enqueue (negative delta), **refund** on failure/cancel. Stripe webhooks
are the **only** path that grants credits, idempotent via a `stripe_events(event_id)` table. The credit
map (per-step costs) is calibrated in code from `VENDOR_COST_USD` at a target margin (see `pricing.md`).
The API checks balance before enqueue; insufficient → 402 with the shortfall.

---

## 9. Frontend API client (`frontend/lib/apiClient.ts`)

Every call goes through one wrapper that attaches the Supabase JWT and surfaces 503 (maintenance/deploy
drain) as a friendly message. Typed endpoint wrappers live in `lib/api.ts`.

```ts
const BASE = process.env.NEXT_PUBLIC_API_BASE_URL ?? "/api";
export async function apiFetch<T = unknown>(path: string, init: RequestInit = {}): Promise<T> {
  const { data: { session } } = await supabase.auth.getSession();
  const headers = new Headers(init.headers);
  headers.set("Content-Type", "application/json");
  if (session?.access_token) headers.set("Authorization", `Bearer ${session.access_token}`);
  const res = await fetch(`${BASE}${path}`, { ...init, headers });
  if (!res.ok) {
    const body = await res.text();
    if (res.status === 503) throw new Error("New update incoming — please try again shortly.");
    throw new Error(`API ${res.status}: ${body}`);
  }
  return res.json() as Promise<T>;
}
```

Authed pages live under `app/(app)/` behind an `AuthGate` that checks the Supabase session.
`NEXT_PUBLIC_*` vars are baked into the bundle **at build time** — compose passes them as build args.

---

## 10. Gotchas to carry over

- **Tests run with no network** — keep every external call behind a provider/repo so mock mode is total.
- **`NEXT_PUBLIC_*` are build-time** — set before `docker compose build`, passed as build args.
- **Request correlation** — mint/propagate an `X-Request-ID` in API middleware, thread it API → enqueue
  → worker → job rows via a `ContextVar` (`core/context.py`) for log correlation.
- **Migrations** numbered `0001…` in `infra/supabase/migrations/`; keep manual rollbacks *outside* that
  dir so a `db push` never applies them.
- **Graceful deploy** — pause new work via a Redis maintenance flag, drain in-flight jobs, rebuild,
  health-gate. Worker `stop_grace_period` must exceed the longest hard task timeout.
- **Backend style** — ruff (line length 100, `E/F/I/UP/B`), `from __future__ import annotations`, 3.12.
