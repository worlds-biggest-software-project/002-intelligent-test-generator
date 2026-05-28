# Intelligent Test Generator — Phased Development Plan

> Project: 002-intelligent-test-generator · Created: 2026-05-11
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12+ | LLM-heavy project with mutation testing, AST parsing, and coverage analysis; Python has the richest ecosystem for all three (tree-sitter bindings, coverage.py, mutmut, openai/anthropic SDKs). The majority of target users are Python/TS/Java/Go developers — Python is the most natural host language for multi-language code analysis. |
| API framework | FastAPI 0.115+ | Native OpenAPI 3.1 schema generation (required for spec-to-test workflows), async support for LLM calls, Pydantic v2 integration for request/response validation, and first-class dependency injection for testing. |
| Database | PostgreSQL 16 (via asyncpg + SQLAlchemy 2.0 async) | The data model (27 tables with complex joins across coverage, mutations, test results) demands a relational database with strong referential integrity. PostgreSQL provides JSONB for settings columns, UUID generation, row-level security for multi-tenancy, and partitioning for high-volume coverage_lines. SQLite is insufficient for concurrent CI workloads. |
| Migrations | Alembic 1.13+ | Standard SQLAlchemy migration tool; generates versioned migration scripts from model changes; supports both online (ALTER TABLE) and offline (SQL script) modes. |
| Task queue | Celery 5.4+ with Redis broker | Test generation and mutation testing are long-running async workloads (10s–5min per session). Celery provides task routing, retry logic, and result backends. Redis is the broker for simplicity; RabbitMQ is an alternative for enterprise deployments. |
| Cache / broker | Redis 7+ | Serves as Celery broker, rate-limit store for API keys, and cache for parsed AST data. |
| LLM SDK | anthropic Python SDK (primary), litellm (adapter layer) | Claude API is the default LLM. litellm provides a unified interface for OpenAI, Gemini, and local models without code changes — important for self-hosted users who cannot use cloud LLMs. |
| Code parsing | tree-sitter (via py-tree-sitter 0.23+) | Language-agnostic AST parsing for Python, TypeScript, Java, and Go. Extracts function signatures, complexity metrics, and dependency graphs without executing the target code. Faster and more reliable than regex-based approaches. |
| Coverage tools | coverage.py (Python), istanbul/c8 (TS/JS), JaCoCo (Java), go tool cover (Go) | These are the standard coverage tools for each language. The system invokes them via subprocess and normalises output into the unified coverage schema. |
| Mutation testing | mutmut (Python), Stryker (TS/JS), PITest (Java), go-mutesting (Go) | Established mutation testing tools for each target language. Invoked via subprocess; results parsed into the Stryker-aligned mutant schema from the data model. |
| Test framework (for this project) | pytest 8+ with pytest-asyncio, pytest-httpx | Standard Python test framework. pytest-asyncio for testing async FastAPI endpoints. pytest-httpx for mocking external HTTP calls (LLM API, VCS webhooks). |
| Code quality | ruff (linter + formatter), mypy (type checker) | ruff replaces flake8+isort+black with a single fast tool. mypy enforces type safety across the codebase. Both run in CI as pre-commit hooks. |
| Containerisation | Docker + docker-compose | Self-hosted deployment requires PostgreSQL, Redis, the API server, and the Celery worker. docker-compose orchestrates all four services. Multi-stage Dockerfile keeps the image small. |
| CLI | typer 0.12+ | Type-safe CLI framework built on Click. Used for the `testgen` CLI that CI pipelines invoke. Generates shell completion and help text automatically. |
| IDE extension | VS Code Extension (TypeScript) | VS Code has the largest market share among target users. The extension communicates with the API server via REST; it does not embed Python. JetBrains plugin is deferred to post-MVP. |
| Package manager | uv (for Python), npm (for VS Code extension) | uv is the fastest Python package manager and supports lockfiles, virtual environments, and PEP 723 inline script metadata. |
| Authentication | OAuth 2.0 (GitHub, GitLab) + API key | OAuth for web/IDE login; API keys (bcrypt-hashed, scoped) for CI/CD pipelines. JWT tokens for session management. |

### Project Structure

```
intelligent-test-generator/
├── pyproject.toml                  # Project metadata, dependencies, tool config
├── uv.lock                        # Locked dependency versions
├── Dockerfile                      # Multi-stage build
├── docker-compose.yml              # PostgreSQL, Redis, API, Worker
├── alembic/                        # Database migrations
│   ├── alembic.ini
│   ├── env.py
│   └── versions/
├── src/
│   └── testgen/
│       ├── __init__.py
│       ├── main.py                 # FastAPI application factory
│       ├── config.py               # Pydantic Settings (env vars, defaults)
│       ├── cli.py                  # Typer CLI entry point
│       ├── models/                 # SQLAlchemy ORM models
│       │   ├── __init__.py
│       │   ├── organization.py
│       │   ├── user.py
│       │   ├── repository.py
│       │   ├── source.py           # source_files, source_functions
│       │   ├── specification.py
│       │   ├── generation.py       # generation_sessions, generated_tests, generated_mocks
│       │   ├── test_run.py         # test_runs, test_results
│       │   ├── coverage.py         # coverage_reports, coverage_files, coverage_lines, coverage_gaps
│       │   ├── mutation.py         # mutation_runs, mutants, mutant_test_links
│       │   ├── repair.py           # test_repairs
│       │   ├── security.py         # security_test_mappings
│       │   ├── pipeline.py         # pipeline_configurations, pipeline_runs
│       │   ├── snapshot.py         # coverage_snapshots
│       │   └── auth.py             # api_keys, webhooks
│       ├── schemas/                # Pydantic request/response schemas
│       │   ├── __init__.py
│       │   ├── repository.py
│       │   ├── generation.py
│       │   ├── coverage.py
│       │   ├── mutation.py
│       │   └── pipeline.py
│       ├── api/                    # FastAPI routers
│       │   ├── __init__.py
│       │   ├── auth.py
│       │   ├── repositories.py
│       │   ├── generation.py
│       │   ├── coverage.py
│       │   ├── mutations.py
│       │   ├── pipelines.py
│       │   └── webhooks.py
│       ├── services/               # Business logic layer
│       │   ├── __init__.py
│       │   ├── parsing.py          # tree-sitter AST parsing
│       │   ├── coverage_analysis.py # Coverage gap identification
│       │   ├── generation.py       # LLM-based test generation
│       │   ├── mutation.py         # Mutation testing orchestration
│       │   ├── repair.py           # Test repair logic
│       │   ├── spec_parser.py      # OpenAPI / Gherkin parsing
│       │   └── runner.py           # Test execution orchestration
│       ├── llm/                    # LLM abstraction layer
│       │   ├── __init__.py
│       │   ├── client.py           # LLM client interface + litellm adapter
│       │   ├── prompts.py          # Prompt templates for generation, repair, analysis
│       │   └── token_budget.py     # Token usage tracking and budgeting
│       ├── parsers/                # Language-specific parsers
│       │   ├── __init__.py
│       │   ├── base.py             # Abstract parser interface
│       │   ├── python_parser.py
│       │   ├── typescript_parser.py
│       │   ├── java_parser.py
│       │   └── go_parser.py
│       ├── runners/                # Language-specific test runners
│       │   ├── __init__.py
│       │   ├── base.py             # Abstract runner interface
│       │   ├── pytest_runner.py
│       │   ├── jest_runner.py
│       │   ├── junit_runner.py
│       │   └── go_test_runner.py
│       ├── coverage_parsers/       # Coverage format parsers
│       │   ├── __init__.py
│       │   ├── lcov.py
│       │   ├── jacoco.py
│       │   ├── coverage_py.py
│       │   └── istanbul.py
│       ├── mutation_parsers/       # Mutation tool output parsers
│       │   ├── __init__.py
│       │   ├── stryker.py
│       │   ├── pitest.py
│       │   ├── mutmut.py
│       │   └── go_mutesting.py
│       ├── tasks/                  # Celery tasks
│       │   ├── __init__.py
│       │   ├── generate.py
│       │   ├── run_tests.py
│       │   ├── coverage.py
│       │   ├── mutation.py
│       │   └── repair.py
│       └── integrations/           # External system connectors
│           ├── __init__.py
│           ├── github.py
│           ├── gitlab.py
│           └── vcs_base.py
├── tests/
│   ├── conftest.py                 # Shared fixtures (db, client, factories)
│   ├── unit/
│   │   ├── test_parsing.py
│   │   ├── test_coverage_analysis.py
│   │   ├── test_generation.py
│   │   ├── test_prompts.py
│   │   ├── test_coverage_parsers.py
│   │   └── test_mutation_parsers.py
│   ├── integration/
│   │   ├── test_api_repositories.py
│   │   ├── test_api_generation.py
│   │   ├── test_api_coverage.py
│   │   ├── test_api_pipelines.py
│   │   └── test_celery_tasks.py
│   ├── e2e/
│   │   ├── test_cli.py
│   │   └── test_generation_workflow.py
│   └── fixtures/
│       ├── python_samples/
│       ├── typescript_samples/
│       ├── java_samples/
│       ├── go_samples/
│       ├── coverage_reports/
│       └── mutation_reports/
├── vscode-extension/               # VS Code extension (Phase 8)
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   └── extension.ts
│   └── test/
└── docs/
    └── openapi-override.yaml       # OpenAPI customizations
```

---

## Phase 1: Foundation — Project Scaffolding, Configuration, and Database

### Purpose
Establish the project skeleton, dependency management, configuration system, database schema, and CI pipeline. After this phase, the project builds, lints, type-checks, runs an empty test suite, and can connect to PostgreSQL with all tables created via Alembic migrations. Every subsequent phase adds functionality on top of this foundation without restructuring.

### Tasks

#### 1.1 — Project Scaffolding and Dependency Management

**What**: Create the `pyproject.toml`, directory structure, and install core dependencies using uv.

**Design**:

`pyproject.toml` (key sections):
```toml
[project]
name = "testgen"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.30",
    "alembic>=1.13",
    "pydantic>=2.9",
    "pydantic-settings>=2.6",
    "celery[redis]>=5.4",
    "redis>=5.2",
    "anthropic>=0.42",
    "litellm>=1.55",
    "typer>=0.12",
    "tree-sitter>=0.23",
    "tree-sitter-python>=0.23",
    "tree-sitter-typescript>=0.23",
    "tree-sitter-java>=0.23",
    "tree-sitter-go>=0.23",
    "httpx>=0.28",
    "pyjwt>=2.10",
    "bcrypt>=4.2",
    "structlog>=24.4",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3",
    "pytest-asyncio>=0.24",
    "pytest-httpx>=0.35",
    "pytest-cov>=6.0",
    "ruff>=0.8",
    "mypy>=1.13",
    "factory-boy>=3.3",
]

[project.scripts]
testgen = "testgen.cli:app"
```

Configuration model (`src/testgen/config.py`):
```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="TESTGEN_", env_file=".env")

    # Database
    database_url: str = "postgresql+asyncpg://testgen:testgen@localhost:5432/testgen"

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # LLM
    llm_provider: str = "anthropic"  # or "openai", "litellm"
    llm_model: str = "claude-sonnet-4-20250514"
    llm_api_key: str = ""
    llm_max_tokens: int = 4096
    llm_temperature: float = 0.2

    # Auth
    jwt_secret: str = "change-me-in-production"
    jwt_algorithm: str = "HS256"
    jwt_expiry_minutes: int = 1440  # 24 hours

    # Server
    host: str = "0.0.0.0"
    port: int = 8000
    debug: bool = False

    # Generation defaults
    default_coverage_threshold: float = 80.0
    default_mutation_score_threshold: float = 60.0
    max_generation_retries: int = 3
```

**Testing**:
- `Unit: Settings loads from environment variables with TESTGEN_ prefix`
- `Unit: Settings uses defaults when no env vars set`
- `Unit: Settings loads from .env file when present`
- `Unit: Missing required llm_api_key raises validation warning (not error — allows CLI-only modes)`

#### 1.2 — Database Models and Migrations

**What**: Implement all 27 SQLAlchemy ORM models matching the data-model-suggestion-1 schema, and generate the initial Alembic migration.

**Design**:

The ORM models mirror the SQL DDL from `data-model-suggestion-1.md` exactly. Each model uses `mapped_column` (SQLAlchemy 2.0 style) with type annotations.

Example model (`src/testgen/models/repository.py`):
```python
from __future__ import annotations
import uuid
from datetime import datetime
from sqlalchemy import ForeignKey, String, Text, Integer, Boolean, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column, relationship
from testgen.models import Base

class Repository(Base):
    __tablename__ = "repositories"
    __table_args__ = (UniqueConstraint("organization_id", "name"),)

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    organization_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organizations.id", ondelete="CASCADE"))
    name: Mapped[str] = mapped_column(String(255))
    url: Mapped[str | None] = mapped_column(Text)
    default_branch: Mapped[str] = mapped_column(String(255), default="main")
    vcs_provider: Mapped[str] = mapped_column(String(50))  # github, gitlab, bitbucket, local
    vcs_provider_id: Mapped[str | None] = mapped_column(String(255))
    primary_language: Mapped[str | None] = mapped_column(String(50))
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    organization: Mapped["Organization"] = relationship(back_populates="repositories")
    commits: Mapped[list["Commit"]] = relationship(back_populates="repository", cascade="all, delete-orphan")
    source_files: Mapped[list["SourceFile"]] = relationship(back_populates="repository")
```

All 27 tables from the data model are implemented as ORM models. The `Base` class is defined in `src/testgen/models/__init__.py` using `DeclarativeBase`.

Alembic configuration (`alembic/env.py`) uses the async engine from the application config.

Initial migration covers all 27 tables with indexes and constraints matching the DDL.

**Testing**:
- `Unit: All 27 model classes are importable and have correct __tablename__`
- `Unit: Repository model enforces unique (organization_id, name) constraint`
- `Unit: CHECK constraints on enum-like columns reject invalid values`
- `Integration (real DB): Alembic upgrade head creates all 27 tables`
- `Integration (real DB): Alembic downgrade base drops all tables cleanly`
- `Integration (real DB): Insert Organization → Repository → Commit → SourceFile chain succeeds with FK integrity`
- `Integration (real DB): Delete Organization cascades to all child records`

#### 1.3 — FastAPI Application Factory and Health Check

**What**: Create the FastAPI app with lifespan management (DB connection pool, Redis), CORS, and a health check endpoint.

**Design**:

```python
# src/testgen/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from testgen.config import Settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = Settings()
    # Create async engine and session factory
    engine = create_async_engine(settings.database_url, pool_size=10, max_overflow=20)
    async_session_factory = async_sessionmaker(engine, expire_on_commit=False)
    app.state.db_session_factory = async_session_factory
    app.state.settings = settings
    yield
    await engine.dispose()

def create_app() -> FastAPI:
    app = FastAPI(
        title="Intelligent Test Generator",
        version="0.1.0",
        lifespan=lifespan,
    )
    app.include_router(health_router)
    return app
```

Health check endpoint:
```
GET /health
Response 200: { "status": "healthy", "database": "connected", "redis": "connected", "version": "0.1.0" }
Response 503: { "status": "unhealthy", "database": "disconnected", ... }
```

**Testing**:
- `Integration: GET /health returns 200 with status "healthy" when DB and Redis are available`
- `Integration: GET /health returns 503 when database is unreachable`
- `Unit: create_app() returns a FastAPI instance with correct title and version`

#### 1.4 — Docker and docker-compose

**What**: Create multi-stage Dockerfile and docker-compose.yml for local development and deployment.

**Design**:

```dockerfile
# Dockerfile
FROM python:3.12-slim AS base
RUN pip install uv

FROM base AS deps
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM base AS runtime
WORKDIR /app
COPY --from=deps /app/.venv /app/.venv
COPY src/ src/
COPY alembic/ alembic/
COPY alembic.ini .
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "testgen.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: testgen
      POSTGRES_PASSWORD: testgen
      POSTGRES_DB: testgen
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  api:
    build: .
    ports: ["8000:8000"]
    depends_on: [db, redis]
    environment:
      TESTGEN_DATABASE_URL: postgresql+asyncpg://testgen:testgen@db:5432/testgen
      TESTGEN_REDIS_URL: redis://redis:6379/0
    command: >
      sh -c "alembic upgrade head && uvicorn testgen.main:create_app --factory --host 0.0.0.0 --port 8000"

  worker:
    build: .
    depends_on: [db, redis]
    environment:
      TESTGEN_DATABASE_URL: postgresql+asyncpg://testgen:testgen@db:5432/testgen
      TESTGEN_REDIS_URL: redis://redis:6379/0
    command: celery -A testgen.tasks worker --loglevel=info

volumes:
  pgdata:
```

**Testing**:
- `E2E: docker-compose up --build completes without errors`
- `E2E: API container responds to GET /health with 200`
- `E2E: Database migrations run successfully on fresh PostgreSQL`
- `Unit: Dockerfile builds without errors (docker build --target runtime)`

#### 1.5 — CI Pipeline and Code Quality

**What**: Configure ruff, mypy, and pytest in pyproject.toml and create a GitHub Actions workflow.

**Design**:

ruff configuration in `pyproject.toml`:
```toml
[tool.ruff]
target-version = "py312"
line-length = 120
[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "SIM", "TCH", "RUF"]

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy", "sqlalchemy.ext.mypy.plugin"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

GitHub Actions workflow (`.github/workflows/ci.yml`):
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
        env: { POSTGRES_USER: testgen, POSTGRES_PASSWORD: testgen, POSTGRES_DB: testgen_test }
        ports: ["5432:5432"]
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run pytest --cov=testgen --cov-report=xml
```

**Testing**:
- `Unit: ruff check passes on all source files`
- `Unit: ruff format --check passes (no formatting changes needed)`
- `Unit: mypy passes with strict mode on all source files`
- `E2E: GitHub Actions workflow runs lint and test jobs successfully`

---

## Phase 2: Code Parsing and Source Analysis

### Purpose
Implement the tree-sitter-based code parsing layer that extracts functions, classes, complexity metrics, and dependency information from Python, TypeScript, Java, and Go source files. After this phase, the system can ingest a repository, parse its source files, and populate the `source_files` and `source_functions` tables with structured metadata. This is the foundation for all coverage gap analysis and test generation in later phases.

### Tasks

#### 2.1 — Abstract Parser Interface

**What**: Define the base parser interface that all language-specific parsers implement.

**Design**:

```python
# src/testgen/parsers/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass, field

@dataclass
class ParsedFunction:
    name: str
    qualified_name: str       # e.g. "MyClass.my_method"
    start_line: int
    end_line: int
    complexity: int           # cyclomatic complexity
    parameter_count: int
    is_public: bool
    docstring: str | None = None
    dependencies: list[str] = field(default_factory=list)  # imported names used in body
    decorators: list[str] = field(default_factory=list)

@dataclass
class ParsedFile:
    file_path: str
    language: str             # "python", "typescript", "java", "go"
    line_count: int
    imports: list[str]
    classes: list[str]
    functions: list[ParsedFunction]
    top_level_code_lines: int  # lines not inside any function (setup, globals)

class BaseParser(ABC):
    @abstractmethod
    def parse(self, source_code: str, file_path: str) -> ParsedFile:
        """Parse source code and return structured metadata."""
        ...

    @abstractmethod
    def supported_extensions(self) -> list[str]:
        """Return file extensions this parser handles (e.g., ['.py', '.pyi'])."""
        ...

    def _calculate_cyclomatic_complexity(self, node) -> int:
        """Count decision points (if, elif, for, while, and, or, except, case) + 1."""
        ...
```

**Testing**:
- `Unit: ParsedFunction dataclass initialises with all required fields`
- `Unit: ParsedFile correctly computes line_count from source`
- `Unit: BaseParser cannot be instantiated directly (ABC enforcement)`

#### 2.2 — Python Parser

**What**: Implement the Python source parser using tree-sitter-python.

**Design**:

```python
# src/testgen/parsers/python_parser.py
from tree_sitter import Language, Parser
import tree_sitter_python as tspython
from testgen.parsers.base import BaseParser, ParsedFile, ParsedFunction

class PythonParser(BaseParser):
    def __init__(self):
        self._language = Language(tspython.language())
        self._parser = Parser(self._language)

    def supported_extensions(self) -> list[str]:
        return [".py", ".pyi"]

    def parse(self, source_code: str, file_path: str) -> ParsedFile:
        tree = self._parser.parse(source_code.encode())
        root = tree.root_node
        functions = []
        classes = []
        imports = []

        for node in self._walk(root):
            if node.type == "function_definition":
                functions.append(self._parse_function(node, parent_class=None))
            elif node.type == "class_definition":
                class_name = self._get_name(node)
                classes.append(class_name)
                for child in self._walk(node):
                    if child.type == "function_definition":
                        functions.append(self._parse_function(child, parent_class=class_name))
            elif node.type in ("import_statement", "import_from_statement"):
                imports.append(node.text.decode())

        return ParsedFile(
            file_path=file_path,
            language="python",
            line_count=source_code.count("\n") + 1,
            imports=imports,
            classes=classes,
            functions=functions,
            top_level_code_lines=self._count_top_level_lines(root, source_code),
        )
```

Key implementation details:
- `_parse_function` extracts name, qualified_name (Class.method), parameters, decorators, docstring, and calculates cyclomatic complexity by counting `if_statement`, `elif_clause`, `for_statement`, `while_statement`, `except_clause`, `boolean_operator (and/or)`, `conditional_expression`, and `match_statement` nodes.
- `is_public` is determined by checking if the function name starts with `_` (private by Python convention).
- `dependencies` are extracted by cross-referencing identifiers in the function body against the file's import list.

**Testing**:
- `Unit: parse simple function → ParsedFunction with correct name, lines, parameter_count=0`
- `Unit: parse function with 3 parameters → parameter_count=3`
- `Unit: parse class with 2 methods → 2 ParsedFunctions with qualified_name "ClassName.method_name"`
- `Unit: cyclomatic complexity of function with if/elif/else → complexity=3`
- `Unit: function starting with underscore → is_public=False`
- `Unit: function with @staticmethod decorator → decorators=["staticmethod"]`
- `Unit: file with "import os" and "from pathlib import Path" → imports list has both`
- `Unit: empty file → ParsedFile with line_count=1, empty functions list`
- `Fixture: tests/fixtures/python_samples/calculator.py → expected ParsedFile with known function list`

#### 2.3 — TypeScript Parser

**What**: Implement the TypeScript/JavaScript source parser using tree-sitter-typescript.

**Design**:

Same structure as PythonParser but handles TypeScript-specific AST nodes:
- `function_declaration`, `arrow_function`, `method_definition` → ParsedFunction
- `class_declaration`, `interface_declaration` → classes
- `import_statement` → imports
- `is_public` determined by absence of `private` or `protected` keyword modifier
- Complexity counting includes `if_statement`, `ternary_expression`, `switch_case`, `for_statement`, `for_in_statement`, `while_statement`, `catch_clause`, `logical_expression (&&, ||)`, `optional_chain_expression (?.)`.
- Handles both `.ts` and `.tsx` extensions; `.js` and `.jsx` use JavaScript grammar variant.

**Testing**:
- `Unit: parse exported function → is_public=True`
- `Unit: parse private class method → is_public=False`
- `Unit: parse arrow function assigned to const → ParsedFunction with correct name`
- `Unit: parse interface (not a function) → appears in classes list, not functions`
- `Unit: complexity of function with ternary + optional chaining → correct count`
- `Fixture: tests/fixtures/typescript_samples/service.ts → expected ParsedFile`

#### 2.4 — Java Parser

**What**: Implement the Java source parser using tree-sitter-java.

**Design**:

Handles Java AST nodes:
- `method_declaration`, `constructor_declaration` → ParsedFunction
- `class_declaration`, `interface_declaration`, `enum_declaration` → classes
- `import_declaration` → imports
- `is_public` determined by presence of `public` modifier; absence of any access modifier defaults to package-private (treated as `is_public=True` for test generation purposes)
- Qualified names include the class hierarchy: `OuterClass.InnerClass.methodName`
- Complexity includes `if_statement`, `for_statement`, `enhanced_for_statement`, `while_statement`, `do_statement`, `catch_clause`, `switch_expression`, `ternary_expression`, `binary_expression (&&, ||)`

**Testing**:
- `Unit: parse public method → is_public=True`
- `Unit: parse private method → is_public=False`
- `Unit: parse nested class method → qualified_name="Outer.Inner.method"`
- `Unit: parse constructor → ParsedFunction with name="ClassName"`
- `Unit: parse interface method (abstract) → ParsedFunction with complexity=1`
- `Fixture: tests/fixtures/java_samples/Calculator.java → expected ParsedFile`

#### 2.5 — Go Parser

**What**: Implement the Go source parser using tree-sitter-go.

**Design**:

Handles Go AST nodes:
- `function_declaration`, `method_declaration` → ParsedFunction
- `type_declaration` (struct, interface) → classes
- `import_declaration` → imports
- `is_public` determined by first character being uppercase (Go convention)
- For methods, qualified_name is `ReceiverType.MethodName`
- Complexity includes `if_statement`, `for_statement`, `switch_statement`, `select_statement`, `case_clause`, `binary_expression (&&, ||)`

**Testing**:
- `Unit: parse exported function (uppercase) → is_public=True`
- `Unit: parse unexported function (lowercase) → is_public=False`
- `Unit: parse method with receiver → qualified_name="ReceiverType.MethodName"`
- `Unit: parse function with multiple return values → parameter_count counts inputs only`
- `Fixture: tests/fixtures/go_samples/calculator.go → expected ParsedFile`

#### 2.6 — Parser Registry and Repository Ingestion Service

**What**: Create a registry that maps file extensions to parsers, and a service that clones/reads a repository and populates the `source_files` and `source_functions` tables.

**Design**:

```python
# src/testgen/services/parsing.py
from testgen.parsers.base import BaseParser, ParsedFile
from testgen.parsers.python_parser import PythonParser
from testgen.parsers.typescript_parser import TypeScriptParser
from testgen.parsers.java_parser import JavaParser
from testgen.parsers.go_parser import GoParser

class ParserRegistry:
    def __init__(self):
        self._parsers: dict[str, BaseParser] = {}
        for parser_cls in [PythonParser, TypeScriptParser, JavaParser, GoParser]:
            parser = parser_cls()
            for ext in parser.supported_extensions():
                self._parsers[ext] = parser

    def get_parser(self, file_path: str) -> BaseParser | None:
        ext = Path(file_path).suffix
        return self._parsers.get(ext)

    def supported_extensions(self) -> list[str]:
        return list(self._parsers.keys())


class RepositoryIngestionService:
    def __init__(self, session: AsyncSession, registry: ParserRegistry):
        self._session = session
        self._registry = registry

    async def ingest(
        self,
        repository_id: uuid.UUID,
        commit_id: uuid.UUID,
        root_path: Path,
        exclude_patterns: list[str] | None = None,
    ) -> IngestionResult:
        """Walk the repository, parse all supported files, and persist to DB."""
        ...

@dataclass
class IngestionResult:
    files_parsed: int
    functions_found: int
    languages: dict[str, int]  # language → file count
    errors: list[str]          # files that failed to parse
```

The service:
1. Walks `root_path` recursively, skipping directories matching `exclude_patterns` (default: `[".git", "node_modules", "__pycache__", "vendor", ".venv", "target", "build", "dist"]`).
2. For each file with a supported extension, invokes the appropriate parser.
3. Creates `SourceFile` and `SourceFunction` records in a single transaction.
4. Returns an `IngestionResult` summary.

**Testing**:
- `Unit: ParserRegistry returns PythonParser for ".py" extension`
- `Unit: ParserRegistry returns None for ".md" extension`
- `Integration (real DB): ingest a directory with 3 Python files → 3 SourceFile rows, N SourceFunction rows`
- `Integration (real DB): ingest skips files matching exclude patterns`
- `Integration (real DB): ingest handles parse errors gracefully (records error, continues)`
- `Fixture: tests/fixtures/python_samples/ directory → known IngestionResult counts`

---

## Phase 3: Coverage Analysis and Gap Identification

### Purpose
Implement coverage report parsing (LCOV, JaCoCo, coverage.py, Istanbul), gap analysis, and the LLM-driven explanation of why specific code paths are uncovered. After this phase, users can upload or generate coverage data, and the system identifies and explains coverage gaps with actionable suggestions for how to make untested code reachable. This is the core differentiating feature described in the README as "Coverage Gap Analysis."

### Tasks

#### 3.1 — Coverage Report Parsers

**What**: Implement parsers for the four major coverage formats that normalize data into the unified `coverage_reports` → `coverage_files` → `coverage_lines` schema.

**Design**:

```python
# src/testgen/coverage_parsers/base.py
from dataclasses import dataclass, field

@dataclass
class CoverageLineData:
    line_number: int
    hit_count: int
    is_branch: bool = False
    branch_taken: int | None = None
    branch_missed: int | None = None

@dataclass
class CoverageFileData:
    file_path: str
    lines: list[CoverageLineData]
    total_lines: int = 0
    covered_lines: int = 0
    total_branches: int = 0
    covered_branches: int = 0

@dataclass
class CoverageReportData:
    source_format: str  # "lcov", "jacoco", "coverage_py", "istanbul"
    files: list[CoverageFileData]
    total_lines: int = 0
    covered_lines: int = 0
    total_branches: int = 0
    covered_branches: int = 0

class BaseCoverageParser:
    def parse(self, raw_data: str | bytes) -> CoverageReportData:
        ...
```

LCOV parser handles `SF:`, `DA:`, `BRDA:`, `BRF:`, `BRH:`, `LF:`, `LH:`, `end_of_record` markers.

JaCoCo parser handles the XML format: `<report>` → `<package>` → `<class>` → `<method>` → `<counter type="LINE|BRANCH" missed="N" covered="N"/>`.

coverage.py parser handles the JSON output from `coverage json`: `{"files": {"path": {"executed_lines": [...], "missing_lines": [...], "excluded_lines": [...]}}}`.

Istanbul parser handles the JSON output: `{"path": {"statementMap": {...}, "s": {...}, "branchMap": {...}, "b": {...}}}`.

**Testing**:
- `Unit: LCOV parser → correct line and branch counts from fixture file`
- `Unit: JaCoCo parser → correct line and branch counts from fixture XML`
- `Unit: coverage.py parser → correct executed/missing lines from fixture JSON`
- `Unit: Istanbul parser → correct statement and branch counts from fixture JSON`
- `Unit: Empty LCOV file → CoverageReportData with zero counts`
- `Unit: Malformed LCOV → raises ParseError with line number`
- `Fixture: tests/fixtures/coverage_reports/sample.lcov → known CoverageReportData`
- `Fixture: tests/fixtures/coverage_reports/sample_jacoco.xml → known CoverageReportData`
- `Fixture: tests/fixtures/coverage_reports/sample_coverage_py.json → known CoverageReportData`
- `Fixture: tests/fixtures/coverage_reports/sample_istanbul.json → known CoverageReportData`

#### 3.2 — Coverage Persistence Service

**What**: Persist parsed coverage data into the database, linking to source files and calculating per-file and aggregate percentages.

**Design**:

```python
# src/testgen/services/coverage_analysis.py
class CoveragePersistenceService:
    async def save_report(
        self,
        session: AsyncSession,
        test_run_id: uuid.UUID,
        repository_id: uuid.UUID,
        commit_id: uuid.UUID,
        report_data: CoverageReportData,
    ) -> CoverageReport:
        """Persist coverage data and compute aggregate metrics."""
        # 1. Create CoverageReport row with aggregate totals
        # 2. For each CoverageFileData, create CoverageFile row
        #    - Attempt to link to SourceFile by matching file_path
        #    - Calculate per-file line_coverage_pct and branch_coverage_pct
        # 3. Bulk-insert CoverageLine rows (batched for performance)
        # 4. Calculate and store report-level percentages
        ...
```

**Testing**:
- `Integration (real DB): save_report with 5 files → 1 CoverageReport, 5 CoverageFile, N CoverageLine rows`
- `Integration (real DB): line_coverage_pct correctly calculated as covered/total * 100`
- `Integration (real DB): CoverageFile linked to existing SourceFile when file_path matches`
- `Integration (real DB): CoverageFile.source_file_id is NULL when no matching SourceFile exists`
- `Unit: Aggregate totals correctly sum across all files`

#### 3.3 — Coverage Gap Analysis Service

**What**: Cross-reference coverage data with parsed source functions to identify uncovered functions and branches, then use the LLM to explain why each gap exists and suggest how to make the code testable.

**Design**:

```python
# src/testgen/services/coverage_analysis.py
class CoverageGapAnalysisService:
    def __init__(self, session: AsyncSession, llm_client: LLMClient):
        self._session = session
        self._llm = llm_client

    async def analyze_gaps(
        self,
        coverage_report_id: uuid.UUID,
        repository_id: uuid.UUID,
        commit_id: uuid.UUID,
    ) -> list[CoverageGap]:
        """
        1. Query source_functions with no corresponding covered lines
        2. Query source_functions with partial branch coverage
        3. For each gap, fetch the function source code and context
        4. Call LLM to classify the gap type and generate explanation
        5. Persist CoverageGap records
        """
        ...
```

LLM prompt template for gap classification:
```
System: You are a test engineering expert. Analyze why the following function lacks test coverage and classify the gap.

User:
Function: {qualified_name}
Language: {language}
Source code:
```{language}
{source_code}
```
Coverage: {covered_lines}/{total_lines} lines covered
Uncovered lines: {uncovered_line_numbers}
File imports: {imports}

Classify this coverage gap into one of:
- uncovered_function: The function has zero test coverage
- uncovered_branch: Some branches are not covered
- complex_setup: The function requires complex test setup (database, file system, etc.)
- external_dependency: The function depends on external services that need mocking
- concurrency: The function involves async/threaded code that's hard to test deterministically
- error_path: The uncovered code is error handling that's hard to trigger

Respond in JSON:
{
  "gap_type": "...",
  "severity": "critical|high|medium|low",
  "explanation": "...",
  "suggested_approach": "..."
}
```

Gap severity is determined by:
- `critical`: Public function with zero coverage and complexity > 5
- `high`: Public function with zero coverage or branch coverage < 50%
- `medium`: Partial coverage (50-80%) or private function with zero coverage
- `low`: Coverage > 80% with a few uncovered branches

**Testing**:
- `Unit (mocked LLM): analyze_gaps identifies uncovered functions from coverage data`
- `Unit (mocked LLM): gap severity correctly assigned based on coverage percentage and complexity`
- `Unit (mocked LLM): LLM response parsed correctly into CoverageGap fields`
- `Integration (mocked LLM, real DB): CoverageGap records persisted with correct foreign keys`
- `Integration (mocked LLM, real DB): Function with 0/10 lines covered → severity="critical" or "high"`
- `Integration (mocked LLM, real DB): Function with 8/10 lines covered → severity="low"`
- `Unit: Malformed LLM response → fallback to default gap_type="uncovered_function" with error logged`

#### 3.4 — Coverage API Endpoints

**What**: REST API endpoints for uploading coverage reports, triggering gap analysis, and querying coverage data.

**Design**:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| POST | `/api/v1/repos/{repo_id}/coverage` | Multipart form: `file` (LCOV/JaCoCo/etc.), `format` (string), `commit_sha` (string) | `201: CoverageReport` |
| GET | `/api/v1/repos/{repo_id}/coverage` | Query: `commit_sha`, `branch`, `limit`, `offset` | `200: list[CoverageReportSummary]` |
| GET | `/api/v1/repos/{repo_id}/coverage/{report_id}` | — | `200: CoverageReportDetail` (includes files) |
| POST | `/api/v1/repos/{repo_id}/coverage/{report_id}/analyze-gaps` | — | `202: { "task_id": "..." }` (async Celery task) |
| GET | `/api/v1/repos/{repo_id}/coverage/{report_id}/gaps` | Query: `severity`, `gap_type`, `limit`, `offset` | `200: list[CoverageGap]` |

Pydantic response schemas:
```python
class CoverageReportSummary(BaseModel):
    id: uuid.UUID
    commit_sha: str
    source_format: str
    line_coverage_pct: float
    branch_coverage_pct: float
    total_files: int
    created_at: datetime

class CoverageGapResponse(BaseModel):
    id: uuid.UUID
    function_name: str
    qualified_name: str
    file_path: str
    gap_type: str
    severity: str
    explanation: str
    suggested_approach: str | None
```

**Testing**:
- `Integration: POST coverage with LCOV file → 201, report created with correct metrics`
- `Integration: POST coverage with invalid format string → 400 error`
- `Integration: GET coverage list → paginated results ordered by created_at DESC`
- `Integration: POST analyze-gaps → 202 with task_id, Celery task enqueued`
- `Integration: GET gaps filtered by severity="critical" → only critical gaps returned`
- `Integration: GET gaps for report with no gaps → empty list`

---

## Phase 4: LLM-Based Test Generation Engine

### Purpose
Implement the core test generation pipeline: given a source function (or a coverage gap), call the LLM to generate a test, validate that it compiles/parses, and persist the result. This is the "heart" of the product. After this phase, users can generate tests for individual functions or for all coverage gaps in a repository.

### Tasks

#### 4.1 — LLM Client Abstraction

**What**: Create the LLM client interface with anthropic SDK as default and litellm as the adapter for alternative providers.

**Design**:

```python
# src/testgen/llm/client.py
from dataclasses import dataclass
from abc import ABC, abstractmethod

@dataclass
class LLMResponse:
    content: str
    model: str
    input_tokens: int
    output_tokens: int
    stop_reason: str

class LLMClient(ABC):
    @abstractmethod
    async def generate(
        self,
        system_prompt: str,
        user_prompt: str,
        max_tokens: int = 4096,
        temperature: float = 0.2,
    ) -> LLMResponse:
        ...

class AnthropicClient(LLMClient):
    def __init__(self, api_key: str, model: str = "claude-sonnet-4-20250514"):
        self._client = anthropic.AsyncAnthropic(api_key=api_key)
        self._model = model

    async def generate(self, system_prompt: str, user_prompt: str, **kwargs) -> LLMResponse:
        response = await self._client.messages.create(
            model=self._model,
            max_tokens=kwargs.get("max_tokens", 4096),
            temperature=kwargs.get("temperature", 0.2),
            system=system_prompt,
            messages=[{"role": "user", "content": user_prompt}],
        )
        return LLMResponse(
            content=response.content[0].text,
            model=response.model,
            input_tokens=response.usage.input_tokens,
            output_tokens=response.usage.output_tokens,
            stop_reason=response.stop_reason,
        )

class LiteLLMClient(LLMClient):
    """Adapter for any provider supported by litellm (OpenAI, Gemini, Ollama, etc.)."""
    def __init__(self, model: str, api_key: str | None = None):
        self._model = model
        self._api_key = api_key

    async def generate(self, system_prompt: str, user_prompt: str, **kwargs) -> LLMResponse:
        import litellm
        response = await litellm.acompletion(
            model=self._model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt},
            ],
            max_tokens=kwargs.get("max_tokens", 4096),
            temperature=kwargs.get("temperature", 0.2),
            api_key=self._api_key,
        )
        return LLMResponse(
            content=response.choices[0].message.content,
            model=response.model,
            input_tokens=response.usage.prompt_tokens,
            output_tokens=response.usage.completion_tokens,
            stop_reason=response.choices[0].finish_reason,
        )

def create_llm_client(settings: Settings) -> LLMClient:
    if settings.llm_provider == "anthropic":
        return AnthropicClient(api_key=settings.llm_api_key, model=settings.llm_model)
    return LiteLLMClient(model=settings.llm_model, api_key=settings.llm_api_key)
```

**Testing**:
- `Unit (mocked): AnthropicClient.generate returns LLMResponse with correct fields`
- `Unit (mocked): LiteLLMClient.generate returns LLMResponse with correct fields`
- `Unit: create_llm_client with provider="anthropic" returns AnthropicClient`
- `Unit: create_llm_client with provider="openai" returns LiteLLMClient`
- `Unit (mocked): Token counts are correctly extracted from API response`

#### 4.2 — Prompt Templates for Test Generation

**What**: Define structured prompt templates for generating unit tests in each target language, with configurable context windows.

**Design**:

```python
# src/testgen/llm/prompts.py
from dataclasses import dataclass

@dataclass
class GenerationContext:
    language: str
    test_framework: str           # "pytest", "jest", "junit5", "go_test"
    function_source: str          # source code of the function under test
    function_name: str
    qualified_name: str
    file_path: str
    file_imports: list[str]       # imports from the source file
    class_context: str | None     # surrounding class if method
    related_functions: list[str]  # other functions in the same file (signatures only)
    existing_tests: str | None    # existing test code for this file if any
    coverage_gaps: list[str]      # specific uncovered lines/branches to target
    dependencies: list[str]       # external dependencies that may need mocking

SYSTEM_PROMPT = """You are an expert test engineer. Generate high-quality unit tests that:
1. Test behavior, not implementation details
2. Cover happy paths, edge cases, error conditions, and boundary values
3. Use descriptive test names that explain what is being tested
4. Generate appropriate mocks/stubs for external dependencies
5. Follow the conventions of the target test framework
6. Include assertions that verify meaningful outcomes, not just that code runs without errors

Output ONLY the test code — no explanations, no markdown fences, no comments about what you're doing."""

USER_PROMPT_TEMPLATE = """Generate unit tests for the following {language} function.

**Test framework**: {test_framework}
**File**: {file_path}

**Imports in source file**:
{imports_block}

{class_context_block}

**Function to test**:
```{language}
{function_source}
```

{coverage_gaps_block}

{dependencies_block}

{existing_tests_block}

Generate comprehensive tests covering:
- Normal/happy path
- Edge cases (empty inputs, None/null, boundary values)
- Error conditions (invalid inputs, exceptions)
- Any uncovered branches listed above

Use {test_framework} conventions. Mock external dependencies appropriately."""


def build_generation_prompt(ctx: GenerationContext) -> tuple[str, str]:
    """Build (system_prompt, user_prompt) from generation context."""
    ...
```

**Testing**:
- `Unit: build_generation_prompt with Python context → includes "pytest" and function source`
- `Unit: build_generation_prompt with empty coverage_gaps → omits coverage section`
- `Unit: build_generation_prompt with class_context → includes class definition`
- `Unit: build_generation_prompt with existing_tests → includes "Existing tests" section`
- `Unit: build_generation_prompt truncates related_functions if total exceeds token budget`

#### 4.3 — Test Generation Service

**What**: Orchestrate the generation-validation-repair loop: generate a test via LLM, validate it parses (using tree-sitter), retry with error feedback if invalid, and persist the result.

**Design**:

```python
# src/testgen/services/generation.py
class TestGenerationService:
    def __init__(
        self,
        session: AsyncSession,
        llm_client: LLMClient,
        parser_registry: ParserRegistry,
    ):
        self._session = session
        self._llm = llm_client
        self._parsers = parser_registry

    async def generate_for_function(
        self,
        session_id: uuid.UUID,
        source_function: SourceFunction,
        context: GenerationContext,
        max_retries: int = 3,
    ) -> GeneratedTest | None:
        """
        Generation-validation-repair loop:
        1. Build prompt from context
        2. Call LLM to generate test code
        3. Validate the generated code parses (tree-sitter)
        4. If parse fails, retry with error message appended to prompt
        5. If all retries fail, return None and log the failure
        6. Persist GeneratedTest record
        """
        for attempt in range(max_retries):
            system_prompt, user_prompt = build_generation_prompt(context)
            if attempt > 0:
                user_prompt += f"\n\nPrevious attempt failed with error:\n{last_error}\nPlease fix the issue."

            response = await self._llm.generate(system_prompt, user_prompt)
            test_code = self._extract_code(response.content)

            parse_result = self._validate_syntax(test_code, context.language)
            if parse_result.is_valid:
                return await self._persist_test(session_id, source_function, context, test_code, response)
            last_error = parse_result.error_message

        return None

    def _validate_syntax(self, code: str, language: str) -> ParseValidation:
        """Use tree-sitter to check if the generated code is syntactically valid."""
        ...

    def _extract_code(self, llm_output: str) -> str:
        """Strip markdown fences and any non-code content from LLM output."""
        ...

    async def generate_for_gaps(
        self,
        session_id: uuid.UUID,
        coverage_gaps: list[CoverageGap],
    ) -> list[GeneratedTest]:
        """Generate tests targeting specific coverage gaps. Uses iterative coverage-guided
        prompting: after generating tests for the first batch of gaps, re-analyze which
        gaps remain and generate additional tests only for those."""
        ...
```

State machine for `GeneratedTest.status`:
```
generated → validated (syntax check passed)
         → stale     (source function changed since generation)
validated → accepted  (user approved or auto-accepted)
         → rejected  (user rejected)
```

**Testing**:
- `Unit (mocked LLM): generate_for_function with valid response → GeneratedTest persisted`
- `Unit (mocked LLM): generate_for_function with invalid syntax → retries with error context`
- `Unit (mocked LLM): generate_for_function with 3 invalid responses → returns None`
- `Unit: _extract_code strips markdown fences from LLM output`
- `Unit: _extract_code handles output with no fences (raw code)`
- `Unit: _validate_syntax accepts valid Python test code`
- `Unit: _validate_syntax rejects Python code with syntax errors`
- `Integration (mocked LLM, real DB): GeneratedTest persisted with correct session_id and status="generated"`
- `Integration (mocked LLM, real DB): LLM token usage recorded in generation_session.llm_tokens_used`

#### 4.4 — Mock and Fixture Generation

**What**: Generate appropriate mocks, stubs, and fixtures for external dependencies detected in the source function.

**Design**:

```python
# Added to generation.py or as part of the prompt:
class MockGenerationService:
    async def generate_mocks(
        self,
        source_function: SourceFunction,
        dependencies: list[str],
        language: str,
        test_framework: str,
    ) -> list[GeneratedMock]:
        """
        Analyze function dependencies and generate:
        - Stubs for database queries (return canned data)
        - Mocks for HTTP clients (return predefined responses)
        - Fakes for file system operations (use tempdir/StringIO)
        - Fixtures for complex object construction
        """
        ...
```

The mock generation is integrated into the test generation prompt — the LLM generates mocks inline with the test code. The `GeneratedMock` records are extracted from the generated test by parsing import statements and fixture/mock declarations.

Mock classification:
- `stub`: Returns a fixed value, no assertions on calls
- `mock`: Returns a value and asserts call count/arguments
- `spy`: Wraps real implementation, records calls
- `fake`: In-memory implementation of an interface
- `fixture`: Test data factory (e.g., pytest fixture, JUnit @BeforeEach)

**Testing**:
- `Unit (mocked LLM): function calling requests.get → GeneratedMock with mock_target="requests.get"`
- `Unit (mocked LLM): function calling database.query → GeneratedMock with mock_type="stub"`
- `Integration (real DB): GeneratedMock records linked to GeneratedTest via FK`

#### 4.5 — Generation API Endpoints and Celery Tasks

**What**: REST API endpoints for triggering test generation and Celery tasks for async execution.

**Design**:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| POST | `/api/v1/repos/{repo_id}/generate` | `{ "commit_sha": "...", "strategy": "coverage_gap", "target_files": [...], "target_functions": [...] }` | `202: { "session_id": "...", "task_id": "..." }` |
| GET | `/api/v1/sessions/{session_id}` | — | `200: GenerationSessionDetail` |
| GET | `/api/v1/sessions/{session_id}/tests` | Query: `status`, `test_type`, `limit`, `offset` | `200: list[GeneratedTestResponse]` |
| GET | `/api/v1/tests/{test_id}` | — | `200: GeneratedTestDetail` (includes test_code, mocks) |
| POST | `/api/v1/tests/{test_id}/accept` | — | `200: GeneratedTestDetail` (status→accepted) |
| POST | `/api/v1/tests/{test_id}/reject` | `{ "reason": "..." }` | `200: GeneratedTestDetail` (status→rejected) |

Celery task:
```python
# src/testgen/tasks/generate.py
@celery_app.task(bind=True, max_retries=2, default_retry_delay=30)
def generate_tests_task(self, session_id: str, repository_id: str, commit_sha: str, strategy: str, targets: dict):
    """
    1. Clone/checkout repository at commit_sha
    2. Parse target files (or all files if no targets specified)
    3. Load coverage data if strategy is "coverage_gap"
    4. Generate tests for each target function
    5. Update session status to "completed" or "failed"
    """
    ...
```

**Testing**:
- `Integration: POST /generate with coverage_gap strategy → 202, session created with status="pending"`
- `Integration: POST /generate with invalid strategy → 400`
- `Integration: GET /sessions/{id} → correct session status and test count`
- `Integration: POST /tests/{id}/accept → test status changes to "accepted"`
- `Integration (mocked LLM): Celery task generates tests and updates session status to "completed"`
- `Integration (mocked LLM): Celery task records token usage in session`

---

## Phase 5: Test Execution and Validation

### Purpose
Implement the ability to actually run generated tests against the target codebase using the appropriate test runner (pytest, Jest, JUnit, Go Test), capture results, and validate that generated tests pass. After this phase, the generation-validation loop is complete: generate → validate syntax → run → capture results → mark as validated or fix.

### Tasks

#### 5.1 — Abstract Test Runner Interface

**What**: Define the base interface for language-specific test runners.

**Design**:

```python
# src/testgen/runners/base.py
from dataclasses import dataclass

@dataclass
class TestCaseResult:
    test_name: str
    class_name: str | None
    status: str          # "passed", "failed", "error", "skipped"
    duration_ms: int
    failure_message: str | None = None
    failure_type: str | None = None
    stack_trace: str | None = None
    system_out: str | None = None

@dataclass
class RunResult:
    runner: str           # "pytest", "jest", "junit", "go_test"
    status: str           # "passed", "failed", "error"
    total: int
    passed: int
    failed: int
    errors: int
    skipped: int
    duration_ms: int
    test_cases: list[TestCaseResult]
    coverage_file: Path | None = None  # path to generated coverage report

class BaseTestRunner(ABC):
    @abstractmethod
    async def run(
        self,
        test_file_path: Path,
        working_dir: Path,
        timeout_seconds: int = 300,
        collect_coverage: bool = True,
    ) -> RunResult:
        ...

    @abstractmethod
    async def install_dependencies(self, working_dir: Path) -> None:
        """Ensure test framework and dependencies are installed."""
        ...
```

**Testing**:
- `Unit: TestCaseResult correctly defaults optional fields to None`
- `Unit: RunResult computes correct total from test_cases list`

#### 5.2 — pytest Runner

**What**: Run generated Python tests using pytest with coverage collection via coverage.py.

**Design**:

```python
# src/testgen/runners/pytest_runner.py
class PytestRunner(BaseTestRunner):
    async def run(self, test_file_path: Path, working_dir: Path, **kwargs) -> RunResult:
        """
        1. Write generated test to a temp file in the working directory
        2. Run: python -m pytest {test_file} --tb=short --junitxml={result_file} --cov={source_dir} --cov-report=lcov
        3. Parse JUnit XML for test results
        4. Parse LCOV for coverage data
        5. Return RunResult
        """
        cmd = [
            sys.executable, "-m", "pytest",
            str(test_file_path),
            f"--junitxml={junit_path}",
            "--tb=short",
            "-v",
        ]
        if collect_coverage:
            cmd.extend([
                f"--cov={source_dir}",
                f"--cov-report=lcov:{lcov_path}",
            ])
        proc = await asyncio.create_subprocess_exec(
            *cmd,
            cwd=str(working_dir),
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        stdout, stderr = await asyncio.wait_for(proc.communicate(), timeout=timeout_seconds)
        return self._parse_junit_xml(junit_path)
```

**Testing**:
- `Integration (real): Run a known-passing Python test file → RunResult with status="passed"`
- `Integration (real): Run a known-failing Python test file → RunResult with status="failed", failure_message populated`
- `Integration (real): Run with coverage collection → coverage_file path set and LCOV file exists`
- `Unit: JUnit XML parsing extracts correct test names, statuses, and durations`
- `Integration (real): Timeout exceeded → RunResult with status="error"`

#### 5.3 — Jest Runner

**What**: Run generated TypeScript/JavaScript tests using Jest or Vitest.

**Design**:

Similar to PytestRunner but invokes `npx jest --json --outputFile={result_file} --coverage --coverageReporters=lcov` and parses Jest JSON output format.

**Testing**:
- `Integration (real): Run a known-passing Jest test → RunResult with status="passed"`
- `Integration (real): Run with coverage → Istanbul LCOV output collected`
- `Unit: Jest JSON result parsing extracts correct test names and statuses`

#### 5.4 — JUnit Runner

**What**: Run generated Java tests using Maven Surefire with JaCoCo coverage.

**Design**:

Invokes `mvn test -pl {module} -Dtest={test_class} -Dsurefire.reportFormat=xml` with JaCoCo agent for coverage. Parses Surefire XML reports.

**Testing**:
- `Integration (real, requires JDK): Run a known-passing JUnit test → RunResult with status="passed"`
- `Unit: Surefire XML parsing extracts correct test results`

#### 5.5 — Go Test Runner

**What**: Run generated Go tests using `go test` with coverage.

**Design**:

Invokes `go test -v -json -coverprofile={cover_file} ./...` and parses the JSON stream output.

**Testing**:
- `Integration (real, requires Go): Run a known-passing Go test → RunResult with status="passed"`
- `Unit: go test JSON output parsing extracts correct test results`

#### 5.6 — Test Execution Orchestration and Result Persistence

**What**: Orchestrate test execution, persist results to `test_runs` and `test_results` tables, and update `generated_tests` status based on results.

**Design**:

```python
# src/testgen/services/runner.py
class TestExecutionService:
    async def execute_and_persist(
        self,
        session_id: uuid.UUID,
        repository_path: Path,
        generated_tests: list[GeneratedTest],
    ) -> TestRun:
        """
        1. Group tests by language/framework
        2. Write test files to the repository working directory
        3. Run each group with the appropriate runner
        4. Persist TestRun and TestResult records
        5. If coverage was collected, invoke CoveragePersistenceService
        6. Update GeneratedTest.status:
           - All assertions pass → "validated"
           - Any failure → leave as "generated" (will retry in generation loop)
        """
        ...
```

**Testing**:
- `Integration (mocked runner): execute_and_persist creates TestRun with correct status`
- `Integration (mocked runner): Individual TestResult records match runner output`
- `Integration (mocked runner): GeneratedTest.status updated to "validated" when test passes`
- `Integration (mocked runner): Coverage data persisted when runner provides coverage file`

---

## Phase 6: CLI Interface

### Purpose
Implement the `testgen` CLI that CI pipelines and developers invoke from the command line. After this phase, users can run `testgen analyze`, `testgen generate`, `testgen run`, and `testgen gate` from any terminal to parse a codebase, generate tests, run them, and enforce coverage thresholds. The CLI communicates with the API server for stateful operations but can also operate in a standalone mode for local development.

### Tasks

#### 6.1 — CLI Framework and Authentication

**What**: Set up the Typer CLI with subcommands, authentication (API key or local mode), and output formatting.

**Design**:

```python
# src/testgen/cli.py
import typer
from rich.console import Console
from rich.table import Table

app = typer.Typer(name="testgen", help="Intelligent Test Generator CLI")
console = Console()

@app.command()
def analyze(
    path: Path = typer.Argument(".", help="Path to repository root"),
    languages: list[str] = typer.Option(["python"], help="Languages to parse"),
    output: str = typer.Option("table", help="Output format: table, json"),
):
    """Analyze source code and report functions, complexity, and coverage gaps."""
    ...

@app.command()
def generate(
    path: Path = typer.Argument(".", help="Path to repository root"),
    target: list[str] = typer.Option([], help="Specific files or functions to target"),
    strategy: str = typer.Option("coverage_gap", help="Generation strategy"),
    framework: str = typer.Option("auto", help="Test framework (auto-detected from project)"),
    output_dir: str = typer.Option("tests/", help="Directory for generated test files"),
    dry_run: bool = typer.Option(False, help="Print generated tests without writing files"),
):
    """Generate tests for source code in the repository."""
    ...

@app.command()
def run(
    path: Path = typer.Argument(".", help="Path to repository root"),
    test_dir: str = typer.Option("tests/", help="Test directory"),
    coverage: bool = typer.Option(True, help="Collect coverage data"),
):
    """Run generated tests and report results."""
    ...

@app.command()
def gate(
    path: Path = typer.Argument(".", help="Path to repository root"),
    min_coverage: float = typer.Option(80.0, help="Minimum line coverage %"),
    min_mutation_score: float = typer.Option(0.0, help="Minimum mutation score %"),
):
    """Check coverage thresholds and exit with non-zero code if gates fail."""
    ...
```

Exit codes:
- `0`: All gates passed
- `1`: Gate failed (coverage below threshold)
- `2`: Error (could not parse, generate, or run)

**Testing**:
- `E2E: testgen analyze ./tests/fixtures/python_samples → table output with function list`
- `E2E: testgen analyze --output json → valid JSON output`
- `E2E: testgen gate --min-coverage 100 → exit code 1 (threshold not met)`
- `E2E: testgen gate --min-coverage 0 → exit code 0`
- `E2E: testgen --help → shows all subcommands`
- `Unit: CLI correctly resolves "auto" framework to "pytest" for Python projects`

#### 6.2 — Standalone Local Mode

**What**: Enable CLI operation without requiring a running API server, for local development and simple CI pipelines.

**Design**:

In local mode, the CLI:
1. Uses SQLite instead of PostgreSQL (via `aiosqlite`)
2. Runs generation synchronously (no Celery)
3. Stores state in `~/.testgen/` or `.testgen/` in the project root
4. Still requires an LLM API key (reads from `TESTGEN_LLM_API_KEY` env var or `~/.testgen/config.toml`)

```python
@app.command()
def init(
    path: Path = typer.Argument(".", help="Initialize testgen in this directory"),
    provider: str = typer.Option("anthropic", help="LLM provider"),
):
    """Initialize testgen configuration in the current project."""
    config_dir = path / ".testgen"
    config_dir.mkdir(exist_ok=True)
    # Write default config.toml
    ...
```

**Testing**:
- `E2E: testgen init → creates .testgen/ directory with config.toml`
- `E2E: testgen generate (local mode) → generates tests without API server`
- `E2E: testgen generate --dry-run → prints test code to stdout without writing files`

---

## Phase 7: CI/CD Integration

### Purpose
Implement GitHub Actions action, pipeline configuration management, and coverage threshold gates that can block merges. After this phase, teams can add the test generator to their CI pipeline with a single YAML file and have it auto-generate tests, run them, and enforce coverage gates on pull requests.

### Tasks

#### 7.1 — GitHub Actions Action

**What**: Create a GitHub Actions composite action that runs testgen in a CI pipeline.

**Design**:

```yaml
# action.yml
name: 'Intelligent Test Generator'
description: 'Generate and validate tests with AI-driven coverage analysis'
inputs:
  api-key:
    description: 'LLM API key'
    required: true
  strategy:
    description: 'Generation strategy'
    default: 'coverage_gap'
  min-coverage:
    description: 'Minimum line coverage threshold'
    default: '80'
  languages:
    description: 'Comma-separated list of languages to target'
    default: 'python'
runs:
  using: 'composite'
  steps:
    - name: Install testgen
      shell: bash
      run: pip install testgen
    - name: Generate tests
      shell: bash
      run: testgen generate --strategy ${{ inputs.strategy }}
      env:
        TESTGEN_LLM_API_KEY: ${{ inputs.api-key }}
    - name: Run tests
      shell: bash
      run: testgen run
    - name: Coverage gate
      shell: bash
      run: testgen gate --min-coverage ${{ inputs.min-coverage }}
```

Usage in a workflow:
```yaml
# .github/workflows/test-gen.yml
jobs:
  generate-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: intelligent-test-generator/action@v1
        with:
          api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          min-coverage: 80
```

**Testing**:
- `E2E: action.yml validates with actionlint`
- `Integration (mocked): Workflow runs generate → run → gate sequence`
- `Unit: Action correctly passes inputs as environment variables`

#### 7.2 — Pipeline Configuration API

**What**: REST API for managing per-repository CI pipeline configurations (thresholds, auto-generation settings).

**Design**:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| GET | `/api/v1/repos/{repo_id}/pipeline-config` | — | `200: PipelineConfiguration` |
| PUT | `/api/v1/repos/{repo_id}/pipeline-config` | `{ "min_line_coverage": 80.0, "min_mutation_score": 60.0, "auto_generate": true, "target_languages": ["python", "typescript"] }` | `200: PipelineConfiguration` |
| GET | `/api/v1/repos/{repo_id}/pipeline-runs` | Query: `status`, `limit`, `offset` | `200: list[PipelineRun]` |

**Testing**:
- `Integration: PUT pipeline config → 200, config persisted`
- `Integration: GET pipeline config → correct thresholds`
- `Integration: GET pipeline runs → paginated list`

#### 7.3 — Webhook Handler for GitHub/GitLab Events

**What**: Receive webhook events from GitHub/GitLab and auto-trigger test generation on pull requests.

**Design**:

```python
# src/testgen/api/webhooks.py
@router.post("/webhooks/github")
async def github_webhook(request: Request):
    """
    Handle GitHub webhook events:
    - pull_request.opened / synchronize → trigger generation for changed files
    - push to default branch → update coverage snapshot
    """
    payload = await request.json()
    signature = request.headers.get("X-Hub-Signature-256")
    if not verify_github_signature(payload, signature, webhook_secret):
        raise HTTPException(status_code=401, detail="Invalid signature")

    event = request.headers.get("X-GitHub-Event")
    if event == "pull_request" and payload["action"] in ("opened", "synchronize"):
        # Enqueue generation task for changed files
        ...
    elif event == "push" and payload["ref"] == f"refs/heads/{default_branch}":
        # Update coverage snapshot
        ...
```

**Testing**:
- `Integration (mocked): Valid GitHub webhook with PR event → generation task enqueued`
- `Integration (mocked): Invalid signature → 401 returned, no task enqueued`
- `Integration (mocked): Push to default branch → coverage snapshot task enqueued`
- `Unit: verify_github_signature correctly validates HMAC-SHA256`

---

## Phase 8: VS Code Extension

### Purpose
Build a VS Code extension that provides inline test generation, coverage gap visualization, and test management directly in the editor. After this phase, developers can right-click a function and generate tests without leaving their IDE. The extension communicates with the testgen API server (or runs in local mode).

### Tasks

#### 8.1 — Extension Scaffolding

**What**: Create the VS Code extension project with TypeScript, webpack bundling, and the extension activation entry point.

**Design**:

```
vscode-extension/
├── package.json          # Extension manifest with commands, menus, configuration
├── tsconfig.json
├── webpack.config.js
├── src/
│   ├── extension.ts      # activate/deactivate entry point
│   ├── api-client.ts     # HTTP client for testgen API
│   ├── commands/
│   │   ├── generate.ts   # "Generate Tests" command
│   │   ├── analyze.ts    # "Analyze Coverage" command
│   │   └── configure.ts  # "Configure TestGen" command
│   ├── providers/
│   │   ├── codelens.ts   # CodeLens "Generate Test" above functions
│   │   └── coverage.ts   # Coverage gutter decoration
│   └── views/
│       └── sidebar.ts    # Sidebar panel with generation history
└── test/
    └── extension.test.ts
```

`package.json` contributes:
- Commands: `testgen.generate`, `testgen.analyze`, `testgen.configure`
- CodeLens provider for supported languages
- Gutter decoration type for coverage visualization
- Configuration: API URL, API key, default strategy

**Testing**:
- `Unit: Extension activates without errors`
- `Unit: Package.json passes vsce package validation`
- `E2E: Install extension in VS Code → commands appear in command palette`

#### 8.2 — CodeLens and Inline Generation

**What**: Show "Generate Test" CodeLens above each function, and handle the generation flow when clicked.

**Design**:

The CodeLens provider uses the VS Code `DocumentSymbolProvider` to find function definitions and renders a clickable "Generate Test | View Coverage" lens above each one.

When clicked:
1. Extract function source code and file context
2. Call testgen API `POST /generate` (or run locally)
3. Show progress notification
4. On completion, open the generated test file in a side-by-side editor
5. Show "Accept / Reject" buttons in the notification

**Testing**:
- `Unit: CodeLens appears above Python function definitions`
- `Unit: CodeLens appears above TypeScript function/arrow function definitions`
- `Integration (mocked API): Clicking CodeLens sends correct request to API`
- `Integration (mocked API): Generated test opens in side-by-side editor`

#### 8.3 — Coverage Gutter Decoration

**What**: Visualize line-level coverage in the editor gutter (green = covered, red = uncovered, yellow = branch partially covered).

**Design**:

The extension fetches coverage data from the API (or reads local LCOV files) and applies `DecorationRenderOptions` to the active editor:
- Green left-border: line is fully covered
- Red left-border: line is not covered
- Yellow left-border: line is covered but some branches are missed

Coverage decorations update when the active editor changes or when new coverage data is available.

**Testing**:
- `Unit: Coverage parser correctly reads LCOV format`
- `Unit: Decorations applied to correct line ranges`
- `Unit: Decorations update when active editor changes`

---

## Phase 9: Mutation Testing Integration

### Purpose
Implement the mutation-score-guided feedback loop: run mutation testing tools, identify surviving mutants, and generate additional assertions specifically targeting those mutants. After this phase, the system produces test suites with genuine defect-detection power measured by mutation score, not just coverage percentage. This is the key differentiator validated by Meta's production deployment.

### Tasks

#### 9.1 — Mutation Tool Orchestration

**What**: Invoke mutation testing tools (mutmut, Stryker, PITest, go-mutesting) via subprocess and capture results.

**Design**:

```python
# src/testgen/services/mutation.py
class MutationTestingService:
    def __init__(self, session: AsyncSession):
        self._session = session
        self._tools = {
            "python": MutmutOrchestrator(),
            "typescript": StrykerOrchestrator(),
            "javascript": StrykerOrchestrator(),
            "java": PITestOrchestrator(),
            "go": GoMutestingOrchestrator(),
        }

    async def run_mutation_testing(
        self,
        session_id: uuid.UUID,
        repository_path: Path,
        language: str,
        source_files: list[str],
        test_files: list[str],
    ) -> MutationRun:
        """
        1. Select appropriate mutation tool for language
        2. Configure and invoke via subprocess
        3. Parse tool output into MutationRun + Mutant records
        4. Link mutants to test results via mutant_test_links
        """
        ...
```

Each orchestrator class handles tool-specific invocation:
- `MutmutOrchestrator`: `mutmut run --paths-to-mutate={files} --tests-dir={test_dir} --runner=pytest`
- `StrykerOrchestrator`: `npx stryker run --mutate={files} --reporters json`
- `PITestOrchestrator`: `mvn org.pitest:pitest-maven-plugin:mutationCoverage`
- `GoMutestingOrchestrator`: `go-mutesting {package}`

**Testing**:
- `Integration (real, Python): Run mutmut on simple Python module → MutationRun with correct killed/survived counts`
- `Unit: Mutmut output parsing extracts correct mutant details`
- `Unit: Stryker JSON report parsing extracts mutants aligned with schema`
- `Unit: PITest XML report parsing extracts correct mutation results`

#### 9.2 — Mutation Report Parsers

**What**: Parse the output of each mutation testing tool into the unified `mutants` table schema (aligned with Stryker mutation-testing-report-schema).

**Design**:

```python
# src/testgen/mutation_parsers/stryker.py
class StrykerReportParser:
    def parse(self, report_json: dict) -> list[MutantData]:
        """
        Parse Stryker JSON report:
        {
          "files": {
            "src/app.ts": {
              "mutants": [
                {
                  "id": "1",
                  "mutatorName": "BooleanSubstitution",
                  "replacement": "false",
                  "location": {"start": {"line": 5, "column": 10}, "end": {"line": 5, "column": 14}},
                  "status": "Killed",
                  "killedBy": ["test1"],
                  "coveredBy": ["test1", "test2"]
                }
              ]
            }
          }
        }
        """
        ...
```

**Testing**:
- `Fixture: tests/fixtures/mutation_reports/stryker_report.json → correct MutantData list`
- `Fixture: tests/fixtures/mutation_reports/pitest_report.xml → correct MutantData list`
- `Fixture: tests/fixtures/mutation_reports/mutmut_results.txt → correct MutantData list`
- `Unit: All Stryker status values mapped correctly to schema enum`

#### 9.3 — Mutation-Guided Test Enhancement

**What**: After initial test generation, identify surviving mutants and generate additional assertions targeting them.

**Design**:

```python
class MutationGuidedEnhancer:
    async def enhance_tests(
        self,
        session_id: uuid.UUID,
        mutation_run: MutationRun,
        surviving_mutants: list[Mutant],
        existing_tests: list[GeneratedTest],
    ) -> list[GeneratedTest]:
        """
        For each surviving mutant:
        1. Extract the original code and the mutation
        2. Identify which test covers the mutated line
        3. Call LLM to generate an additional assertion that would kill this mutant
        4. Validate the new assertion kills the mutant (re-run mutation on just that mutant)
        5. Persist as additional GeneratedTest or amendment to existing test
        """
        ...
```

LLM prompt for mutation-killing assertion:
```
The following mutation survived (was not detected by any test):

Original code ({language}):
```{language}
{original_code}
```

Mutated code:
```{language}
{mutated_code}
```

Mutation type: {mutator_name} (changed {description})
Location: {file_path}:{start_line}

Existing test that covers this line:
```{language}
{existing_test_code}
```

Generate ONE additional assertion (or a new test case) that would fail if the mutation were applied.
The assertion must pass against the original code and fail against the mutated code.
```

**Testing**:
- `Unit (mocked LLM): enhance_tests generates assertion for surviving BooleanSubstitution mutant`
- `Unit (mocked LLM): enhance_tests skips mutants with status "no_coverage" (need new test, not new assertion)`
- `Integration (mocked LLM, real DB): New GeneratedTest records persisted for each enhancement`
- `Integration (real, Python): Generated assertion kills the target mutant when re-run`

#### 9.4 — Mutation API Endpoints

**What**: REST API for triggering mutation testing and viewing results.

**Design**:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| POST | `/api/v1/repos/{repo_id}/mutations` | `{ "commit_sha": "...", "source_files": [...], "test_files": [...] }` | `202: { "mutation_run_id": "...", "task_id": "..." }` |
| GET | `/api/v1/repos/{repo_id}/mutations/{run_id}` | — | `200: MutationRunDetail` |
| GET | `/api/v1/repos/{repo_id}/mutations/{run_id}/mutants` | Query: `status`, `mutator_name`, `limit`, `offset` | `200: list[MutantResponse]` |
| POST | `/api/v1/repos/{repo_id}/mutations/{run_id}/enhance` | — | `202: { "session_id": "...", "task_id": "..." }` |

**Testing**:
- `Integration: POST mutations → 202, mutation run created`
- `Integration: GET mutation run → correct killed/survived counts`
- `Integration: GET mutants filtered by status="survived" → only surviving mutants`
- `Integration: POST enhance → enhancement session created`

---

## Phase 10: Spec-to-Test Generation

### Purpose
Enable test generation from OpenAPI 3.x specifications and Gherkin feature files before implementation exists. After this phase, teams practising TDD or API-first development can generate a complete test suite from their specification, catching requirement ambiguities early and enabling true test-driven development at scale.

### Tasks

#### 10.1 — OpenAPI Spec Parser

**What**: Parse OpenAPI 3.x specifications and extract endpoint definitions, request/response schemas, and security requirements into testable scenarios.

**Design**:

```python
# src/testgen/services/spec_parser.py
from dataclasses import dataclass, field

@dataclass
class SpecEndpoint:
    method: str                    # GET, POST, PUT, DELETE, PATCH
    path: str                      # /users/{id}
    operation_id: str | None
    summary: str | None
    parameters: list[SpecParameter]
    request_body: SpecSchema | None
    responses: dict[str, SpecResponse]  # status_code → response schema
    security: list[dict[str, list[str]]]  # security schemes required

@dataclass
class SpecParameter:
    name: str
    location: str                  # path, query, header, cookie
    required: bool
    schema: SpecSchema

@dataclass
class SpecSchema:
    type: str                      # string, integer, object, array
    properties: dict[str, "SpecSchema"]
    required_fields: list[str]
    enum_values: list[str] | None
    minimum: float | None
    maximum: float | None
    pattern: str | None            # regex pattern
    example: Any | None

class OpenAPISpecParser:
    def parse(self, spec_content: str) -> list[SpecEndpoint]:
        """Parse OpenAPI 3.x YAML/JSON and extract endpoints with schemas."""
        ...

    def generate_test_scenarios(self, endpoint: SpecEndpoint) -> list[TestScenario]:
        """
        For each endpoint, generate scenarios based on ISO 29119 test design techniques:
        - Equivalence partitioning: valid/invalid values for each parameter type
        - Boundary value analysis: min, min+1, max-1, max for numeric params
        - Error guessing: missing required fields, wrong types, empty strings
        - Security: missing auth, invalid tokens, wrong scopes
        - HTTP semantics (RFC 9110): correct status codes for each scenario
        """
        ...
```

**Testing**:
- `Fixture: Parse Petstore OpenAPI spec → correct endpoint list`
- `Unit: generate_test_scenarios for POST /users → scenarios for valid creation, missing required field, invalid email format, duplicate username`
- `Unit: generate_test_scenarios for GET /users/{id} → scenarios for valid ID, non-existent ID (404), invalid ID format (400)`
- `Unit: Boundary value analysis for integer parameter with min=1, max=100 → test values [0, 1, 2, 99, 100, 101]`
- `Unit: Security scenarios generated when endpoint requires OAuth`

#### 10.2 — Gherkin Parser

**What**: Parse Gherkin feature files and convert Given-When-Then scenarios into executable test code.

**Design**:

```python
class GherkinParser:
    def parse(self, feature_content: str) -> list[GherkinScenario]:
        """
        Parse Gherkin feature file:
        Feature: User Registration
          Scenario: Successful registration
            Given a valid email "user@example.com"
            When the user submits the registration form
            Then a new user account is created
            And a confirmation email is sent
        """
        ...

    def generate_test_code(
        self,
        scenario: GherkinScenario,
        language: str,
        test_framework: str,
    ) -> str:
        """Generate test code from a Gherkin scenario."""
        ...
```

**Testing**:
- `Fixture: Parse sample feature file → correct scenario list with Given/When/Then steps`
- `Unit: Scenario Outline with Examples table → generates parameterised test`
- `Unit: generate_test_code for Python → pytest test with descriptive name`
- `Unit: generate_test_code for TypeScript → Jest test with descriptive name`

#### 10.3 — Spec-to-Test Generation API

**What**: API endpoint for uploading specs and generating tests before implementation.

**Design**:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| POST | `/api/v1/repos/{repo_id}/specs` | `{ "spec_type": "openapi", "content": "..." }` | `201: Specification` |
| POST | `/api/v1/repos/{repo_id}/specs/{spec_id}/generate` | `{ "language": "python", "test_framework": "pytest" }` | `202: { "session_id": "...", "task_id": "..." }` |

**Testing**:
- `Integration: Upload OpenAPI spec → 201, specification persisted`
- `Integration: Generate tests from OpenAPI spec → test code generated for each endpoint`
- `Integration: Upload Gherkin feature → tests generated matching Given/When/Then structure`

---

## Phase 11: Test Maintenance and Auto-Repair

### Purpose
Implement the capability to detect broken tests after code changes and auto-repair them using semantic diff analysis. After this phase, when production code evolves and tests break, the system can distinguish between stale assertions that need updating and genuine regressions that reveal real bugs. This is the capability that Diffblue Cover provides commercially with no open-source equivalent.

### Tasks

#### 11.1 — Semantic Diff Analysis

**What**: Compare two versions of a source function (before and after a code change) and classify the nature of the change.

**Design**:

```python
# src/testgen/services/repair.py
from dataclasses import dataclass
from enum import Enum

class ChangeType(Enum):
    SIGNATURE_CHANGE = "signature_change"     # Parameters added/removed/renamed
    RETURN_TYPE_CHANGE = "return_type_change"  # Return type changed
    BEHAVIORAL_CHANGE = "behavioral_change"    # Logic changed (if/else, loops)
    REFACTOR = "refactor"                      # Same behavior, different structure
    DEPENDENCY_CHANGE = "dependency_change"    # Imports or external calls changed
    DELETION = "deletion"                      # Function removed entirely

@dataclass
class SemanticDiff:
    old_source: str
    new_source: str
    change_types: list[ChangeType]
    changed_lines: list[int]
    summary: str                # LLM-generated summary of what changed
    is_breaking: bool           # Whether the change likely breaks existing tests
    is_intentional: bool | None # LLM assessment: intentional behavior change vs. bug?

class SemanticDiffService:
    async def analyze_diff(
        self,
        old_function: SourceFunction,
        new_function: SourceFunction,
        old_source: str,
        new_source: str,
        commit_message: str | None = None,
    ) -> SemanticDiff:
        """
        1. Parse both versions with tree-sitter
        2. Compare AST structure for structural changes
        3. Call LLM to classify the semantic nature of changes
        4. Determine if changes are likely to break tests
        """
        ...
```

**Testing**:
- `Unit: Function parameter renamed → ChangeType.SIGNATURE_CHANGE`
- `Unit: Function body refactored (same behavior) → ChangeType.REFACTOR, is_breaking=False`
- `Unit: New if-branch added → ChangeType.BEHAVIORAL_CHANGE, is_breaking=True`
- `Unit: Function deleted → ChangeType.DELETION`
- `Unit (mocked LLM): LLM correctly classifies intentional behavior change vs. accidental`

#### 11.2 — Test Repair Service

**What**: Given a broken test and a semantic diff, generate a repaired version of the test.

**Design**:

```python
class TestRepairService:
    async def repair_test(
        self,
        broken_test: GeneratedTest,
        test_result: TestResult,   # the failing result
        semantic_diff: SemanticDiff,
        session_id: uuid.UUID,
    ) -> TestRepair:
        """
        1. If diff is REFACTOR → update test to match new structure (rename, reorder)
        2. If diff is SIGNATURE_CHANGE → update test calls to match new signature
        3. If diff is BEHAVIORAL_CHANGE:
           a. Ask LLM: is the old assertion still correct? (regression detection)
           b. If regression → create TestRepair with is_regression=True, status="proposed"
           c. If intentional change → generate new assertion matching new behavior
        4. Persist TestRepair record
        5. Validate repaired test compiles and passes
        """
        ...
```

LLM prompt for repair:
```
A test broke after a code change. Determine whether this is a regression (bug) or an intentional change, then repair the test if appropriate.

Original function:
```{language}
{old_source}
```

Updated function:
```{language}
{new_source}
```

Change summary: {semantic_diff.summary}

Broken test:
```{language}
{broken_test.test_code}
```

Failure message: {test_result.failure_message}

If this is a REGRESSION (the old behavior was correct and the change introduced a bug):
- Set is_regression=true
- Explain why the old behavior was correct
- Do NOT repair the test — the code needs fixing, not the test

If this is an INTENTIONAL CHANGE:
- Set is_regression=false
- Generate the repaired test code that matches the new behavior
- Explain what changed in the assertion

Respond in JSON:
{
  "is_regression": true/false,
  "confidence": 0.0-1.0,
  "explanation": "...",
  "repaired_test_code": "..." // only if is_regression=false
}
```

**Testing**:
- `Unit (mocked LLM): Signature change → test updated with new parameter names`
- `Unit (mocked LLM): Behavioral regression → is_regression=True, test NOT modified`
- `Unit (mocked LLM): Intentional change → repaired test with updated assertions`
- `Integration (real DB): TestRepair record persisted with correct repair_type`
- `Integration (real DB): TestRepair links to correct breaking commit`
- `Unit: Repaired test validates (parses) successfully`

#### 11.3 — Repair API and Automation

**What**: API endpoints for triggering repair and auto-repair Celery task for CI integration.

**Design**:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| POST | `/api/v1/repos/{repo_id}/repair` | `{ "commit_sha": "...", "base_sha": "..." }` | `202: { "session_id": "...", "task_id": "..." }` |
| GET | `/api/v1/repos/{repo_id}/repairs` | Query: `status`, `is_regression`, `limit` | `200: list[TestRepair]` |
| POST | `/api/v1/repairs/{repair_id}/accept` | — | `200: TestRepair` (status→accepted) |
| POST | `/api/v1/repairs/{repair_id}/reject` | — | `200: TestRepair` (status→rejected) |

Auto-repair Celery task is triggered by the webhook handler when `pipeline_configurations.auto_repair = true`.

**Testing**:
- `Integration: POST repair → 202, repair session created`
- `Integration: GET repairs filtered by is_regression=true → only regression repairs returned`
- `Integration: Accept repair → test_repairs.status updated, generated_test.test_code updated`

---

## Phase 12: Coverage Trend Tracking and Dashboards

### Purpose
Implement coverage snapshot tracking across commits, trend visualization data, and regression alerts. After this phase, engineering managers and team leads can track coverage and mutation score trends over time, set up alerts when coverage drops, and understand the ROI of the test generator across their repositories.

### Tasks

#### 12.1 — Coverage Snapshot Service

**What**: After each test run on the default branch, create a coverage snapshot for trend tracking.

**Design**:

```python
class CoverageSnapshotService:
    async def create_snapshot(
        self,
        repository_id: uuid.UUID,
        commit_id: uuid.UUID,
        branch_name: str,
        coverage_report: CoverageReport,
        mutation_run: MutationRun | None,
    ) -> CoverageSnapshot:
        """Create a point-in-time coverage snapshot for trend tracking."""
        ...

    async def get_trend(
        self,
        repository_id: uuid.UUID,
        branch_name: str = "main",
        days: int = 30,
    ) -> list[CoverageSnapshot]:
        """Get coverage snapshots for the last N days."""
        ...

    async def detect_regression(
        self,
        repository_id: uuid.UUID,
        current_snapshot: CoverageSnapshot,
        threshold_drop_pct: float = 2.0,
    ) -> RegressionAlert | None:
        """Alert if coverage dropped more than threshold_drop_pct from the previous snapshot."""
        ...
```

**Testing**:
- `Integration (real DB): create_snapshot persists correct metrics`
- `Integration (real DB): get_trend returns snapshots in chronological order`
- `Integration (real DB): detect_regression returns alert when coverage drops by 3% (threshold=2%)`
- `Integration (real DB): detect_regression returns None when coverage is stable`

#### 12.2 — Trend API Endpoints

**What**: REST API for querying coverage trends and regression alerts.

**Design**:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| GET | `/api/v1/repos/{repo_id}/trends` | Query: `branch`, `days`, `metric` (line_coverage, branch_coverage, mutation_score) | `200: list[TrendPoint]` |
| GET | `/api/v1/repos/{repo_id}/alerts` | Query: `status`, `limit` | `200: list[RegressionAlert]` |

```python
class TrendPoint(BaseModel):
    commit_sha: str
    timestamp: datetime
    line_coverage_pct: float
    branch_coverage_pct: float
    mutation_score_pct: float | None
    total_tests: int
```

**Testing**:
- `Integration: GET trends for 30 days → list of TrendPoints`
- `Integration: GET trends for empty repo → empty list`
- `Integration: GET alerts → regression alerts for coverage drops`

#### 12.3 — OWASP Security Test Mapping

**What**: When generating tests for security-sensitive code, map generated tests to OWASP WSTG test IDs.

**Design**:

The generation prompt is augmented for functions that interact with authentication, authorization, input validation, or session management. The LLM is asked to classify which OWASP WSTG category the test addresses, and the mapping is persisted in `security_test_mappings`.

OWASP categories mapped:
- `WSTG-AUTHN-*`: Authentication testing
- `WSTG-AUTHZ-*`: Authorization testing
- `WSTG-INPVAL-*`: Input validation testing
- `WSTG-SESS-*`: Session management testing
- `WSTG-CRYPST-*`: Cryptography testing

**Testing**:
- `Unit (mocked LLM): Test for login function → mapped to WSTG-AUTHN-01`
- `Unit (mocked LLM): Test for SQL query → mapped to WSTG-INPVAL-05 (SQL Injection)`
- `Integration (real DB): security_test_mappings records persisted with correct FK`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Code Parsing                  ─── requires Phase 1
    │
Phase 3: Coverage Analysis             ─── requires Phase 2
    │
Phase 4: Test Generation Engine        ─── requires Phase 2, Phase 3
    │
Phase 5: Test Execution                ─── requires Phase 4
    │
    ├── Phase 6: CLI Interface         ─── requires Phase 5; can parallel with Phase 7, 8
    ├── Phase 7: CI/CD Integration     ─── requires Phase 5; can parallel with Phase 6, 8
    └── Phase 8: VS Code Extension     ─── requires Phase 5; can parallel with Phase 6, 7
         │
Phase 9: Mutation Testing              ─── requires Phase 5
    │
Phase 10: Spec-to-Test                 ─── requires Phase 4; can parallel with Phase 9
    │
Phase 11: Test Repair                  ─── requires Phase 5, Phase 9
    │
Phase 12: Trends & Security            ─── requires Phase 3, Phase 9
```

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specifications.
2. All unit tests pass (`pytest tests/unit/`).
3. All integration tests pass (`pytest tests/integration/`).
4. Linting passes (`ruff check src/ tests/`).
5. Formatting passes (`ruff format --check src/ tests/`).
6. Type checking passes (`mypy src/`).
7. Docker build succeeds (`docker build .`).
8. Feature works end-to-end with a manual smoke test (documented in task).
9. New configuration options documented in `config.py` with defaults.
10. New API endpoints appear in the auto-generated OpenAPI spec (`/docs`).
11. Database migrations created and tested (upgrade + downgrade).
12. Test coverage of new code is >= 80% line coverage.
