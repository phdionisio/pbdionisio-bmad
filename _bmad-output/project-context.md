---
project: pbdionisio
generated: 2026-05-22
sections_completed: [stack, structure, backend-patterns, db-patterns, testing-patterns, frontend-patterns, config-patterns]
---

# Project Context — pbdionisio

LLM-optimized reference for AI agents implementing code in this project. Focus is on non-obvious rules and conventions that prevent implementation mistakes.

---

## Technology Stack

| Layer | Technology | Version |
|---|---|---|
| Backend language | Python | ≥ 3.9 |
| Backend framework | FastAPI | ≥ 0.115 |
| Settings | pydantic-settings | ≥ 2.4 |
| ORM | SQLAlchemy | (via `pbdionisio-db`) |
| Migrations | Alembic | (via `pbdionisio-db`) |
| Database (prod) | PostgreSQL | — |
| Database (test) | SQLite | — |
| Server | uvicorn | ≥ 0.30 |
| Frontend framework | React + TypeScript | — |
| Frontend build | Vite | — |
| Frontend router | React Router | — |

---

## Repository Structure

```
pbdionisio/
├── apps/
│   ├── pbdionisio-api/          # FastAPI backend
│   │   ├── src/pbdionisio_api/
│   │   │   ├── config.py        # pydantic-settings config
│   │   │   ├── dependencies.py  # FastAPI dependency providers (@lru_cache)
│   │   │   ├── main.py          # App factory, middleware, router inclusion
│   │   │   ├── routers/         # One file per feature domain
│   │   │   ├── services/        # Business logic layer
│   │   │   ├── repositories/    # Abstract + SQL implementations
│   │   │   ├── schemas/         # Pydantic request/response models
│   │   │   └── seeders/         # Dev/test data seeders
│   │   └── tests/               # API-level tests (TestClient + stubs)
│   ├── pbdionisio-internal-web/ # React + TypeScript frontend
│   │   └── src/
│   │       ├── App.tsx          # Route definitions
│   │       ├── components/      # Reusable components
│   │       │   └── auth/        # Auth guards (RequireAuth)
│   │       ├── lib/             # Shared API helpers
│   │       └── pages/           # Pages grouped by section
│   │           ├── Tools/       # Internal tools pages
│   │           └── AuthPages/   # Sign-in, sign-up
│   └── docker-compose.dev.yml
└── libs/
    ├── activity-log/            # Activity event lib (source of truth for event model)
    ├── pbdionisio-db/           # SQLAlchemy Base, engine, session, Alembic migrations
    └── google-auth/             # Google OAuth helpers
```

---

## Backend Implementation Rules

### Universal File Header
Every Python file (source and test) **must** begin with:
```python
from __future__ import annotations
```
This is non-negotiable — the codebase uses `str | None` union syntax that requires this on Python 3.9.

### Feature Layer Pattern
Every new feature domain follows exactly this stack:

```
routers/<domain>.py       # APIRouter — HTTP boundary only
services/<domain>_service.py   # Business logic, no DB knowledge
repositories/<domain>_repository.py   # Abstract interface (ABC or Protocol)
repositories/sql_<domain>_repository.py  # SQLAlchemy implementation
schemas/<domain>.py       # Pydantic request + response models
```

Do not skip layers. Routers call services; services call repositories. Services do not touch SQLAlchemy directly.

### Router Registration
Routers are registered in `main.py` with `app.include_router(..., prefix="/api/v1")`. The router's own `APIRouter` sets the sub-prefix and tags:
```python
router = APIRouter(prefix="/<domain>", tags=["<domain>"])
```

### Dependency Injection
All singleton dependencies (services, clients) are created in `dependencies.py` using `@lru_cache(maxsize=1)`. FastAPI routes receive them via `Depends()`. Never instantiate services inside routers directly.

### Pagination Convention
List endpoints return results with:
- `limit` (default 25, max 100) and `offset` query params
- `X-Total-Count` response header with the total unfiltered count
- `X-Total-Count` must be exposed in CORS `expose_headers` (already configured in `main.py`)

### Config (API)
`config.py` uses `pydantic_settings.BaseSettings` with `env_prefix="PBDIONISIO_API_"`. All API env vars use this prefix (e.g. `PBDIONISIO_API_ENV`, `PBDIONISIO_API_CORS_ORIGINS`). The DB config uses no prefix and reads `DATABASE_URL` directly.

---

## Database Implementation Rules

### All Models Inherit From `Base`
```python
from pbdionisio_db.base import Base

class MyModel(Base):
    __tablename__ = "my_table"
```
`Base` enforces the global `NAMING_CONVENTION` for constraints and indexes. Never define a `DeclarativeBase` outside of `pbdionisio-db`.

### SQLAlchemy Naming Convention
Constraint and index names follow the convention in `pbdionisio_db.base.NAMING_CONVENTION`:
- Index: `ix_<column_0_label>`
- Unique: `uq_<table_name>_<column_0_name>`
- FK: `fk_<table_name>_<column_0_name>_<referred_table_name>`
- PK: `pk_<table_name>`
- Check: `ck_<table_name>_<constraint_name>`

### Migrations Live in `libs/pbdionisio-db/migrations/versions/`
Filename convention: `YYYYMMDD_NNNN_<description>.py`
Example: `20260430_0001_create_activity_events.py`

Revision IDs match the date prefix: `revision = "20260430_0001"`.

In migration `upgrade()`, always use `op.f()` for auto-named constraints/indexes:
```python
op.create_index(op.f("ix_my_table_col"), "my_table", ["col"], unique=False)
op.create_table(..., sa.PrimaryKeyConstraint("id", name=op.f("pk_my_table")))
```

### Session Factory Pattern
Use `create_session_factory()` from `pbdionisio_db.engine` to get a `sessionmaker`. Repositories receive `session_factory: sessionmaker[Session]` in `__init__`. Sessions are opened with `with self._session_factory() as session:`.

### JSON Columns
Structured data stored in JSON columns should use the `_json` suffix (e.g., `payload_json`, `indexes_json`) to distinguish from the Pydantic-facing field name in schemas.

---

## Testing Rules

### API Tests: Stub Services, No DB
All API-level tests in `apps/pbdionisio-api/tests/` use `TestClient` with dependency overrides. Services are replaced with stub classes — no database, no real sessions. Pattern:
```python
app.dependency_overrides[get_my_service] = lambda: stub_service
with TestClient(app) as client:
    ...
app.dependency_overrides.clear()
```

Always clear overrides after the test (use a fixture with `yield`).

### Library Tests: SQLite
Shared library tests (in `libs/`) may use real SQLite databases for integration coverage. PostgreSQL tests run when `DATABASE_URL` env var points to Postgres.

### Test File Naming
`test_<subject>.py` in `tests/` directory alongside the package being tested. Fixtures go in `conftest.py`.

### Pytest Config
`pyproject.toml` sets `testpaths = ["tests"]`. Run from the package root.

---

## Frontend Implementation Rules

### Route Registration
New pages are added as `<Route>` entries in `src/App.tsx` inside the `RequireAuth`-wrapped `AppLayout` route group. All internal tool pages require authentication — never add pages outside the `RequireAuth` wrapper unless they are public auth pages.

### Page Location Convention
Pages are organized by section under `src/pages/<Section>/PageName.tsx` using PascalCase. Tools live in `src/pages/Tools/`. New pages follow this structure.

### API Calls
The Vite dev server proxies `/api/*` to the backend (configured in `vite.config.ts`). Use relative `/api/v1/...` paths in fetch calls — no hardcoded base URLs.

### Auth
Session auth is cookie-based, managed by the backend (`pbdionisio_session` cookie). The `RequireAuth` component calls `/api/v1/auth/google/me` to check auth state. Auth state is not stored in localStorage or React context — it is always server-authoritative.

---

## Key Business Domain Facts

### Application Types
The system processes firearms-related government applications:
- **LTOPF** — License to Own and Possess Firearm (New / Renewal)
- **FA Registration** — Firearm Registration (New / Renewal / Transfer)
- **Airgun Registration**
- **PTT** — Permit to Transport

### Branches
- **Roces** (main) — generates 6 separate summary reports by application type
- **Crame** — generates 1 unified report (Representation, Others, Piso Piso)
- **Davao** — separate department, tracked in same reports as Davao entries

### Staff Assignment by Category
| Category | Staff |
|---|---|
| LTOPF (New/Renewal) | Jo |
| FA Registration (New) | Ellen, Carla |
| FA Registration (Renewal) | Paul |
| Airgun Registration | Paul |

### Payment Methods
- **Landbank online** — Tax/ID (PTT), PTT fee
- **Cash** — Notary, Drug Test/Neuro, GSS, Courier, Representation, Others
- **FEO cash** — up to PHP 20,000 for PTC, IT, FEO matters

### Signatories (Current)
- **Prepared by:** assigned staff member
- **Noted / Endorsed by:** Cherry S. Bencito (Processing Operation Head)
- **Validated by:** Cel (Finance/Acctg — currently uses Excel/Access)
- **Received by:** Accounting Department / Finance Department
