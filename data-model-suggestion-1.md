# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Intelligent Test Generator · Created: 2026-05-11

## Philosophy

This model follows a fully normalized relational design where every concept in the domain has its own dedicated table with strict foreign key constraints. The source of truth is always the current state of each row. Entities are modeled around the natural domain boundaries of the test generation lifecycle: source code repositories, analysis sessions, generated tests, coverage measurements, and mutation testing results.

The normalized approach reflects how established test management platforms (TestRail, Helix ALM, Qodo) model their data internally: discrete, queryable entities with explicit relationships. It prioritizes data integrity, complex cross-entity queries (e.g., "which functions in this repository have never had a mutation-killing test generated?"), and straightforward migration paths as the schema evolves.

This model is best suited for teams that value query flexibility and referential integrity over write throughput, and where the expected data volume per tenant is moderate (thousands of repositories, millions of test cases, not billions of events).

**Best for:** Teams building a stable, query-heavy platform where complex cross-entity reporting and coverage analytics are core requirements.

**Trade-offs:**
- (+) Full referential integrity prevents orphaned records and inconsistent state
- (+) Complex analytical queries (joins across coverage, mutations, test results) are natural SQL
- (+) Standard ORM mapping; straightforward for most backend developers
- (+) Schema evolution through standard migrations (ALTER TABLE)
- (-) More tables to maintain (40+); schema migrations require coordination
- (-) Write-heavy operations (bulk test result ingestion) may bottleneck on foreign key checks
- (-) Temporal queries ("what was the coverage on March 15?") require explicit snapshot tables
- (-) Multi-language metadata variations (Python vs. Java AST structures) require nullable columns or many small extension tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC/IEEE 29119 Part 3 | Test plan, test case, and test result entities mirror the 29119 documentation hierarchy |
| IEEE 829 | Test artifact structure (plan → design → case → procedure → log → report) maps to table relationships |
| ISO/IEC 25010 (SQuaRE) | Quality metrics tables align with reliability sub-characteristics; mutation score operationalizes defect-detection power |
| OpenAPI 3.1 | `api_specifications` table stores parsed OpenAPI schemas for spec-to-test generation |
| Gherkin/BDD | `specifications` table supports Gherkin feature files as input artifacts |
| Stryker Mutation Testing Report Schema | `mutants` and `mutation_runs` tables align with the Stryker schema fields (id, mutatorName, status, location, killedBy) |
| JUnit XML Format | `test_results` table captures the attributes from JUnit XML (name, classname, time, failure message) |
| OWASP WSTG | `security_test_mappings` table links generated tests to OWASP test IDs |
| LCOV / JaCoCo / coverage.py | `coverage_lines` table normalizes coverage data from all three formats into a unified schema |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- ORGANIZATIONS & USERS
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'team', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    auth_provider   TEXT NOT NULL CHECK (auth_provider IN ('github', 'gitlab', 'google', 'email')),
    auth_provider_id TEXT,
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

CREATE INDEX idx_org_members_user ON organization_members(user_id);
```

## Repository & Source Code Management

```sql
-- ============================================================
-- REPOSITORIES & SOURCE FILES
-- ============================================================

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    url             TEXT,                          -- git remote URL
    default_branch  TEXT NOT NULL DEFAULT 'main',
    vcs_provider    TEXT NOT NULL CHECK (vcs_provider IN ('github', 'gitlab', 'bitbucket', 'local')),
    vcs_provider_id TEXT,                          -- external repo ID
    primary_language TEXT,                         -- detected primary language
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE INDEX idx_repos_org ON repositories(organization_id);

CREATE TABLE commits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    sha             TEXT NOT NULL,
    message         TEXT,
    author_name     TEXT,
    author_email    TEXT,
    committed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, sha)
);

CREATE INDEX idx_commits_repo_sha ON commits(repository_id, sha);

CREATE TABLE source_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    commit_id       UUID NOT NULL REFERENCES commits(id) ON DELETE CASCADE,
    file_path       TEXT NOT NULL,                 -- relative path within repo
    language        TEXT NOT NULL CHECK (language IN ('python', 'typescript', 'javascript', 'java', 'go', 'rust', 'other')),
    content_hash    TEXT NOT NULL,                 -- SHA-256 of file content
    line_count      INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (commit_id, file_path)
);

CREATE INDEX idx_source_files_repo ON source_files(repository_id);
CREATE INDEX idx_source_files_commit ON source_files(commit_id);
CREATE INDEX idx_source_files_language ON source_files(language);

CREATE TABLE source_functions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_file_id  UUID NOT NULL REFERENCES source_files(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    qualified_name  TEXT NOT NULL,                 -- e.g. "MyClass.my_method" or "myModule.myFunction"
    start_line      INTEGER NOT NULL,
    end_line        INTEGER NOT NULL,
    complexity      INTEGER,                       -- cyclomatic complexity
    parameter_count INTEGER,
    is_public       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_source_functions_file ON source_functions(source_file_id);
```

## Specifications (Input Artifacts)

```sql
-- ============================================================
-- SPECIFICATIONS (OpenAPI, Gherkin, Requirements)
-- ============================================================

CREATE TABLE specifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    spec_type       TEXT NOT NULL CHECK (spec_type IN ('openapi', 'gherkin', 'natural_language', 'adr')),
    file_path       TEXT,                          -- path within repo if file-based
    title           TEXT NOT NULL,
    content         TEXT NOT NULL,                 -- raw spec content
    parsed_version  TEXT,                          -- e.g. "3.1.0" for OpenAPI
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_specifications_repo ON specifications(repository_id);
CREATE INDEX idx_specifications_type ON specifications(spec_type);
```

## Test Generation & Management

```sql
-- ============================================================
-- GENERATION SESSIONS & GENERATED TESTS
-- ============================================================

CREATE TABLE generation_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    commit_id       UUID NOT NULL REFERENCES commits(id) ON DELETE CASCADE,
    triggered_by    UUID REFERENCES users(id),
    trigger_source  TEXT NOT NULL CHECK (trigger_source IN ('ide', 'cli', 'ci', 'api', 'scheduled')),
    strategy        TEXT NOT NULL CHECK (strategy IN (
                        'coverage_gap', 'mutation_guided', 'spec_to_test',
                        'full_suite', 'incremental', 'repair'
                    )),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
    llm_model       TEXT,                          -- e.g. "claude-opus-4-20250514"
    llm_tokens_used INTEGER DEFAULT 0,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gen_sessions_repo ON generation_sessions(repository_id);
CREATE INDEX idx_gen_sessions_status ON generation_sessions(status);
CREATE INDEX idx_gen_sessions_created ON generation_sessions(created_at DESC);

CREATE TABLE generated_tests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES generation_sessions(id) ON DELETE CASCADE,
    source_function_id UUID REFERENCES source_functions(id),
    specification_id UUID REFERENCES specifications(id),
    test_framework  TEXT NOT NULL CHECK (test_framework IN ('pytest', 'jest', 'junit4', 'junit5', 'go_test', 'vitest')),
    test_file_path  TEXT NOT NULL,                 -- intended output path
    test_name       TEXT NOT NULL,                 -- test function/method name
    test_class_name TEXT,                          -- for JUnit-style frameworks
    test_code       TEXT NOT NULL,                 -- generated test source code
    description     TEXT,                          -- plain-language description of what this test verifies
    test_type       TEXT NOT NULL DEFAULT 'unit'
                    CHECK (test_type IN ('unit', 'integration', 'e2e', 'security', 'property')),
    status          TEXT NOT NULL DEFAULT 'generated'
                    CHECK (status IN ('generated', 'validated', 'accepted', 'rejected', 'stale')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_generated_tests_session ON generated_tests(session_id);
CREATE INDEX idx_generated_tests_function ON generated_tests(source_function_id);
CREATE INDEX idx_generated_tests_status ON generated_tests(status);

-- Mocks and fixtures generated alongside tests
CREATE TABLE generated_mocks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    generated_test_id UUID NOT NULL REFERENCES generated_tests(id) ON DELETE CASCADE,
    mock_target     TEXT NOT NULL,                 -- what is being mocked (e.g. "database.query", "http.get")
    mock_type       TEXT NOT NULL CHECK (mock_type IN ('stub', 'mock', 'spy', 'fake', 'fixture')),
    mock_code       TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_generated_mocks_test ON generated_mocks(generated_test_id);
```

## Test Execution & Results

```sql
-- ============================================================
-- TEST RUNS & RESULTS
-- ============================================================

CREATE TABLE test_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    commit_id       UUID NOT NULL REFERENCES commits(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES generation_sessions(id),
    run_source      TEXT NOT NULL CHECK (run_source IN ('ci', 'ide', 'cli', 'validation')),
    runner          TEXT NOT NULL CHECK (runner IN ('pytest', 'jest', 'junit', 'go_test', 'vitest')),
    status          TEXT NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'passed', 'failed', 'error', 'cancelled')),
    total_tests     INTEGER NOT NULL DEFAULT 0,
    passed          INTEGER NOT NULL DEFAULT 0,
    failed          INTEGER NOT NULL DEFAULT 0,
    errors          INTEGER NOT NULL DEFAULT 0,
    skipped         INTEGER NOT NULL DEFAULT 0,
    duration_ms     INTEGER,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_test_runs_repo ON test_runs(repository_id);
CREATE INDEX idx_test_runs_commit ON test_runs(commit_id);
CREATE INDEX idx_test_runs_created ON test_runs(created_at DESC);

-- Individual test case results (aligned with JUnit XML schema)
CREATE TABLE test_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    test_run_id     UUID NOT NULL REFERENCES test_runs(id) ON DELETE CASCADE,
    generated_test_id UUID REFERENCES generated_tests(id),
    test_name       TEXT NOT NULL,                 -- JUnit XML: name attribute
    class_name      TEXT,                          -- JUnit XML: classname attribute
    file_path       TEXT,
    status          TEXT NOT NULL CHECK (status IN ('passed', 'failed', 'error', 'skipped')),
    duration_ms     INTEGER,
    failure_message TEXT,                          -- JUnit XML: <failure> message
    failure_type    TEXT,                          -- JUnit XML: <failure> type
    stack_trace     TEXT,                          -- JUnit XML: <failure> body
    system_out      TEXT,                          -- JUnit XML: <system-out>
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_test_results_run ON test_results(test_run_id);
CREATE INDEX idx_test_results_generated ON test_results(generated_test_id);
CREATE INDEX idx_test_results_status ON test_results(status);
```

## Coverage Analysis

```sql
-- ============================================================
-- COVERAGE DATA
-- (Unified model normalizing LCOV, JaCoCo XML, coverage.py)
-- ============================================================

CREATE TABLE coverage_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    test_run_id     UUID NOT NULL REFERENCES test_runs(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    commit_id       UUID NOT NULL REFERENCES commits(id) ON DELETE CASCADE,
    source_format   TEXT NOT NULL CHECK (source_format IN ('lcov', 'jacoco', 'coverage_py', 'cobertura', 'istanbul')),
    total_lines     INTEGER NOT NULL DEFAULT 0,
    covered_lines   INTEGER NOT NULL DEFAULT 0,
    total_branches  INTEGER NOT NULL DEFAULT 0,
    covered_branches INTEGER NOT NULL DEFAULT 0,
    line_coverage_pct  NUMERIC(5,2),              -- e.g. 78.50
    branch_coverage_pct NUMERIC(5,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_reports_run ON coverage_reports(test_run_id);
CREATE INDEX idx_coverage_reports_repo_commit ON coverage_reports(repository_id, commit_id);

CREATE TABLE coverage_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coverage_report_id UUID NOT NULL REFERENCES coverage_reports(id) ON DELETE CASCADE,
    source_file_id  UUID REFERENCES source_files(id),
    file_path       TEXT NOT NULL,
    total_lines     INTEGER NOT NULL DEFAULT 0,
    covered_lines   INTEGER NOT NULL DEFAULT 0,
    total_branches  INTEGER NOT NULL DEFAULT 0,
    covered_branches INTEGER NOT NULL DEFAULT 0,
    line_coverage_pct  NUMERIC(5,2),
    branch_coverage_pct NUMERIC(5,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_files_report ON coverage_files(coverage_report_id);
CREATE INDEX idx_coverage_files_source ON coverage_files(source_file_id);

-- Per-line coverage detail (high volume; consider partitioning)
CREATE TABLE coverage_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coverage_file_id UUID NOT NULL REFERENCES coverage_files(id) ON DELETE CASCADE,
    line_number     INTEGER NOT NULL,
    hit_count       INTEGER NOT NULL DEFAULT 0,    -- LCOV: DA line execution count
    is_branch       BOOLEAN NOT NULL DEFAULT false,
    branch_taken    INTEGER,                       -- LCOV: BRDA taken count; JaCoCo: cb
    branch_missed   INTEGER,                       -- JaCoCo: mb
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (coverage_file_id, line_number)
);

CREATE INDEX idx_coverage_lines_file ON coverage_lines(coverage_file_id);

-- Coverage gap analysis results
CREATE TABLE coverage_gaps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coverage_report_id UUID NOT NULL REFERENCES coverage_reports(id) ON DELETE CASCADE,
    source_function_id UUID NOT NULL REFERENCES source_functions(id),
    gap_type        TEXT NOT NULL CHECK (gap_type IN (
                        'uncovered_function', 'uncovered_branch', 'complex_setup',
                        'external_dependency', 'concurrency', 'error_path'
                    )),
    severity        TEXT NOT NULL DEFAULT 'medium'
                    CHECK (severity IN ('critical', 'high', 'medium', 'low')),
    explanation     TEXT NOT NULL,                 -- LLM-generated explanation of why this gap exists
    suggested_approach TEXT,                       -- LLM-suggested scaffolding approach
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_gaps_report ON coverage_gaps(coverage_report_id);
CREATE INDEX idx_coverage_gaps_function ON coverage_gaps(source_function_id);
CREATE INDEX idx_coverage_gaps_severity ON coverage_gaps(severity);
```

## Mutation Testing

```sql
-- ============================================================
-- MUTATION TESTING
-- (Aligned with Stryker mutation-testing-report-schema)
-- ============================================================

CREATE TABLE mutation_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES generation_sessions(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    commit_id       UUID NOT NULL REFERENCES commits(id) ON DELETE CASCADE,
    tool            TEXT NOT NULL CHECK (tool IN ('pitest', 'stryker', 'mutmut', 'custom')),
    status          TEXT NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'completed', 'failed', 'cancelled')),
    total_mutants   INTEGER NOT NULL DEFAULT 0,
    killed          INTEGER NOT NULL DEFAULT 0,
    survived        INTEGER NOT NULL DEFAULT 0,
    no_coverage     INTEGER NOT NULL DEFAULT 0,
    timeout         INTEGER NOT NULL DEFAULT 0,
    compile_error   INTEGER NOT NULL DEFAULT 0,
    runtime_error   INTEGER NOT NULL DEFAULT 0,
    ignored         INTEGER NOT NULL DEFAULT 0,
    mutation_score  NUMERIC(5,2),                  -- killed / (killed + survived) * 100
    duration_ms     INTEGER,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mutation_runs_session ON mutation_runs(session_id);
CREATE INDEX idx_mutation_runs_repo ON mutation_runs(repository_id);

-- Individual mutants (aligned with Stryker schema fields)
CREATE TABLE mutants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mutation_run_id UUID NOT NULL REFERENCES mutation_runs(id) ON DELETE CASCADE,
    source_file_id  UUID REFERENCES source_files(id),
    mutator_name    TEXT NOT NULL,                 -- Stryker: mutatorName (e.g. "BooleanSubstitution", "ArithmeticOperator")
    description     TEXT,                          -- human-readable description
    original_code   TEXT,                          -- original source fragment
    replacement     TEXT,                          -- Stryker: replacement
    file_path       TEXT NOT NULL,
    start_line      INTEGER NOT NULL,              -- Stryker: location.start.line
    start_column    INTEGER,                       -- Stryker: location.start.column
    end_line        INTEGER NOT NULL,              -- Stryker: location.end.line
    end_column      INTEGER,                       -- Stryker: location.end.column
    status          TEXT NOT NULL CHECK (status IN (
                        'killed', 'survived', 'no_coverage', 'compile_error',
                        'runtime_error', 'timeout', 'ignored', 'pending'
                    )),                            -- Stryker: status enum
    status_reason   TEXT,                          -- Stryker: statusReason / description
    test_duration_ms INTEGER,                      -- Stryker: testsCompleted net time
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mutants_run ON mutants(mutation_run_id);
CREATE INDEX idx_mutants_status ON mutants(status);
CREATE INDEX idx_mutants_file ON mutants(source_file_id);

-- Which tests killed or covered each mutant
CREATE TABLE mutant_test_links (
    mutant_id       UUID NOT NULL REFERENCES mutants(id) ON DELETE CASCADE,
    test_result_id  UUID NOT NULL REFERENCES test_results(id) ON DELETE CASCADE,
    relationship    TEXT NOT NULL CHECK (relationship IN ('killed_by', 'covered_by')),
    PRIMARY KEY (mutant_id, test_result_id, relationship)
);

CREATE INDEX idx_mutant_tests_mutant ON mutant_test_links(mutant_id);
CREATE INDEX idx_mutant_tests_result ON mutant_test_links(test_result_id);
```

## Test Maintenance & Repair

```sql
-- ============================================================
-- TEST REPAIR & MAINTENANCE
-- ============================================================

CREATE TABLE test_repairs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    generated_test_id UUID NOT NULL REFERENCES generated_tests(id) ON DELETE CASCADE,
    session_id      UUID NOT NULL REFERENCES generation_sessions(id) ON DELETE CASCADE,
    breaking_commit_id UUID REFERENCES commits(id),
    original_code   TEXT NOT NULL,                 -- test code before repair
    repaired_code   TEXT NOT NULL,                 -- test code after repair
    repair_type     TEXT NOT NULL CHECK (repair_type IN (
                        'assertion_update', 'mock_update', 'import_fix',
                        'api_change', 'signature_change', 'deleted_dependency'
                    )),
    change_summary  TEXT NOT NULL,                 -- LLM explanation of what changed and why
    is_regression   BOOLEAN,                       -- true if the failing test reveals a real bug
    confidence      NUMERIC(3,2),                  -- 0.00 to 1.00 confidence in the repair
    status          TEXT NOT NULL DEFAULT 'proposed'
                    CHECK (status IN ('proposed', 'accepted', 'rejected', 'applied')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_test_repairs_test ON test_repairs(generated_test_id);
CREATE INDEX idx_test_repairs_session ON test_repairs(session_id);
CREATE INDEX idx_test_repairs_status ON test_repairs(status);
```

## Security Test Mappings

```sql
-- ============================================================
-- SECURITY TEST MAPPINGS (OWASP WSTG)
-- ============================================================

CREATE TABLE security_test_mappings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    generated_test_id UUID NOT NULL REFERENCES generated_tests(id) ON DELETE CASCADE,
    owasp_test_id   TEXT NOT NULL,                 -- e.g. "WSTG-AUTHN-01", "WSTG-INPVAL-05"
    owasp_category  TEXT NOT NULL,                 -- e.g. "Authentication", "Input Validation"
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (generated_test_id, owasp_test_id)
);

CREATE INDEX idx_security_mappings_owasp ON security_test_mappings(owasp_test_id);
```

## CI/CD Integration

```sql
-- ============================================================
-- CI/CD PIPELINES & THRESHOLD GATES
-- ============================================================

CREATE TABLE pipeline_configurations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    ci_provider     TEXT NOT NULL CHECK (ci_provider IN ('github_actions', 'gitlab_ci', 'jenkins', 'circleci', 'generic')),
    min_line_coverage   NUMERIC(5,2),              -- threshold gate: minimum line coverage %
    min_branch_coverage NUMERIC(5,2),              -- threshold gate: minimum branch coverage %
    min_mutation_score  NUMERIC(5,2),              -- threshold gate: minimum mutation score %
    auto_generate   BOOLEAN NOT NULL DEFAULT false, -- auto-generate tests on PR
    auto_repair     BOOLEAN NOT NULL DEFAULT false, -- auto-repair broken tests on PR
    target_languages TEXT[] NOT NULL DEFAULT '{}',  -- languages to target
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, ci_provider)
);

CREATE TABLE pipeline_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_config_id UUID NOT NULL REFERENCES pipeline_configurations(id) ON DELETE CASCADE,
    commit_id       UUID NOT NULL REFERENCES commits(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES generation_sessions(id),
    test_run_id     UUID REFERENCES test_runs(id),
    external_run_id TEXT,                          -- CI provider's run ID
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'passed', 'failed', 'error')),
    gate_result     TEXT CHECK (gate_result IN ('passed', 'failed', 'skipped')),
    gate_details    TEXT,                          -- explanation of why gate passed/failed
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pipeline_runs_config ON pipeline_runs(pipeline_config_id);
CREATE INDEX idx_pipeline_runs_commit ON pipeline_runs(commit_id);
CREATE INDEX idx_pipeline_runs_status ON pipeline_runs(status);
```

## Coverage Trend Tracking

```sql
-- ============================================================
-- COVERAGE SNAPSHOTS (for trend tracking across commits)
-- ============================================================

CREATE TABLE coverage_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    commit_id       UUID NOT NULL REFERENCES commits(id) ON DELETE CASCADE,
    branch_name     TEXT NOT NULL DEFAULT 'main',
    line_coverage_pct   NUMERIC(5,2),
    branch_coverage_pct NUMERIC(5,2),
    mutation_score_pct  NUMERIC(5,2),
    total_tests     INTEGER NOT NULL DEFAULT 0,
    total_functions INTEGER NOT NULL DEFAULT 0,
    covered_functions INTEGER NOT NULL DEFAULT 0,
    snapshot_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_snapshots_repo ON coverage_snapshots(repository_id, snapshot_at DESC);
CREATE INDEX idx_coverage_snapshots_branch ON coverage_snapshots(repository_id, branch_name, snapshot_at DESC);
```

## API Keys & Webhooks

```sql
-- ============================================================
-- API KEYS & WEBHOOKS
-- ============================================================

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    created_by      UUID NOT NULL REFERENCES users(id),
    name            TEXT NOT NULL,
    key_hash        TEXT NOT NULL,                 -- bcrypt hash of the API key
    key_prefix      TEXT NOT NULL,                 -- first 8 chars for identification
    scopes          TEXT[] NOT NULL DEFAULT '{read}',
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_org ON api_keys(organization_id);
CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,               -- e.g. '{generation.completed, test_run.failed}'
    secret_hash     TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhooks_org ON webhooks(organization_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organizations, users, organization_members |
| Repository & Source Code | 4 | repositories, commits, source_files, source_functions |
| Specifications | 1 | specifications (OpenAPI, Gherkin, natural language) |
| Test Generation | 3 | generation_sessions, generated_tests, generated_mocks |
| Test Execution | 2 | test_runs, test_results |
| Coverage Analysis | 4 | coverage_reports, coverage_files, coverage_lines, coverage_gaps |
| Mutation Testing | 3 | mutation_runs, mutants, mutant_test_links |
| Test Maintenance | 1 | test_repairs |
| Security Mappings | 1 | security_test_mappings |
| CI/CD Integration | 2 | pipeline_configurations, pipeline_runs |
| Coverage Trends | 1 | coverage_snapshots |
| API & Webhooks | 2 | api_keys, webhooks |
| **Total** | **27** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation without coordination, safe for multi-tenant SaaS and future sharding on `organization_id`.

2. **Commit-anchored source files** — every `source_file` is tied to a specific `commit`, allowing the system to reason about exactly which version of the code was analyzed. This avoids ambiguity when coverage changes between commits.

3. **Unified coverage model** — rather than storing raw LCOV/JaCoCo/coverage.py files, coverage data is normalized into `coverage_reports` → `coverage_files` → `coverage_lines`. This enables cross-language, cross-format queries without parsing at read time.

4. **Stryker-aligned mutant schema** — the `mutants` table fields (mutatorName, status enum, location with line/column, killedBy/coveredBy via junction table) directly mirror the Stryker mutation-testing-report-schema, ensuring interoperability with the Stryker dashboard and ecosystem.

5. **Separate generation sessions from test runs** — a generation session produces test code; a test run executes tests. These are distinct lifecycle phases. A single session may produce tests that are run many times, and a test run may include both generated and hand-written tests.

6. **Coverage gaps as first-class entities** — the `coverage_gaps` table stores LLM-generated explanations of why specific functions are hard to cover, making gap analysis queryable and trackable over time rather than ephemeral.

7. **Test repair with regression detection** — the `test_repairs` table captures the full before/after of each repair with a `is_regression` boolean, enabling the system to surface genuine bugs discovered through test maintenance rather than silently fixing stale assertions.

8. **Pipeline threshold gates** — coverage and mutation score thresholds are stored per-repository per-CI-provider, making it possible to enforce different standards for different repositories and track gate pass/fail history.

9. **No JSONB for core entities** — every field is explicitly typed and indexed. This trades flexibility for query performance and schema enforcement, making it impossible to store malformed data in core fields.

10. **Row-level security ready** — all tenant-scoped tables have `organization_id` (directly or via parent FK), enabling PostgreSQL RLS policies for tenant isolation without application-level filtering.
