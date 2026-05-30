# Personal Finance Management — Phased Development Plan

> Project: 217-personal-finance-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | Python 3.12+ | LLM integration is the core differentiator; Python has the best LLM SDK ecosystem (Anthropic SDK, LangChain), plus strong financial libraries (pandas, numpy). Plaid, MX, and Finicity all provide official Python SDKs. |
| API framework | FastAPI 0.115+ | Automatic OpenAPI 3.1 spec generation (required for the public API); async support for concurrent bank sync and LLM calls; Pydantic v2 for request/response validation that doubles as data model documentation. |
| Database | PostgreSQL 16 | JSONB columns for aggregator-specific metadata (Hybrid Relational + JSONB data model); GIN indexes for tag and JSONB queries; NUMERIC type for precise monetary arithmetic; array types for tags. Mature, proven in fintech. |
| ORM / query builder | SQLAlchemy 2.0 + Alembic | SQLAlchemy 2.0's typed ORM maps cleanly to the hybrid schema; Alembic handles migrations including JSONB column additions. Native JSONB and array type support. |
| Task queue | Celery 5.x + Redis | Bank sync, AI categorisation, recurring pattern detection, and forecast generation are all async workloads. Celery provides task scheduling (beat), retry logic, and rate limiting. Redis as broker is lightweight and doubles as cache. |
| Cache | Redis 7 | Session cache, rate limiting, LLM response caching, and Celery broker in a single dependency. |
| LLM provider | Anthropic Claude API (claude-sonnet-4-20250514) | Best cost/quality ratio for financial coaching and categorisation. Sonnet for categorisation (high throughput, low cost), Opus for coaching conversations (deeper reasoning). Prompt caching for repeated system prompts. |
| Frontend | Next.js 15 (React 19) + TypeScript | Server components for dashboard rendering; app router for nested layouts (dashboard, budget, goals, settings). TypeScript enforces API contract alignment with OpenAPI-generated types. |
| UI component library | shadcn/ui + Tailwind CSS 4 | Accessible, unstyled primitives that match the clean, modern aesthetic of Monarch/Copilot. No vendor lock-in — components are copied into the project. |
| Charting | Recharts 2 | React-native charting for net worth trends, spending breakdowns, cash flow projections, and goal progress. Lightweight, composable, responsive. |
| Mobile | React Native (Expo) | Shared TypeScript codebase with web frontend; Expo for zero-config builds to iOS and Android. Deferred to Phase 10 — web-first MVP. |
| Authentication | NextAuth.js 5 + OAuth 2.0 | Supports email/password, Google, Apple sign-in. JWT sessions with refresh tokens. PKCE for mobile flows. |
| Encryption | libsodium (PyNaCl) | AES-256-GCM for at-rest encryption of aggregator access tokens and sensitive financial data. Key management via environment variable or HashiCorp Vault in production. |
| Testing (backend) | pytest + pytest-asyncio + factory_boy | pytest for unit and integration tests; factory_boy for generating realistic financial test data; pytest-asyncio for async endpoint testing. |
| Testing (frontend) | Vitest + React Testing Library + Playwright | Vitest for unit tests; React Testing Library for component tests; Playwright for end-to-end browser tests. |
| Code quality | Ruff (lint + format) + mypy (strict) + ESLint + Prettier | Ruff replaces flake8, isort, and black in a single fast tool. mypy strict mode catches type errors in financial arithmetic. |
| Containerisation | Docker + docker-compose | Self-hosted deployment is a key differentiator over closed-source competitors. Single `docker-compose up` for the full stack (API, worker, PostgreSQL, Redis, frontend). |
| CI/CD | GitHub Actions | Lint, type-check, test, build Docker images, and deploy on every push. Matrix testing for Python 3.12+ and Node 22+. |
| API documentation | Swagger UI (built into FastAPI) + Redoc | Auto-generated from OpenAPI spec. Public API modelled after YNAB's API structure per standards.md. |
| Package manager (Python) | uv | Fastest Python package manager; deterministic lockfile; replaces pip, pip-tools, and virtualenv. |
| Package manager (JS) | pnpm 9 | Deterministic, fast, disk-efficient. Workspace support for monorepo (frontend + shared types). |

### Project Structure

```
personal-finance-management/
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.frontend
├── pyproject.toml                    # Python backend config (uv)
├── alembic.ini
├── alembic/
│   └── versions/                     # Database migrations
├── src/
│   ├── __init__.py
│   ├── main.py                       # FastAPI application factory
│   ├── config.py                     # Pydantic Settings (env-based config)
│   ├── database.py                   # SQLAlchemy engine, session factory
│   ├── models/                       # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── user.py                   # User, Household, HouseholdMember
│   │   ├── account.py                # Account, aggregator_data JSONB
│   │   ├── transaction.py            # Transaction, extra JSONB
│   │   ├── category.py               # Category (self-referencing hierarchy)
│   │   ├── budget.py                 # Budget, BudgetPeriod, BudgetAllocation
│   │   ├── goal.py                   # Goal, modeling_data JSONB
│   │   ├── recurring.py              # RecurringPattern, detection_data JSONB
│   │   ├── investment.py             # InvestmentHolding, security_data JSONB
│   │   ├── net_worth.py              # NetWorthSnapshot, breakdown JSONB
│   │   ├── ai.py                     # AIConversation, AIMessage, AIInsight
│   │   ├── categorisation_rule.py    # CategorisationRule, rule_definition JSONB
│   │   ├── import_job.py             # ImportJob, results JSONB
│   │   └── api_token.py              # APIToken
│   ├── schemas/                      # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── account.py
│   │   ├── transaction.py
│   │   ├── category.py
│   │   ├── budget.py
│   │   ├── goal.py
│   │   ├── recurring.py
│   │   ├── insight.py
│   │   └── common.py                 # Pagination, error responses
│   ├── api/                          # FastAPI routers
│   │   ├── __init__.py
│   │   ├── deps.py                   # Dependency injection (auth, db session)
│   │   ├── auth.py                   # /auth/* endpoints
│   │   ├── accounts.py               # /accounts/* endpoints
│   │   ├── transactions.py           # /transactions/* endpoints
│   │   ├── categories.py             # /categories/* endpoints
│   │   ├── budgets.py                # /budgets/* endpoints
│   │   ├── goals.py                  # /goals/* endpoints
│   │   ├── insights.py               # /insights/* endpoints
│   │   ├── coaching.py               # /coaching/* endpoints
│   │   ├── sync.py                   # /sync/* endpoints
│   │   └── import_export.py          # /import/*, /export/* endpoints
│   ├── services/                     # Business logic layer
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── account_service.py
│   │   ├── transaction_service.py
│   │   ├── budget_service.py
│   │   ├── goal_service.py
│   │   ├── categorisation_service.py
│   │   ├── recurring_service.py
│   │   ├── net_worth_service.py
│   │   ├── forecast_service.py
│   │   ├── insight_service.py
│   │   ├── coaching_service.py
│   │   └── import_service.py
│   ├── aggregators/                  # Bank data sync adapters
│   │   ├── __init__.py
│   │   ├── base.py                   # Abstract aggregator interface
│   │   ├── plaid_adapter.py
│   │   ├── manual_adapter.py
│   │   └── csv_adapter.py
│   ├── ai/                           # AI/LLM integration layer
│   │   ├── __init__.py
│   │   ├── categoriser.py            # AI transaction categorisation
│   │   ├── coach.py                  # Financial coaching conversation
│   │   ├── anomaly_detector.py       # Subscription/fee anomaly detection
│   │   ├── forecaster.py             # Cash flow forecast generation
│   │   ├── prompts/                  # Prompt templates
│   │   │   ├── categorisation.py
│   │   │   ├── coaching.py
│   │   │   ├── anomaly.py
│   │   │   └── forecast.py
│   │   └── tools.py                  # MCP tool definitions
│   ├── workers/                      # Celery task definitions
│   │   ├── __init__.py
│   │   ├── celery_app.py
│   │   ├── sync_tasks.py
│   │   ├── categorisation_tasks.py
│   │   ├── recurring_tasks.py
│   │   ├── forecast_tasks.py
│   │   ├── insight_tasks.py
│   │   └── snapshot_tasks.py
│   ├── encryption.py                 # Token encryption/decryption
│   └── utils/
│       ├── __init__.py
│       ├── money.py                  # Decimal arithmetic helpers
│       └── dates.py                  # Date range, fiscal period helpers
├── tests/
│   ├── conftest.py                   # Shared fixtures, test database
│   ├── factories/                    # factory_boy factories
│   │   ├── __init__.py
│   │   ├── user_factory.py
│   │   ├── account_factory.py
│   │   ├── transaction_factory.py
│   │   └── budget_factory.py
│   ├── unit/
│   │   ├── test_money.py
│   │   ├── test_categoriser.py
│   │   ├── test_budget_service.py
│   │   ├── test_forecast_service.py
│   │   └── ...
│   ├── integration/
│   │   ├── test_auth_api.py
│   │   ├── test_accounts_api.py
│   │   ├── test_transactions_api.py
│   │   ├── test_plaid_adapter.py
│   │   └── ...
│   ├── e2e/
│   │   └── ...
│   └── fixtures/
│       ├── sample_transactions.csv
│       ├── sample_ofx.ofx
│       └── plaid_webhook_payloads.json
├── frontend/
│   ├── package.json
│   ├── next.config.ts
│   ├── tsconfig.json
│   ├── tailwind.config.ts
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx              # Landing / login
│   │   │   ├── (auth)/
│   │   │   │   ├── login/page.tsx
│   │   │   │   └── register/page.tsx
│   │   │   └── (dashboard)/
│   │   │       ├── layout.tsx        # Authenticated shell
│   │   │       ├── page.tsx          # Dashboard home
│   │   │       ├── accounts/
│   │   │       ├── transactions/
│   │   │       ├── budget/
│   │   │       ├── goals/
│   │   │       ├── net-worth/
│   │   │       ├── insights/
│   │   │       ├── coaching/
│   │   │       └── settings/
│   │   ├── components/
│   │   │   ├── ui/                   # shadcn/ui primitives
│   │   │   ├── dashboard/
│   │   │   ├── transactions/
│   │   │   ├── budget/
│   │   │   ├── charts/
│   │   │   └── coaching/
│   │   ├── lib/
│   │   │   ├── api-client.ts         # Typed fetch wrapper from OpenAPI spec
│   │   │   ├── auth.ts               # NextAuth config
│   │   │   └── utils.ts
│   │   └── types/
│   │       └── api.ts                # Generated from OpenAPI spec
│   ├── tests/
│   │   ├── components/
│   │   └── e2e/
│   └── public/
└── scripts/
    ├── seed_categories.py            # Default category hierarchy
    ├── generate_api_types.sh         # OpenAPI → TypeScript types
    └── create_dev_data.py            # Development seed data
```

---

## Phase 1: Foundation — Project Setup, Configuration & Database Schema

### Purpose
Establish the project skeleton with all tooling, configuration, database schema, and CI pipeline. After this phase, a developer can clone the repo, run `docker-compose up`, have PostgreSQL and Redis running, execute migrations to create all tables, and run the test suite. No business logic yet — this is pure infrastructure.

### Tasks

#### 1.1 — Python Project Initialisation

**What**: Create the Python project with uv, configure all development tools, and establish the module structure.

**Design**:

`pyproject.toml` dependencies (core):
```toml
[project]
name = "personal-finance-management"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "sqlalchemy>=2.0",
    "alembic>=1.14",
    "psycopg[binary]>=3.2",
    "pydantic>=2.10",
    "pydantic-settings>=2.6",
    "celery[redis]>=5.4",
    "redis>=5.2",
    "httpx>=0.28",
    "pynacl>=1.5",
    "python-jose[cryptography]>=3.3",
    "passlib[bcrypt]>=1.7",
    "python-multipart>=0.0.18",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3",
    "pytest-asyncio>=0.25",
    "pytest-cov>=6.0",
    "factory-boy>=3.3",
    "httpx>=0.28",
    "ruff>=0.8",
    "mypy>=1.13",
    "sqlalchemy[mypy]>=2.0",
]
```

`src/config.py` — Pydantic Settings:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Application
    app_name: str = "Personal Finance Management"
    debug: bool = False
    api_prefix: str = "/api/v1"
    secret_key: str  # Required — no default

    # Database
    database_url: str = "postgresql+psycopg://pfm:pfm@localhost:5432/pfm"
    database_pool_size: int = 10
    database_max_overflow: int = 20

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Encryption
    encryption_key: str  # 32-byte hex key for AES-256-GCM

    # Celery
    celery_broker_url: str = "redis://localhost:6379/1"
    celery_result_backend: str = "redis://localhost:6379/2"

    # LLM
    anthropic_api_key: str = ""
    llm_model_categorisation: str = "claude-sonnet-4-20250514"
    llm_model_coaching: str = "claude-sonnet-4-20250514"

    # Plaid
    plaid_client_id: str = ""
    plaid_secret: str = ""
    plaid_environment: str = "sandbox"  # 'sandbox', 'development', 'production'

    # Auth
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 30

    model_config = {"env_file": ".env", "env_prefix": "PFM_"}
```

Ruff configuration in `pyproject.toml`:
```toml
[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "W", "I", "N", "UP", "S", "B", "A", "C4", "DTZ", "T20", "RUF"]
ignore = ["S101"]  # Allow assert in tests

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["sqlalchemy.ext.mypy.plugin", "pydantic.mypy"]
```

**Testing**:
- Unit: `uv sync` completes without errors
- Unit: `ruff check src/` passes with zero violations on empty module structure
- Unit: `mypy src/` passes with zero errors on empty module structure
- Unit: `pytest --co` collects zero tests without errors (test discovery works)

---

#### 1.2 — Docker & docker-compose Configuration

**What**: Create Docker configuration for the full development stack.

**Design**:

`docker-compose.yml`:
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: pfm
      POSTGRES_PASSWORD: pfm
      POSTGRES_DB: pfm
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pfm"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src

  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src

volumes:
  pgdata:
```

`Dockerfile.api`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev
COPY src/ src/
COPY alembic/ alembic/
COPY alembic.ini .
CMD ["uv", "run", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Testing**:
- Integration: `docker-compose up -d db redis` starts PostgreSQL and Redis; health checks pass within 30 seconds
- Integration: `docker-compose build api worker` completes without errors
- Integration: API container starts and responds to `GET /health` with `200 OK`

---

#### 1.3 — Database Schema & Migrations

**What**: Implement the complete Hybrid Relational + JSONB schema (Data Model Suggestion 3) as SQLAlchemy ORM models with Alembic migrations.

**Design**:

SQLAlchemy model for the core `User` entity (pattern for all models):
```python
# src/models/user.py
from __future__ import annotations
import uuid
from datetime import datetime
from sqlalchemy import String, Boolean, DateTime, func
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship
from src.database import Base

class User(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    display_name: Mapped[str] = mapped_column(String(255), nullable=False)
    password_hash: Mapped[str | None] = mapped_column(String(255))
    email_verified: Mapped[bool] = mapped_column(Boolean, default=False)
    preferred_currency: Mapped[str] = mapped_column(String(3), default="USD")
    locale: Mapped[str] = mapped_column(String(10), default="en-US")
    timezone: Mapped[str] = mapped_column(String(50), default="UTC")
    preferences: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    household_memberships: Mapped[list[HouseholdMember]] = relationship(back_populates="user")
```

SQLAlchemy model for `Transaction` with JSONB `extra` field:
```python
# src/models/transaction.py
class Transaction(Base):
    __tablename__ = "transactions"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    household_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("households.id", ondelete="CASCADE"), nullable=False)
    account_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("accounts.id", ondelete="CASCADE"), nullable=False)
    category_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("categories.id", ondelete="SET NULL"))
    date: Mapped[date] = mapped_column(Date, nullable=False)
    amount: Mapped[Decimal] = mapped_column(Numeric(15, 2), nullable=False)
    currency_code: Mapped[str] = mapped_column(String(3), default="USD")
    payee_name: Mapped[str | None] = mapped_column(String(500))
    merchant_name: Mapped[str | None] = mapped_column(String(255))
    description: Mapped[str | None] = mapped_column(Text)
    memo: Mapped[str | None] = mapped_column(Text)
    cleared_status: Mapped[str] = mapped_column(String(20), default="uncleared")
    is_pending: Mapped[bool] = mapped_column(Boolean, default=False)
    is_split_parent: Mapped[bool] = mapped_column(Boolean, default=False)
    parent_transaction_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("transactions.id", ondelete="CASCADE"))
    transfer_pair_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("transactions.id", ondelete="SET NULL"))
    flag_color: Mapped[str | None] = mapped_column(String(20))
    tags: Mapped[list[str]] = mapped_column(ARRAY(String), default=list)
    ai_category_confidence: Mapped[Decimal | None] = mapped_column(Numeric(3, 2))
    ai_categorized_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    user_recategorized_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    extra: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
```

All 19 tables from Data Model Suggestion 3 must be implemented as ORM models:
1. `users` — User identity and preferences (JSONB)
2. `households` — Tenant boundary with settings (JSONB)
3. `household_members` — User-household membership with permissions (JSONB)
4. `accounts` — Financial accounts with aggregator_data (JSONB)
5. `categories` — Self-referencing hierarchy with category_mapping (JSONB)
6. `transactions` — Core transaction ledger with extra (JSONB) and tags (ARRAY)
7. `recurring_patterns` — Detected subscriptions with detection_data (JSONB)
8. `budgets` — Budget definitions with config (JSONB)
9. `budget_periods` — Monthly budget periods
10. `budget_allocations` — Per-category budget amounts with goal_target (JSONB)
11. `goals` — Financial goals with modeling_data (JSONB)
12. `investment_holdings` — Securities with security_data (JSONB)
13. `net_worth_snapshots` — Point-in-time net worth with breakdown (JSONB)
14. `ai_conversations` — Coaching conversation metadata with context (JSONB)
15. `ai_messages` — Individual messages with tool_calls (JSONB)
16. `ai_insights` — Proactive insights with insight_data (JSONB)
17. `categorisation_rules` — AI learning rules with rule_definition (JSONB)
18. `import_jobs` — Data import tracking with results (JSONB)
19. `api_tokens` — Public API authentication

Database session factory in `src/database.py`:
```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

engine = create_async_engine(settings.database_url, pool_size=settings.database_pool_size)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session
```

Index strategy — all indexes from Data Model Suggestion 3:
- `idx_accounts_household` on `accounts(household_id)`
- `idx_accounts_type` on `accounts(household_id, account_type)`
- `idx_accounts_aggregator` GIN on `accounts(aggregator_data)`
- `idx_categories_household` on `categories(household_id)`
- `idx_categories_parent` on `categories(parent_id)`
- `idx_categories_mapping` GIN on `categories(category_mapping)`
- `idx_tx_household_date` on `transactions(household_id, date DESC)`
- `idx_tx_account_date` on `transactions(account_id, date DESC)`
- `idx_tx_category` on `transactions(category_id)`
- `idx_tx_pending` partial on `transactions(account_id, is_pending) WHERE is_pending = TRUE`
- `idx_tx_parent` partial on `transactions(parent_transaction_id) WHERE parent_transaction_id IS NOT NULL`
- `idx_tx_tags` GIN on `transactions(tags)`
- `idx_tx_extra` GIN on `transactions(extra)`
- `idx_tx_payee` on `transactions(household_id, payee_name)`
- Plus all remaining indexes from the data model suggestion

**Testing**:
- Unit: All 19 ORM model classes can be imported without errors
- Integration: `alembic upgrade head` creates all 19 tables in a clean database
- Integration: `alembic downgrade base` drops all tables cleanly
- Integration: `alembic upgrade head && alembic check` reports no pending migrations
- Unit: Each model's `__tablename__` matches the expected table name
- Integration: Insert a User, Household, HouseholdMember, Account, and Transaction; verify foreign key constraints are enforced
- Integration: Insert a transaction with JSONB `extra` containing Plaid-style location data; query it back using JSONB operators
- Integration: Insert transactions with tags; query using `@>` array operator

---

#### 1.4 — FastAPI Application Shell & Health Endpoint

**What**: Create the FastAPI application factory with middleware, CORS, and a health check endpoint.

**Design**:

```python
# src/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

def create_app() -> FastAPI:
    app = FastAPI(
        title="Personal Finance Management API",
        version="0.1.0",
        docs_url="/api/docs",
        openapi_url="/api/openapi.json",
    )
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["http://localhost:3000"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    app.include_router(health_router)
    return app

app = create_app()
```

Health endpoint:
```python
# GET /health
# Response: {"status": "ok", "database": "connected", "redis": "connected", "version": "0.1.0"}
```

The health endpoint pings PostgreSQL (`SELECT 1`) and Redis (`PING`) to verify connectivity.

**Testing**:
- Integration: `GET /health` returns `200` with `status: "ok"` when DB and Redis are running
- Integration: `GET /health` returns `503` with `database: "disconnected"` when DB is down
- Integration: `GET /api/docs` returns the Swagger UI HTML page
- Integration: `GET /api/openapi.json` returns a valid OpenAPI 3.1 spec

---

#### 1.5 — CI Pipeline

**What**: Create a GitHub Actions workflow that runs on every push and PR.

**Design**:

`.github/workflows/ci.yml`:
```yaml
name: CI
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run ruff check src/ tests/
      - run: uv run ruff format --check src/ tests/
      - run: uv run mypy src/

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_USER: pfm, POSTGRES_PASSWORD: pfm, POSTGRES_DB: pfm_test }
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run pytest tests/ --cov=src --cov-report=term-missing
        env:
          PFM_DATABASE_URL: postgresql+psycopg://pfm:pfm@localhost:5432/pfm_test
          PFM_SECRET_KEY: test-secret-key-not-for-production
          PFM_ENCRYPTION_KEY: "0000000000000000000000000000000000000000000000000000000000000000"

  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose build
```

**Testing**:
- Integration: Push to a branch triggers the workflow; all three jobs (lint, test, docker) pass

---

#### 1.6 — Default Category Seed Data

**What**: Create a seed script that populates the default category hierarchy matching Plaid's Personal Finance Category (PFC) taxonomy.

**Design**:

`scripts/seed_categories.py` creates the following category group / category hierarchy for a household:

```python
DEFAULT_CATEGORIES = {
    "Income": {
        "is_income": True,
        "categories": ["Salary", "Freelance", "Investment Income", "Refunds", "Other Income"],
    },
    "Housing": {
        "categories": ["Rent", "Mortgage", "Home Insurance", "Property Tax", "Home Maintenance", "Utilities"],
    },
    "Transportation": {
        "categories": ["Gas", "Car Payment", "Car Insurance", "Public Transit", "Parking", "Ride Share"],
    },
    "Food & Drink": {
        "categories": ["Groceries", "Restaurants", "Coffee Shops", "Fast Food", "Alcohol & Bars"],
    },
    "Shopping": {
        "categories": ["Clothing", "Electronics", "Home Goods", "Personal Care", "Gifts"],
    },
    "Entertainment": {
        "categories": ["Streaming Services", "Movies & TV", "Music", "Games", "Hobbies", "Events"],
    },
    "Health & Fitness": {
        "categories": ["Doctor", "Dentist", "Pharmacy", "Gym & Fitness", "Vision", "Mental Health"],
    },
    "Financial": {
        "categories": ["Bank Fees", "Interest", "ATM Fees", "Late Fees", "Financial Advisor"],
    },
    "Education": {
        "categories": ["Tuition", "Books", "Student Loans", "Courses"],
    },
    "Travel": {
        "categories": ["Flights", "Hotels", "Car Rental", "Vacation"],
    },
    "Subscriptions": {
        "categories": ["Software", "News", "Cloud Storage", "Other Subscriptions"],
    },
    "Insurance": {
        "categories": ["Life Insurance", "Health Insurance", "Disability Insurance"],
    },
    "Taxes": {
        "categories": ["Federal Tax", "State Tax", "Property Tax"],
    },
    "Transfers": {
        "is_system": True,
        "categories": ["Account Transfer", "Credit Card Payment"],
    },
}
```

Each category row includes `category_mapping` JSONB with Plaid PFC v2 codes:
```json
{
    "plaid_pfc_v2": {
        "primary": "FOOD_AND_DRINK",
        "detailed": ["FOOD_AND_DRINK_GROCERIES", "FOOD_AND_DRINK_SUPERMARKETS_AND_GROCERIES"]
    }
}
```

The script is idempotent — it skips categories that already exist (matched by `household_id` + `name`).

**Testing**:
- Unit: Running the seed script on a household creates the expected number of categories (14 groups + ~55 child categories)
- Unit: Running the seed script twice produces no duplicates
- Unit: Each category has valid `category_mapping` JSONB matching Plaid PFC codes

---

## Phase 2: Authentication, User Management & Household Setup

### Purpose
Enable user registration, login, and household creation so that all subsequent features have proper authentication and tenant isolation. After this phase, users can create accounts, log in, create or join households, and all API endpoints are protected by JWT authentication with household-scoped authorization.

### Tasks

#### 2.1 — User Registration & Login

**What**: Implement email/password registration, login with JWT access/refresh tokens, and password hashing.

**Design**:

Pydantic schemas:
```python
class UserCreate(BaseModel):
    email: EmailStr
    display_name: str = Field(min_length=1, max_length=255)
    password: str = Field(min_length=8, max_length=128)
    preferred_currency: str = Field(default="USD", pattern=r"^[A-Z]{3}$")

class UserLogin(BaseModel):
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int  # seconds

class UserResponse(BaseModel):
    id: uuid.UUID
    email: str
    display_name: str
    preferred_currency: str
    locale: str
    timezone: str
    email_verified: bool
    created_at: datetime
```

API endpoints:
```
POST /api/v1/auth/register        → UserResponse (201)
POST /api/v1/auth/login           → TokenResponse (200)
POST /api/v1/auth/refresh         → TokenResponse (200)
POST /api/v1/auth/logout          → 204
GET  /api/v1/auth/me              → UserResponse (200)
PATCH /api/v1/auth/me             → UserResponse (200)
```

Password hashing: bcrypt via passlib with cost factor 12.

JWT tokens: `python-jose` with HS256 algorithm. Access token payload:
```json
{
    "sub": "<user_id>",
    "exp": "<30min from now>",
    "type": "access"
}
```

Refresh token: 30-day expiry, stored as `type: "refresh"` in payload.

Auth dependency for all protected endpoints:
```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    payload = decode_jwt(token)
    if payload.get("type") != "access":
        raise HTTPException(401, "Invalid token type")
    user = await db.get(User, payload["sub"])
    if not user:
        raise HTTPException(401, "User not found")
    return user
```

**Testing**:
- Unit: `hash_password("test123")` produces a bcrypt hash; `verify_password("test123", hash)` returns True
- Unit: `create_access_token(user_id)` produces a JWT with correct sub, exp, and type claims
- Unit: Expired token raises `HTTPException(401)`
- Integration: `POST /auth/register` with valid data creates user, returns 201 with UserResponse
- Integration: `POST /auth/register` with duplicate email returns 409 Conflict
- Integration: `POST /auth/register` with password < 8 chars returns 422 Validation Error
- Integration: `POST /auth/login` with correct credentials returns TokenResponse with valid JWT
- Integration: `POST /auth/login` with wrong password returns 401
- Integration: `GET /auth/me` with valid access token returns the user's profile
- Integration: `GET /auth/me` without token returns 401
- Integration: `POST /auth/refresh` with valid refresh token returns new access token
- Integration: `POST /auth/refresh` with expired refresh token returns 401

---

#### 2.2 — Household Management

**What**: Implement household creation, member invitation, and household-scoped authorization.

**Design**:

Pydantic schemas:
```python
class HouseholdCreate(BaseModel):
    name: str = Field(min_length=1, max_length=255)
    default_currency: str = Field(default="USD", pattern=r"^[A-Z]{3}$")
    country_code: str = Field(default="US", pattern=r"^[A-Z]{2}$")

class HouseholdResponse(BaseModel):
    id: uuid.UUID
    name: str
    default_currency: str
    country_code: str
    owner_user_id: uuid.UUID
    member_count: int
    created_at: datetime

class HouseholdMemberResponse(BaseModel):
    user_id: uuid.UUID
    display_name: str
    email: str
    role: str  # 'owner', 'member', 'viewer'
    accepted_at: datetime | None

class InviteMember(BaseModel):
    email: EmailStr
    role: str = Field(default="member", pattern=r"^(member|viewer)$")
```

API endpoints:
```
POST   /api/v1/households              → HouseholdResponse (201)
GET    /api/v1/households               → list[HouseholdResponse] (200)
GET    /api/v1/households/{id}          → HouseholdResponse (200)
PATCH  /api/v1/households/{id}          → HouseholdResponse (200)
GET    /api/v1/households/{id}/members  → list[HouseholdMemberResponse] (200)
POST   /api/v1/households/{id}/members  → HouseholdMemberResponse (201)
DELETE /api/v1/households/{id}/members/{user_id} → 204
```

Business rules:
- Creating a household automatically adds the creator as `owner`.
- A user can belong to multiple households.
- Only the `owner` can invite members or change household settings.
- All subsequent data operations require a `household_id` header or path parameter.
- Household-scoped authorization dependency:

```python
async def get_current_household(
    household_id: uuid.UUID,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> tuple[User, Household]:
    membership = await db.execute(
        select(HouseholdMember)
        .where(HouseholdMember.household_id == household_id)
        .where(HouseholdMember.user_id == user.id)
    )
    member = membership.scalar_one_or_none()
    if not member:
        raise HTTPException(403, "Not a member of this household")
    household = await db.get(Household, household_id)
    return user, household
```

**Testing**:
- Integration: `POST /households` creates a household; creator is automatically added as owner
- Integration: `GET /households` returns only households the user belongs to
- Integration: `POST /households/{id}/members` with owner token adds a new member
- Integration: `POST /households/{id}/members` with non-owner token returns 403
- Integration: A user who is not a member of household X cannot access `GET /households/X`
- Integration: Removing a member prevents their subsequent access to the household
- Unit: `get_current_household` raises 403 for non-members

---

## Phase 3: Account Management & Manual Transactions

### Purpose
Enable users to add financial accounts (manually or via bank sync stub), view account balances, and create/edit/delete transactions. This is the first phase with real financial data entry. After this phase, users can track their spending manually — the foundation for budgeting and AI features.

### Tasks

#### 3.1 — Account CRUD

**What**: Implement account creation, listing, updating, and soft-deletion.

**Design**:

Pydantic schemas:
```python
class AccountCreate(BaseModel):
    name: str = Field(min_length=1, max_length=255)
    account_type: Literal["checking", "savings", "credit_card", "loan", "investment", "mortgage", "other"]
    account_subtype: str | None = None
    currency_code: str = Field(default="USD", pattern=r"^[A-Z]{3}$")
    current_balance: Decimal = Decimal("0.00")
    credit_limit: Decimal | None = None
    interest_rate: Decimal | None = None
    is_manual: bool = True

class AccountUpdate(BaseModel):
    name: str | None = None
    is_hidden: bool | None = None
    display_order: int | None = None
    current_balance: Decimal | None = None

class AccountResponse(BaseModel):
    id: uuid.UUID
    name: str
    official_name: str | None
    account_type: str
    account_subtype: str | None
    currency_code: str
    current_balance: Decimal | None
    available_balance: Decimal | None
    credit_limit: Decimal | None
    mask: str | None
    is_hidden: bool
    is_manual: bool
    display_order: int
    last_synced_at: datetime | None
    created_at: datetime
```

API endpoints:
```
POST   /api/v1/households/{hid}/accounts           → AccountResponse (201)
GET    /api/v1/households/{hid}/accounts           → list[AccountResponse] (200)
GET    /api/v1/households/{hid}/accounts/{id}      → AccountResponse (200)
PATCH  /api/v1/households/{hid}/accounts/{id}      → AccountResponse (200)
DELETE /api/v1/households/{hid}/accounts/{id}       → 204 (soft-delete: sets closed_at)
```

All account operations are scoped to the authenticated user's household.

**Testing**:
- Integration: `POST /accounts` with valid data creates account and returns 201
- Integration: `GET /accounts` returns only accounts belonging to the household
- Integration: `GET /accounts` excludes hidden accounts unless `?include_hidden=true`
- Integration: `PATCH /accounts/{id}` updates name and balance
- Integration: `DELETE /accounts/{id}` sets `closed_at` but does not delete the row
- Integration: Account from household A is not accessible from household B

---

#### 3.2 — Transaction CRUD

**What**: Implement manual transaction creation, listing with filtering/pagination, editing, and deletion.

**Design**:

Pydantic schemas:
```python
class TransactionCreate(BaseModel):
    account_id: uuid.UUID
    date: date
    amount: Decimal  # negative = outflow, positive = inflow
    payee_name: str | None = None
    category_id: uuid.UUID | None = None
    memo: str | None = None
    tags: list[str] = []
    is_pending: bool = False

class TransactionUpdate(BaseModel):
    date: date | None = None
    amount: Decimal | None = None
    payee_name: str | None = None
    category_id: uuid.UUID | None = None
    memo: str | None = None
    cleared_status: Literal["uncleared", "cleared", "reconciled"] | None = None
    tags: list[str] | None = None
    flag_color: str | None = None

class TransactionResponse(BaseModel):
    id: uuid.UUID
    account_id: uuid.UUID
    date: date
    amount: Decimal
    currency_code: str
    payee_name: str | None
    merchant_name: str | None
    category_id: uuid.UUID | None
    category_name: str | None  # denormalized for display
    memo: str | None
    cleared_status: str
    is_pending: bool
    is_split_parent: bool
    flag_color: str | None
    tags: list[str]
    ai_category_confidence: float | None
    created_at: datetime

class TransactionList(BaseModel):
    items: list[TransactionResponse]
    total: int
    page: int
    page_size: int
    has_more: bool
```

API endpoints:
```
POST   /api/v1/households/{hid}/transactions                → TransactionResponse (201)
GET    /api/v1/households/{hid}/transactions                → TransactionList (200)
GET    /api/v1/households/{hid}/transactions/{id}           → TransactionResponse (200)
PATCH  /api/v1/households/{hid}/transactions/{id}           → TransactionResponse (200)
DELETE /api/v1/households/{hid}/transactions/{id}           → 204
```

Query parameters for `GET /transactions`:
- `account_id` — filter by account (UUID)
- `category_id` — filter by category (UUID)
- `date_from` / `date_to` — date range filter
- `min_amount` / `max_amount` — amount range filter
- `search` — full-text search on payee_name, merchant_name, memo
- `tags` — filter by tag (comma-separated)
- `cleared_status` — filter by status
- `page` / `page_size` — pagination (default: page=1, page_size=50, max=200)
- `sort` — `date_desc` (default), `date_asc`, `amount_desc`, `amount_asc`

When a transaction is created, `account.current_balance` is updated accordingly.

**Testing**:
- Integration: `POST /transactions` creates a transaction and updates account balance
- Integration: Creating a transaction with amount -50.00 on an account with balance 100.00 results in balance 50.00
- Integration: `GET /transactions?account_id=X` returns only transactions for that account
- Integration: `GET /transactions?date_from=2026-01-01&date_to=2026-01-31` returns only January transactions
- Integration: `GET /transactions?tags=groceries` returns only transactions tagged "groceries"
- Integration: `GET /transactions?search=whole+foods` matches payee_name containing "whole foods"
- Integration: Pagination returns correct `total`, `has_more`, and `page` values
- Integration: `DELETE /transactions/{id}` removes the transaction and adjusts account balance
- Unit: Transaction amount precision: `Decimal("0.01")` through `Decimal("9999999999999.99")` are all stored and retrieved correctly

---

#### 3.3 — Split Transactions

**What**: Implement split transactions where a parent transaction is broken into multiple child transactions with different categories.

**Design**:

```python
class SplitTransactionCreate(BaseModel):
    account_id: uuid.UUID
    date: date
    total_amount: Decimal
    payee_name: str | None = None
    memo: str | None = None
    splits: list[SplitItem] = Field(min_length=2)

class SplitItem(BaseModel):
    amount: Decimal
    category_id: uuid.UUID | None = None
    memo: str | None = None
```

API endpoint:
```
POST /api/v1/households/{hid}/transactions/split → TransactionResponse (201)
```

Business rules:
- Sum of split amounts must equal total_amount (validated in Pydantic)
- Parent transaction has `is_split_parent=True` and `amount=total_amount`
- Child transactions reference `parent_transaction_id`
- Editing a child recalculates the parent if needed
- Deleting the parent cascades to children

**Testing**:
- Integration: Creating a split with two items produces 1 parent + 2 child transactions
- Unit: Split amounts summing to a different total than `total_amount` returns 422
- Integration: Child transactions reference the parent via `parent_transaction_id`
- Integration: Deleting the parent deletes all children
- Integration: `GET /transactions` returns parent with `is_split_parent=true`; children available via `GET /transactions/{parent_id}/splits`

---

#### 3.4 — Category CRUD

**What**: Implement category management with the self-referencing hierarchy.

**Design**:

```python
class CategoryCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    parent_id: uuid.UUID | None = None  # None = top-level group
    icon: str | None = None
    color: str | None = Field(None, pattern=r"^#[0-9a-fA-F]{6}$")

class CategoryResponse(BaseModel):
    id: uuid.UUID
    name: str
    parent_id: uuid.UUID | None
    display_order: int
    is_system: bool
    is_income: bool
    is_hidden: bool
    icon: str | None
    color: str | None
    children: list[CategoryResponse] | None  # populated when requesting tree
```

API endpoints:
```
POST   /api/v1/households/{hid}/categories           → CategoryResponse (201)
GET    /api/v1/households/{hid}/categories           → list[CategoryResponse] (200)
GET    /api/v1/households/{hid}/categories/tree      → list[CategoryResponse] (200, nested)
PATCH  /api/v1/households/{hid}/categories/{id}      → CategoryResponse (200)
DELETE /api/v1/households/{hid}/categories/{id}       → 204
```

System categories (`is_system=True`) cannot be deleted or renamed.

**Testing**:
- Integration: `POST /categories` creates a top-level group when `parent_id` is null
- Integration: `POST /categories` with `parent_id` creates a child category
- Integration: `GET /categories/tree` returns a nested structure with groups containing children
- Integration: Deleting a system category returns 403
- Integration: Deleting a group cascades to child categories (ON DELETE CASCADE)

---

## Phase 4: Budgeting Engine

### Purpose
Implement the core budgeting system supporting zero-based, envelope, and 50/30/20 methods. After this phase, users can create monthly budgets, allocate money to categories, and see budget-vs-actual comparisons. This is the highest-engagement feature for daily users.

### Tasks

#### 4.1 — Budget Creation & Period Management

**What**: Implement budget creation and monthly period generation.

**Design**:

```python
class BudgetCreate(BaseModel):
    name: str = Field(default="My Budget", max_length=255)
    method: Literal["zero_based", "envelope", "50_30_20"] = "zero_based"
    currency_code: str = Field(default="USD", pattern=r"^[A-Z]{3}$")

class BudgetPeriodResponse(BaseModel):
    id: uuid.UUID
    month: date  # first of month
    income_total: Decimal
    ready_to_assign: Decimal
    age_of_money_days: int | None
    allocations: list[BudgetAllocationResponse]
```

API endpoints:
```
POST  /api/v1/households/{hid}/budgets                          → BudgetResponse (201)
GET   /api/v1/households/{hid}/budgets                          → list[BudgetResponse] (200)
GET   /api/v1/households/{hid}/budgets/{bid}/months/{month}     → BudgetPeriodResponse (200)
```

When a budget period is requested for a month that does not exist, auto-create it with carryover from the previous month (for `zero_based` and `envelope` methods).

**Testing**:
- Integration: `POST /budgets` creates a budget with `method=zero_based`
- Integration: `GET /budgets/{bid}/months/2026-05` auto-creates the May 2026 period
- Integration: Auto-created period carries over available amounts from April 2026
- Unit: `50_30_20` method validates that allocations respect the 50/30/20 ratio

---

#### 4.2 — Budget Allocation & Tracking

**What**: Implement budget allocation per category per month, with real-time activity tracking from transactions.

**Design**:

```python
class AllocateBudget(BaseModel):
    category_id: uuid.UUID
    budgeted: Decimal

class BudgetAllocationResponse(BaseModel):
    category_id: uuid.UUID
    category_name: str
    category_group_name: str
    budgeted: Decimal
    activity: Decimal       # sum of transactions in this category this month
    available: Decimal      # budgeted + carryover + activity
    carryover: Decimal
    is_overspent: bool
    goal_target: dict | None  # from JSONB
```

API endpoints:
```
PUT   /api/v1/households/{hid}/budgets/{bid}/months/{month}/allocations  → list[BudgetAllocationResponse] (200)
GET   /api/v1/households/{hid}/budgets/{bid}/months/{month}/allocations  → list[BudgetAllocationResponse] (200)
```

`PUT` accepts a list of `AllocateBudget` items and upserts allocations. The `activity` field is computed by summing transactions for the category in the given month. `available = carryover + budgeted + activity` (activity is negative for outflows).

The `ready_to_assign` for a month is calculated as:
```
ready_to_assign = income_total - sum(budgeted for all categories)
```

Business logic in `BudgetService`:
```python
async def compute_allocation(
    self, household_id: UUID, category_id: UUID, month: date
) -> BudgetAllocationComputed:
    # 1. Get budgeted amount from budget_allocations
    # 2. Sum transactions for this category in this month
    # 3. Get carryover from previous month's available
    # 4. available = carryover + budgeted + activity
    ...
```

**Testing**:
- Integration: Allocating $500 to "Groceries" for May 2026 creates/updates the allocation row
- Integration: A transaction of -$42.50 in "Groceries" in May 2026 appears as `activity: -42.50`
- Integration: `available` correctly computes as `carryover + budgeted + activity`
- Integration: `ready_to_assign` decreases when allocations are made
- Integration: Overspent category (available < 0) has `is_overspent: true`
- Unit: Carryover from previous month is correctly applied (positive available carries forward; negative available is handled per method)
- Unit: `zero_based` method subtracts overspend from next month's ready_to_assign
- Unit: `envelope` method does not subtract overspend (envelopes are independent)

---

#### 4.3 — Budget Summary & Spending Plan View

**What**: Implement the "Spending Plan" dashboard view showing income minus committed amounts equals available-to-spend.

**Design**:

```python
class SpendingPlanResponse(BaseModel):
    month: date
    total_income: Decimal
    total_bills: Decimal          # sum of budgeted amounts for "needs" categories
    total_savings_goals: Decimal  # sum of budgeted amounts linked to goals
    available_to_spend: Decimal   # income - bills - savings
    category_groups: list[CategoryGroupSummary]

class CategoryGroupSummary(BaseModel):
    group_name: str
    budgeted: Decimal
    activity: Decimal
    available: Decimal
    categories: list[BudgetAllocationResponse]
```

API endpoint:
```
GET /api/v1/households/{hid}/budgets/{bid}/months/{month}/summary → SpendingPlanResponse (200)
```

This endpoint aggregates allocations by category group and computes the spending plan formula.

**Testing**:
- Integration: Summary correctly sums category groups
- Integration: `available_to_spend` = income - bills - savings goals
- Integration: Adding a new transaction updates the next summary request's activity
- Unit: Empty month (no allocations) returns zeroes for all fields

---

## Phase 5: Bank Sync — Plaid Integration

### Purpose
Connect real bank accounts via Plaid to import transactions automatically. This replaces manual entry for users who want automated tracking. After this phase, users can link their bank accounts and see transactions appear automatically.

### Tasks

#### 5.1 — Plaid Link Token & Item Creation

**What**: Implement the Plaid Link flow: generate a link token, handle the public token exchange, and store the encrypted access token.

**Design**:

Aggregator interface (abstract base):
```python
# src/aggregators/base.py
from abc import ABC, abstractmethod

class AggregatorAdapter(ABC):
    @abstractmethod
    async def create_link_token(self, user_id: str, household_id: str) -> str: ...

    @abstractmethod
    async def exchange_public_token(self, public_token: str) -> AggregatorConnection: ...

    @abstractmethod
    async def sync_transactions(self, connection: AggregatorConnection) -> TransactionSyncResult: ...

    @abstractmethod
    async def get_accounts(self, connection: AggregatorConnection) -> list[AggregatorAccount]: ...
```

Plaid adapter:
```python
# src/aggregators/plaid_adapter.py
class PlaidAdapter(AggregatorAdapter):
    def __init__(self, client_id: str, secret: str, environment: str):
        self.client = PlaidApi(ApiClient(Configuration(
            host=PlaidEnvironment[environment],
            api_key={"clientId": client_id, "secret": secret, "plaidVersion": "2020-09-14"},
        )))

    async def create_link_token(self, user_id: str, household_id: str) -> str:
        response = self.client.link_token_create(LinkTokenCreateRequest(
            user=LinkTokenCreateRequestUser(client_user_id=user_id),
            client_name="Personal Finance Management",
            products=[Products("transactions")],
            country_codes=[CountryCode("US")],
            language="en",
        ))
        return response.link_token

    async def exchange_public_token(self, public_token: str) -> AggregatorConnection:
        response = self.client.item_public_token_exchange(
            ItemPublicTokenExchangeRequest(public_token=public_token)
        )
        # Encrypt access_token before storage
        encrypted_token = encrypt(response.access_token)
        return AggregatorConnection(
            aggregator="plaid",
            aggregator_item_id=response.item_id,
            access_token_encrypted=encrypted_token,
        )
```

API endpoints:
```
POST /api/v1/households/{hid}/sync/link-token      → {"link_token": "link-sandbox-..."}
POST /api/v1/households/{hid}/sync/exchange-token   → list[AccountResponse]
```

The exchange endpoint:
1. Exchanges the public token for an access token
2. Encrypts the access token with AES-256-GCM
3. Stores it in the `accounts.aggregator_data` JSONB field
4. Fetches accounts from Plaid and creates `Account` rows
5. Returns the list of created accounts

**Testing**:
- Integration (mocked Plaid): `POST /sync/link-token` returns a link token string
- Integration (mocked Plaid): `POST /sync/exchange-token` creates Account rows with aggregator_data populated
- Unit: Access token is encrypted before storage; decrypting yields the original token
- Unit: Invalid public token returns 400 with Plaid error code
- Integration (mocked Plaid): Multiple accounts from one Plaid item create multiple Account rows

---

#### 5.2 — Transaction Sync

**What**: Implement incremental transaction sync using Plaid's `/transactions/sync` endpoint.

**Design**:

```python
# Called by Celery task or API endpoint
async def sync_transactions(self, connection: AggregatorConnection) -> TransactionSyncResult:
    access_token = decrypt(connection.access_token_encrypted)
    cursor = connection.aggregator_data.get("last_cursor")

    added, modified, removed = [], [], []
    has_more = True

    while has_more:
        response = self.client.transactions_sync(TransactionsSyncRequest(
            access_token=access_token,
            cursor=cursor,
        ))
        added.extend(response.added)
        modified.extend(response.modified)
        removed.extend(response.removed)
        cursor = response.next_cursor
        has_more = response.has_more

    # Update cursor in aggregator_data
    connection.aggregator_data["last_cursor"] = cursor

    return TransactionSyncResult(added=added, modified=modified, removed=removed)
```

Transaction mapping from Plaid to internal model:
```python
def map_plaid_transaction(plaid_tx, account_id: UUID, household_id: UUID) -> Transaction:
    return Transaction(
        household_id=household_id,
        account_id=account_id,
        date=plaid_tx.date,
        amount=Decimal(str(-plaid_tx.amount)),  # Plaid uses positive for outflows
        payee_name=plaid_tx.merchant_name or plaid_tx.name,
        merchant_name=plaid_tx.merchant_name,
        description=plaid_tx.name,
        is_pending=plaid_tx.pending,
        cleared_status="uncleared" if plaid_tx.pending else "cleared",
        extra={
            "aggregator_id": plaid_tx.transaction_id,
            "personal_finance_category": plaid_tx.personal_finance_category.dict() if plaid_tx.personal_finance_category else None,
            "location": plaid_tx.location.dict() if plaid_tx.location else None,
            "payment_channel": plaid_tx.payment_channel,
        },
    )
```

Deduplication: transactions are matched by `extra->>'aggregator_id'` within an account. Modified transactions update existing rows. Removed transactions are soft-deleted.

API endpoint:
```
POST /api/v1/households/{hid}/sync/{account_id}  → TransactionSyncResult (200)
```

Celery task for background sync:
```python
@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def sync_account_transactions(self, account_id: str):
    ...
```

**Testing**:
- Integration (mocked Plaid): Sync imports 50 transactions from Plaid sandbox
- Integration (mocked Plaid): Incremental sync (with cursor) imports only new transactions
- Integration (mocked Plaid): Modified transactions update existing rows (matched by aggregator_id)
- Integration (mocked Plaid): Removed transactions are soft-deleted
- Unit: `map_plaid_transaction` correctly inverts Plaid's amount sign convention
- Unit: `map_plaid_transaction` stores location and PFC data in the `extra` JSONB
- Integration (mocked Plaid): Pending transactions are imported with `is_pending=True`
- Integration (mocked Plaid): Account balance is updated after sync

---

#### 5.3 — Plaid Webhook Handler

**What**: Implement webhook processing for real-time transaction updates.

**Design**:

```
POST /api/v1/webhooks/plaid  → 200
```

Webhook verification: validate the JWT in the `Plaid-Verification` header using Plaid's public key.

Supported webhook types:
- `TRANSACTIONS` / `SYNC_UPDATES_AVAILABLE` — enqueue a sync task
- `ITEM` / `ERROR` — update account status, generate an insight
- `ITEM` / `PENDING_EXPIRATION` — notify user to re-authenticate

```python
async def handle_plaid_webhook(payload: dict):
    webhook_type = payload["webhook_type"]
    webhook_code = payload["webhook_code"]

    if webhook_type == "TRANSACTIONS" and webhook_code == "SYNC_UPDATES_AVAILABLE":
        item_id = payload["item_id"]
        account = await find_account_by_plaid_item(item_id)
        sync_account_transactions.delay(str(account.id))

    elif webhook_type == "ITEM" and webhook_code == "ERROR":
        item_id = payload["item_id"]
        error = payload["error"]
        await update_account_status(item_id, status="error", error=error)
```

**Testing**:
- Integration (mocked): Valid webhook triggers a sync task
- Integration (mocked): Invalid webhook signature returns 401
- Integration (mocked): `ITEM` / `ERROR` webhook updates account status
- Unit: Webhook payload parsing extracts item_id and webhook_code correctly

---

## Phase 6: AI Transaction Categorisation

### Purpose
Implement AI-powered automatic transaction categorisation using Claude. This is the first AI-native feature and a primary differentiator. After this phase, imported transactions are automatically categorised with confidence scores, and user corrections improve future accuracy.

### Tasks

#### 6.1 — AI Categoriser

**What**: Build the categorisation pipeline that assigns categories to uncategorised transactions using Claude.

**Design**:

```python
# src/ai/categoriser.py
import anthropic

class TransactionCategoriser:
    def __init__(self, client: anthropic.AsyncAnthropic, model: str):
        self.client = client
        self.model = model

    async def categorise_batch(
        self,
        transactions: list[Transaction],
        categories: list[Category],
        rules: list[CategorisationRule],
        recent_corrections: list[tuple[str, str, str]],  # (merchant, old_cat, new_cat)
    ) -> list[CategorisationResult]:
        # Step 1: Apply rule-based matching first
        rule_matched, unmatched = self._apply_rules(transactions, rules)

        # Step 2: For unmatched, call Claude
        if unmatched:
            llm_results = await self._categorise_with_llm(unmatched, categories, recent_corrections)
        else:
            llm_results = []

        return rule_matched + llm_results
```

Prompt template for categorisation:
```python
# src/ai/prompts/categorisation.py
SYSTEM_PROMPT = """You are a personal finance transaction categoriser. Given a list of transactions
and a list of available categories, assign each transaction to the most appropriate category.

Rules:
1. Use ONLY the category IDs provided. Never invent categories.
2. For each transaction, return the category_id and a confidence score (0.0-1.0).
3. If uncertain (confidence < 0.5), return category_id as null.
4. Consider the merchant name, description, and amount when categorising.
5. Learn from the user's recent corrections shown below.

Available categories:
{categories_json}

Recent user corrections (learn from these patterns):
{corrections_json}
"""

USER_PROMPT = """Categorise these transactions:

{transactions_json}

Return a JSON array with objects: {"transaction_id": "...", "category_id": "...", "confidence": 0.XX}
"""
```

Batching: transactions are categorised in batches of 50 to stay within token limits. Each batch includes the full category list and the 20 most recent user corrections for context.

Result type:
```python
@dataclass
class CategorisationResult:
    transaction_id: uuid.UUID
    category_id: uuid.UUID | None
    confidence: float  # 0.0 to 1.0
    source: str  # "rule" or "ai"
```

**Testing**:
- Unit (mocked LLM): Categoriser assigns "Groceries" to a transaction with merchant "Whole Foods" with confidence > 0.8
- Unit (mocked LLM): Categoriser returns null category for ambiguous transactions with confidence < 0.5
- Unit: Rule-based matching applies before LLM — transaction matching a rule skips the LLM call
- Unit: Batch of 100 transactions is split into 2 LLM calls of 50 each
- Integration (mocked LLM): Categorised transactions have `ai_category_confidence` and `ai_categorized_at` set
- Unit: Prompt includes the user's recent corrections for learning

---

#### 6.2 — User Correction & Learning Loop

**What**: When a user recategorises a transaction, create a categorisation rule that improves future accuracy.

**Design**:

When `PATCH /transactions/{id}` changes `category_id` and the transaction was AI-categorised:
1. Set `user_recategorized_at` on the transaction
2. Create or update a `CategorisationRule`:

```python
async def record_correction(
    self, transaction: Transaction, new_category_id: UUID
):
    # Create a rule: merchant_name → new_category
    merchant = transaction.merchant_name or transaction.payee_name
    if merchant:
        rule = CategorisationRule(
            household_id=transaction.household_id,
            category_id=new_category_id,
            priority=10,  # User corrections take priority over AI
            rule_definition={
                "type": "merchant_pattern",
                "conditions": [
                    {"field": "merchant_name", "op": "contains", "value": merchant.lower()}
                ],
                "source": "user_correction",
                "learned_from_transaction_id": str(transaction.id),
            },
        )
        # Upsert — if a rule for this merchant already exists, update the category
        await self.upsert_rule(rule)

    # Retroactive application: find other transactions with the same merchant
    # that have lower AI confidence, and offer to recategorise them
    similar = await self.find_similar_uncorrected(transaction, new_category_id)
    return similar  # returned to frontend for user confirmation
```

API endpoint:
```
POST /api/v1/households/{hid}/transactions/{id}/recategorise → RecategoriseResponse
```

`RecategoriseResponse` includes `similar_transactions: list[TransactionResponse]` — transactions with the same merchant that the user can bulk-recategorise.

**Testing**:
- Integration: Recategorising a transaction creates a CategorisationRule
- Integration: New transactions from the same merchant are matched by the rule in subsequent categorisation
- Integration: Rule priority ensures user corrections override AI suggestions
- Integration: Retroactive suggestions return transactions with the same merchant
- Unit: Rules are scoped to the household — household A's corrections don't affect household B

---

#### 6.3 — Background Categorisation Task

**What**: Celery task that runs after every bank sync to categorise new transactions.

**Design**:

```python
@celery_app.task
def categorise_new_transactions(household_id: str):
    # 1. Fetch all transactions where category_id IS NULL and ai_categorized_at IS NULL
    # 2. Fetch household categories and rules
    # 3. Fetch recent corrections (last 50)
    # 4. Run categoriser.categorise_batch()
    # 5. Update transactions with results
    # 6. Log categorisation metrics (count, avg confidence, rule vs AI breakdown)
```

Triggered automatically after `sync_account_transactions` completes:
```python
@celery_app.task
def sync_account_transactions(account_id: str):
    result = sync(account_id)
    if result.added:
        categorise_new_transactions.delay(result.household_id)
```

**Testing**:
- Integration (mocked LLM): After sync, uncategorised transactions are categorised within the Celery task
- Integration: Transactions that already have a category are not re-categorised
- Integration: Categorisation task is idempotent — running it twice does not change already-categorised transactions

---

## Phase 7: Recurring Transaction Detection & Subscription Management

### Purpose
Automatically detect recurring transactions (subscriptions, bills, income) and alert users to anomalies like price increases and forgotten subscriptions. This is the second AI-native feature and directly addresses an underserved area identified in the feature survey.

### Tasks

#### 7.1 — Recurring Pattern Detection Engine

**What**: Analyse transaction history to identify recurring patterns using frequency analysis and LLM assistance.

**Design**:

```python
# src/services/recurring_service.py
class RecurringDetectionService:
    async def detect_patterns(self, household_id: UUID) -> list[RecurringPattern]:
        # 1. Group transactions by normalized payee_name
        # 2. For each group with 3+ transactions, analyze:
        #    - Date intervals (are they regular? weekly/monthly/quarterly?)
        #    - Amount consistency (same amount or close?)
        #    - Is it income or expense?
        # 3. Score each potential pattern on regularity
        # 4. For ambiguous cases, use LLM to classify

        groups = await self._group_by_payee(household_id)
        patterns = []
        for payee, transactions in groups.items():
            if len(transactions) < 3:
                continue

            intervals = self._compute_intervals(transactions)
            frequency = self._classify_frequency(intervals)
            if frequency is None:
                continue

            amount_stats = self._compute_amount_stats(transactions)
            confidence = self._score_pattern(frequency, amount_stats, intervals)

            if confidence >= 0.7:
                pattern = RecurringPattern(
                    household_id=household_id,
                    account_id=transactions[0].account_id,
                    payee_name=payee,
                    expected_amount=amount_stats.median,
                    amount_variance=amount_stats.std_dev,
                    frequency=frequency,
                    next_expected_date=self._predict_next(transactions, frequency),
                    is_income=amount_stats.median > 0,
                    is_subscription=self._is_subscription(payee, amount_stats),
                    auto_detected=True,
                    detection_data={
                        "confidence": confidence,
                        "sample_transaction_ids": [str(t.id) for t in transactions[-5:]],
                        "amount_history": [float(t.amount) for t in transactions[-12:]],
                        "date_history": [t.date.isoformat() for t in transactions[-12:]],
                    },
                )
                patterns.append(pattern)
        return patterns

    def _classify_frequency(self, intervals_days: list[int]) -> str | None:
        median_interval = statistics.median(intervals_days)
        if 5 <= median_interval <= 9:
            return "weekly"
        elif 12 <= median_interval <= 16:
            return "biweekly"
        elif 26 <= median_interval <= 35:
            return "monthly"
        elif 85 <= median_interval <= 100:
            return "quarterly"
        elif 350 <= median_interval <= 380:
            return "yearly"
        return None
```

API endpoints:
```
GET  /api/v1/households/{hid}/recurring                    → list[RecurringPatternResponse] (200)
POST /api/v1/households/{hid}/recurring/detect             → list[RecurringPatternResponse] (200)
PATCH /api/v1/households/{hid}/recurring/{id}              → RecurringPatternResponse (200)
DELETE /api/v1/households/{hid}/recurring/{id}              → 204
```

**Testing**:
- Unit: 6 monthly transactions at $15.99 detected as `frequency=monthly` with confidence > 0.8
- Unit: 3 transactions with irregular intervals (15, 45, 22 days) returns None frequency
- Unit: Weekly transactions (7-day intervals) detected as `frequency=weekly`
- Unit: Income transactions (positive amounts) marked as `is_income=True`
- Unit: `next_expected_date` prediction is within 3 days of the actual pattern
- Integration: `POST /recurring/detect` creates RecurringPattern rows in the database
- Integration: Existing patterns are not duplicated on re-detection

---

#### 7.2 — Anomaly Detection (Fee Creep & Forgotten Subscriptions)

**What**: Detect subscription price increases and subscriptions with no corresponding usage.

**Design**:

```python
class AnomalyDetector:
    async def detect_anomalies(self, household_id: UUID) -> list[AIInsight]:
        patterns = await self.get_active_patterns(household_id)
        insights = []

        for pattern in patterns:
            # Fee creep: compare recent amount to historical median
            amounts = pattern.detection_data.get("amount_history", [])
            if len(amounts) >= 3:
                historical_median = statistics.median(amounts[:-1])
                latest = amounts[-1]
                if latest > historical_median * 1.05:  # 5% increase threshold
                    fee_increase = latest - historical_median
                    insights.append(AIInsight(
                        household_id=household_id,
                        insight_type="fee_creep",
                        title=f"{pattern.payee_name} price increased",
                        description=f"Your {pattern.payee_name} charge increased from "
                                    f"${historical_median:.2f} to ${latest:.2f} "
                                    f"(+${fee_increase:.2f}/month, ${fee_increase * 12:.2f}/year).",
                        severity="warning",
                        insight_data={
                            "related_entity_type": "recurring_pattern",
                            "related_entity_id": str(pattern.id),
                            "old_amount": float(historical_median),
                            "new_amount": float(latest),
                            "annual_impact": float(fee_increase * 12),
                            "confidence": 0.9,
                        },
                    ))

            # Forgotten subscription: no transaction in 2x the expected interval
            if pattern.next_expected_date and pattern.next_expected_date < date.today() - timedelta(days=self._interval_days(pattern.frequency)):
                # Subscription may have been cancelled by bank but not by user
                insights.append(AIInsight(
                    household_id=household_id,
                    insight_type="subscription_alert",
                    title=f"Missing {pattern.payee_name} charge",
                    description=f"Expected a {pattern.payee_name} charge around "
                                f"{pattern.next_expected_date} but none appeared. "
                                f"This may have been cancelled, or the charge is late.",
                    severity="info",
                    insight_data={...},
                ))

        return insights
```

**Testing**:
- Unit: Price increase from $15.99 to $17.99 generates a "fee_creep" insight
- Unit: Price decrease does not generate an insight
- Unit: Missing charge past 2x the interval generates a "subscription_alert"
- Unit: Annual impact calculation: $2.00/month increase = $24.00/year
- Integration: Generated insights are stored in the `ai_insights` table
- Integration: Duplicate insights (same pattern, same type, within 30 days) are deduplicated

---

#### 7.3 — Insights API

**What**: API endpoints for reading, dismissing, and acting on AI-generated insights.

**Design**:

```python
class InsightResponse(BaseModel):
    id: uuid.UUID
    insight_type: str
    title: str
    description: str
    severity: str  # 'info', 'warning', 'critical'
    is_read: bool
    is_dismissed: bool
    insight_data: dict
    created_at: datetime
```

API endpoints:
```
GET    /api/v1/households/{hid}/insights              → list[InsightResponse] (200)
GET    /api/v1/households/{hid}/insights/unread-count  → {"count": N} (200)
PATCH  /api/v1/households/{hid}/insights/{id}          → InsightResponse (200)
POST   /api/v1/households/{hid}/insights/{id}/dismiss  → 204
```

Query parameters for `GET /insights`:
- `insight_type` — filter by type
- `severity` — filter by severity
- `is_read` — filter read/unread
- `limit` / `offset` — pagination

**Testing**:
- Integration: `GET /insights` returns insights ordered by `created_at DESC`
- Integration: `GET /insights?is_read=false` returns only unread insights
- Integration: `GET /insights/unread-count` returns the correct count
- Integration: `PATCH /insights/{id}` with `is_read: true` marks as read
- Integration: `POST /insights/{id}/dismiss` sets `is_dismissed: true` and excludes from future queries

---

## Phase 8: Net Worth Tracking & Cash Flow Forecasting

### Purpose
Implement the net worth dashboard and forward-looking cash flow projections. After this phase, users can see their total financial picture and get warnings about upcoming cash shortfalls.

### Tasks

#### 8.1 — Net Worth Snapshots

**What**: Generate daily net worth snapshots and provide a historical net worth API.

**Design**:

Celery beat task runs nightly:
```python
@celery_app.task
def generate_net_worth_snapshot(household_id: str):
    accounts = get_all_accounts(household_id)

    total_assets = sum(a.current_balance for a in accounts if a.account_type in ASSET_TYPES and a.current_balance)
    total_liabilities = sum(abs(a.current_balance) for a in accounts if a.account_type in LIABILITY_TYPES and a.current_balance)

    breakdown = {
        "by_type": {},
        "by_account": [],
    }
    for account in accounts:
        if account.is_hidden or account.closed_at:
            continue
        type_key = account.account_type
        breakdown["by_type"][type_key] = breakdown["by_type"].get(type_key, 0) + float(account.current_balance or 0)
        breakdown["by_account"].append({
            "id": str(account.id),
            "name": account.name,
            "balance": float(account.current_balance or 0),
            "type": account.account_type,
        })

    snapshot = NetWorthSnapshot(
        household_id=household_id,
        snapshot_date=date.today(),
        total_assets=total_assets,
        total_liabilities=total_liabilities,
        net_worth=total_assets - total_liabilities,
        breakdown=breakdown,
    )
    upsert(snapshot)  # UNIQUE(household_id, snapshot_date) handles idempotency
```

API endpoints:
```
GET /api/v1/households/{hid}/net-worth                → NetWorthCurrentResponse (200)
GET /api/v1/households/{hid}/net-worth/history         → list[NetWorthSnapshotResponse] (200)
```

Query parameters for history: `from_date`, `to_date`, `interval` (`daily`, `weekly`, `monthly`).

**Testing**:
- Integration: Snapshot correctly sums asset and liability accounts
- Unit: Credit card balance of -$4,200 counted as $4,200 liability
- Unit: Hidden and closed accounts excluded from snapshot
- Integration: Net worth history endpoint returns data points for the requested date range
- Integration: Monthly interval aggregates daily snapshots to month-end values
- Unit: Idempotent — running twice on the same day updates the existing snapshot

---

#### 8.2 — Cash Flow Forecasting

**What**: Project future account balances based on recurring transactions and historical spending patterns.

**Design**:

```python
# src/ai/forecaster.py
class CashFlowForecaster:
    async def generate_forecast(
        self, household_id: UUID, days_ahead: int = 90
    ) -> list[ForecastDay]:
        # 1. Get current total balance across checking/savings accounts
        current_balance = await self._get_liquid_balance(household_id)

        # 2. Get all active recurring patterns
        recurring = await self._get_recurring_patterns(household_id)

        # 3. Get historical monthly spending by category (last 6 months)
        monthly_avg = await self._get_monthly_spending_average(household_id)

        # 4. Project forward day by day
        forecast = []
        balance = current_balance
        for day_offset in range(days_ahead):
            forecast_date = date.today() + timedelta(days=day_offset)
            daily_inflows = Decimal("0")
            daily_outflows = Decimal("0")

            # Add known recurring items due on this date
            for pattern in recurring:
                if self._is_due_on(pattern, forecast_date):
                    if pattern.is_income:
                        daily_inflows += pattern.expected_amount
                    else:
                        daily_outflows += abs(pattern.expected_amount)

            # Distribute average discretionary spending across weekdays
            if forecast_date.weekday() < 5:  # weekday spending
                daily_outflows += monthly_avg.daily_discretionary

            balance = balance + daily_inflows - daily_outflows
            forecast.append(ForecastDay(
                date=forecast_date,
                projected_balance=balance,
                projected_inflows=daily_inflows,
                projected_outflows=daily_outflows,
                has_shortfall_risk=balance < Decimal("0"),
            ))

        return forecast
```

API endpoint:
```
GET /api/v1/households/{hid}/forecast?days=90 → list[ForecastDayResponse] (200)
```

Shortfall detection: if any day in the forecast has `projected_balance < 0`, generate an `AIInsight` with `severity=critical`.

**Testing**:
- Unit: Recurring income of $5,000 on the 15th appears on the correct forecast day
- Unit: Recurring expense of $2,100 rent on the 1st appears correctly
- Unit: Balance below zero triggers `has_shortfall_risk=True`
- Unit: 90-day forecast produces exactly 90 data points
- Integration: Shortfall generates a critical insight
- Integration: Forecast endpoint returns data in chronological order

---

## Phase 9: Financial Coaching — LLM-Powered Conversations

### Purpose
Implement the natural language financial assistant that answers questions about the user's financial data and provides proactive coaching. This is the centerpiece AI feature. After this phase, users can ask questions like "How much did I spend on dining out last quarter?" and receive contextual, data-grounded answers.

### Tasks

#### 9.1 — Coaching Conversation Engine

**What**: Build the conversation engine that grounds LLM responses in the user's actual financial data.

**Design**:

```python
# src/ai/coach.py
class FinancialCoach:
    def __init__(self, client: anthropic.AsyncAnthropic):
        self.client = client

    async def chat(
        self,
        household_id: UUID,
        user_id: UUID,
        conversation_id: UUID | None,
        message: str,
    ) -> CoachingResponse:
        # 1. Load or create conversation
        conversation = await self._get_or_create_conversation(conversation_id, user_id, household_id)

        # 2. Build context: recent transactions, budget status, goals, insights
        context = await self._build_financial_context(household_id)

        # 3. Build message history
        history = await self._get_message_history(conversation.id)

        # 4. Call Claude with tools for data retrieval
        response = await self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            system=COACHING_SYSTEM_PROMPT.format(
                current_date=date.today().isoformat(),
                financial_summary=context.summary,
            ),
            messages=history + [{"role": "user", "content": message}],
            tools=FINANCIAL_TOOLS,
        )

        # 5. Handle tool use (data queries)
        while response.stop_reason == "tool_use":
            tool_results = await self._execute_tools(response, household_id)
            response = await self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=2048,
                system=COACHING_SYSTEM_PROMPT.format(...),
                messages=history + [
                    {"role": "user", "content": message},
                    {"role": "assistant", "content": response.content},
                    {"role": "user", "content": tool_results},
                ],
                tools=FINANCIAL_TOOLS,
            )

        # 6. Save messages and return
        await self._save_messages(conversation.id, message, response)
        return CoachingResponse(
            conversation_id=conversation.id,
            message=response.content[0].text,
        )
```

Coaching system prompt:
```python
COACHING_SYSTEM_PROMPT = """You are a personal financial coach. You have access to the user's
actual financial data through tools. Always ground your advice in their real numbers.

Guidelines:
- Be specific: "You spent $847 on dining out in April, up 23% from March" not "You spent a lot on dining out"
- Be actionable: Suggest concrete changes with dollar amounts
- Be empathetic: Money is emotional. Acknowledge feelings before giving advice.
- Never recommend specific investments or financial products — you are not a licensed advisor.
- When asked about data, use tools to retrieve actual figures. Never estimate or hallucinate numbers.

Current date: {current_date}

User's financial summary:
{financial_summary}
"""
```

Tool definitions for Claude:
```python
FINANCIAL_TOOLS = [
    {
        "name": "query_transactions",
        "description": "Search the user's transaction history with filters",
        "input_schema": {
            "type": "object",
            "properties": {
                "date_from": {"type": "string", "format": "date"},
                "date_to": {"type": "string", "format": "date"},
                "category_name": {"type": "string"},
                "payee_name": {"type": "string"},
                "min_amount": {"type": "number"},
                "max_amount": {"type": "number"},
            },
        },
    },
    {
        "name": "get_spending_summary",
        "description": "Get total spending by category for a date range",
        "input_schema": {
            "type": "object",
            "properties": {
                "date_from": {"type": "string", "format": "date"},
                "date_to": {"type": "string", "format": "date"},
            },
            "required": ["date_from", "date_to"],
        },
    },
    {
        "name": "get_budget_status",
        "description": "Get the current month's budget allocations and spending",
        "input_schema": {"type": "object", "properties": {}},
    },
    {
        "name": "get_goal_progress",
        "description": "Get progress on all financial goals",
        "input_schema": {"type": "object", "properties": {}},
    },
    {
        "name": "get_net_worth_trend",
        "description": "Get net worth over a date range",
        "input_schema": {
            "type": "object",
            "properties": {
                "date_from": {"type": "string", "format": "date"},
                "date_to": {"type": "string", "format": "date"},
            },
        },
    },
]
```

API endpoints:
```
POST  /api/v1/households/{hid}/coaching/chat            → CoachingResponse (200)
GET   /api/v1/households/{hid}/coaching/conversations    → list[ConversationSummary] (200)
GET   /api/v1/households/{hid}/coaching/conversations/{id}/messages → list[MessageResponse] (200)
DELETE /api/v1/households/{hid}/coaching/conversations/{id}         → 204
```

**Testing**:
- Integration (mocked LLM): "How much did I spend on groceries last month?" triggers `query_transactions` tool and returns a data-grounded response
- Integration (mocked LLM): Conversation history is persisted and included in subsequent messages
- Integration (mocked LLM): Tool use loop executes the tool and feeds results back to the LLM
- Unit: Financial context builder summarises account balances, budget status, and goal progress
- Unit: Tool execution is scoped to the user's household — cannot access other households' data
- Integration: Creating a new conversation generates a title from the first message
- Integration (mocked LLM): Response includes actual dollar amounts from the database, not hallucinated figures

---

#### 9.2 — Goal Management & Trade-off Modelling

**What**: Implement financial goals with AI-powered scenario modelling.

**Design**:

```python
class GoalCreate(BaseModel):
    name: str = Field(max_length=255)
    goal_type: Literal["savings", "debt_payoff", "emergency_fund", "custom"]
    target_amount: Decimal = Field(gt=0)
    target_date: date | None = None
    linked_account_id: uuid.UUID | None = None
    linked_category_id: uuid.UUID | None = None
    monthly_contribution_target: Decimal | None = None
    priority: int = Field(default=0, ge=0)

class GoalResponse(BaseModel):
    id: uuid.UUID
    name: str
    goal_type: str
    target_amount: Decimal
    current_amount: Decimal
    currency_code: str
    target_date: date | None
    monthly_contribution_target: Decimal | None
    percent_complete: float
    on_track: bool
    projected_completion_date: date | None
    status: str
    modeling_data: dict
```

API endpoints:
```
POST   /api/v1/households/{hid}/goals             → GoalResponse (201)
GET    /api/v1/households/{hid}/goals             → list[GoalResponse] (200)
GET    /api/v1/households/{hid}/goals/{id}        → GoalResponse (200)
PATCH  /api/v1/households/{hid}/goals/{id}        → GoalResponse (200)
DELETE /api/v1/households/{hid}/goals/{id}        → 204
POST   /api/v1/households/{hid}/goals/model       → GoalModelResponse (200)
```

Goal modelling endpoint accepts all goals and returns trade-off recommendations:
```python
class GoalModelRequest(BaseModel):
    monthly_available: Decimal  # total available for goals
    scenarios: list[ScenarioInput] | None = None

class GoalModelResponse(BaseModel):
    recommended_allocation: list[GoalAllocation]
    scenarios: list[Scenario]
    trade_off_summary: str  # LLM-generated natural language summary

class GoalAllocation(BaseModel):
    goal_id: uuid.UUID
    monthly_contribution: Decimal
    projected_completion_date: date
    months_to_goal: int
```

The modelling service uses the LLM to generate natural language trade-off explanations:
```
"Allocating $500/month to your emergency fund and $300/month to your house deposit 
means the emergency fund is fully funded by November 2026, then $800/month can shift 
to the house deposit, reaching your $40,000 target by March 2028."
```

**Testing**:
- Integration: Creating a goal with `target_amount=10000` and `current_amount=0` shows `percent_complete=0.0`
- Integration: Manual contribution updates `current_amount` and `percent_complete`
- Unit: `on_track` calculation: current_amount >= expected_amount_by_now based on linear projection
- Unit: `projected_completion_date` based on average monthly contribution rate
- Integration (mocked LLM): Goal model endpoint returns allocation recommendations
- Integration (mocked LLM): Trade-off summary is a natural language explanation with specific dollar amounts

---

## Phase 10: Frontend — Next.js Dashboard

### Purpose
Build the web frontend that presents all backend features in a clean, modern dashboard. After this phase, users have a fully functional web application for managing their finances.

### Tasks

#### 10.1 — Frontend Project Setup & Authentication UI

**What**: Initialise the Next.js project with shadcn/ui, configure API client generation from OpenAPI spec, and build login/register pages.

**Design**:

API client generation:
```bash
# scripts/generate_api_types.sh
npx openapi-typescript http://localhost:8000/api/openapi.json -o frontend/src/types/api.ts
```

Typed fetch wrapper:
```typescript
// frontend/src/lib/api-client.ts
import type { paths } from "@/types/api";
import createClient from "openapi-fetch";

export const api = createClient<paths>({
  baseUrl: process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000",
});
```

NextAuth.js configuration with credentials provider (email/password) against the backend `/auth/login` endpoint.

Pages:
- `/login` — email/password form with error handling
- `/register` — registration form with password strength indicator
- Protected route middleware that redirects unauthenticated users to `/login`

**Testing**:
- E2E (Playwright): Navigate to `/login`, enter valid credentials, see dashboard
- E2E (Playwright): Navigate to `/register`, create account, redirected to dashboard
- E2E (Playwright): Unauthenticated user visiting `/` is redirected to `/login`
- Component (Vitest): Login form validates email format and password length

---

#### 10.2 — Dashboard Home, Accounts & Transactions Views

**What**: Build the main dashboard, account list, and transaction table with filtering.

**Design**:

Dashboard layout (`/(dashboard)/layout.tsx`):
- Sidebar navigation: Dashboard, Accounts, Transactions, Budget, Goals, Net Worth, Insights, Coaching, Settings
- Header: household name, user avatar, notification bell with unread insight count

Dashboard home (`/(dashboard)/page.tsx`):
- Net worth card with sparkline chart (last 30 days)
- Budget status card (this month's ready-to-assign, overspent categories)
- Recent transactions (last 10)
- Active insights (unread, ordered by severity)
- Goal progress bars

Accounts page (`/(dashboard)/accounts/page.tsx`):
- Grouped by type (Checking, Savings, Credit Cards, Loans, Investments)
- Each account shows name, balance, last sync time
- "Add Account" button (manual or Plaid Link)

Transactions page (`/(dashboard)/transactions/page.tsx`):
- Filterable table with columns: Date, Payee, Category, Amount, Account, Tags
- Filter bar: date range, category, account, search text
- Inline category editing (click category to change)
- Bulk operations: categorise, tag, delete

Key components:
```typescript
// frontend/src/components/transactions/TransactionTable.tsx
interface TransactionTableProps {
  transactions: TransactionResponse[];
  onCategoryChange: (txId: string, categoryId: string) => void;
  onBulkAction: (txIds: string[], action: BulkAction) => void;
}
```

**Testing**:
- E2E (Playwright): Dashboard loads with account balances, recent transactions, and budget summary
- E2E (Playwright): Transaction table filters by date range
- E2E (Playwright): Clicking a transaction category opens a category picker; selecting a new category updates it
- Component (Vitest): TransactionTable renders correct columns and data
- Component (Vitest): Account cards display correct balance formatting (negative balances for credit cards)

---

#### 10.3 — Budget View

**What**: Build the monthly budget view with allocation controls and spending-plan summary.

**Design**:

Budget page (`/(dashboard)/budget/page.tsx`):
- Month selector (prev/next month arrows, month picker)
- Spending plan header: Income | Bills | Savings | = Available to Spend
- Category group accordion with nested categories
- Each category row: name, budgeted (editable input), activity, available (with color: green positive, red negative)
- "Ready to Assign" banner at top (green if positive, red if negative)

Category allocation component:
```typescript
// frontend/src/components/budget/CategoryRow.tsx
interface CategoryRowProps {
  allocation: BudgetAllocationResponse;
  onBudgetChange: (categoryId: string, amount: number) => void;
}
```

Inline editing: clicking the "budgeted" cell reveals a numeric input. On blur or Enter, the value is saved via `PUT /budgets/{bid}/months/{month}/allocations`.

**Testing**:
- E2E (Playwright): Navigate to budget, see category groups with allocations
- E2E (Playwright): Change a budgeted amount; "Ready to Assign" updates in real time
- E2E (Playwright): Navigate to previous month; data loads correctly
- Component (Vitest): Overspent category row displays red background
- Component (Vitest): Spending plan summary correctly computes available-to-spend

---

#### 10.4 — Goals, Net Worth, Insights & Coaching Views

**What**: Build the remaining dashboard pages.

**Design**:

Goals page (`/(dashboard)/goals/page.tsx`):
- Card per goal: name, progress bar, current/target amount, projected completion date
- "Add Goal" form: name, type, target amount, target date, linked account
- Goal modelling panel: slider for monthly contribution, live-updating projected completion

Net Worth page (`/(dashboard)/net-worth/page.tsx`):
- Line chart: net worth over time (6m / 1y / 3y / all time)
- Stacked area chart: asset vs. liability breakdown
- Account-level breakdown table

Insights page (`/(dashboard)/insights/page.tsx`):
- Card per insight with severity indicator (info/warning/critical)
- "Mark as read" and "Dismiss" buttons
- Filter by type and severity

Coaching page (`/(dashboard)/coaching/page.tsx`):
- Chat interface: message list + input field
- Conversation sidebar: list of past conversations
- Streaming response display (progressive text rendering)
- Suggested questions: "How much did I spend last month?", "Am I on track for my savings goal?", "What subscriptions can I cancel?"

**Testing**:
- E2E (Playwright): Goals page shows progress bars with correct percentages
- E2E (Playwright): Net worth chart renders with data points
- E2E (Playwright): Insights page shows unread insights; clicking "dismiss" removes them
- E2E (Playwright): Coaching chat sends a message and receives a response
- Component (Vitest): Chat message component renders user and assistant messages with correct styling

---

## Phase 11: Data Import & Export

### Purpose
Enable users to import historical data from CSV, OFX/QFX files, and competitor exports (YNAB, Mint), and export their data for portability. This captures users with years of financial history in other tools.

### Tasks

#### 11.1 — CSV Import

**What**: Import transactions from CSV files with column mapping.

**Design**:

```python
class CSVImportRequest(BaseModel):
    account_id: uuid.UUID
    column_mapping: dict[str, str]  # {"date": "Date", "amount": "Amount", ...}
    date_format: str = "%Y-%m-%d"
    amount_sign_convention: Literal["standard", "inverted"] = "standard"

class ImportResultResponse(BaseModel):
    job_id: uuid.UUID
    status: str
    total_rows: int
    imported: int
    duplicates_skipped: int
    errors: int
    error_details: list[dict]
```

API endpoint:
```
POST /api/v1/households/{hid}/import/csv  → ImportResultResponse (201)
    Content-Type: multipart/form-data
    Body: file (CSV), config (JSON)
```

Deduplication: transactions are considered duplicates if they match on (account_id, date, amount, payee_name).

**Testing**:
- Integration: Import a 100-row CSV; verify 100 transactions created
- Integration: Import the same CSV again; verify 100 duplicates skipped
- Integration: CSV with invalid date format returns error details per row
- Integration: Column mapping correctly maps non-standard column names
- Fixture: `tests/fixtures/sample_transactions.csv` with 50 sample transactions

---

#### 11.2 — OFX/QFX Import

**What**: Import transactions from Open Financial Exchange (OFX) files commonly exported by banks and Quicken.

**Design**:

Use the `ofxparse` library to parse OFX/QFX files:
```python
async def import_ofx(file: UploadFile, account_id: UUID, household_id: UUID) -> ImportResult:
    ofx = OfxParser.parse(file.file)
    transactions = []
    for account in ofx.accounts:
        for tx in account.statement.transactions:
            transactions.append(Transaction(
                household_id=household_id,
                account_id=account_id,
                date=tx.date.date(),
                amount=Decimal(str(tx.amount)),
                payee_name=tx.payee,
                description=tx.memo,
                extra={"ofx_id": tx.id, "ofx_type": tx.type},
            ))
    return await self._bulk_insert_with_dedup(transactions)
```

API endpoint:
```
POST /api/v1/households/{hid}/import/ofx → ImportResultResponse (201)
```

**Testing**:
- Integration: Import a sample OFX file; verify transactions created with correct dates and amounts
- Fixture: `tests/fixtures/sample_ofx.ofx` with 20 sample transactions

---

#### 11.3 — CSV Export

**What**: Export all transactions for a household as a CSV file.

**Design**:

API endpoint:
```
GET /api/v1/households/{hid}/export/csv?date_from=&date_to=&account_id= → CSV file (200)
    Content-Type: text/csv
    Content-Disposition: attachment; filename="transactions_2026-05.csv"
```

CSV columns: Date, Account, Payee, Category, Amount, Memo, Tags, Cleared

**Testing**:
- Integration: Export produces a valid CSV with correct headers
- Integration: Date range filter limits exported rows
- Integration: Exported CSV can be re-imported via the CSV import endpoint

---

## Phase 12: Public API, MCP Server & Investment Tracking

### Purpose
Implement the public OAuth 2.0 API for third-party integrations, an MCP server for AI agent access, and investment portfolio tracking. These are the extensibility and advanced features that complete the v1.0 product.

### Tasks

#### 12.1 — Public API with Personal Access Tokens

**What**: Implement API token management and scoped access for third-party integrations, modelled after the YNAB API per standards.md.

**Design**:

Token scopes:
```python
VALID_SCOPES = [
    "read:accounts",
    "read:transactions",
    "write:transactions",
    "read:budgets",
    "write:budgets",
    "read:goals",
    "read:insights",
    "read:net_worth",
]
```

API endpoints:
```
POST   /api/v1/tokens           → {"token": "pfm_...", "id": "uuid"} (201)
GET    /api/v1/tokens           → list[TokenSummary] (200)
DELETE /api/v1/tokens/{id}      → 204
```

Token format: `pfm_` prefix + 32 random bytes (base62 encoded). Only the hash is stored. The plaintext token is shown once at creation.

Authentication: API tokens are accepted via `Authorization: Bearer pfm_...` header. The auth dependency checks both JWT and API token formats.

**Testing**:
- Integration: Create a token; use it to access `GET /accounts`; verify scoped access
- Integration: Token with only `read:accounts` scope cannot access `POST /transactions` (403)
- Integration: Deleted token returns 401 on subsequent use
- Unit: Token hash is irreversible — cannot recover plaintext from stored hash

---

#### 12.2 — MCP Server

**What**: Implement a Model Context Protocol (MCP) server that exposes the user's financial data to AI agents, as specified in standards.md.

**Design**:

MCP tools exposed:
- `get_accounts` — list all accounts with balances
- `get_transactions` — query transactions with filters
- `get_budget_status` — current month's budget
- `get_net_worth` — current net worth and history
- `get_goals` — goal progress
- `get_insights` — unread insights

The MCP server authenticates via API token (scope: all `read:*` scopes required).

Implementation using the `mcp` Python library:
```python
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("personal-finance-management")

@server.tool()
async def get_accounts(token: str) -> list[dict]:
    """Get all financial accounts with current balances."""
    user = await authenticate_token(token)
    accounts = await account_service.list_accounts(user.household_id)
    return [account.model_dump() for account in accounts]
```

**Testing**:
- Integration: MCP client connects and lists available tools
- Integration: `get_accounts` tool returns account data for the authenticated user
- Integration: Invalid token returns an authentication error
- Integration: Tool results contain actual financial data matching the database

---

#### 12.3 — Investment Holdings

**What**: Track investment holdings (securities, quantities, values) within investment accounts.

**Design**:

```python
class InvestmentHoldingResponse(BaseModel):
    id: uuid.UUID
    account_id: uuid.UUID
    ticker_symbol: str | None
    security_name: str
    security_type: str | None
    quantity: Decimal
    cost_basis_per_unit: Decimal | None
    current_price: Decimal | None
    current_value: Decimal | None
    gain_loss: Decimal | None
    gain_loss_pct: float | None
    security_data: dict
```

API endpoints:
```
GET  /api/v1/households/{hid}/accounts/{aid}/holdings    → list[InvestmentHoldingResponse] (200)
GET  /api/v1/households/{hid}/investments/summary        → InvestmentSummaryResponse (200)
```

Investment summary includes:
- Total portfolio value
- Total gain/loss
- Asset allocation breakdown (stocks, bonds, ETFs, cash)
- Top 10 holdings by value
- Expense ratio impact (annual cost of fund fees)

Holdings are synced from Plaid's `/investments/holdings/get` endpoint for linked accounts, or manually entered.

**Testing**:
- Integration: Holdings synced from Plaid appear with correct quantities and values
- Integration: Manual holdings can be created for non-linked accounts
- Integration: Investment summary correctly computes total portfolio value
- Unit: Gain/loss calculation: `(current_price - cost_basis_per_unit) * quantity`
- Unit: Expense ratio impact: `total_value * expense_ratio` per year

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (6 tasks)
    │
Phase 2: Auth & Households (2 tasks)  ─── requires Phase 1
    │
Phase 3: Accounts & Transactions (4 tasks)  ─── requires Phase 2
    │
    ├── Phase 4: Budgeting (3 tasks)  ─── requires Phase 3
    │       │
    ├── Phase 5: Bank Sync (3 tasks)  ─── requires Phase 3
    │       │
    │       └── Phase 6: AI Categorisation (3 tasks)  ─── requires Phase 5
    │               │
    │               └── Phase 7: Recurring & Subscriptions (3 tasks)  ─── requires Phase 6
    │
    ├── Phase 8: Net Worth & Cash Flow (2 tasks)  ─── requires Phase 3 + Phase 7
    │
    └── Phase 9: Coaching & Goals (2 tasks)  ─── requires Phase 3 + Phase 6
        │
Phase 10: Frontend (4 tasks)  ─── requires Phases 4, 5, 7, 8, 9
    │
Phase 11: Import/Export (3 tasks)  ─── can parallel with Phase 10 after Phase 3
    │
Phase 12: Public API, MCP & Investments (3 tasks)  ─── can parallel with Phase 10 after Phase 5
```

**Parallelism opportunities:**
- Phases 4 (Budgeting) and 5 (Bank Sync) can be developed concurrently after Phase 3
- Phase 11 (Import/Export) can be developed concurrently with Phase 10 (Frontend)
- Phase 12 (Public API, MCP, Investments) can be developed concurrently with Phase 10

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented with production-quality code
2. All unit tests pass (`pytest tests/unit/`)
3. All integration tests pass (`pytest tests/integration/`)
4. Ruff lint passes with zero violations (`ruff check src/ tests/`)
5. Ruff format passes (`ruff format --check src/ tests/`)
6. mypy strict mode passes (`mypy src/`)
7. Docker build succeeds (`docker compose build`)
8. The feature works end-to-end via API (manual or automated verification)
9. New configuration options are documented in `.env.example`
10. New API endpoints appear in the auto-generated OpenAPI spec (`/api/openapi.json`)
11. Database migrations are created and tested (`alembic upgrade head` + `alembic downgrade -1` + `alembic upgrade head`)
12. No secrets, credentials, or API keys are committed to the repository
13. All JSONB columns have documented example structures in the ORM model docstrings
14. Test coverage for new code is above 80%
