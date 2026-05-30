# Trade Compliance Platform — Phased Development Plan

> Project: 215-trade-compliance-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | Python 3.12+ | LLM-heavy classification workflows rely on the Python AI/ML ecosystem (LangChain, OpenAI SDK, sentence-transformers). Python also has mature libraries for XML parsing (OFAC lists), fuzzy string matching (thefuzz, rapidfuzz), and data pipelines. |
| API framework | FastAPI 0.115+ | Automatic OpenAPI 3.1 generation (required by spec), async support for concurrent screening, Pydantic v2 for request/response validation, and dependency-injection for tenant/auth context. |
| Database | PostgreSQL 16+ | Hybrid relational + JSONB model (Data Model Suggestion 3) provides normalized core with flexible JSONB for jurisdiction-varying fields. PostgreSQL GIN indexes on JSONB, Row-Level Security for multi-tenancy, and full-text search for name matching. |
| ORM / query builder | SQLAlchemy 2.0 + Alembic | SQLAlchemy 2.0's typed ORM with Alembic migrations. Native JSONB column support and PostgreSQL dialect features. |
| Task queue | Celery 5 + Redis | Async workloads: batch screening, watchlist ingestion, regulatory digest generation, continuous re-screening. Redis as broker and result backend. |
| Caching | Redis 7 | Screening result caching, rate limiting, session storage, and Celery broker. Single Redis instance serves multiple purposes for MVP. |
| Search / matching | PostgreSQL pg_trgm + rapidfuzz | pg_trgm trigram indexes for database-level fuzzy name search; rapidfuzz for in-memory scoring during screening. Avoids Elasticsearch dependency for MVP. |
| LLM integration | OpenAI API (GPT-4o) via openai SDK | ECCN/HS classification assistance, regulatory digest generation, licence determination reasoning. Abstracted behind a provider interface for future multi-provider support. |
| Frontend | Next.js 15 (React 19) + Tailwind CSS + shadcn/ui | Dashboard-heavy compliance UI with server components for initial load performance, client components for interactive screening queues. TypeScript throughout. |
| Authentication | JWT + OAuth 2.0 (via python-jose + authlib) | API key auth for programmatic access, JWT for web sessions, OAuth 2.0/OIDC for enterprise SSO integration. |
| Containerisation | Docker + Docker Compose | Self-hosted deployment target. Compose for local development with PostgreSQL, Redis, and the application services. |
| Testing | pytest + pytest-asyncio + httpx | pytest for unit/integration tests, httpx for async API testing against FastAPI's TestClient, factory_boy for test fixtures. |
| Frontend testing | Vitest + Playwright | Vitest for component unit tests, Playwright for E2E browser tests against the compliance dashboard. |
| Code quality | ruff (lint + format) + mypy (strict) | ruff replaces flake8+isort+black in one tool. mypy in strict mode catches type errors in Pydantic models and SQLAlchemy queries. |
| Package manager | uv | Fast Python package management with lockfile support. Replaces pip + pip-tools. |
| API documentation | Auto-generated OpenAPI 3.1 via FastAPI | OpenAPI spec published at /openapi.json and interactive docs at /docs (Swagger UI). Aligns with OpenAPI 3.x standard requirement. |

### Project Structure

```
trade-compliance-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/
├── src/
│   └── tcp/                              # trade-compliance-platform package
│       ├── __init__.py
│       ├── main.py                       # FastAPI app entrypoint
│       ├── config.py                     # Pydantic Settings (env-based config)
│       ├── database.py                   # SQLAlchemy engine + session factory
│       ├── auth/
│       │   ├── __init__.py
│       │   ├── dependencies.py           # FastAPI auth dependencies
│       │   ├── jwt.py                    # JWT token creation/validation
│       │   ├── models.py                 # User, Role, Tenant ORM models
│       │   └── router.py                 # /auth endpoints
│       ├── parties/
│       │   ├── __init__.py
│       │   ├── models.py                 # Party ORM model
│       │   ├── schemas.py                # Pydantic request/response schemas
│       │   ├── service.py                # Business logic
│       │   └── router.py                 # /parties endpoints
│       ├── watchlists/
│       │   ├── __init__.py
│       │   ├── models.py                 # WatchlistSource, WatchlistEntry ORM
│       │   ├── schemas.py
│       │   ├── ingestors/
│       │   │   ├── __init__.py
│       │   │   ├── base.py               # Abstract ingestor interface
│       │   │   ├── ofac.py               # OFAC SDN XML parser
│       │   │   ├── bis.py                # BIS Entity List parser
│       │   │   ├── eu.py                 # EU Consolidated List parser
│       │   │   └── un.py                 # UN sanctions list parser
│       │   ├── service.py
│       │   └── router.py
│       ├── screening/
│       │   ├── __init__.py
│       │   ├── models.py                 # Screening ORM model
│       │   ├── schemas.py
│       │   ├── matchers/
│       │   │   ├── __init__.py
│       │   │   ├── fuzzy.py              # Fuzzy name matching (rapidfuzz)
│       │   │   ├── phonetic.py           # Phonetic matching (metaphone/soundex)
│       │   │   ├── identifier.py         # Exact identifier matching
│       │   │   └── composite.py          # Multi-algorithm scorer
│       │   ├── service.py
│       │   ├── tasks.py                  # Celery tasks (batch, continuous)
│       │   └── router.py
│       ├── classification/
│       │   ├── __init__.py
│       │   ├── models.py
│       │   ├── schemas.py
│       │   ├── ai/
│       │   │   ├── __init__.py
│       │   │   ├── provider.py           # LLM provider abstraction
│       │   │   ├── eccn_classifier.py    # ECCN classification prompts + logic
│       │   │   ├── hs_classifier.py      # HS code classification
│       │   │   └── prompts.py            # Prompt templates
│       │   ├── reference_data.py         # HS code + ECCN reference table loaders
│       │   ├── service.py
│       │   └── router.py
│       ├── licensing/
│       │   ├── __init__.py
│       │   ├── models.py
│       │   ├── schemas.py
│       │   ├── determination.py          # Licence requirement logic
│       │   ├── service.py
│       │   └── router.py
│       ├── regulatory/
│       │   ├── __init__.py
│       │   ├── models.py                 # EmbargoRule, RegulatoryUpdate
│       │   ├── schemas.py
│       │   ├── embargo.py                # Embargo checking logic
│       │   ├── digest.py                 # AI regulatory digest generator
│       │   └── router.py
│       ├── transactions/
│       │   ├── __init__.py
│       │   ├── models.py
│       │   ├── schemas.py
│       │   ├── compliance_check.py       # Orchestrates screening + licence + embargo checks
│       │   ├── service.py
│       │   └── router.py
│       ├── audit/
│       │   ├── __init__.py
│       │   ├── models.py
│       │   ├── middleware.py             # Auto-audit middleware
│       │   └── router.py
│       └── common/
│           ├── __init__.py
│           ├── pagination.py             # Cursor/offset pagination
│           ├── exceptions.py             # Domain exceptions
│           ├── tenant.py                 # Tenant context + RLS
│           └── types.py                  # Shared type aliases
├── tests/
│   ├── conftest.py                       # Shared fixtures, test DB setup
│   ├── factories.py                      # factory_boy model factories
│   ├── fixtures/
│   │   ├── ofac_sdn_sample.xml           # Sample OFAC XML for testing
│   │   ├── bis_entity_list_sample.csv    # Sample BIS data
│   │   ├── eu_consolidated_sample.xml    # Sample EU list data
│   │   └── sample_products.json          # Test product data
│   ├── unit/
│   │   ├── test_fuzzy_matcher.py
│   │   ├── test_phonetic_matcher.py
│   │   ├── test_eccn_classifier.py
│   │   ├── test_hs_classifier.py
│   │   ├── test_licence_determination.py
│   │   └── test_embargo_check.py
│   ├── integration/
│   │   ├── test_screening_api.py
│   │   ├── test_classification_api.py
│   │   ├── test_party_api.py
│   │   ├── test_watchlist_ingestion.py
│   │   └── test_audit_trail.py
│   └── e2e/
│       ├── test_screening_workflow.py
│       └── test_classification_workflow.py
└── frontend/                             # Next.js application (Phase 8)
    ├── package.json
    ├── tsconfig.json
    ├── next.config.ts
    ├── src/
    │   ├── app/
    │   │   ├── layout.tsx
    │   │   ├── page.tsx
    │   │   ├── dashboard/
    │   │   ├── screening/
    │   │   ├── classification/
    │   │   ├── licensing/
    │   │   └── settings/
    │   ├── components/
    │   ├── lib/
    │   │   └── api-client.ts             # Generated from OpenAPI spec
    │   └── hooks/
    ├── tests/
    │   └── e2e/
    └── playwright.config.ts
```

---

## Phase 1: Project Foundation and Core Infrastructure

### Purpose
Establish the project skeleton, build tooling, database connectivity, multi-tenant authentication, and the audit trail. After this phase, the application boots, connects to PostgreSQL/Redis, authenticates users, and logs all state changes. Every subsequent phase builds on this foundation.

### Tasks

#### 1.1 — Project Scaffolding and Build Configuration

**What**: Create the Python project with uv, configure linting/formatting/typing, and set up Docker Compose for local development.

**Design**:

`pyproject.toml`:
```toml
[project]
name = "trade-compliance-platform"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "sqlalchemy[asyncio]>=2.0.36",
    "asyncpg>=0.30.0",
    "alembic>=1.14.0",
    "pydantic>=2.10.0",
    "pydantic-settings>=2.6.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "celery[redis]>=5.4.0",
    "redis>=5.2.0",
    "httpx>=0.28.0",
    "openai>=1.60.0",
    "rapidfuzz>=3.10.0",
    "phonetics>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "httpx>=0.28.0",
    "factory-boy>=3.3.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
    "sqlalchemy-stubs>=0.4",
]
```

`docker-compose.yml`:
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: tcp
      POSTGRES_USER: tcp
      POSTGRES_PASSWORD: tcp_dev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  api:
    build: .
    command: uvicorn tcp.main:app --host 0.0.0.0 --port 8000 --reload
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://tcp:tcp_dev@db:5432/tcp
      REDIS_URL: redis://redis:6379/0
    depends_on: [db, redis]
    volumes: ["./src:/app/src"]
  worker:
    build: .
    command: celery -A tcp.tasks worker -l info
    environment:
      DATABASE_URL: postgresql+asyncpg://tcp:tcp_dev@db:5432/tcp
      REDIS_URL: redis://redis:6379/0
    depends_on: [db, redis]

volumes:
  pgdata:
```

`Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen
COPY src/ ./src/
COPY alembic.ini alembic/ ./
ENV PYTHONPATH=/app/src
```

**Testing**:
- Unit: `pyproject.toml` is valid and `uv sync` resolves all dependencies without conflicts
- Integration: `docker compose up` starts all services; `curl http://localhost:8000/healthz` returns `200 OK`
- Unit: `ruff check src/` and `mypy src/` pass with zero errors on empty project

---

#### 1.2 — Application Configuration

**What**: Pydantic Settings model loading configuration from environment variables with sensible defaults.

**Design**:

```python
# src/tcp/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    model_config = {"env_prefix": "TCP_"}

    # Database
    database_url: str = "postgresql+asyncpg://tcp:tcp_dev@localhost:5432/tcp"
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Auth
    jwt_secret_key: str  # Required — no default
    jwt_algorithm: str = "HS256"
    jwt_access_token_expire_minutes: int = 60
    jwt_refresh_token_expire_days: int = 30

    # LLM
    openai_api_key: str = ""
    openai_model: str = "gpt-4o"
    openai_max_tokens: int = 4096

    # Screening
    screening_default_threshold: float = 0.80
    screening_auto_clear_below: float = 0.30
    screening_auto_escalate_above: float = 0.95

    # Watchlist updates
    watchlist_update_schedule_hours: int = 24

    # Application
    app_name: str = "Trade Compliance Platform"
    app_version: str = "0.1.0"
    debug: bool = False
    cors_origins: list[str] = ["http://localhost:3000"]
```

Environment variables: `TCP_DATABASE_URL`, `TCP_JWT_SECRET_KEY`, `TCP_OPENAI_API_KEY`, etc.

**Testing**:
- Unit: Settings loads with all env vars set → all fields populated
- Unit: Settings with missing `TCP_JWT_SECRET_KEY` → `ValidationError` with clear field name
- Unit: Settings with only required vars → defaults applied correctly for all optional fields

---

#### 1.3 — Database Setup and Core Schema Migration

**What**: SQLAlchemy 2.0 async engine, session factory, and Alembic migration for tenant, user, role tables with Row-Level Security policies.

**Design**:

```python
# src/tcp/database.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from tcp.config import Settings

def create_engine(settings: Settings):
    return create_async_engine(
        settings.database_url,
        pool_size=settings.database_pool_size,
        max_overflow=settings.database_max_overflow,
    )

def create_session_factory(engine) -> async_sessionmaker[AsyncSession]:
    return async_sessionmaker(engine, expire_on_commit=False)
```

ORM models (from Data Model Suggestion 3 — hybrid relational + JSONB):

```python
# src/tcp/auth/models.py
from __future__ import annotations
import uuid
from datetime import datetime
from sqlalchemy import String, Boolean, ARRAY, Text
from sqlalchemy.dialects.postgresql import UUID, JSONB, TIMESTAMPTZ
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class Tenant(Base):
    __tablename__ = "tenant"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    slug: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    jurisdictions: Mapped[list[str]] = mapped_column(ARRAY(String(3)), default=list)
    config: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow, onupdate=datetime.utcnow)

    users: Mapped[list[User]] = relationship(back_populates="tenant")

class User(Base):
    __tablename__ = "user"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    email: Mapped[str] = mapped_column(String(255), nullable=False)
    display_name: Mapped[str] = mapped_column(String(255), nullable=False)
    password_hash: Mapped[str | None] = mapped_column(String(255))
    roles: Mapped[list[str]] = mapped_column(ARRAY(Text), default=list)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    last_login_at: Mapped[datetime | None] = mapped_column(TIMESTAMPTZ)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow, onupdate=datetime.utcnow)

    tenant: Mapped[Tenant] = relationship(back_populates="users")
```

Row-Level Security policy (in Alembic migration raw SQL):
```sql
ALTER TABLE "user" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "user"
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

**Testing**:
- Integration: Alembic `upgrade head` succeeds on clean database; `downgrade base` succeeds
- Integration: Tenant and User records insert and query correctly via SQLAlchemy async session
- Unit: RLS policy blocks cross-tenant user queries when `app.current_tenant_id` is set

---

#### 1.4 — Authentication and Authorisation

**What**: JWT-based authentication with API key support, role-based access control, and FastAPI dependency injection for current user/tenant context.

**Design**:

```python
# src/tcp/auth/jwt.py
from datetime import datetime, timedelta
from jose import jwt, JWTError
from tcp.config import Settings

def create_access_token(user_id: str, tenant_id: str, roles: list[str], settings: Settings) -> str:
    payload = {
        "sub": user_id,
        "tenant_id": tenant_id,
        "roles": roles,
        "exp": datetime.utcnow() + timedelta(minutes=settings.jwt_access_token_expire_minutes),
        "type": "access",
    }
    return jwt.encode(payload, settings.jwt_secret_key, algorithm=settings.jwt_algorithm)

def decode_token(token: str, settings: Settings) -> dict:
    """Raises JWTError on invalid/expired token."""
    return jwt.decode(token, settings.jwt_secret_key, algorithms=[settings.jwt_algorithm])
```

```python
# src/tcp/auth/dependencies.py
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, APIKeyHeader

bearer_scheme = HTTPBearer()
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def get_current_user(
    token: str = Security(bearer_scheme),
    session: AsyncSession = Depends(get_session),
) -> User:
    """Extracts user from JWT. Sets tenant context on DB session."""
    payload = decode_token(token.credentials, settings)
    user = await session.get(User, payload["sub"])
    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="User inactive or not found")
    # Set RLS context
    await session.execute(text(f"SET LOCAL app.current_tenant_id = '{user.tenant_id}'"))
    return user

def require_role(*roles: str):
    """Dependency factory for role-based access control."""
    async def checker(user: User = Depends(get_current_user)):
        if not any(r in user.roles for r in roles):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return checker
```

API endpoints:
- `POST /auth/login` — email + password → access + refresh tokens
- `POST /auth/refresh` — refresh token → new access token
- `POST /auth/api-keys` — create API key (admin role required)
- `GET /auth/me` — current user profile

**Testing**:
- Unit: `create_access_token` → valid JWT with correct claims (sub, tenant_id, roles, exp)
- Unit: `decode_token` with expired token → `JWTError`
- Unit: `decode_token` with tampered token → `JWTError`
- Integration: `POST /auth/login` with valid credentials → 200, returns access + refresh tokens
- Integration: `POST /auth/login` with wrong password → 401
- Integration: `GET /auth/me` with valid token → 200, user profile returned
- Integration: `GET /auth/me` with expired token → 401
- Integration: Endpoint with `require_role("admin")` called by non-admin → 403

---

#### 1.5 — Audit Trail Middleware

**What**: Automatic audit logging that captures all state changes across all entities with old/new values, user context, and timestamps.

**Design**:

```python
# src/tcp/audit/models.py
class AuditLog(Base):
    __tablename__ = "audit_log"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    user_id: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("user.id"))
    action: Mapped[str] = mapped_column(String(100), nullable=False)  # 'created', 'updated', 'deleted'
    entity_type: Mapped[str] = mapped_column(String(100), nullable=False)
    entity_id: Mapped[uuid.UUID] = mapped_column(UUID, nullable=False)
    old_values: Mapped[dict | None] = mapped_column(JSONB)
    new_values: Mapped[dict | None] = mapped_column(JSONB)
    ip_address: Mapped[str | None] = mapped_column(String(45))
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
```

```python
# src/tcp/audit/middleware.py
from sqlalchemy import event

def audit_after_insert(mapper, connection, target):
    """SQLAlchemy event listener: log INSERT as 'created' audit entry."""
    ...

def audit_after_update(mapper, connection, target):
    """SQLAlchemy event listener: log UPDATE with old/new JSONB diff."""
    ...

def register_audit_listeners(model_class: type):
    """Register audit event listeners on an ORM model class."""
    event.listen(model_class, "after_insert", audit_after_insert)
    event.listen(model_class, "after_update", audit_after_update)
```

API endpoints:
- `GET /audit?entity_type=screening&entity_id=...` — query audit log with filters
- `GET /audit?start_date=2026-01-01&end_date=2026-03-31` — date-range audit report

**Testing**:
- Integration: Create a User → audit_log contains one row with action='created', entity_type='user', new_values with user fields
- Integration: Update a User's `display_name` → audit_log contains old_values.display_name and new_values.display_name
- Integration: `GET /audit?entity_type=user&entity_id=<id>` → returns ordered list of audit entries
- Unit: Audit entries include ip_address and user_id from request context

---

#### 1.6 — Health Check and OpenAPI Base

**What**: Health check endpoint, CORS configuration, and base FastAPI application with OpenAPI metadata.

**Design**:

```python
# src/tcp/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from tcp.config import Settings

settings = Settings()

app = FastAPI(
    title="Trade Compliance Platform",
    version=settings.app_version,
    description="AI-native trade compliance: denied party screening, export classification, and licence management.",
    openapi_url="/openapi.json",
    docs_url="/docs",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/healthz", tags=["system"])
async def healthz():
    return {"status": "ok", "version": settings.app_version}

@app.get("/readyz", tags=["system"])
async def readyz(session: AsyncSession = Depends(get_session)):
    """Checks database and Redis connectivity."""
    await session.execute(text("SELECT 1"))
    # Check Redis
    redis = Redis.from_url(settings.redis_url)
    await redis.ping()
    return {"status": "ready", "database": "ok", "redis": "ok"}
```

**Testing**:
- Integration: `GET /healthz` → 200 with `{"status": "ok"}`
- Integration: `GET /readyz` with DB up → 200 with all services "ok"
- Integration: `GET /readyz` with DB down → 503
- Integration: `GET /openapi.json` → valid OpenAPI 3.1 document
- Integration: CORS preflight from allowed origin → 200 with correct headers
- Integration: CORS preflight from disallowed origin → no CORS headers

---

## Phase 2: Party Management and Watchlist Ingestion

### Purpose
Build the counterparty data model and the watchlist ingestion pipeline. After this phase, the platform can store internal counterparties (customers, suppliers, end-users) and ingest government sanctions lists (OFAC SDN, BIS Entity List, EU Consolidated List, UN sanctions) into a searchable watchlist database. This is the prerequisite for screening.

### Tasks

#### 2.1 — Party CRUD API

**What**: Full REST API for managing internal counterparties with aliases, addresses, identifiers, and flexible properties.

**Design**:

```python
# src/tcp/parties/models.py
class Party(Base):
    __tablename__ = "party"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    party_type: Mapped[str] = mapped_column(String(50), nullable=False)  # 'individual', 'organisation', 'vessel', 'aircraft'
    primary_name: Mapped[str] = mapped_column(String(500), nullable=False)
    country_code: Mapped[str | None] = mapped_column(String(3))  # ISO 3166-1 alpha-3
    aliases: Mapped[list] = mapped_column(JSONB, default=list)
    addresses: Mapped[list] = mapped_column(JSONB, default=list)
    identifiers: Mapped[list] = mapped_column(JSONB, default=list)
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    risk_score: Mapped[float | None] = mapped_column(Numeric(5, 2))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow, onupdate=datetime.utcnow)
```

```python
# src/tcp/parties/schemas.py
from pydantic import BaseModel, Field
from typing import Literal

class AliasSchema(BaseModel):
    type: Literal["aka", "fka", "dba", "original_script"]
    name: str = Field(max_length=500)
    script: str | None = Field(None, max_length=10)  # ISO 15924

class AddressSchema(BaseModel):
    type: Literal["registered", "operating", "shipping"]
    line1: str | None = None
    line2: str | None = None
    city: str | None = None
    state_province: str | None = None
    postal_code: str | None = None
    country_code: str = Field(max_length=3)  # ISO 3166-1 alpha-3
    is_primary: bool = False

class IdentifierSchema(BaseModel):
    type: str  # 'lei', 'duns', 'vat', 'passport', 'imo_number', 'mmsi'
    value: str
    issuing_country: str | None = None

class PartyCreate(BaseModel):
    party_type: Literal["individual", "organisation", "vessel", "aircraft"]
    primary_name: str = Field(max_length=500)
    country_code: str | None = Field(None, max_length=3)
    aliases: list[AliasSchema] = []
    addresses: list[AddressSchema] = []
    identifiers: list[IdentifierSchema] = []
    properties: dict = {}

class PartyResponse(PartyCreate):
    id: uuid.UUID
    risk_score: float | None = None
    is_active: bool
    created_at: datetime
    updated_at: datetime
```

API endpoints:
- `POST /parties` — create party
- `GET /parties` — list parties (paginated, filterable by type, country, name search)
- `GET /parties/{id}` — get party detail
- `PUT /parties/{id}` — update party
- `DELETE /parties/{id}` — soft-delete (set is_active=false)
- `GET /parties/{id}/screenings` — list screening history for party

**Testing**:
- Integration: `POST /parties` with valid individual → 201, party created with UUID returned
- Integration: `POST /parties` with valid organisation including aliases, addresses, identifiers → 201, all nested data persisted
- Integration: `GET /parties?party_type=organisation&country_code=DEU` → filtered results
- Integration: `GET /parties?search=acme` → fuzzy name search returns matching parties
- Integration: `PUT /parties/{id}` adding new alias → 200, alias appended
- Integration: `DELETE /parties/{id}` → 200, party marked inactive; `GET /parties/{id}` still returns it with `is_active=false`
- Integration: Tenant A cannot see Tenant B's parties (RLS isolation)
- Unit: `PartyCreate` with `party_type="invalid"` → `ValidationError`
- Unit: `PartyCreate` with `country_code="XXXX"` (4 chars) → `ValidationError`

---

#### 2.2 — Watchlist Source and Entry Models

**What**: Database models for watchlist sources (OFAC, BIS, EU, UN) and their entries, following the hybrid relational+JSONB pattern.

**Design**:

```python
# src/tcp/watchlists/models.py
class WatchlistSource(Base):
    __tablename__ = "watchlist_source"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    code: Mapped[str] = mapped_column(String(50), unique=True, nullable=False)  # 'OFAC_SDN', 'BIS_EL', etc.
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    administering_body: Mapped[str] = mapped_column(String(255), nullable=False)
    country_code: Mapped[str | None] = mapped_column(String(3))
    url: Mapped[str | None] = mapped_column(Text)
    schema_mapping: Mapped[dict] = mapped_column(JSONB, default=dict)
    last_fetched_at: Mapped[datetime | None] = mapped_column(TIMESTAMPTZ)
    entry_count: Mapped[int] = mapped_column(Integer, default=0)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)

class WatchlistEntry(Base):
    __tablename__ = "watchlist_entry"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    source_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("watchlist_source.id"), nullable=False)
    source_uid: Mapped[str] = mapped_column(String(255), nullable=False)
    entry_type: Mapped[str] = mapped_column(String(50), nullable=False)  # 'individual', 'organisation', 'vessel', 'aircraft'
    primary_name: Mapped[str] = mapped_column(String(500), nullable=False)
    programme: Mapped[str | None] = mapped_column(String(255))
    listing_date: Mapped[date | None] = mapped_column(Date)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    aliases: Mapped[list] = mapped_column(JSONB, default=list)
    addresses: Mapped[list] = mapped_column(JSONB, default=list)
    identifiers: Mapped[list] = mapped_column(JSONB, default=list)
    source_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow, onupdate=datetime.utcnow)

    __table_args__ = (UniqueConstraint("source_id", "source_uid"),)
```

Indexes (in Alembic migration):
```sql
CREATE INDEX idx_watchlist_entry_name ON watchlist_entry(primary_name);
CREATE INDEX idx_watchlist_entry_active ON watchlist_entry(is_active) WHERE is_active = true;
CREATE INDEX idx_watchlist_aliases ON watchlist_entry USING GIN(aliases jsonb_path_ops);
CREATE INDEX idx_watchlist_identifiers ON watchlist_entry USING GIN(identifiers jsonb_path_ops);
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_watchlist_name_trgm ON watchlist_entry USING GIN(primary_name gin_trgm_ops);
```

**Testing**:
- Integration: Alembic migration creates watchlist tables with all indexes
- Integration: Insert WatchlistSource + WatchlistEntry → queryable; unique constraint enforced on (source_id, source_uid)
- Integration: pg_trgm index supports `SELECT ... WHERE primary_name % 'search_term'` with similarity threshold

---

#### 2.3 — OFAC SDN List Ingestor

**What**: Parse the OFAC SDN XML file (Advanced Sanctions List Standard format) and upsert entries into the watchlist database.

**Design**:

```python
# src/tcp/watchlists/ingestors/base.py
from abc import ABC, abstractmethod

class WatchlistIngestor(ABC):
    """Abstract base class for all watchlist ingestors."""

    @abstractmethod
    async def fetch(self) -> bytes:
        """Download the raw watchlist data."""
        ...

    @abstractmethod
    async def parse(self, raw_data: bytes) -> list[WatchlistEntryCreate]:
        """Parse raw data into normalized WatchlistEntry schemas."""
        ...

    @abstractmethod
    async def ingest(self, session: AsyncSession) -> IngestResult:
        """Full pipeline: fetch → parse → upsert. Returns counts."""
        ...

class IngestResult(BaseModel):
    source_code: str
    entries_added: int
    entries_updated: int
    entries_deactivated: int
    total_active: int
    duration_seconds: float
```

```python
# src/tcp/watchlists/ingestors/ofac.py
import xml.etree.ElementTree as ET

class OFACSDNIngestor(WatchlistIngestor):
    SOURCE_CODE = "OFAC_SDN"
    SDN_XML_URL = "https://www.treasury.gov/ofac/downloads/sanctions/1.0/sdn_advanced.xml"

    async def fetch(self) -> bytes:
        async with httpx.AsyncClient() as client:
            resp = await client.get(self.SDN_XML_URL, timeout=120)
            resp.raise_for_status()
            return resp.content

    async def parse(self, raw_data: bytes) -> list[WatchlistEntryCreate]:
        """Parse OFAC Advanced Sanctions List Standard XML.
        
        Maps:
        - <sdnEntry> → WatchlistEntry with source_uid from <uid>
        - <aka> elements → aliases array with quality attribute
        - <address> elements → addresses array
        - <id> elements → identifiers array with type and country
        - <programme> → programme field
        - Vessel/aircraft info → source_data JSONB
        """
        root = ET.fromstring(raw_data)
        ns = {"sdn": "https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/ADVANCED_XML"}
        entries = []
        for entry_elem in root.findall(".//sdn:sdnEntry", ns):
            uid = entry_elem.find("sdn:uid", ns).text
            # Parse name parts, aliases, addresses, identifiers, programme, vessel info
            # Map OFAC entry type to normalized party_type
            ...
            entries.append(WatchlistEntryCreate(...))
        return entries

    async def ingest(self, session: AsyncSession) -> IngestResult:
        raw = await self.fetch()
        entries = await self.parse(raw)
        # Upsert: INSERT ON CONFLICT (source_id, source_uid) DO UPDATE
        # Deactivate entries present in DB but not in current file
        ...
```

**Testing**:
- Unit: `parse()` with `tests/fixtures/ofac_sdn_sample.xml` (50-entry extract) → correctly extracts names, aliases, addresses, identifiers, programmes
- Unit: OFAC entry with type "Individual" → `party_type="individual"`; type "Entity" → `party_type="organisation"`; type "Vessel" → `party_type="vessel"` with vessel info in `source_data`
- Unit: Alias quality attribute preserved ("strong", "weak", "low")
- Integration: `ingest()` on empty database → all entries inserted; `IngestResult.entries_added == len(entries)`
- Integration: `ingest()` called twice with same data → zero entries_added, zero entries_updated (idempotent)
- Integration: `ingest()` with entry removed from XML → that entry's `is_active` set to false; `entries_deactivated == 1`
- Integration (real, optional): Full OFAC SDN XML download and parse completes within 60 seconds

---

#### 2.4 — BIS Entity List Ingestor

**What**: Parse the BIS Entity List (CSV format) and upsert entries.

**Design**:

The BIS Entity List is published as a CSV with columns: Name, Aliases, Addresses, Federal Register Citation, Licence Requirement, Licence Policy, Country. The ingestor downloads from `https://www.bis.doc.gov/index.php/documents/consolidated-entity-list/` and parses the CSV.

```python
# src/tcp/watchlists/ingestors/bis.py
import csv
import io

class BISEntityListIngestor(WatchlistIngestor):
    SOURCE_CODE = "BIS_EL"

    async def parse(self, raw_data: bytes) -> list[WatchlistEntryCreate]:
        reader = csv.DictReader(io.StringIO(raw_data.decode("utf-8")))
        entries = []
        for row in reader:
            # Map CSV columns to WatchlistEntry fields
            # Name → primary_name
            # Aliases (semicolon-separated) → aliases JSONB array
            # Country → addresses[0].country_code (map country name to ISO 3166-1 alpha-3)
            # Federal Register Citation → source_data.citation
            # Licence Requirement → source_data.licence_requirement
            # Licence Policy → source_data.licence_policy
            ...
        return entries
```

**Testing**:
- Unit: Parse `tests/fixtures/bis_entity_list_sample.csv` (20 rows) → correct name, country, alias extraction
- Unit: Multi-alias entries (semicolon-separated) correctly split into alias array
- Unit: Country name "China" maps to "CHN", "Russia" maps to "RUS"
- Integration: Full ingest cycle inserts all entries; re-ingest is idempotent

---

#### 2.5 — EU Consolidated List and UN Sanctions Ingestors

**What**: Parse the EU Consolidated List (XML format) and UN Security Council sanctions list.

**Design**:

The EU Consolidated List is published at `https://webgate.ec.europa.eu/fsd/fsf/public/files/xmlFullSanctionsList_1_1/content` in a proprietary XML schema. The UN list is published at `https://scsanctions.un.org/resources/xml/en/consolidated.xml`.

Each ingestor follows the same `WatchlistIngestor` interface. Entity-type mapping, alias extraction, and address parsing are source-specific.

**Testing**:
- Unit: EU XML parser extracts `nameAlias`, `address`, `identification` elements correctly
- Unit: UN XML parser extracts `INDIVIDUAL` and `ENTITY` entries with all listed aliases
- Integration: Both ingestors complete full cycle; entries searchable by name

---

#### 2.6 — Watchlist Update Scheduler

**What**: Celery periodic task that runs all ingestors on a configurable schedule (default: every 24 hours).

**Design**:

```python
# src/tcp/watchlists/tasks.py
from celery import shared_task
from celery.schedules import crontab

@shared_task(name="tcp.watchlists.update_all")
async def update_all_watchlists():
    """Run all registered ingestors and log results."""
    ingestors = [OFACSDNIngestor(), BISEntityListIngestor(), EUConsolidatedIngestor(), UNSanctionsIngestor()]
    results = []
    for ingestor in ingestors:
        try:
            result = await ingestor.ingest(session)
            results.append(result)
        except Exception as e:
            # Log error but continue with remaining sources
            logger.error(f"Watchlist ingestion failed for {ingestor.SOURCE_CODE}: {e}")
    return results
```

Celery beat schedule:
```python
app.conf.beat_schedule = {
    "update-watchlists": {
        "task": "tcp.watchlists.update_all",
        "schedule": crontab(hour="*/24"),
    },
}
```

API endpoints:
- `POST /watchlists/update` — trigger manual watchlist update (admin only)
- `GET /watchlists/sources` — list all watchlist sources with last_fetched_at and entry_count
- `GET /watchlists/entries?source=OFAC_SDN&search=...` — search watchlist entries

**Testing**:
- Integration: `POST /watchlists/update` triggers all ingestors; returns aggregated IngestResult
- Integration: `GET /watchlists/sources` returns all sources with correct entry counts
- Integration: `GET /watchlists/entries?search=iran` returns matching entries across all sources
- Unit: Celery beat schedule is configured with correct interval

---

## Phase 3: Denied Party Screening Engine

### Purpose
Build the core screening engine that matches internal counterparties against watchlist entries using multiple algorithms (fuzzy name, phonetic, exact identifier). After this phase, users can screen parties in real-time or batch mode with configurable match thresholds, review hits, and disposition them. This is the platform's primary table-stakes feature.

### Tasks

#### 3.1 — Fuzzy Name Matcher

**What**: String similarity matcher using multiple algorithms with configurable thresholds.

**Design**:

```python
# src/tcp/screening/matchers/fuzzy.py
from rapidfuzz import fuzz, process
from dataclasses import dataclass

@dataclass
class MatchResult:
    entry_id: uuid.UUID
    entry_name: str
    score: float  # 0.0 to 1.0
    algorithm: str
    matched_field: str  # 'primary_name', 'alias'

class FuzzyNameMatcher:
    """Multi-algorithm fuzzy name matching using rapidfuzz.
    
    Algorithms applied (highest score wins):
    1. Token sort ratio — handles word reordering ("John Smith" vs "Smith, John")
    2. Token set ratio — handles partial matches ("ACME Corp" vs "ACME Corporation International")
    3. Weighted ratio — standard Levenshtein-based similarity
    """

    def __init__(self, threshold: float = 0.80):
        self.threshold = threshold

    async def match(
        self,
        query_name: str,
        entries: list[WatchlistEntry],
    ) -> list[MatchResult]:
        """Match query_name against all entry names and aliases.
        
        Returns matches above threshold, sorted by score descending.
        """
        results = []
        for entry in entries:
            # Check primary name
            names_to_check = [entry.primary_name]
            # Add all alias names
            for alias in entry.aliases:
                names_to_check.append(alias["name"])

            best_score = 0.0
            best_field = "primary_name"
            for i, name in enumerate(names_to_check):
                token_sort = fuzz.token_sort_ratio(query_name, name) / 100.0
                token_set = fuzz.token_set_ratio(query_name, name) / 100.0
                weighted = fuzz.WRatio(query_name, name) / 100.0
                score = max(token_sort, token_set, weighted)
                if score > best_score:
                    best_score = score
                    best_field = "primary_name" if i == 0 else f"alias[{i-1}]"

            if best_score >= self.threshold:
                results.append(MatchResult(
                    entry_id=entry.id,
                    entry_name=entry.primary_name,
                    score=best_score,
                    algorithm="fuzzy_name",
                    matched_field=best_field,
                ))
        return sorted(results, key=lambda r: r.score, reverse=True)
```

**Testing**:
- Unit: "John Smith" vs "SMITH, JOHN" → score >= 0.90 (token sort handles reordering)
- Unit: "ACME Corp" vs "ACME Corporation" → score >= 0.85 (token set handles partial)
- Unit: "ACME Corp" vs "Totally Different Company" → score < 0.30 (below threshold, not returned)
- Unit: Match found via alias → `matched_field` reflects the alias index
- Unit: Threshold 0.95 → fewer matches than threshold 0.70 on same dataset
- Unit: Empty query name → empty results, no error
- Unit: Entry with 10 aliases → all aliases checked, best score returned

---

#### 3.2 — Phonetic Matcher

**What**: Phonetic matching using Double Metaphone and Soundex for transliterated and misspelled names.

**Design**:

```python
# src/tcp/screening/matchers/phonetic.py
from phonetics import dmetaphone

class PhoneticMatcher:
    """Phonetic matching for names that sound similar but are spelled differently.
    
    Use cases:
    - Transliterated Arabic/Cyrillic names: "Muammar Gaddafi" vs "Moammar Qadhafi"
    - Misspellings: "Mohammad" vs "Mohammed" vs "Muhammad"
    - Anglicised variants: "Mikhail" vs "Michael"
    """

    def match(self, query_name: str, entries: list[WatchlistEntry]) -> list[MatchResult]:
        query_codes = self._get_phonetic_codes(query_name)
        results = []
        for entry in entries:
            entry_codes = self._get_phonetic_codes(entry.primary_name)
            if query_codes & entry_codes:  # set intersection
                score = len(query_codes & entry_codes) / max(len(query_codes | entry_codes), 1)
                results.append(MatchResult(
                    entry_id=entry.id,
                    entry_name=entry.primary_name,
                    score=score,
                    algorithm="phonetic",
                    matched_field="primary_name",
                ))
        return results

    def _get_phonetic_codes(self, name: str) -> set[str]:
        """Generate phonetic codes for each word in the name."""
        codes = set()
        for word in name.upper().split():
            primary, secondary = dmetaphone(word)
            if primary: codes.add(primary)
            if secondary: codes.add(secondary)
        return codes
```

**Testing**:
- Unit: "Muammar Gaddafi" vs "Moammar Qadhafi" → phonetic match (shared metaphone codes)
- Unit: "Mohammad" vs "Mohammed" → phonetic match
- Unit: "John Smith" vs "Jane Doe" → no phonetic match
- Unit: Single-word names handled correctly

---

#### 3.3 — Identifier Matcher

**What**: Exact-match on structured identifiers (passport numbers, IMO numbers, VAT IDs, LEIs).

**Design**:

```python
# src/tcp/screening/matchers/identifier.py
class IdentifierMatcher:
    """Exact match on structured identifiers stored in JSONB."""

    async def match(
        self,
        query_identifiers: list[dict],
        session: AsyncSession,
    ) -> list[MatchResult]:
        """Query watchlist_entry.identifiers JSONB for exact identifier matches.
        
        Uses PostgreSQL @> containment operator with GIN index.
        """
        results = []
        for qid in query_identifiers:
            stmt = select(WatchlistEntry).where(
                WatchlistEntry.is_active == True,
                WatchlistEntry.identifiers.op("@>")(
                    cast([{"type": qid["type"], "value": qid["value"]}], JSONB)
                ),
            )
            entries = (await session.execute(stmt)).scalars().all()
            for entry in entries:
                results.append(MatchResult(
                    entry_id=entry.id,
                    entry_name=entry.primary_name,
                    score=1.0,  # exact match
                    algorithm="exact_identifier",
                    matched_field=f"identifier:{qid['type']}",
                ))
        return results
```

**Testing**:
- Integration: Party with passport "AB1234567" matched against watchlist entry with same passport → score 1.0
- Integration: Party with IMO number → matched against vessel entry with same IMO
- Integration: No identifier match → empty results
- Integration: GIN index used (EXPLAIN shows index scan, not seq scan)

---

#### 3.4 — Composite Screening Service

**What**: Orchestrates all matchers, deduplicates results, and produces a final scored screening result.

**Design**:

```python
# src/tcp/screening/matchers/composite.py
class CompositeScreener:
    """Combines fuzzy, phonetic, and identifier matchers.
    
    Scoring strategy:
    - Exact identifier match: score = 1.0 (always a confirmed match candidate)
    - Fuzzy name match: score from rapidfuzz (0.0-1.0)
    - Phonetic match: boosts fuzzy score by 0.05 if both match
    - Multiple algorithm matches on same entry: take max score, note all algorithms
    """

    async def screen(
        self,
        query: ScreeningQuery,
        sources: list[str],
        session: AsyncSession,
        settings: Settings,
    ) -> list[ScreeningHit]:
        # 1. Load active watchlist entries for requested sources
        entries = await self._load_entries(sources, session)

        # 2. Run all matchers in parallel
        fuzzy_results = await self.fuzzy_matcher.match(query.name, entries)
        phonetic_results = self.phonetic_matcher.match(query.name, entries)
        id_results = await self.identifier_matcher.match(query.identifiers, session) if query.identifiers else []

        # 3. Merge and deduplicate by entry_id
        merged = self._merge_results(fuzzy_results, phonetic_results, id_results)

        # 4. Apply auto-disposition based on thresholds
        for hit in merged:
            if hit.score < settings.screening_auto_clear_below:
                hit.auto_disposition = "auto_cleared"
            elif hit.score >= settings.screening_auto_escalate_above:
                hit.auto_disposition = "auto_escalated"

        return merged
```

```python
# src/tcp/screening/schemas.py
class ScreeningQuery(BaseModel):
    name: str = Field(max_length=500)
    country_code: str | None = Field(None, max_length=3)
    party_type: Literal["individual", "organisation", "vessel", "aircraft"] | None = None
    identifiers: list[IdentifierSchema] | None = None

class ScreeningHitResponse(BaseModel):
    entry_id: uuid.UUID
    source_code: str
    entry_name: str
    programme: str | None
    match_score: float
    algorithms: list[str]
    matched_fields: dict
    status: str  # 'potential_match', 'confirmed_match', 'false_positive', 'auto_cleared', 'auto_escalated'
    reviewed_by: uuid.UUID | None = None
    reviewed_at: datetime | None = None
    disposition_note: str | None = None
```

**Testing**:
- Integration: Screen "ACME Corp" against watchlist containing "ACME Corporation" → hit with fuzzy score >= 0.85
- Integration: Screen with matching passport number → hit with score 1.0, algorithm "exact_identifier"
- Integration: Screen with both name and identifier match → single hit with max score, both algorithms listed
- Integration: Screen with no matches → empty hits, status "clear"
- Unit: Hit with score 0.25 and auto_clear_below=0.30 → auto_disposition="auto_cleared"
- Unit: Hit with score 0.97 and auto_escalate_above=0.95 → auto_disposition="auto_escalated"

---

#### 3.5 — Screening API

**What**: REST API for real-time and batch screening with hit disposition workflow.

**Design**:

```python
# src/tcp/screening/models.py
class Screening(Base):
    __tablename__ = "screening"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    party_id: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("party.id"))
    mode: Mapped[str] = mapped_column(String(20), nullable=False)  # 'realtime', 'batch', 'continuous'
    query: Mapped[dict] = mapped_column(JSONB, nullable=False)
    sources_checked: Mapped[list[str]] = mapped_column(ARRAY(String(50)), nullable=False)
    status: Mapped[str] = mapped_column(String(50), nullable=False, default="pending")
    hit_count: Mapped[int] = mapped_column(Integer, default=0)
    hits: Mapped[list] = mapped_column(JSONB, default=list)
    overall_disposition: Mapped[str | None] = mapped_column(String(50))
    disposition_note: Mapped[str | None] = mapped_column(Text)
    requested_by: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("user.id"), nullable=False)
    requested_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
    completed_at: Mapped[datetime | None] = mapped_column(TIMESTAMPTZ)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
```

API endpoints:
- `POST /screenings` — real-time screening (synchronous response)
- `POST /screenings/batch` — batch screening of multiple parties (async via Celery, returns job ID)
- `GET /screenings/{id}` — get screening result with all hits
- `GET /screenings?status=potential_match` — list screenings needing review
- `PUT /screenings/{id}/hits/{hit_index}/dispose` — set hit disposition (false_positive, confirmed_match, escalated) with note
- `GET /screenings/stats` — screening statistics (total, pending, cleared, matched by source)

Hit disposition request:
```python
class HitDisposition(BaseModel):
    status: Literal["false_positive", "confirmed_match", "escalated"]
    note: str = Field(min_length=10, max_length=2000)  # Regulatory requirement: must document reasoning
```

When all hits are disposed, the screening `overall_disposition` is automatically set:
- All hits false_positive → "cleared"
- Any hit confirmed_match → "blocked"
- Any hit escalated (none confirmed) → "escalated"

**Testing**:
- Integration: `POST /screenings` with a name matching OFAC entry → 200, hits returned with scores
- Integration: `POST /screenings` with no matches → 200, status="clear", hits=[]
- Integration: `POST /screenings/batch` with 100 parties → 202, job_id returned; poll until complete
- Integration: `PUT /screenings/{id}/hits/0/dispose` with status="false_positive" → hit updated; when all hits disposed → overall_disposition set
- Integration: Disposition note shorter than 10 chars → 422 validation error
- Integration: `GET /screenings?status=potential_match` returns only pending screenings
- Integration: Screening creates audit_log entry
- Integration: Batch screening handles 1000 parties within 60 seconds (with sample watchlist of 5000 entries)

---

## Phase 4: AI-Assisted Export Classification

### Purpose
Build the ECCN and HS code classification engine powered by LLM-assisted reasoning. After this phase, users can submit product descriptions and receive AI-generated classification suggestions with explainable rationale and cited regulatory sources. This is the platform's key AI-native differentiator.

### Tasks

#### 4.1 — Reference Data Loader (HS Codes and ECCNs)

**What**: Load and index the HS code hierarchy and Commerce Control List ECCNs into the database for lookup and classification context.

**Design**:

```python
# src/tcp/classification/reference_data.py
from tcp.classification.models import HSCode, ECCN

async def load_hs_codes(session: AsyncSession, data_path: str):
    """Load HS code hierarchy from WCO-published data.
    
    Each HS code has: code (up to 10 digits), chapter (2), heading (4),
    subheading (6), description, edition ('HS2022').
    """
    ...

async def load_eccns(session: AsyncSession, data_path: str):
    """Load Commerce Control List ECCNs.
    
    Each ECCN has: code (5 chars, e.g., '3A001'), category (0-9),
    product_group (A-E), control_reason, description, licence_requirements.
    """
    ...
```

```python
# src/tcp/classification/models.py
class HSCode(Base):
    __tablename__ = "ref_hs_code"
    code: Mapped[str] = mapped_column(String(10), primary_key=True)
    chapter: Mapped[str] = mapped_column(String(2), nullable=False)
    heading: Mapped[str] = mapped_column(String(4), nullable=False)
    subheading: Mapped[str] = mapped_column(String(6), nullable=False)
    description: Mapped[str] = mapped_column(Text, nullable=False)
    unit_of_measure: Mapped[str | None] = mapped_column(String(50))
    edition: Mapped[str] = mapped_column(String(10), default="HS2022")

class ECCN(Base):
    __tablename__ = "ref_eccn"
    code: Mapped[str] = mapped_column(String(5), primary_key=True)
    category: Mapped[str] = mapped_column(String(1), nullable=False)
    product_group: Mapped[str] = mapped_column(String(1), nullable=False)
    control_reason: Mapped[str | None] = mapped_column(String(100))
    description: Mapped[str] = mapped_column(Text, nullable=False)
    ccl_heading: Mapped[str | None] = mapped_column(Text)
    licence_requirements: Mapped[str | None] = mapped_column(Text)
    related_usml: Mapped[str | None] = mapped_column(String(50))
```

**Testing**:
- Integration: Load HS codes → all chapters 01-99 present; total count matches published HS2022 dataset
- Integration: Load ECCNs → categories 0-9 and product groups A-E represented; total matches published CCL
- Integration: `SELECT * FROM ref_hs_code WHERE code = '8471300000'` → "Portable automatic data-processing machines..."
- Integration: `SELECT * FROM ref_eccn WHERE code = '3A001'` → correct description and control reason

---

#### 4.2 — LLM Provider Abstraction

**What**: Provider-agnostic interface for LLM calls with structured output, token tracking, and error handling.

**Design**:

```python
# src/tcp/classification/ai/provider.py
from abc import ABC, abstractmethod
from pydantic import BaseModel

class ClassificationSuggestion(BaseModel):
    code: str                    # The suggested HS or ECCN code
    confidence: float            # 0.0 to 1.0
    rationale: str               # Step-by-step reasoning
    cited_sources: list[str]     # Regulatory references (e.g., "EAR §742.6(a)(1)")
    alternative_codes: list[dict[str, str]]  # [{code, reason}] for alternatives considered

class LLMProvider(ABC):
    @abstractmethod
    async def classify(
        self,
        product_description: str,
        classification_type: str,  # 'HS' or 'ECCN'
        context: str,              # Relevant regulatory text
    ) -> ClassificationSuggestion:
        ...

class OpenAIProvider(LLMProvider):
    def __init__(self, api_key: str, model: str = "gpt-4o"):
        self.client = AsyncOpenAI(api_key=api_key)
        self.model = model

    async def classify(self, product_description, classification_type, context):
        response = await self.client.chat.completions.create(
            model=self.model,
            response_format={"type": "json_object"},
            messages=[
                {"role": "system", "content": self._system_prompt(classification_type)},
                {"role": "user", "content": self._user_prompt(product_description, context)},
            ],
            temperature=0.1,  # Low temperature for consistency
            max_tokens=4096,
        )
        return ClassificationSuggestion.model_validate_json(response.choices[0].message.content)
```

**Testing**:
- Unit (mocked): OpenAIProvider.classify with mocked API response → ClassificationSuggestion with all fields populated
- Unit (mocked): API error (rate limit, timeout) → appropriate exception raised, not swallowed
- Unit: System prompt contains clear instructions for structured JSON output with rationale
- Integration (optional, real API): Classify "laptop computer" as HS → returns code starting with "8471"

---

#### 4.3 — ECCN Classification Engine

**What**: AI-assisted ECCN classification with CCL context retrieval and explainable output.

**Design**:

```python
# src/tcp/classification/ai/eccn_classifier.py
class ECCNClassifier:
    """ECCN classification using LLM with CCL context.
    
    Workflow:
    1. Receive product description + technical specs
    2. Retrieve relevant CCL categories based on keyword matching
    3. Build context prompt with relevant ECCN descriptions
    4. Call LLM for classification with structured output
    5. Return classification with rationale and cited EAR sections
    """

    async def classify(
        self,
        product: ProductClassifyRequest,
        session: AsyncSession,
    ) -> ClassificationSuggestion:
        # Step 1: Retrieve candidate ECCNs from reference data
        candidates = await self._retrieve_candidate_eccns(product.description, session)

        # Step 2: Build context with CCL descriptions
        context = self._build_ccl_context(candidates)

        # Step 3: Call LLM
        suggestion = await self.llm_provider.classify(
            product_description=self._format_product(product),
            classification_type="ECCN",
            context=context,
        )

        return suggestion

    async def _retrieve_candidate_eccns(self, description: str, session: AsyncSession) -> list[ECCN]:
        """Retrieve ECCNs whose descriptions are most relevant to the product.
        
        Uses PostgreSQL full-text search on ref_eccn.description to find
        the top 20 candidate ECCNs, then passes them as context to the LLM.
        """
        stmt = (
            select(ECCN)
            .where(func.to_tsvector("english", ECCN.description).match(
                func.plainto_tsquery("english", description)
            ))
            .limit(20)
        )
        return (await session.execute(stmt)).scalars().all()
```

System prompt template (in `prompts.py`):
```python
ECCN_SYSTEM_PROMPT = """You are an expert export control classifier specializing in the US 
Export Administration Regulations (EAR) and the Commerce Control List (CCL).

Given a product description and a set of candidate ECCN entries from the CCL, determine 
the most appropriate ECCN classification.

Rules:
1. If the product does not meet the technical parameters of any specific ECCN, classify it as EAR99.
2. Always cite the specific CCL entry number and the technical parameter that determines the classification.
3. Consider both the item's technical specifications and its intended end-use.
4. If the item could fall under multiple ECCNs, explain why one is more appropriate.
5. Reference specific EAR sections (e.g., §742.6, §744.11) when relevant.

Output format: JSON with fields: code, confidence, rationale, cited_sources, alternative_codes."""
```

**Testing**:
- Unit (mocked LLM): Product "high-performance GPU with 16GB VRAM" → ECCN suggestion in category 3 or 4 with rationale citing CCL parameters
- Unit (mocked LLM): Product "wooden pencil" → EAR99 with rationale explaining no CCL entry applies
- Unit: Context retrieval with "encryption" → candidate ECCNs include 5A002, 5D002
- Unit: Context retrieval with empty/nonsense description → returns empty candidates, LLM falls back to EAR99
- Integration (mocked): Full classify flow → ClassificationSuggestion with confidence, rationale, cited_sources

---

#### 4.4 — HS Code Classification Engine

**What**: AI-assisted HS/HTS classification from product descriptions.

**Design**:

Same architecture as ECCN but with HS-specific context retrieval and prompting. The system prompt references WCO General Interpretive Rules (GIRs) and the HS chapter/heading structure.

```python
# src/tcp/classification/ai/hs_classifier.py
class HSClassifier:
    """HS code classification using LLM with WCO context.
    
    Workflow:
    1. Retrieve candidate HS codes by chapter/heading text search
    2. Build context with HS descriptions and GIR rules
    3. Call LLM for classification
    """

    async def classify(self, product: ProductClassifyRequest, session: AsyncSession) -> ClassificationSuggestion:
        candidates = await self._retrieve_candidate_hs(product.description, session)
        context = self._build_hs_context(candidates)
        return await self.llm_provider.classify(
            product_description=self._format_product(product),
            classification_type="HS",
            context=context,
        )
```

HS system prompt references:
- WCO General Interpretive Rules (GIR 1-6)
- Section and chapter notes
- Explanatory notes where available

**Testing**:
- Unit (mocked LLM): "Laptop computer" → HS code starting with "8471"
- Unit (mocked LLM): "Cotton T-shirt" → HS code starting with "6109"
- Unit: Context retrieval for "computer" → HS chapter 84 candidates included
- Integration (mocked): Full HS classification flow works end-to-end

---

#### 4.5 — Classification API and Decision Recording

**What**: REST API for product classification with decision history and audit trail.

**Design**:

```python
# src/tcp/classification/schemas.py
class ProductClassifyRequest(BaseModel):
    name: str = Field(max_length=500)
    description: str = Field(max_length=5000)
    part_number: str | None = None
    manufacturer: str | None = None
    specs: dict = {}  # Technical specifications as key-value pairs
    classification_type: Literal["HS", "ECCN", "both"]

class ClassificationResponse(BaseModel):
    product_id: uuid.UUID
    classifications: dict[str, ClassificationSuggestion]  # keyed by regime ("HS", "ECCN")
    product: ProductResponse
```

API endpoints:
- `POST /products` — create product
- `GET /products/{id}` — get product with current classifications
- `POST /products/{id}/classify` — trigger AI classification (HS, ECCN, or both)
- `PUT /products/{id}/classifications/{regime}` — manually set/override classification with rationale
- `GET /products/{id}/classification-history` — full classification decision history
- `GET /products?classification_status=unclassified` — list products needing classification

When a classification is made (AI or manual), a `classification_history` record is created with:
- regime, old_code, new_code, rationale, cited_sources, ai_assisted, ai_confidence, decided_by, decided_at

The product's current classification fields (hs_code, eccn, classification_status) are updated.

**Testing**:
- Integration: `POST /products` → 201, product created with `classification_status="unclassified"`
- Integration: `POST /products/{id}/classify` with type="ECCN" → 200, product updated with ECCN, classification_history record created
- Integration: `POST /products/{id}/classify` with type="both" → 200, both HS and ECCN suggestions returned
- Integration: `PUT /products/{id}/classifications/ECCN` manual override → classification_history records old and new
- Integration: `GET /products/{id}/classification-history` → ordered list of all decisions
- Integration: All classification actions create audit_log entries

---

## Phase 5: Export Licence Determination and Management

### Purpose
Build licence requirement determination logic and licence lifecycle management. After this phase, users can check whether a specific product-destination-end-user combination requires an export licence, manage licence applications, track consumption against approved values, and receive expiry alerts.

### Tasks

#### 5.1 — Licence Determination Engine

**What**: Given a product (with ECCN), destination country, and end-user, determine whether an export licence is required and which exceptions may apply.

**Design**:

```python
# src/tcp/licensing/determination.py
class LicenceDetermination(BaseModel):
    licence_required: bool
    basis: str                    # Regulatory basis for the determination
    licence_exception: str | None  # e.g., 'LVS', 'TMP', 'TSR', 'ENC'
    exception_conditions: str | None
    embargo_check: EmbargoCheckResult
    ai_assisted: bool

class LicenceDeterminationEngine:
    """Determines licence requirements based on EAR rules.
    
    Decision flow:
    1. Check if destination country is under comprehensive embargo → licence required, no exceptions
    2. Look up ECCN on the Commerce Country Chart for destination country
    3. Check if the control reason for the ECCN triggers a licence requirement for that destination
    4. Check applicable licence exceptions (LVS, TMP, TSR, ENC, etc.)
    5. Check end-user restrictions (Entity List, Denied Persons List)
    6. Return determination with full regulatory basis
    """

    async def determine(
        self,
        product_id: uuid.UUID,
        destination_country: str,  # ISO 3166-1 alpha-3
        end_user_party_id: uuid.UUID | None,
        end_use_description: str | None,
        session: AsyncSession,
    ) -> LicenceDetermination:
        product = await session.get(Product, product_id)

        # Step 1: Embargo check
        embargo = await self.embargo_service.check(destination_country, session)
        if embargo.is_comprehensive:
            return LicenceDetermination(
                licence_required=True,
                basis=f"Comprehensive embargo on {destination_country}: {embargo.programmes}",
                licence_exception=None,
                embargo_check=embargo,
                ai_assisted=False,
            )

        # Step 2: EAR99 products generally don't require licence (with exceptions)
        if product.ear99:
            # Check end-user restrictions
            if end_user_party_id:
                screening = await self.screener.screen_party(end_user_party_id, session)
                if screening.has_matches:
                    return LicenceDetermination(
                        licence_required=True,
                        basis="End-user appears on restricted party list",
                        ...
                    )
            return LicenceDetermination(licence_required=False, basis="Product classified EAR99; no end-user restrictions", ...)

        # Step 3: ECCN-based determination using Commerce Country Chart
        eccn = await session.get(ECCN, product.eccn)
        chart_result = self._check_commerce_country_chart(eccn, destination_country)

        # Step 4: Check licence exceptions
        exception = self._check_exceptions(eccn, destination_country, product)

        return LicenceDetermination(
            licence_required=chart_result.requires_licence and exception is None,
            basis=chart_result.basis,
            licence_exception=exception,
            ...
        )
```

**Testing**:
- Unit: EAR99 product to Canada → licence_required=False
- Unit: 3A001 product to Iran → licence_required=True (comprehensive embargo)
- Unit: 3A001 product to Germany → depends on Commerce Country Chart control reasons
- Unit: EAR99 product to non-embargoed country but end-user on Entity List → licence_required=True
- Unit: Product eligible for LVS exception → licence_exception="LVS" with conditions
- Integration: Full determination flow with database lookups

---

#### 5.2 — Licence CRUD and Lifecycle Management

**What**: Full licence lifecycle from draft through approval, consumption tracking, and expiry.

**Design**:

```python
# src/tcp/licensing/models.py
class Licence(Base):
    __tablename__ = "licence"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    licence_number: Mapped[str | None] = mapped_column(String(100))
    licence_type: Mapped[str] = mapped_column(String(50), nullable=False)  # 'individual', 'bulk', 'general'
    authority: Mapped[str] = mapped_column(String(100), nullable=False)  # 'BIS', 'DDTC', 'EU_member_state'
    status: Mapped[str] = mapped_column(String(50), nullable=False, default="draft")
    # Status lifecycle: draft → submitted → approved → [consumed|expired|revoked]
    product_id: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("product.id"))
    destination_country: Mapped[str] = mapped_column(String(3), nullable=False)
    end_user_party_id: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("party.id"))
    approved_value: Mapped[Decimal | None] = mapped_column(Numeric(18, 2))
    consumed_value: Mapped[Decimal] = mapped_column(Numeric(18, 2), default=0)
    approved_quantity: Mapped[Decimal | None] = mapped_column(Numeric(18, 4))
    consumed_quantity: Mapped[Decimal] = mapped_column(Numeric(18, 4), default=0)
    currency_code: Mapped[str] = mapped_column(String(3), default="USD")
    issued_date: Mapped[date | None] = mapped_column(Date)
    expiry_date: Mapped[date | None] = mapped_column(Date)
    authority_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    consumption_log: Mapped[list] = mapped_column(JSONB, default=list)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow, onupdate=datetime.utcnow)
```

API endpoints:
- `POST /licences` — create licence application (draft)
- `GET /licences` — list licences (filterable by status, authority, destination)
- `GET /licences/{id}` — get licence detail with consumption history
- `PUT /licences/{id}` — update licence (status transitions, approval details)
- `POST /licences/{id}/consume` — record consumption against licence
- `GET /licences/expiring?days=30` — licences expiring within N days
- `POST /licence-determinations` — run licence determination for product+destination+end-user

Consumption request:
```python
class LicenceConsumption(BaseModel):
    transaction_ref: str
    value_consumed: Decimal | None = None
    quantity_consumed: Decimal | None = None
```

Consumption validation:
- `consumed_value + value_consumed <= approved_value` (or reject)
- `consumed_quantity + quantity_consumed <= approved_quantity` (or reject)
- Append to `consumption_log` JSONB array

**Testing**:
- Integration: Create licence → status="draft"; update to "submitted" → "approved" with approved_value
- Integration: Consume against approved licence → consumed_value updated, consumption_log entry added
- Integration: Consume beyond approved_value → 400 error "Consumption would exceed approved value"
- Integration: `GET /licences/expiring?days=30` returns licences with expiry_date within 30 days
- Integration: Status transition "draft" → "approved" without going through "submitted" → 400 error
- Integration: All licence operations create audit_log entries

---

#### 5.3 — Embargo Rule Engine

**What**: Embargo rule database and checking service used by licence determination and transaction compliance checks.

**Design**:

```python
# src/tcp/regulatory/embargo.py
class EmbargoCheckResult(BaseModel):
    is_embargoed: bool
    is_comprehensive: bool  # Full embargo vs. sectoral
    programmes: list[str]
    restrictions: list[str]
    authorities: list[str]

class EmbargoService:
    async def check(self, country_code: str, session: AsyncSession) -> EmbargoCheckResult:
        """Check all active embargo rules for a country."""
        rules = await session.execute(
            select(EmbargoRule)
            .where(EmbargoRule.target_country == country_code)
            .where(EmbargoRule.is_active == True)
            .where(or_(EmbargoRule.expiry_date == None, EmbargoRule.expiry_date > date.today()))
        )
        ...
```

Seed data includes current embargo rules for: Iran, North Korea, Syria, Cuba, Russia (sectoral), Crimea/Donetsk/Luhansk, Belarus, Myanmar, Venezuela (sectoral).

**Testing**:
- Unit: `check("IRN")` → `is_embargoed=True, is_comprehensive=True`
- Unit: `check("RUS")` → `is_embargoed=True, is_comprehensive=False` (sectoral)
- Unit: `check("DEU")` → `is_embargoed=False`
- Unit: Expired embargo rule → not returned
- Integration: Seed data loaded; all comprehensive embargo countries return correct results

---

## Phase 6: Trade Transaction Compliance Orchestration

### Purpose
Build the trade transaction model that ties together screening, classification, licence checks, and embargo checks into a single compliance workflow. After this phase, users can submit export transactions and receive a comprehensive compliance assessment covering all required checks before shipment.

### Tasks

#### 6.1 — Trade Transaction Model and API

**What**: Trade transaction with line items, linked compliance checks, and status workflow.

**Design**:

```python
# src/tcp/transactions/models.py
class TradeTransaction(Base):
    __tablename__ = "trade_transaction"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    transaction_ref: Mapped[str] = mapped_column(String(255), nullable=False)
    transaction_type: Mapped[str] = mapped_column(String(50), nullable=False)  # 'export', 'import', 'reexport'
    status: Mapped[str] = mapped_column(String(50), nullable=False, default="draft")
    # Status lifecycle: draft → screening → compliance_review → approved → shipped | blocked
    exporter_party_id: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("party.id"))
    consignee_party_id: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("party.id"))
    end_user_party_id: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("party.id"))
    destination_country: Mapped[str] = mapped_column(String(3), nullable=False)
    total_value: Mapped[Decimal | None] = mapped_column(Numeric(18, 2))
    currency_code: Mapped[str] = mapped_column(String(3), default="USD")
    line_items: Mapped[list] = mapped_column(JSONB, default=list)
    screening_id: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("screening.id"))
    licence_id: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("licence.id"))
    compliance_checks: Mapped[dict] = mapped_column(JSONB, default=dict)
    filing_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow, onupdate=datetime.utcnow)
```

Line item schema:
```python
class LineItem(BaseModel):
    product_id: uuid.UUID
    quantity: Decimal
    unit_value: Decimal
    hs_code: str | None = None
    eccn: str | None = None
    schedule_b: str | None = None
    licence_exception: str | None = None
```

API endpoints:
- `POST /transactions` — create transaction
- `GET /transactions` — list (paginated, filterable by status, type, destination)
- `GET /transactions/{id}` — get transaction with compliance results
- `PUT /transactions/{id}` — update transaction details
- `POST /transactions/{id}/check` — run full compliance check (orchestrates all sub-checks)

**Testing**:
- Integration: Create transaction with 3 line items → 201, line_items JSONB populated
- Integration: `GET /transactions?status=draft&destination_country=IRN` → filtered results
- Integration: All CRUD operations create audit_log entries

---

#### 6.2 — Compliance Check Orchestrator

**What**: Orchestrates screening, classification verification, licence determination, and embargo checks for a transaction in a single workflow.

**Design**:

```python
# src/tcp/transactions/compliance_check.py
class ComplianceCheckResult(BaseModel):
    transaction_id: uuid.UUID
    overall_status: Literal["approved", "blocked", "needs_review"]
    checks: dict  # {screening, embargo, classification, licence}

class ComplianceOrchestrator:
    """Runs all compliance checks for a trade transaction.
    
    Check sequence:
    1. Embargo check on destination country
    2. Screen consignee and end-user parties
    3. Verify all line item products are classified
    4. Run licence determination for each line item
    5. Aggregate results into overall compliance status
    """

    async def check(self, transaction_id: uuid.UUID, session: AsyncSession) -> ComplianceCheckResult:
        tx = await session.get(TradeTransaction, transaction_id)

        results = {}

        # 1. Embargo
        embargo = await self.embargo_service.check(tx.destination_country, session)
        results["embargo"] = {"status": "blocked" if embargo.is_comprehensive else "passed", ...}

        # 2. Screening (consignee + end-user)
        for party_field in ["consignee_party_id", "end_user_party_id"]:
            party_id = getattr(tx, party_field)
            if party_id:
                screening = await self.screening_service.screen_party(party_id, session)
                results[f"screening_{party_field}"] = screening.dict()

        # 3. Classification verification
        unclassified = []
        for item in tx.line_items:
            product = await session.get(Product, item["product_id"])
            if product.classification_status == "unclassified":
                unclassified.append(product.name)
        if unclassified:
            results["classification"] = {"status": "incomplete", "unclassified_products": unclassified}

        # 4. Licence determination per line item
        licence_results = []
        for item in tx.line_items:
            det = await self.licence_engine.determine(
                item["product_id"], tx.destination_country, tx.end_user_party_id, None, session
            )
            licence_results.append(det.dict())
        results["licence"] = licence_results

        # 5. Aggregate
        overall = self._aggregate(results)

        # Update transaction
        tx.compliance_checks = results
        tx.status = overall
        session.add(tx)

        return ComplianceCheckResult(transaction_id=tx.id, overall_status=overall, checks=results)
```

**Testing**:
- Integration: Transaction to embargoed country → overall_status="blocked", embargo check shows blocked
- Integration: Transaction with clean parties and EAR99 products to Canada → overall_status="approved"
- Integration: Transaction with unclassified product → overall_status="needs_review", classification check shows incomplete
- Integration: Transaction where consignee matches SDN entry → overall_status="blocked"
- Integration: Transaction with product requiring licence but no licence attached → overall_status="needs_review"
- Integration: Compliance check creates audit_log entry
- Integration: `POST /transactions/{id}/check` → 200, full ComplianceCheckResult returned

---

## Phase 7: Regulatory Intelligence and Continuous Monitoring

### Purpose
Build continuous re-screening, regulatory change monitoring, and AI-generated digest capabilities. After this phase, the platform actively monitors for changes in watchlists and regulations, alerts users when existing counterparties appear on new lists, and produces natural-language summaries of regulatory updates.

### Tasks

#### 7.1 — Continuous Re-Screening

**What**: When watchlists are updated, automatically re-screen all active parties and generate alerts for new matches.

**Design**:

```python
# src/tcp/screening/tasks.py
@shared_task(name="tcp.screening.continuous_rescreen")
async def continuous_rescreen(tenant_id: str):
    """Re-screen all active parties for a tenant against current watchlists.
    
    Triggered after every watchlist update. Compares results with previous
    screenings to identify NEW matches only.
    """
    parties = await get_active_parties(tenant_id, session)
    for party in parties:
        new_screening = await screener.screen(
            ScreeningQuery(name=party.primary_name, ...),
            sources=["OFAC_SDN", "BIS_EL", "EU_CONSOLIDATED", "UN_SC"],
            session=session,
        )
        # Compare with most recent screening for this party
        prev = await get_latest_screening(party.id, session)
        new_hits = identify_new_hits(new_screening, prev)
        if new_hits:
            await create_alert(tenant_id, party, new_hits, session)
```

Alert model:
```python
class Alert(Base):
    __tablename__ = "alert"
    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    alert_type: Mapped[str] = mapped_column(String(50), nullable=False)  # 'new_screening_match', 'licence_expiry', 'regulatory_update'
    severity: Mapped[str] = mapped_column(String(20), nullable=False)  # 'critical', 'high', 'medium', 'low'
    title: Mapped[str] = mapped_column(String(500), nullable=False)
    detail: Mapped[dict] = mapped_column(JSONB, nullable=False)
    entity_type: Mapped[str] = mapped_column(String(100))
    entity_id: Mapped[uuid.UUID | None] = mapped_column(UUID)
    is_read: Mapped[bool] = mapped_column(Boolean, default=False)
    is_resolved: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
```

API endpoints:
- `GET /alerts` — list alerts (filterable by type, severity, read/unread)
- `PUT /alerts/{id}/read` — mark alert as read
- `PUT /alerts/{id}/resolve` — mark alert as resolved with note
- `GET /alerts/stats` — alert counts by type and severity

**Testing**:
- Integration: Add new entry to watchlist matching existing party → alert created with type "new_screening_match"
- Integration: Re-screen with no new matches → no new alerts
- Integration: `GET /alerts?severity=critical&is_read=false` → only unread critical alerts
- Integration: Alert resolution creates audit_log entry

---

#### 7.2 — Regulatory Update Monitor and AI Digest

**What**: Ingest regulatory updates from BIS, OFAC, and EU sources; generate AI-summarised digests flagging affected products and counterparties.

**Design**:

```python
# src/tcp/regulatory/digest.py
class RegulatoryDigestGenerator:
    """Generates natural-language summaries of regulatory updates.
    
    Workflow:
    1. Fetch new regulatory updates (from RSS feeds, Federal Register API)
    2. Parse and store the raw update
    3. Use LLM to generate plain-language summary
    4. Cross-reference affected ECCNs and countries with tenant's products and parties
    5. Generate impact analysis and create alerts for affected items
    """

    async def generate_digest(self, update: RegulatoryUpdate, tenant_id: uuid.UUID, session: AsyncSession):
        # LLM generates summary
        summary = await self.llm_provider.summarize_regulatory_update(update.title, update.raw_text)

        # Impact analysis: find tenant's products with affected ECCNs
        affected_products = await self._find_affected_products(
            update.affected_eccns, tenant_id, session
        )
        affected_parties = await self._find_affected_parties(
            update.affected_countries, tenant_id, session
        )

        # Store digest
        update.ai_digest = summary
        update.impact_analysis = {
            "affected_products": [p.name for p in affected_products],
            "affected_parties": [p.primary_name for p in affected_parties],
            "severity": self._assess_severity(affected_products, affected_parties),
            "recommended_actions": self._recommend_actions(update, affected_products),
        }

        # Create alert if impact found
        if affected_products or affected_parties:
            await create_alert(tenant_id, ...)
```

API endpoints:
- `GET /regulatory-updates` — list updates (filterable by source, date range)
- `GET /regulatory-updates/{id}` — get update with AI digest and impact analysis
- `POST /regulatory-updates/refresh` — trigger manual check for new updates

**Testing**:
- Integration (mocked LLM): Regulatory update about new China export controls on ECCNs 3A001, 5A002 → AI digest generated; products with those ECCNs flagged as affected
- Integration: Update with no matching products/parties → no alert created
- Integration: `GET /regulatory-updates?source=BIS&start_date=2026-01-01` → filtered results

---

#### 7.3 — Licence Expiry Alerts

**What**: Celery periodic task that checks for licences expiring within configurable thresholds and creates alerts.

**Design**:

```python
@shared_task(name="tcp.licensing.check_expiries")
async def check_licence_expiries():
    """Check for expiring licences and create alerts.
    
    Thresholds:
    - 90 days: severity='low' (advance notice)
    - 30 days: severity='medium' (action needed)
    - 7 days: severity='high' (urgent)
    - Expired: severity='critical'
    """
    ...
```

**Testing**:
- Integration: Licence expiring in 25 days → alert with severity="medium"
- Integration: Already-expired licence → alert with severity="critical"
- Integration: Licence expiring in 100 days → no alert (outside 90-day window)
- Integration: Duplicate check → does not create duplicate alerts for same licence/threshold

---

## Phase 8: Web Dashboard (Frontend)

### Purpose
Build the compliance officer dashboard for screening, classification, licence management, and regulatory monitoring. After this phase, users interact with the platform through a modern web interface rather than API calls alone.

### Tasks

#### 8.1 — Next.js Project Setup and API Client Generation

**What**: Initialize Next.js 15 project with TypeScript, Tailwind CSS, shadcn/ui, and generate a typed API client from the OpenAPI spec.

**Design**:

```
frontend/
├── package.json
├── tsconfig.json
├── next.config.ts
├── tailwind.config.ts
├── src/
│   ├── app/
│   │   ├── layout.tsx           # Root layout with sidebar navigation
│   │   ├── page.tsx             # Dashboard home
│   │   ├── login/page.tsx
│   │   ├── dashboard/page.tsx
│   │   ├── screening/
│   │   │   ├── page.tsx         # Screening queue
│   │   │   └── [id]/page.tsx    # Screening detail with hit review
│   │   ├── classification/
│   │   │   ├── page.tsx         # Product list
│   │   │   └── [id]/page.tsx    # Product detail with classification
│   │   ├── licensing/
│   │   │   ├── page.tsx         # Licence list
│   │   │   └── [id]/page.tsx    # Licence detail
│   │   ├── parties/
│   │   │   ├── page.tsx         # Party list
│   │   │   └── [id]/page.tsx    # Party detail
│   │   └── alerts/page.tsx      # Alert center
│   ├── components/
│   │   ├── ui/                  # shadcn/ui components
│   │   ├── screening/
│   │   │   ├── screening-queue.tsx
│   │   │   ├── hit-review-card.tsx
│   │   │   └── screening-form.tsx
│   │   ├── classification/
│   │   │   ├── classify-dialog.tsx
│   │   │   └── classification-history.tsx
│   │   └── common/
│   │       ├── data-table.tsx
│   │       ├── sidebar.tsx
│   │       └── alert-badge.tsx
│   ├── lib/
│   │   ├── api-client.ts        # Generated from OpenAPI spec (openapi-typescript)
│   │   └── auth.ts              # JWT token management
│   └── hooks/
│       ├── use-screenings.ts
│       ├── use-parties.ts
│       └── use-alerts.ts
```

API client generation using `openapi-typescript` and `openapi-fetch`:
```bash
npx openapi-typescript http://localhost:8000/openapi.json -o src/lib/api-types.ts
```

**Testing**:
- Integration: `npm run build` succeeds with zero TypeScript errors
- Integration: API client generated from OpenAPI spec has typed methods for all endpoints
- E2E: Login page renders; submit credentials → redirected to dashboard

---

#### 8.2 — Screening Dashboard

**What**: Screening queue view showing pending screenings, hit review interface with disposition controls, and new screening form.

**Design**:

Key screens:
1. **Screening Queue** — table of screenings with status filters (pending, potential_match, cleared, blocked), sortable by date and score
2. **Screening Detail** — shows all hits with match scores, matched fields, watchlist entry details, and disposition controls (false_positive / confirmed_match / escalated with required note)
3. **New Screening Form** — name input, optional country, party type, and identifier fields; submit triggers real-time screening; results displayed inline

UX patterns:
- Match scores displayed as color-coded badges (green < 0.5, yellow 0.5-0.8, red > 0.8)
- Hit disposition requires a text note (minimum 10 characters) for regulatory compliance
- When all hits disposed, overall screening status auto-updates
- Batch screening upload via CSV

**Testing**:
- E2E (Playwright): Navigate to /screening → queue table renders with mock data
- E2E: Submit new screening → results appear with hit cards
- E2E: Dispose hit as false_positive with note → hit card updates to show disposition
- E2E: Dispose all hits → screening status changes to "cleared"
- E2E: Try to dispose without note → validation error shown

---

#### 8.3 — Classification Interface

**What**: Product list with classification status, AI classification trigger, and manual override interface.

**Design**:

Key screens:
1. **Product List** — filterable by classification_status, sortable by name
2. **Product Detail** — shows current HS/ECCN classification with rationale, classification history timeline, and "Re-classify" button
3. **Classify Dialog** — shows AI suggestion with confidence score, rationale, and cited sources; user can accept, modify, or reject

UX patterns:
- Classification rationale displayed in expandable card with cited regulatory sources as linked references
- Confidence score shown as percentage with color coding
- Classification history as a vertical timeline

**Testing**:
- E2E: Navigate to /classification → product table renders
- E2E: Click "Classify" on unclassified product → dialog shows AI suggestion
- E2E: Accept AI suggestion → product updated, history entry added
- E2E: Manually override classification → dialog accepts custom code and rationale

---

#### 8.4 — Licence Management Interface

**What**: Licence list, detail view with consumption tracking, and expiry alerts.

**Design**:

Key screens:
1. **Licence List** — filterable by status, authority, destination; shows consumption bar (consumed/approved)
2. **Licence Detail** — full licence info, consumption log as table, "Record Consumption" form
3. **Expiring Licences** — filtered view of licences expiring within 30/60/90 days

**Testing**:
- E2E: Navigate to /licensing → licence table with consumption progress bars
- E2E: Record consumption → balance updates, log entry appears
- E2E: Attempt to over-consume → error message displayed

---

#### 8.5 — Alert Center and Dashboard Home

**What**: Unified alert center and dashboard home page with compliance status overview.

**Design**:

Dashboard home shows:
- Active alert count by severity (critical/high/medium/low)
- Screening queue depth (pending reviews)
- Licence expiry warnings
- Recent regulatory updates
- Compliance check pass/fail rates

Alert center:
- Filterable by type, severity, read/unread status
- Inline resolve with note
- Link to related entity (screening, licence, party)

**Testing**:
- E2E: Dashboard renders with correct counts from API
- E2E: Click alert → navigates to related entity
- E2E: Mark alert as resolved → alert moves to resolved section

---

## Phase 9: Continuous Re-Screening and Batch Operations

### Purpose
Build production-grade batch screening, scheduled continuous re-screening, and performance optimizations for large watchlist datasets. After this phase, the platform handles enterprise-scale workloads.

### Tasks

#### 9.1 — Batch Screening Pipeline

**What**: Upload CSV of parties for batch screening; process asynchronously via Celery; stream results.

**Design**:

```python
# src/tcp/screening/tasks.py
@shared_task(name="tcp.screening.batch_screen")
def batch_screen(job_id: str, parties: list[dict], tenant_id: str, sources: list[str]):
    """Process batch screening job.
    
    - Processes parties in chunks of 50
    - Updates job progress in Redis (for polling)
    - Creates individual Screening records for each party
    - Generates summary report on completion
    """
    ...
```

API endpoints:
- `POST /screenings/batch/upload` — upload CSV, returns job_id
- `GET /screenings/batch/{job_id}/status` — poll job progress (percentage, processed/total)
- `GET /screenings/batch/{job_id}/results` — download results when complete

CSV format:
```csv
name,country,type,identifier_type,identifier_value
"ACME Corp","DEU","organisation","vat","DE123456789"
"John Smith","USA","individual","passport","AB1234567"
```

**Testing**:
- Integration: Upload 100-party CSV → job created; poll until complete; results contain all parties
- Integration: Upload malformed CSV → 400 with specific parsing errors
- Integration: Job progress updates in real-time via Redis
- Integration: 1000-party batch completes within 5 minutes (sample watchlist)

---

#### 9.2 — Screening Performance Optimization

**What**: Database-level optimizations for screening against large watchlists (100K+ entries).

**Design**:

Optimizations:
1. **Pre-filtered candidate set**: Use pg_trgm similarity threshold to reduce candidate set before running full matching
2. **Trigram index**: `CREATE INDEX ... USING GIN (primary_name gin_trgm_ops)` on watchlist_entry
3. **Materialized search view**: Pre-compute name + alias flat list for faster scanning
4. **Connection pooling**: PgBouncer for handling concurrent screening requests
5. **Redis caching**: Cache watchlist entries per source with TTL matching update frequency

```sql
-- Materialized view for fast name searching
CREATE MATERIALIZED VIEW watchlist_search AS
SELECT
    e.id AS entry_id,
    e.source_id,
    e.primary_name AS name,
    'primary' AS name_type,
    e.is_active
FROM watchlist_entry e
WHERE e.is_active = true
UNION ALL
SELECT
    e.id AS entry_id,
    e.source_id,
    a.value->>'name' AS name,
    'alias' AS name_type,
    e.is_active
FROM watchlist_entry e,
     jsonb_array_elements(e.aliases) AS a(value)
WHERE e.is_active = true;

CREATE INDEX idx_watchlist_search_trgm ON watchlist_search USING GIN (name gin_trgm_ops);
REFRESH MATERIALIZED VIEW CONCURRENTLY watchlist_search;
```

**Testing**:
- Integration: Screen against 100K watchlist entries → response within 3 seconds for single party
- Integration: Materialized view refresh completes within 30 seconds
- Integration: EXPLAIN ANALYZE shows trigram index usage, not sequential scan

---

## Phase 10: ERP Integration and Webhook API

### Purpose
Build integration connectors for SAP and Oracle ERP systems, plus a webhook system for real-time event notifications to external systems. After this phase, the platform connects to enterprise ERP systems for automated compliance checks within order/shipment workflows.

### Tasks

#### 10.1 — Webhook Event System

**What**: Outbound webhook delivery for compliance events (screening results, classification changes, licence updates, alerts).

**Design**:

```python
# src/tcp/integrations/webhooks/models.py
class WebhookEndpoint(Base):
    __tablename__ = "webhook_endpoint"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    url: Mapped[str] = mapped_column(Text, nullable=False)
    secret: Mapped[str] = mapped_column(String(255), nullable=False)  # HMAC signing secret
    events: Mapped[list[str]] = mapped_column(ARRAY(Text), nullable=False)
    # Events: 'screening.completed', 'screening.match_found', 'classification.decided',
    #         'licence.expiring', 'licence.consumed', 'alert.created', 'watchlist.updated'
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)
```

Webhook delivery:
- HMAC-SHA256 signature in `X-TCP-Signature` header
- Retry with exponential backoff (3 attempts: 10s, 60s, 300s)
- Delivery log with status codes

API endpoints:
- `POST /webhooks` — register webhook endpoint
- `GET /webhooks` — list registered webhooks
- `DELETE /webhooks/{id}` — remove webhook
- `POST /webhooks/{id}/test` — send test event
- `GET /webhooks/{id}/deliveries` — delivery history with status

**Testing**:
- Integration: Register webhook → send test event → delivery logged with 200 status
- Integration: Webhook with invalid URL → delivery logged with error status, retries scheduled
- Integration: HMAC signature verification — compute expected signature from payload + secret
- Integration: Screening completion triggers webhook to registered endpoints subscribed to 'screening.completed'

---

#### 10.2 — SAP Integration Connector

**What**: REST API connector for SAP S/4HANA to trigger compliance checks from SAP sales orders and deliveries.

**Design**:

SAP integration pattern:
1. SAP calls TCP's REST API when a sales order or delivery is created/changed
2. TCP runs screening on the business partner, classification check on materials, licence determination
3. TCP returns pass/fail result
4. SAP can block the document if TCP returns a compliance hold

Dedicated SAP-facing endpoints:
- `POST /integrations/sap/screen-partner` — screen SAP business partner by name/ID
- `POST /integrations/sap/check-delivery` — run full compliance check on delivery (materials + partner + destination)
- `GET /integrations/sap/compliance-status/{sap_doc_number}` — get compliance status for SAP document

SAP-specific field mapping:
```python
class SAPDeliveryCheck(BaseModel):
    sap_doc_number: str
    sap_doc_type: str  # 'LIKP' (delivery), 'VBAK' (sales order)
    business_partner: str
    ship_to_country: str  # SAP country code → mapped to ISO 3166-1 alpha-3
    materials: list[SAPMaterial]

class SAPMaterial(BaseModel):
    material_number: str
    description: str
    quantity: Decimal
    unit: str
    value: Decimal
```

**Testing**:
- Integration: SAP delivery check with clean partner and EAR99 materials → compliance status "approved"
- Integration: SAP delivery check with partner matching SDN → compliance status "blocked"
- Integration: SAP country code mapping (e.g., "US" → "USA", "DE" → "DEU")

---

#### 10.3 — Oracle ERP Integration Connector

**What**: REST API connector for Oracle ERP Cloud with similar compliance check endpoints.

**Design**:

Same pattern as SAP but with Oracle-specific field mappings and document types.

- `POST /integrations/oracle/check-order` — check Oracle sales order
- `GET /integrations/oracle/compliance-status/{oracle_doc_id}` — get status

**Testing**:
- Integration: Oracle order check flow works end-to-end with mock Oracle data
- Integration: Field mappings correctly translate Oracle document structure to TCP entities

---

## Phase 11: Advanced AI Features

### Purpose
Build advanced AI-native capabilities: false-positive reduction (AI-assisted screening triage), natural-language licence determination, and MCP server for AI agent integration. These are the differentiating features that set the platform apart from legacy incumbents.

### Tasks

#### 11.1 — AI-Assisted False-Positive Triage

**What**: LLM-powered analysis of screening hits to automatically classify likely false positives, reducing the manual review queue.

**Design**:

```python
# src/tcp/screening/ai_triage.py
class AITriageResult(BaseModel):
    hit_index: int
    recommendation: Literal["likely_false_positive", "likely_true_match", "uncertain"]
    confidence: float
    reasoning: str
    factors: list[str]  # e.g., ["different_country", "different_entity_type", "partial_name_only"]

class AIScreeningTriager:
    """Analyses screening hits using LLM to identify likely false positives.
    
    Factors considered:
    - Name match quality (exact vs. partial vs. fuzzy)
    - Country match/mismatch
    - Entity type match (individual vs. organisation)
    - Identifier matches (exact ID match is strong positive signal)
    - Watchlist programme relevance to the transaction context
    - Date of birth match for individuals
    - Address similarity
    """

    async def triage(
        self,
        screening: Screening,
        party: Party,
    ) -> list[AITriageResult]:
        prompt = self._build_triage_prompt(screening, party)
        response = await self.llm_provider.analyze(prompt)
        return [AITriageResult(**r) for r in response["results"]]
```

System prompt focuses on:
- Comparing party details (full address, DOB, identifiers) against watchlist entry details
- Explaining which factors increase or decrease match likelihood
- Never auto-clearing confirmed matches — only recommending

API endpoint:
- `POST /screenings/{id}/ai-triage` — run AI triage on screening hits; results stored alongside human review

**Testing**:
- Integration (mocked LLM): Hit with same name but different country and entity type → "likely_false_positive"
- Integration (mocked LLM): Hit with exact name, same country, matching identifier → "likely_true_match"
- Integration: AI triage results stored in screening record but do not auto-dispose (human must confirm)

---

#### 11.2 — Natural-Language Licence Determination

**What**: Chat-style interface for licence determination queries ("Do I need a licence to export this product to this country?").

**Design**:

```python
# src/tcp/licensing/nl_determination.py
class NLLicenceDetermination:
    """Natural-language licence determination.
    
    User input: "Do I need a licence to export AES-256 encryption modules to India?"
    
    System:
    1. Extract product description, destination, end-use from natural language
    2. Look up or classify the product (ECCN)
    3. Run licence determination engine
    4. Generate natural-language explanation
    """

    async def query(self, question: str, tenant_id: uuid.UUID, session: AsyncSession) -> NLResponse:
        # Step 1: LLM extracts structured parameters
        params = await self._extract_params(question)
        # params: {product_description: "AES-256 encryption modules", destination: "IND", ...}

        # Step 2: Find or classify product
        product = await self._find_or_classify(params, session)

        # Step 3: Run determination
        determination = await self.engine.determine(product.id, params.destination, ...)

        # Step 4: Generate explanation
        explanation = await self._generate_explanation(question, determination)

        return NLResponse(
            answer=explanation,
            determination=determination,
            product=product,
        )
```

API endpoint:
- `POST /licence-determinations/ask` — natural-language query → structured determination + explanation

**Testing**:
- Integration (mocked LLM): "Export encryption to Iran" → licence required, embargo cited
- Integration (mocked LLM): "Ship wooden pencils to Canada" → no licence required, EAR99 explanation
- Integration: Extracted parameters match expected product/destination structure

---

#### 11.3 — MCP Server for AI Agent Integration

**What**: Model Context Protocol server exposing screening, classification, and licence determination as tools for AI agents.

**Design**:

MCP server endpoints (per MCP specification):
1. **screen_party** — screen a party name against watchlists
2. **classify_product** — get ECCN/HS classification for a product description
3. **check_licence** — determine if a licence is required for a product+destination
4. **get_embargo_status** — check if a country is under embargo

This enables AI assistants to invoke trade compliance checks programmatically — a first-mover capability noted in the standards research.

**Testing**:
- Integration: MCP client calls `screen_party` tool → receives screening results
- Integration: MCP client calls `classify_product` → receives classification suggestion
- Integration: MCP authentication via API key

---

## Phase 12: Production Hardening and Deployment

### Purpose
Prepare the platform for production deployment with security hardening, monitoring, performance testing, and deployment automation.

### Tasks

#### 12.1 — Security Hardening

**What**: Rate limiting, input sanitisation, HTTPS enforcement, security headers, and API key rotation.

**Design**:
- Rate limiting via Redis (per-tenant, per-endpoint)
- Input sanitisation on all text fields (prevent SQL injection via ORM, XSS via CSP headers)
- Security headers: HSTS, X-Content-Type-Options, X-Frame-Options, CSP
- API key rotation endpoint
- Password hashing with bcrypt (already configured)
- CORS tightening for production origins

**Testing**:
- Integration: Rate limit exceeded → 429 with Retry-After header
- Integration: Security headers present on all responses
- Integration: API key rotation → old key rejected, new key works

---

#### 12.2 — Monitoring and Observability

**What**: Structured logging, metrics, and health monitoring.

**Design**:
- Structured JSON logging via `structlog`
- Prometheus metrics endpoint (`/metrics`) with key counters:
  - `tcp_screenings_total` (by status, source)
  - `tcp_classifications_total` (by type, ai_assisted)
  - `tcp_watchlist_entries_total` (by source)
  - `tcp_api_requests_total` (by endpoint, status_code)
  - `tcp_screening_duration_seconds` (histogram)
- Celery task monitoring via Flower

**Testing**:
- Integration: `/metrics` returns Prometheus-formatted metrics
- Integration: After screening, `tcp_screenings_total` counter incremented
- Integration: Structured log output is valid JSON with correlation IDs

---

#### 12.3 — Production Docker and CI/CD

**What**: Multi-stage Docker build, production Docker Compose, and GitHub Actions CI pipeline.

**Design**:

Multi-stage Dockerfile:
```dockerfile
# Build stage
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

# Runtime stage
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY src/ ./src/
COPY alembic.ini alembic/ ./
ENV PATH="/app/.venv/bin:$PATH"
ENV PYTHONPATH=/app/src
EXPOSE 8000
CMD ["uvicorn", "tcp.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

GitHub Actions CI:
1. Lint (ruff check)
2. Type check (mypy --strict)
3. Unit tests (pytest tests/unit/)
4. Integration tests (pytest tests/integration/ with Docker Compose services)
5. Docker build
6. Frontend build (npm run build)

**Testing**:
- Integration: Multi-stage Docker build produces image < 200MB
- Integration: CI pipeline passes on clean main branch
- Integration: Docker Compose production config starts all services with health checks

---

## Phase Summary & Dependencies

```
Phase 1: Foundation              ─── required by everything
    │
Phase 2: Parties & Watchlists    ─── requires Phase 1
    │
Phase 3: Screening Engine        ─── requires Phase 2
    │
Phase 4: Classification Engine   ─── requires Phase 1 (independent of Phase 3)
    │
Phase 5: Licence Management      ─── requires Phase 3 + Phase 4
    │
Phase 6: Transaction Orchestration ─── requires Phase 5
    │
Phase 7: Regulatory Intelligence  ─── requires Phase 3 + Phase 4
    │                                   (can parallel with Phase 6)
Phase 8: Web Dashboard            ─── requires Phase 6
    │                                   (can start after Phase 5, incrementally)
Phase 9: Batch & Performance      ─── requires Phase 3
    │                                   (can parallel with Phase 6-8)
Phase 10: ERP Integration         ─── requires Phase 6
    │                                   (can parallel with Phase 8)
Phase 11: Advanced AI             ─── requires Phase 3 + Phase 4 + Phase 5
    │                                   (can parallel with Phase 8-10)
Phase 12: Production Hardening    ─── requires all prior phases
```

### Parallelism Opportunities

- **Phases 4 and 3** can be developed concurrently after Phase 2 (classification does not depend on screening)
- **Phases 6 and 7** can be developed concurrently after Phase 5
- **Phases 8, 9, 10, and 11** can all be developed concurrently after Phase 6
- **Frontend (Phase 8)** can begin incrementally after Phase 5, adding screens as backend phases complete

---

## Definition of Done (per phase)

1. All tasks implemented and code committed.
2. All unit tests pass (`pytest tests/unit/`).
3. All integration tests pass (`pytest tests/integration/`).
4. Linting passes (`ruff check src/`).
5. Formatting passes (`ruff format --check src/`).
6. Type checking passes (`mypy src/ --strict`).
7. Docker build succeeds (`docker build .`).
8. Feature works end-to-end (manual verification or E2E test).
9. New configuration options documented in `config.py` with defaults.
10. New API endpoints appear in auto-generated OpenAPI spec at `/openapi.json`.
11. Database migrations created and tested (upgrade + downgrade).
12. Audit trail records created for all state-changing operations.
13. No security vulnerabilities introduced (no secrets in code, no SQL injection, inputs validated).
