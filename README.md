# Backend Implementation Roadmap: Self-Management &amp; Goal Tracker

This specification outlines the backend architecture, data model, and engineering task breakdown for the self-management application using the primary recommended stack: FastAPI, Strawberry GraphQL, PostgreSQL, SQLAlchemy 2.0 (Async), `asyncpg`, Alembic, `fastapi-users`, and ARQ with Redis.

---

## 1\. Primary Technology Stack Specification

| Layer                          | Selected Technology | Package / Version                 | Primary Justification                                                                                                                                   |
| ------------------------------ | ------------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **API Framework**              | FastAPI             | `fastapi &gt;= 0.110.0`           | Asynchronous request handling, Pydantic v2 validation, lightweight dependency injection, and native async performance.                                  |
| **GraphQL Engine**             | Strawberry GraphQL  | `strawberry-graphql[fastapi]`     | Python 3 dataclass-driven code-first schema design, native async resolver support, seamless Apollo Client integration, and built-in DataLoader support. |
| **Database**                   | PostgreSQL          | `PostgreSQL 16+`                  | Native support for Recursive Common Table Expressions (CTEs), window functions for velocity analytics, and JSONB for unstructured check-in data.        |
| **ORM Layer**                  | SQLAlchemy          | `sqlalchemy &gt;= 2.0.0`          | Modern 2.0 explicit async syntax (`select()`, `AsyncSession`), mature relationship mapping, and native integration with Alembic.                        |
| **Database Driver**            | asyncpg             | `asyncpg &gt;= 0.29.0`            | High-throughput asynchronous driver for PostgreSQL.                                                                                                     |
| **Migrations**                 | Alembic             | `alembic &gt;= 1.13.0`            | Version-controlled database schema evolution compatible with async SQLAlchemy models.                                                                   |
| **Authentication &amp; AuthZ** | `fastapi-users`     | `fastapi-users[sqlalchemy,oauth]` | Turnkey authentication handling local JWT issuance/rotation, password hashing (Argon2/bcrypt), and OAuth2 providers (Google, GitHub).                   |
| **Task Queue &amp; Scheduler** | ARQ                 | `arq &gt;= 0.25.0`                | Asynchronous, lightweight job queue built on Redis for calculating end-of-day metrics, lagging-task detection, and reminders.                           |
| **Cache &amp; State Store**    | Redis               | `redis &gt;= 5.0.0`               | In-memory cache for aggregated goal metrics and broker for ARQ jobs.                                                                                    |

---

## 2\. Hierarchical Data Architecture

The application requires tracking both long-term goals (e.g., a 6-month roadmap broken into months, weeks, days) and short-term sprints (e.g., a 10-day sprint with daily check-ins).

### 2.1 Entity Relationship Model

The core relational structure uses an Adjacency List pattern for arbitrary depth hierarchy combined with targeted leaf tasks:

- **`User`**: Account identity, OAuth profiles, and global user settings.
- **`Goal`**: Top-level objective (e.g., "Full-Stack AI Engineer Role" or "10-Day Deep Work Sprint"). Defines `timeframe_type` (`FIXED_DAYS`, `MULTI_MONTH`), `start_date`, and `target_date`.
- **`PlanUnit`**: Recursive breakdown node using an Adjacency List (`parent_id` referencing another `PlanUnit`) to cleanly model `Month -&gt; Week -&gt; Day` or flat day lists (`Day 1..10`).
- **`Task`**: Concrete action item assigned to a specific `PlanUnit` or `Goal`. Holds status (`TODO`, `IN_PROGRESS`, `COMPLETED`, `SKIPPED`), priority, estimated minutes, and actual minutes.
- **`DailyCheckIn`**: User reflection responses for a given day (e.g., "What do you want to achieve today?", "What blocked you?").
- **`GoalMetricSnapshot`**: Pre-aggregated historical records capturing completion rates, task counts, and velocity over time for dashboard graphs.

---

## 3\. Task-by-Task Implementation Breakdown

### Phase 1: Environment Setup &amp; Infrastructure Baseline

#### Task 1.1: Project Scaffolding &amp; Virtual Environment

- Initialize Python 3.11+ project using `poetry` or `uv` for deterministic dependency management.
- Configure tooling configs: `ruff` (linting &amp; formatting), `mypy` (strict type checking), `pytest` with `pytest-asyncio`.
- Define standardized directory layout:
  - `app/core/`: Config, database connections, security.
  - `app/models/`: SQLAlchemy 2.0 ORM models.
  - `app/schemas/`: Pydantic schemas &amp; Strawberry GraphQL types.
  - `app/graphql/`: Queries, Mutations, Resolvers, Loaders.
  - `app/services/`: Business logic (analytics, hierarchy calculation).
  - `app/workers/`: ARQ background jobs.
  - `app/main.py`: FastAPI application entrypoint.
  - `alembic/`: Database migration scripts.

#### Task 1.2: Environment Configuration &amp; Secret Management

- Implement `app/core/config.py` using `pydantic-settings` to manage environment variables (`DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`).
- Set up Docker Compose for local development: PostgreSQL 16 container and Redis 7 container with persistent volumes and healthchecks.

---

### Phase 2: Database Schema &amp; Migration Pipeline

#### Task 2.1: Async SQLAlchemy Session Architecture

- Configure async database engine using `create_async_engine` with connection pooling (`pool_size=20`, `max_overflow=10`).
- Create `async_sessionmaker` factory for dependency-injected sessions in FastAPI and Strawberry context.

#### Task 2.2: Relational ORM Models Creation

- Implement `User` model using `fastapi-users` base classes (`SQLAlchemyBaseUserTableUUID`).
- Implement `Goal` model:
  - `id` (UUID), `user_id` (FK to `users.id`), `title`, `description`, `category`, `duration_days`, `start_date`, `target_date`, `status`, `created_at`, `updated_at`.
- Implement `PlanUnit` model (Hierarchy Node):
  - `id`, `goal_id` (FK to `goals.id`), `parent_id` (Self-referencing FK to `plan_units.id`, nullable), `unit_type` (`MONTH`, `WEEK`, `DAY`), `sequence_index` (int), `title`, `target_date`.
- Implement `Task` model:
  - `id`, `goal_id`, `plan_unit_id` (FK to `plan_units.id`), `title`, `description`, `status` (`TODO`, `IN_PROGRESS`, `COMPLETED`, `SKIPPED`), `due_date`, `completed_at`.
- Implement `DailyCheckIn` model:
  - `id`, `user_id`, `plan_unit_id`, `date`, `prompt_responses` (JSONB), `mood_rating` (int), `reflection_note` (text).
- Add composite indexes:
  - `ix_tasks_plan_unit_status` on `(plan_unit_id, status)`
  - `ix_plan_units_goal_parent` on `(goal_id, parent_id, sequence_index)`
  - `ix_checkins_user_date` on `(user_id, date)`

#### Task 2.3: Alembic Configuration &amp; Initial Migration

- Initialize Alembic environment in async mode (`env.py` using `run_migrations_online` with `asyncio`).
- Generate and verify initial baseline migration script covering all tables, indexes, and foreign keys.

---

### Phase 3: Authentication &amp; Authorization Engine

#### Task 3.1: `fastapi-users` Setup &amp; JWT Token Management

- Configure password hasher using Argon2id.
- Implement JWT authentication backend with custom claims, access token expiry, and refresh token rotation.
- Implement `UserManager` class handling user lifecycle events (registration, password resets).

#### Task 3.2: OAuth2 Integration (Google &amp; GitHub)

- Configure OAuth2 clients in FastAPI using `fastapi-users` OAuth router.
- Implement authorization callback endpoints linking OAuth accounts to internal `User` records.

#### Task 3.3: GraphQL Context &amp; Authentication Injection

- Construct custom Strawberry context provider (`get_context`) extracting the `Authorization: Bearer
