# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Intelligent Test Generator · Created: 2026-05-11

## Philosophy

This model uses a conventional relational backbone for the core entities that are well-understood and stable (organizations, repositories, users, generation sessions) but delegates language-specific, framework-specific, and rapidly-evolving metadata to JSONB columns. The key insight for an AI test generator is that Python, TypeScript, Java, and Go have fundamentally different AST structures, test framework conventions, coverage report formats, and mutation operator sets. Trying to normalize all of these into fixed columns produces either a sparse, nullable schema or an explosion of small per-language extension tables.

Instead, the hybrid approach stores the universal fields (file path, coverage percentage, mutant status) as typed relational columns with indexes and constraints, and stores the variable fields (language-specific AST metadata, framework-specific test configuration, tool-specific mutation operators) in JSONB columns with GIN indexes for containment queries. This mirrors how modern SaaS platforms like Stripe (metadata JSONB on every core entity) and GitHub (API responses with typed core fields plus extensible properties) handle domain variability.

The result is a schema with fewer tables than the fully normalized model, faster development velocity for MVP (adding a new language requires no schema migration -- just new JSONB structures), and the ability to query across languages using the relational core while drilling into language-specific details via JSONB operators.

**Best for:** Teams building an MVP that must support multiple languages from day one, where the metadata structures per language/framework are not yet fully known, and where development speed matters more than schema rigidity.

**Trade-offs:**
- (+) Fewer tables (20 vs. 27+); simpler migrations during rapid development
- (+) Adding a new language or framework requires no ALTER TABLE -- just new JSONB shapes
- (+) Universal queries (coverage across all languages) work on relational columns; deep queries use JSONB operators
- (+) JSONB GIN indexes support efficient containment queries (`@>` operator)
- (+) Natural fit for storing raw tool output (JaCoCo XML, LCOV, Stryker JSON) alongside parsed summaries
- (-) JSONB columns lack column-level constraints; validation must happen in application code
- (-) JSONB queries are slower than relational lookups for high-cardinality filtering
- (-) Schema documentation is critical -- without it, JSONB columns become opaque blobs
- (-) Type safety is weaker; ORM mapping of JSONB requires explicit type definitions
- (-) Storage overhead: JSONB stores field names repeatedly in every row

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC/IEEE 29119 Part 3 | Core test lifecycle entities (session, test, run, result) are relational; 29119 metadata extensions go in JSONB |
| Stryker Mutation Testing Report Schema | Raw Stryker JSON reports stored in `mutation_runs.raw_report`; parsed summary fields are relational columns |
| JUnit XML Format | Test results parsed into relational columns (name, status, duration); raw XML fragments in JSONB for framework-specific attributes |
| LCOV / JaCoCo / coverage.py | Coverage summary (line %, branch %) is relational; format-specific details (JaCoCo instruction counts, LCOV function hits) in JSONB |
| OpenAPI 3.1 / Gherkin | Specification content and parsed structures stored in JSONB; spec type and version are relational columns |
| ISO/IEC 25010 (SQuaRE) | Quality metric definitions stored as JSONB config; computed scores stored as relational NUMERIC columns |
| OWASP WSTG | Security mappings stored in a JSONB array on generated tests, avoiding a separate junction table |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- ORGANIZATIONS & USERS
-- Standard relational -- no JSONB needed for identity.
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'team', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_llm_model": "claude-opus-4-20250514",
    --   "max_tokens_per_session": 50000,
    --   "allowed_languages": ["python", "typescript", "java", "go"],
    --   "notification_channels": [{"type": "slack", "webhook_url": "..."}]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    auth_provider   TEXT NOT NULL,
    auth_provider_id TEXT,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "default_framework": "pytest",
    --   "ide": "vscode",
    --   "theme": "dark",
    --   "auto_accept_repairs": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, user_id)
);
```

## Repositories & Source Code

```sql
-- ============================================================
-- REPOSITORIES & SOURCE CODE
-- Relational core with JSONB for language-specific metadata.
-- ============================================================

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    url             TEXT,
    default_branch  TEXT NOT NULL DEFAULT 'main',
    vcs_provider    TEXT NOT NULL,
    vcs_provider_id TEXT,
    primary_language TEXT,
    language_breakdown JSONB NOT NULL DEFAULT '{}',
    -- language_breakdown example:
    -- {
    --   "python": {"files": 120, "lines": 15000, "percentage": 45.2},
    --   "typescript": {"files": 80, "lines": 12000, "percentage": 36.1},
    --   "java": {"files": 30, "lines": 6200, "percentage": 18.7}
    -- }
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "test_frameworks": {"python": "pytest", "typescript": "jest", "java": "junit5"},
    --   "coverage_tools": {"python": "coverage_py", "typescript": "istanbul", "java": "jacoco"},
    --   "mutation_tools": {"python": "mutmut", "typescript": "stryker", "java": "pitest"},
    --   "ignore_patterns": ["**/vendor/**", "**/node_modules/**", "**/*_test.go"],
    --   "min_coverage": 80.0,
    --   "min_mutation_score": 60.0
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE INDEX idx_repos_org ON repositories(organization_id);
CREATE INDEX idx_repos_language ON repositories(primary_language);

CREATE TABLE source_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    commit_sha      TEXT NOT NULL,
    commit_message  TEXT,
    author_name     TEXT,
    author_email    TEXT,
    committed_at    TIMESTAMPTZ,
    file_count      INTEGER NOT NULL DEFAULT 0,
    function_count  INTEGER NOT NULL DEFAULT 0,
    language_stats  JSONB NOT NULL DEFAULT '{}',
    -- language_stats: same structure as language_breakdown on repository
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, commit_sha)
);

CREATE INDEX idx_snapshots_repo ON source_snapshots(repository_id, committed_at DESC);

-- Source files: relational core + JSONB for AST and language-specific metadata
CREATE TABLE source_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_id     UUID NOT NULL REFERENCES source_snapshots(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    file_path       TEXT NOT NULL,
    language        TEXT NOT NULL,
    content_hash    TEXT NOT NULL,
    line_count      INTEGER NOT NULL,
    function_count  INTEGER NOT NULL DEFAULT 0,
    complexity_avg  NUMERIC(5,2),                  -- average cyclomatic complexity
    functions       JSONB NOT NULL DEFAULT '[]',
    -- functions example (Python):
    -- [
    --   {
    --     "name": "validate_token",
    --     "qualified_name": "auth.service.validate_token",
    --     "start_line": 45, "end_line": 72,
    --     "complexity": 6,
    --     "parameters": ["token", "audience"],
    --     "return_type": "TokenClaims",
    --     "decorators": ["@require_auth"],
    --     "is_async": true,
    --     "docstring": "Validates a JWT token..."
    --   }
    -- ]
    --
    -- functions example (Java):
    -- [
    --   {
    --     "name": "validateToken",
    --     "qualified_name": "com.auth.TokenService.validateToken",
    --     "start_line": 45, "end_line": 72,
    --     "complexity": 6,
    --     "parameters": [{"name": "token", "type": "String"}, {"name": "audience", "type": "String"}],
    --     "return_type": "TokenClaims",
    --     "annotations": ["@Override", "@Transactional"],
    --     "visibility": "public",
    --     "is_static": false,
    --     "throws": ["InvalidTokenException"]
    --   }
    -- ]
    imports         JSONB NOT NULL DEFAULT '[]',
    -- imports example:
    -- ["jwt", "datetime", "typing.Optional"]  (Python)
    -- ["java.util.Date", "io.jsonwebtoken.Jwts"]  (Java)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (snapshot_id, file_path)
);

CREATE INDEX idx_source_files_snapshot ON source_files(snapshot_id);
CREATE INDEX idx_source_files_language ON source_files(language);
CREATE INDEX idx_source_files_functions ON source_files USING GIN (functions);
```

## Test Generation

```sql
-- ============================================================
-- GENERATION SESSIONS & GENERATED TESTS
-- Session is relational; test code and metadata use JSONB
-- for framework-specific configuration.
-- ============================================================

CREATE TABLE generation_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    snapshot_id     UUID NOT NULL REFERENCES source_snapshots(id) ON DELETE CASCADE,
    triggered_by    UUID REFERENCES users(id),
    trigger_source  TEXT NOT NULL CHECK (trigger_source IN ('ide', 'cli', 'ci', 'api', 'scheduled')),
    strategy        TEXT NOT NULL CHECK (strategy IN (
                        'coverage_gap', 'mutation_guided', 'spec_to_test',
                        'full_suite', 'incremental', 'repair'
                    )),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
    target_languages TEXT[] NOT NULL DEFAULT '{}',
    llm_config      JSONB NOT NULL DEFAULT '{}',
    -- llm_config example:
    -- {
    --   "model": "claude-opus-4-20250514",
    --   "temperature": 0.2,
    --   "max_tokens": 4096,
    --   "system_prompt_version": "v2.3",
    --   "context_strategy": "coverage_guided_iterative"
    -- }
    results_summary JSONB NOT NULL DEFAULT '{}',
    -- results_summary (populated on completion):
    -- {
    --   "tests_generated": 12,
    --   "by_language": {"python": 5, "typescript": 4, "java": 3},
    --   "by_type": {"unit": 10, "integration": 2},
    --   "tokens_used": 28500,
    --   "duration_ms": 45000,
    --   "coverage_delta": {"line": "+5.2%", "branch": "+3.1%"}
    -- }
    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_repo ON generation_sessions(repository_id);
CREATE INDEX idx_sessions_status ON generation_sessions(status);
CREATE INDEX idx_sessions_created ON generation_sessions(created_at DESC);

CREATE TABLE generated_tests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES generation_sessions(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    source_file_id  UUID REFERENCES source_files(id),
    -- Core relational fields (queryable, indexable)
    language        TEXT NOT NULL,
    test_framework  TEXT NOT NULL,
    test_file_path  TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    test_code       TEXT NOT NULL,
    test_type       TEXT NOT NULL DEFAULT 'unit'
                    CHECK (test_type IN ('unit', 'integration', 'e2e', 'security', 'property')),
    status          TEXT NOT NULL DEFAULT 'generated'
                    CHECK (status IN ('generated', 'validated', 'accepted', 'rejected', 'stale', 'repaired')),
    -- JSONB for variable metadata
    target_function JSONB,
    -- target_function example:
    -- {
    --   "name": "validate_token",
    --   "qualified_name": "auth.service.validate_token",
    --   "file_path": "src/auth/service.py",
    --   "start_line": 45
    -- }
    description     TEXT,
    test_config     JSONB NOT NULL DEFAULT '{}',
    -- test_config example (pytest):
    -- {
    --   "fixtures": ["db_session", "mock_redis"],
    --   "markers": ["@pytest.mark.asyncio"],
    --   "parametrize": [
    --     {"params": ["expired_token", "malformed_token"], "ids": ["expired", "malformed"]}
    --   ]
    -- }
    --
    -- test_config example (JUnit):
    -- {
    --   "annotations": ["@ExtendWith(MockitoExtension.class)", "@DisplayName(\"Token Validation\")"],
    --   "mock_fields": [
    --     {"type": "TokenRepository", "name": "tokenRepo"}
    --   ],
    --   "parameterized_source": "@CsvSource({\"expired,false\", \"valid,true\"})"
    -- }
    mocks           JSONB NOT NULL DEFAULT '[]',
    -- mocks example:
    -- [
    --   {"target": "database.query", "type": "mock", "return_value": "[]"},
    --   {"target": "http_client.get", "type": "stub", "response": {"status": 200, "body": "{}"}}
    -- ]
    security_mappings JSONB NOT NULL DEFAULT '[]',
    -- security_mappings example:
    -- [
    --   {"owasp_id": "WSTG-AUTHN-01", "category": "Authentication", "description": "..."},
    --   {"owasp_id": "WSTG-INPVAL-05", "category": "Input Validation", "description": "..."}
    -- ]
    repair_history  JSONB NOT NULL DEFAULT '[]',
    -- repair_history example:
    -- [
    --   {
    --     "repair_id": "uuid",
    --     "breaking_commit": "abc123",
    --     "repair_type": "assertion_update",
    --     "original_code": "assert result == 42",
    --     "repaired_code": "assert result == 43",
    --     "is_regression": false,
    --     "confidence": 0.95,
    --     "repaired_at": "2026-05-10T14:30:00Z"
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tests_session ON generated_tests(session_id);
CREATE INDEX idx_tests_repo ON generated_tests(repository_id);
CREATE INDEX idx_tests_language ON generated_tests(language);
CREATE INDEX idx_tests_status ON generated_tests(status);
CREATE INDEX idx_tests_type ON generated_tests(test_type);
CREATE INDEX idx_tests_security ON generated_tests USING GIN (security_mappings);
```

## Test Execution & Coverage

```sql
-- ============================================================
-- TEST RUNS, RESULTS & COVERAGE
-- Summary fields are relational; per-line detail and
-- format-specific data in JSONB.
-- ============================================================

CREATE TABLE test_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    snapshot_id     UUID NOT NULL REFERENCES source_snapshots(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES generation_sessions(id),
    run_source      TEXT NOT NULL CHECK (run_source IN ('ci', 'ide', 'cli', 'validation')),
    runner          TEXT NOT NULL,
    -- Relational summary (queryable)
    status          TEXT NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'passed', 'failed', 'error', 'cancelled')),
    total_tests     INTEGER NOT NULL DEFAULT 0,
    passed          INTEGER NOT NULL DEFAULT 0,
    failed          INTEGER NOT NULL DEFAULT 0,
    errors          INTEGER NOT NULL DEFAULT 0,
    skipped         INTEGER NOT NULL DEFAULT 0,
    duration_ms     INTEGER,
    -- JSONB for per-test-case results (avoids a separate high-volume table)
    test_results    JSONB NOT NULL DEFAULT '[]',
    -- test_results example:
    -- [
    --   {
    --     "test_name": "test_validate_token_expired",
    --     "class_name": "TestTokenValidation",
    --     "file_path": "tests/test_auth.py",
    --     "status": "passed",
    --     "duration_ms": 45,
    --     "generated_test_id": "uuid-or-null"
    --   },
    --   {
    --     "test_name": "test_validate_token_malformed",
    --     "class_name": "TestTokenValidation",
    --     "file_path": "tests/test_auth.py",
    --     "status": "failed",
    --     "duration_ms": 12,
    --     "failure_message": "AssertionError: expected 401 got 200",
    --     "failure_type": "AssertionError",
    --     "stack_trace": "...",
    --     "generated_test_id": "uuid-or-null"
    --   }
    -- ]
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runs_repo ON test_runs(repository_id);
CREATE INDEX idx_runs_snapshot ON test_runs(snapshot_id);
CREATE INDEX idx_runs_status ON test_runs(status);
CREATE INDEX idx_runs_created ON test_runs(created_at DESC);
CREATE INDEX idx_runs_results ON test_runs USING GIN (test_results);

CREATE TABLE coverage_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    test_run_id     UUID NOT NULL REFERENCES test_runs(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    snapshot_id     UUID NOT NULL REFERENCES source_snapshots(id) ON DELETE CASCADE,
    -- Relational summary (for trend queries and threshold checks)
    source_format   TEXT NOT NULL,                 -- 'lcov', 'jacoco', 'coverage_py', 'cobertura', 'istanbul'
    total_lines     INTEGER NOT NULL DEFAULT 0,
    covered_lines   INTEGER NOT NULL DEFAULT 0,
    total_branches  INTEGER NOT NULL DEFAULT 0,
    covered_branches INTEGER NOT NULL DEFAULT 0,
    line_coverage_pct   NUMERIC(5,2),
    branch_coverage_pct NUMERIC(5,2),
    -- JSONB for per-file coverage detail
    file_coverage   JSONB NOT NULL DEFAULT '[]',
    -- file_coverage example:
    -- [
    --   {
    --     "file_path": "src/auth/service.py",
    --     "source_file_id": "uuid",
    --     "lines_total": 200,
    --     "lines_covered": 156,
    --     "branches_total": 40,
    --     "branches_covered": 28,
    --     "line_pct": 78.00,
    --     "branch_pct": 70.00,
    --     "uncovered_lines": [45, 46, 67, 68, 69, 70, 71, 72],
    --     "uncovered_branches": [
    --       {"line": 45, "block": 0, "branch": 1},
    --       {"line": 67, "block": 0, "branch": 0}
    --     ],
    --     "format_specific": {
    --       "jacoco": {"mi": 44, "ci": 156, "mb": 12, "cb": 28}
    --     }
    --   }
    -- ]
    -- Raw report stored for debugging and re-parsing
    raw_report      TEXT,                          -- original LCOV/XML/JSON content
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_run ON coverage_reports(test_run_id);
CREATE INDEX idx_coverage_repo ON coverage_reports(repository_id);
CREATE INDEX idx_coverage_snapshot ON coverage_reports(snapshot_id);
CREATE INDEX idx_coverage_files ON coverage_reports USING GIN (file_coverage);

-- Coverage gaps: relational for queryability
CREATE TABLE coverage_gaps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coverage_report_id UUID NOT NULL REFERENCES coverage_reports(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    file_path       TEXT NOT NULL,
    function_name   TEXT NOT NULL,
    function_qualified_name TEXT NOT NULL,
    gap_type        TEXT NOT NULL CHECK (gap_type IN (
                        'uncovered_function', 'uncovered_branch', 'complex_setup',
                        'external_dependency', 'concurrency', 'error_path'
                    )),
    severity        TEXT NOT NULL DEFAULT 'medium'
                    CHECK (severity IN ('critical', 'high', 'medium', 'low')),
    explanation     TEXT NOT NULL,
    suggested_approach TEXT,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "uncovered_lines": [67, 68, 69, 70],
    --   "blocking_dependencies": ["redis_client", "external_api"],
    --   "suggested_mocks": ["redis_client.get", "external_api.fetch"],
    --   "complexity_score": 8,
    --   "estimated_effort": "medium"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gaps_report ON coverage_gaps(coverage_report_id);
CREATE INDEX idx_gaps_repo ON coverage_gaps(repository_id);
CREATE INDEX idx_gaps_severity ON coverage_gaps(severity);
CREATE INDEX idx_gaps_resolved ON coverage_gaps(resolved) WHERE NOT resolved;
```

## Mutation Testing

```sql
-- ============================================================
-- MUTATION TESTING
-- Summary relational; per-mutant detail in JSONB.
-- Aligned with Stryker mutation-testing-report-schema.
-- ============================================================

CREATE TABLE mutation_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES generation_sessions(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    snapshot_id     UUID NOT NULL REFERENCES source_snapshots(id) ON DELETE CASCADE,
    tool            TEXT NOT NULL,                  -- 'pitest', 'stryker', 'mutmut', 'custom'
    -- Relational summary
    status          TEXT NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'completed', 'failed', 'cancelled')),
    total_mutants   INTEGER NOT NULL DEFAULT 0,
    killed          INTEGER NOT NULL DEFAULT 0,
    survived        INTEGER NOT NULL DEFAULT 0,
    no_coverage     INTEGER NOT NULL DEFAULT 0,
    timeout         INTEGER NOT NULL DEFAULT 0,
    compile_error   INTEGER NOT NULL DEFAULT 0,
    mutation_score  NUMERIC(5,2),
    duration_ms     INTEGER,
    -- JSONB for per-mutant results (aligned with Stryker schema)
    mutants         JSONB NOT NULL DEFAULT '[]',
    -- mutants example:
    -- [
    --   {
    --     "id": "mutant-001",
    --     "mutatorName": "BooleanSubstitution",
    --     "replacement": "false",
    --     "description": "Replaced 'is_valid' with 'false'",
    --     "location": {
    --       "file": "src/auth/service.py",
    --       "start": {"line": 52, "column": 12},
    --       "end": {"line": 52, "column": 20}
    --     },
    --     "status": "Killed",
    --     "statusReason": "AssertionError in test_validate_token_expired",
    --     "killedBy": ["test-uuid-123"],
    --     "coveredBy": ["test-uuid-123", "test-uuid-456"],
    --     "testsCompleted": 2,
    --     "netTime": 45
    --   },
    --   {
    --     "id": "mutant-002",
    --     "mutatorName": "ConditionalBoundary",
    --     "replacement": "<= instead of <",
    --     "location": {
    --       "file": "src/auth/service.py",
    --       "start": {"line": 58, "column": 8},
    --       "end": {"line": 58, "column": 15}
    --     },
    --     "status": "Survived",
    --     "coveredBy": ["test-uuid-789"],
    --     "testsCompleted": 1,
    --     "netTime": 30
    --   }
    -- ]
    -- Store the original Stryker/PITest report for interoperability
    raw_report      JSONB,                         -- original tool output
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "mutation_operators": ["BooleanSubstitution", "ArithmeticOperator", "ConditionalBoundary"],
    --   "target_files": ["src/auth/**"],
    --   "excluded_mutators": ["StringLiteral"],
    --   "timeout_factor": 1.5
    -- }
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mutation_session ON mutation_runs(session_id);
CREATE INDEX idx_mutation_repo ON mutation_runs(repository_id);
CREATE INDEX idx_mutation_status ON mutation_runs(status);
CREATE INDEX idx_mutation_mutants ON mutation_runs USING GIN (mutants);
```

## Specifications

```sql
-- ============================================================
-- SPECIFICATIONS (OpenAPI, Gherkin, Natural Language)
-- Relational envelope; parsed content in JSONB.
-- ============================================================

CREATE TABLE specifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    spec_type       TEXT NOT NULL CHECK (spec_type IN ('openapi', 'gherkin', 'natural_language', 'adr')),
    file_path       TEXT,
    title           TEXT NOT NULL,
    version         TEXT,                          -- spec version (e.g. "3.1.0" for OpenAPI)
    raw_content     TEXT NOT NULL,                 -- original file content
    parsed          JSONB NOT NULL DEFAULT '{}',
    -- parsed example (OpenAPI):
    -- {
    --   "openapi": "3.1.0",
    --   "paths": {
    --     "/auth/token": {
    --       "post": {
    --         "operationId": "createToken",
    --         "parameters": [...],
    --         "requestBody": {...},
    --         "responses": {"200": {...}, "401": {...}}
    --       }
    --     }
    --   },
    --   "schemas": {
    --     "TokenRequest": {"type": "object", "properties": {...}},
    --     "TokenResponse": {"type": "object", "properties": {...}}
    --   }
    -- }
    --
    -- parsed example (Gherkin):
    -- {
    --   "feature": "Token Validation",
    --   "scenarios": [
    --     {
    --       "name": "Valid token is accepted",
    --       "steps": [
    --         {"keyword": "Given", "text": "a valid JWT token"},
    --         {"keyword": "When", "text": "the token is validated"},
    --         {"keyword": "Then", "text": "the response status is 200"}
    --       ]
    --     }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_specs_repo ON specifications(repository_id);
CREATE INDEX idx_specs_type ON specifications(spec_type);
CREATE INDEX idx_specs_parsed ON specifications USING GIN (parsed);
```

## CI/CD & Pipeline Integration

```sql
-- ============================================================
-- CI/CD PIPELINE CONFIGURATION & RESULTS
-- Config uses JSONB for CI-provider-specific settings.
-- ============================================================

CREATE TABLE pipeline_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    ci_provider     TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    thresholds      JSONB NOT NULL DEFAULT '{}',
    -- thresholds example:
    -- {
    --   "min_line_coverage": 80.0,
    --   "min_branch_coverage": 60.0,
    --   "min_mutation_score": 50.0,
    --   "max_surviving_critical_mutants": 0,
    --   "require_security_tests": true
    -- }
    triggers        JSONB NOT NULL DEFAULT '{}',
    -- triggers example:
    -- {
    --   "on_pr": true,
    --   "on_push_to_main": true,
    --   "on_schedule": "0 2 * * 1",
    --   "auto_generate": true,
    --   "auto_repair": false,
    --   "target_languages": ["python", "typescript"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, ci_provider)
);

CREATE TABLE pipeline_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_config_id UUID NOT NULL REFERENCES pipeline_configs(id) ON DELETE CASCADE,
    snapshot_id     UUID NOT NULL REFERENCES source_snapshots(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES generation_sessions(id),
    test_run_id     UUID REFERENCES test_runs(id),
    external_run_id TEXT,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'passed', 'failed', 'error')),
    gate_evaluation JSONB NOT NULL DEFAULT '{}',
    -- gate_evaluation example:
    -- {
    --   "gate_result": "failed",
    --   "checks": [
    --     {"metric": "line_coverage", "threshold": 80.0, "actual": 75.3, "passed": false},
    --     {"metric": "branch_coverage", "threshold": 60.0, "actual": 62.1, "passed": true},
    --     {"metric": "mutation_score", "threshold": 50.0, "actual": 55.2, "passed": true}
    --   ],
    --   "summary": "Line coverage 75.3% is below threshold 80.0%"
    -- }
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pipeline_runs_config ON pipeline_runs(pipeline_config_id);
CREATE INDEX idx_pipeline_runs_status ON pipeline_runs(status);
```

## Coverage Trends

```sql
-- ============================================================
-- COVERAGE SNAPSHOTS (for dashboard trend charts)
-- Fully relational for efficient time-series queries.
-- ============================================================

CREATE TABLE coverage_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    snapshot_id     UUID NOT NULL REFERENCES source_snapshots(id) ON DELETE CASCADE,
    branch_name     TEXT NOT NULL DEFAULT 'main',
    line_coverage_pct   NUMERIC(5,2),
    branch_coverage_pct NUMERIC(5,2),
    mutation_score_pct  NUMERIC(5,2),
    total_tests         INTEGER NOT NULL DEFAULT 0,
    total_functions     INTEGER NOT NULL DEFAULT 0,
    covered_functions   INTEGER NOT NULL DEFAULT 0,
    per_language        JSONB NOT NULL DEFAULT '{}',
    -- per_language example:
    -- {
    --   "python": {"line_pct": 82.0, "branch_pct": 65.0, "mutation_pct": 58.0, "tests": 120},
    --   "typescript": {"line_pct": 74.0, "branch_pct": 55.0, "mutation_pct": 42.0, "tests": 85}
    -- }
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trend_repo ON coverage_snapshots(repository_id, recorded_at DESC);
CREATE INDEX idx_trend_branch ON coverage_snapshots(repository_id, branch_name, recorded_at DESC);
```

## API Keys

```sql
-- ============================================================
-- API KEYS
-- ============================================================

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    created_by      UUID NOT NULL REFERENCES users(id),
    name            TEXT NOT NULL,
    key_hash        TEXT NOT NULL,
    key_prefix      TEXT NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{read}',
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_org ON api_keys(organization_id);
CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
```

---

## Example: Querying Surviving Mutants Across Languages

```sql
-- Find all surviving mutants in a repository, across all languages
SELECT
    mr.tool,
    m->>'mutatorName' AS mutator,
    m->'location'->>'file' AS file_path,
    (m->'location'->'start'->>'line')::int AS line,
    m->>'replacement' AS replacement,
    m->>'status' AS status
FROM mutation_runs mr,
     jsonb_array_elements(mr.mutants) AS m
WHERE mr.repository_id = 'repo-uuid'
  AND mr.status = 'completed'
  AND m->>'status' = 'Survived'
ORDER BY file_path, line;
```

## Example: Coverage by Language Over Time

```sql
-- Coverage trend for Python in a repository over the last 30 days
SELECT
    cs.recorded_at::date AS date,
    (cs.per_language->'python'->>'line_pct')::numeric AS python_line_coverage,
    (cs.per_language->'python'->>'mutation_pct')::numeric AS python_mutation_score
FROM coverage_snapshots cs
WHERE cs.repository_id = 'repo-uuid'
  AND cs.branch_name = 'main'
  AND cs.recorded_at >= now() - interval '30 days'
ORDER BY cs.recorded_at;
```

## Example: Finding Tests with OWASP Security Mappings

```sql
-- Find all security-mapped tests for OWASP Authentication category
SELECT
    gt.test_name,
    gt.language,
    gt.test_file_path,
    sm->>'owasp_id' AS owasp_id,
    sm->>'description' AS mapping_description
FROM generated_tests gt,
     jsonb_array_elements(gt.security_mappings) AS sm
WHERE gt.repository_id = 'repo-uuid'
  AND sm->>'category' = 'Authentication';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organizations, users, organization_members |
| Repository & Source Code | 3 | repositories, source_snapshots, source_files |
| Specifications | 1 | specifications (with parsed JSONB) |
| Test Generation | 2 | generation_sessions, generated_tests |
| Test Execution & Coverage | 3 | test_runs, coverage_reports, coverage_gaps |
| Mutation Testing | 1 | mutation_runs (mutants in JSONB array) |
| CI/CD Integration | 2 | pipeline_configs, pipeline_runs |
| Coverage Trends | 1 | coverage_snapshots |
| API Keys | 1 | api_keys |
| **Total** | **17** | Compared to 27 in the normalized model |

---

## Key Design Decisions

1. **Functions as JSONB arrays on source_files** -- rather than a separate `source_functions` table, function metadata is stored as a JSONB array on each source file. This eliminates a high-volume table and keeps the "file + its functions" as a single document. GIN indexes support containment queries for finding specific functions.

2. **Test results as JSONB array on test_runs** -- individual test case results (passed, failed, error, skipped) are stored as a JSONB array on the test run rather than in a separate table. For a typical test run with 50-500 results, the JSONB array is efficient and avoids thousands of rows per run. The summary counts (total, passed, failed) remain as relational columns for fast aggregation.

3. **Mutants as JSONB array on mutation_runs** -- the full mutant list is stored as a JSONB array directly aligned with the Stryker mutation-testing-report-schema structure. This means the raw Stryker report can be stored almost verbatim, and the tool's native JSON format drives the JSONB structure. Summary counts are relational columns.

4. **Per-file coverage in JSONB, summary in relational columns** -- `coverage_reports.file_coverage` stores per-file detail (including uncovered line numbers and format-specific data like JaCoCo instruction counts) as JSONB, while the aggregate `line_coverage_pct` and `branch_coverage_pct` are relational NUMERIC columns. This enables fast threshold checks (relational) and detailed drill-down (JSONB).

5. **Repair history as JSONB array on generated_tests** -- rather than a separate `test_repairs` table, the repair history is an append-only JSONB array on each generated test. This keeps the test and its repair history together as a single document, simplifying queries like "show me this test and everything that has happened to it."

6. **Security mappings as JSONB array** -- OWASP WSTG mappings are stored as a JSONB array on `generated_tests` rather than in a junction table. This avoids a separate table for what is typically 0-3 mappings per test, and GIN indexes support efficient containment queries.

7. **Language-specific metadata in JSONB, language identifier in relational column** -- the `language` column is always a typed relational field (indexed, constrained). The framework-specific details (pytest fixtures vs. JUnit annotations, Python decorators vs. Java annotations) live in JSONB. Adding Rust support requires no schema migration -- just a new JSONB shape.

8. **Repository config as JSONB** -- test framework, coverage tool, and mutation tool configuration per language is stored as JSONB on the repository. This avoids a `repository_language_configs` junction table and keeps all repo configuration in one place.

9. **Raw reports preserved** -- `coverage_reports.raw_report` and `mutation_runs.raw_report` store the original tool output (LCOV text, JaCoCo XML, Stryker JSON). This enables re-parsing if the JSONB extraction logic improves, and provides a debugging escape hatch.

10. **Fewer tables, same query power** -- 17 tables vs. 27 in the normalized model. The trade-off is that JSONB queries are slightly slower for high-cardinality filtering, but for the expected data volumes (thousands of repos, not millions), GIN indexes make JSONB containment queries fast enough for interactive dashboards.
