# ExpenseFlow — Production Readiness Gap Audit

Audited against a production bar across ten areas. Each entry states the gap as it actually exists in the code today, classifies it as **Blocking** (must close before real, unsupervised production traffic) or **Deferrable** (can ship without it and close later, with a stated risk), and gives a rough effort estimate. Effort estimates assume one engineer already familiar with this codebase; they exclude any legal/compliance review time, which is called out separately where relevant.

This complements `REVIEW.md` (code-level findings) and `docs/HANDOFF.md` (operational notes) — some gaps are named in both places; this document exists to size and prioritize them for a go/no-go decision, not to re-litigate them.

---

## 1. Authentication and key rotation

**Gap:** No authentication or authorization exists on any endpoint (`app/routes.py`). `submitted_by` and `decided_by` are unauthenticated free-text strings — any caller can submit, approve, or reject any expense and self-report as anyone. Separately, the one real secret in the system (`ANTHROPIC_API_KEY`) is read once via `load_dotenv()` at process start (`app/insights.py`); rotating it requires a process restart, and there is no documented rotation procedure. The key currently sitting in this repo's `.env` is a live, unrotated credential.

**Classification:** **Blocking.** A financial approval workflow with no access control has no security meaning — "approved" doesn't mean anything if anyone can claim to be the approver. Rotating the currently-exposed key is also an immediate, blocking action item, independent of code changes.

**Effort estimate:**
- Minimal gate (shared API key / bearer token, no per-user identity): 0.5–1 day.
- Real authentication with role separation (submitter vs. approver, since the approve/reject step is meaningless without knowing who's allowed to decide): 3–5 days, including a user/identity model and wiring it through `decided_by`/`submitted_by`.
- Key rotation runbook + restart-on-rotate automation: 0.5 day.

---

## 2. Input validation

**Gap:** `ExpenseCreate.description` (`app/schemas.py`) has no length or emptiness constraint — empty strings and arbitrarily large payloads are accepted and stored. `currency` is validated only as "3 alphabetic characters" (`currency_must_be_three_letters`), not against a real ISO 4217 list, so nonsense codes like `"ZZZ"` pass. `category`, `submitted_by`, and `decided_by` are unconstrained free text with no length caps. `amount_minor` has a lower bound (`gt=0`) but no upper bound. There is no request body size limit configured anywhere (FastAPI/Starlette default has none).

**Classification:** Mixed.
- Missing length caps on free-text fields and no request size limit: **Blocking** — cheap to fix, and this is the easiest lever for a single caller to bloat the database or send outsized payloads.
- Real ISO 4217 currency validation: **Deferrable** — a data-quality issue, not a stability or security one, given conversion isn't implemented yet anyway (see `docs/HANDOFF.md`).

**Effort estimate:** 2–4 hours — add `Field(max_length=...)` constraints on free-text fields, add a static ISO 4217 code set and validate against it, add a body size limit, cover with tests.

---

## 3. Rate limiting

**Gap:** No rate limiting exists anywhere — no middleware, no per-IP or per-caller throttling on any endpoint. `GET /reports/insights` is the sharpest edge: it is unauthenticated (see §1) and each call triggers up to two real, billed Anthropic API calls (`MAX_RETRIES = 1` in `app/insights.py`). An unthrottled caller can drive real, unbounded API spend with no limit.

**Classification:** Mixed.
- Rate limiting on `/reports/insights` specifically: **Blocking** — direct, uncapped financial exposure per request with no auth gate in front of it.
- General per-endpoint rate limiting on the CRUD endpoints: **Deferrable** — can ship with basic limits and tune later, once there's an authenticated caller identity to key the limiter on.

**Effort estimate:**
- In-memory limiter (single instance) via a Starlette middleware or a library like `slowapi`: 0.5 day.
- Shared-store limiter (Redis-backed) for multi-instance deployment: add 1–1.5 days, including standing up the Redis dependency.

---

## 4. Observability and logging

**Gap:** The only logging in the codebase is inside `app/insights.py`, and only on the failure path. `create_expense`, `approve_expense`, `list_expenses`, and `reject_expense` (`app/routes.py`) have no logging at all — no record of who submitted, approved, or rejected what, beyond what's already in the database rows themselves. There are no metrics (request counts, latencies, error rates), no correlation/request IDs, no structured (JSON) logging for aggregation, and nothing wired to an alerting system.

**Classification:** **Blocking.** Operating a financial approval workflow with zero visibility into its core actions — separate from the data itself — makes incident response and audit effectively impossible.

**Effort estimate:**
- Structured logging (request-scoped, JSON) added to all five endpoints: ~1 day.
- Basic metrics endpoint (e.g. `prometheus-fastapi-instrumentator`) exposing request counts/latencies: 0.5 day.
- Full distributed tracing (OpenTelemetry): **Deferrable** — 2–3 days, only worth it once there's more than one service to trace across.

---

## 5. Error handling

**Gap:** `generate_insight`'s retry loop (`app/insights.py`) catches bare `Exception`, so a network timeout and a genuine programming bug are logged and swallowed identically — already noted in `REVIEW.md`. Separately, and more consequentially: `create_expense`, `approve_expense`, and `reject_expense` call `db.commit()` with no surrounding `try`/`except` — a database error mid-transaction (lock timeout, disk full, constraint violation) propagates as an unhandled exception, and the session is closed (via `get_db()`'s `finally`) without an explicit `rollback()` first. There is also no global FastAPI exception handler, so every unhandled error falls through to FastAPI's default response shape with no consistent error contract and no logging of the failure.

**Classification:** **Blocking** for the core routes (no rollback discipline, no global handler, no logging on failure — this is the money workflow). The bare-`except Exception` in `insights.py` is **Deferrable** — it's an isolated, already-fallback-guarded path, not the core workflow.

**Effort estimate:** ~1 day — add a global exception handler for a consistent error response shape and logging, and wrap each mutating route's commit in try/except with explicit rollback.

---

## 6. Database migrations and pooling

**Gap:** There is no migration tool. `app/db.py`'s `init_db()` runs `Base.metadata.create_all()` (which only creates missing tables, never alters existing ones) followed by `_add_missing_columns()` — a hand-written check for exactly one column (`category`) that runs a raw `ALTER TABLE` if it's absent. This doesn't generalize: every future schema change needs a new hand-written check in that same function, there's no migration history or version tracking, and there's no rollback path if a migration is wrong. Separately, the datastore itself is SQLite, a single-file, single-writer database; `connect_args={"check_same_thread": False}` only works around SQLAlchemy's same-thread check for FastAPI's threaded request handling — it does not provide real connection pooling or safe concurrent writes the way a client-server database does.

**Classification:** **Blocking.** Two independent blockers: (a) there's no safe way to evolve the schema in production without manual, unversioned intervention, and (b) SQLite is a genuine ceiling on concurrent access for a multi-user approval workflow — not a configuration tweak away from production-ready.

**Effort estimate:**
- Introduce Alembic for versioned migrations against the existing SQLite setup: ~1 day to set up, plus ongoing discipline per change.
- Migrate to a client-server database (Postgres): 2–3 days — schema port, connection pool configuration, provisioning the database, and a one-time data migration script from the existing `expenseflow.db`.

---

## 7. Secrets management

**Gap:** `.env` currently holds a real, live `ANTHROPIC_API_KEY` in plaintext. There is no `.gitignore` anywhere in the repository, so nothing stops that file (or `expenseflow.db`, which holds real submitted data) from being committed by a future `git add .` / `git add -A`. There is no integration with any secrets manager (Vault, AWS/GCP Secrets Manager, etc.) — `python-dotenv` reading a flat file is the entire mechanism — and no distinction between dev/staging/prod secrets.

**Classification:** **Blocking.** There is a real, unrotated secret sitting unprotected in the project right now.

**Effort estimate:**
- Add `.gitignore`, rotate the exposed key: under an hour.
- Wire up a real secrets manager for the target deployment platform (fetch at startup instead of reading `.env`): 0.5–1 day, platform-dependent.

---

## 8. Tests and coverage

**Gap:** `python -m pytest -q` is the configured test command (`CLAUDE.md`, `README.md`), but there are zero test files anywhere in this repository. No coverage tooling is configured. There is no CI pipeline (no `.github/workflows`, no equivalent) running anything on a commit or PR.

**Classification:** **Blocking**, unambiguously — there is no automated safety net at all for a financial approval workflow.

**Effort estimate:**
- Baseline test suite covering all six endpoints (creation validation, approve/reject transitions including the 409 case, 404s, insights fallback behavior with the Anthropic client mocked) using FastAPI's `TestClient` against a throwaway test database: 2–3 days for solid coverage of the core workflow.
- Wire up CI to run that suite on every PR: 0.5 day.

---

## 9. Deployment and health checks

**Gap:** There is no Dockerfile, process manager configuration (systemd unit, Procfile), or documented production ASGI serving setup — `README.md` documents `uvicorn app.main:app --reload`, and `--reload` is explicitly a development-only flag. There is no dedicated `/health` or `/ready` endpoint; the closest proxy is `GET /expenses`, which touches the database and returns real data as a side effect of a liveness check. There's no graceful-shutdown handling beyond FastAPI/Starlette's defaults (no explicit `engine.dispose()` on shutdown), and no per-environment configuration — `DATABASE_URL` is a single hardcoded value (`app/db.py`).

**Classification:** **Blocking** — there is currently no defined way to run this as a service (only as a foreground dev command) and no way for an orchestrator to know if it's healthy.

**Effort estimate:**
- Add `/health` (liveness) and `/ready` (readiness, including a DB ping) endpoints: 2–3 hours.
- Containerize (Dockerfile, plus a deployment manifest or compose file): 0.5–1 day.
- Production ASGI server configuration (workers, graceful shutdown): 0.5 day — note this is capped at a single worker until §6's SQLite limitation is resolved.

---

## 10. Data privacy for expense data

**Gap:** `GET /reports/insights` (`app/insights.py`, `app/routes.py`) sends every stored expense's amount, category, and status to a third-party API (Anthropic) with no data minimization, no anonymization, and no opt-out — this happens on every call, unconditionally. There is no data retention or deletion policy — rows persist indefinitely with no archival or purge mechanism, and no delete endpoint exists at all. `expenseflow.db` is an unencrypted flat file; anyone with filesystem access to it can read every expense, including free-text `submitted_by`/`decided_by` fields that may contain names. There is no data classification anywhere in the code or docs marking which fields are sensitive.

**Classification:** Mixed.
- Sending expense data to a third party with no opt-out or minimization: **Blocking** if any of this data is genuinely sensitive or regulated — at minimum this needs an explicit, documented decision before going live, which doesn't exist today.
- Retention/deletion policy and encryption at rest: **Deferrable** for an initial launch, provided the hosting platform's disk encryption covers the at-rest case — but should be tracked, not forgotten.

**Effort estimate:**
- Add a config flag to disable `/reports/insights` entirely for privacy-sensitive deployments, and document the data flow it creates: 0.5 day.
- Retention/purge job: ~1 day.
- Full data classification and regulatory scope review: not an engineering estimate — requires legal/compliance input before it can be sized.
