# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Intelligent Test Generator · Created: 2026-05-11

## Philosophy

This model treats the system as a sequence of immutable events rather than mutable rows. Every meaningful action -- a generation session starting, a test being generated, a coverage report being ingested, a mutant being killed -- is recorded as an append-only event in a central event store. The current state of any entity is derived by replaying its events, and read-optimized materialized views serve queries without touching the event store.

This architecture is a natural fit for a test generation platform where temporal queries are essential: "What was the coverage of this repository on March 15?", "How has the mutation score trended over the last 20 commits?", "What sequence of generation-validation-repair iterations produced the final test suite?" Every question about history is answerable because the event store *is* the history. The full audit trail also satisfies compliance requirements (NIST SP 800-218 PW.8) without adding separate audit tables.

The CQRS (Command Query Responsibility Segregation) pattern separates write operations (appending events) from read operations (querying materialized views), allowing each side to be optimized independently. Write throughput scales through append-only inserts; read performance scales through purpose-built projections. This is the architecture behind Meta's production-scale LLM+mutation testing deployment (FSE 2025), which processes millions of mutation results as events.

**Best for:** Teams building a platform where temporal analysis, complete audit trails, and high-throughput event ingestion (bulk coverage/mutation data from CI pipelines) are primary requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail of every system action -- no data is ever lost
- (+) Temporal queries are trivial: replay events up to any point in time
- (+) Write throughput scales well: append-only inserts with no update contention
- (+) Event replay enables rebuilding views, fixing bugs in projections, and adding new analytics after the fact
- (+) Natural fit for CI/CD webhook-driven architectures (events arrive as webhooks)
- (-) Higher storage cost: events are never deleted, and materialized views duplicate data
- (-) Eventual consistency between event store and read models requires careful design
- (-) More complex application code: developers must think in terms of events and projections, not CRUD
- (-) Schema evolution of events requires versioning and upcasting strategies
- (-) Debugging current state requires understanding the event chain, not just reading a row

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC/IEEE 29119 Part 3 | Test documentation lifecycle (plan → execute → report) maps naturally to event sequences |
| NIST SP 800-218 (PW.8) | Immutable event store provides the audit trail required for compliance; no separate audit mechanism needed |
| Stryker Mutation Testing Report Schema | Mutation results ingested as events with Stryker-aligned fields (mutatorName, status, location) |
| JUnit XML Format | Test results parsed into structured events with JUnit-compatible fields |
| ISO/IEC 25010 (SQuaRE) | Quality metrics computed as projections over event streams; mutation score history is native |
| OpenAPI 3.1 | Spec ingestion events capture parsed OpenAPI schemas for spec-to-test generation workflows |
| OWASP WSTG | Security test mapping events link generated tests to OWASP identifiers |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- CORE EVENT STORE
-- The single source of truth. All state is derived from events.
-- ============================================================

-- Aggregate registry: tracks all aggregates and their current version
-- for optimistic concurrency control
CREATE TABLE aggregates (
    aggregate_id    UUID PRIMARY KEY,
    aggregate_type  TEXT NOT NULL CHECK (aggregate_type IN (
                        'organization', 'repository', 'generation_session',
                        'test_suite', 'coverage_analysis', 'mutation_analysis',
                        'pipeline'
                    )),
    current_version BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_aggregates_type ON aggregates(aggregate_type);

-- The event store: append-only, immutable
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id    UUID NOT NULL REFERENCES aggregates(aggregate_id),
    aggregate_type  TEXT NOT NULL,
    event_type      TEXT NOT NULL,                 -- e.g. 'TestGenerated', 'CoverageReportIngested'
    event_version   BIGINT NOT NULL,               -- monotonically increasing per aggregate
    payload         JSONB NOT NULL,                -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',   -- correlation IDs, user context, timestamps
    -- metadata example:
    -- {
    --   "user_id": "uuid",
    --   "correlation_id": "uuid",
    --   "causation_id": "uuid",     -- which event/command caused this
    --   "source": "ci",
    --   "ip_address": "10.0.0.1"
    -- }
    schema_version  INTEGER NOT NULL DEFAULT 1,    -- for event upcasting
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)           -- optimistic concurrency guard
);

-- Primary query path: replay events for an aggregate
CREATE INDEX idx_events_aggregate ON events(aggregate_id, event_version);

-- Secondary: query by event type across all aggregates
CREATE INDEX idx_events_type ON events(event_type, created_at);

-- Partition by month for storage management
-- (In production, use declarative partitioning on created_at)

-- Global event ordering for projections
CREATE SEQUENCE global_event_position;

CREATE TABLE event_positions (
    event_id        UUID PRIMARY KEY REFERENCES events(event_id),
    global_position BIGINT NOT NULL DEFAULT nextval('global_event_position'),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_event_positions_global ON event_positions(global_position);
```

## Event Type Catalog

```sql
-- ============================================================
-- EVENT TYPE CATALOG
-- Documents all event types and their payload schemas.
-- This is metadata, not operational data.
-- ============================================================

CREATE TABLE event_type_catalog (
    event_type      TEXT PRIMARY KEY,
    aggregate_type  TEXT NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,                -- JSON Schema for the payload
    current_version INTEGER NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payload structures:
--
-- 'OrganizationCreated'     → { name, slug, plan }
-- 'RepositoryRegistered'    → { org_id, name, url, vcs_provider, default_branch }
-- 'CommitAnalyzed'          → { repo_id, sha, message, author, file_count, language_breakdown }
-- 'SourceFileIndexed'       → { repo_id, commit_sha, file_path, language, line_count, functions[] }
-- 'SpecificationIngested'   → { repo_id, spec_type, file_path, title, parsed_version }
--
-- 'GenerationSessionStarted'  → { repo_id, commit_sha, strategy, trigger_source, llm_model }
-- 'GenerationSessionCompleted'→ { session_id, tests_generated, duration_ms, tokens_used }
-- 'GenerationSessionFailed'   → { session_id, error_message }
-- 'TestGenerated'            → { session_id, function_id, framework, test_name, test_code, description, test_type }
-- 'MockGenerated'            → { test_id, mock_target, mock_type, mock_code }
--
-- 'TestRunStarted'           → { repo_id, commit_sha, runner, run_source }
-- 'TestCasePassed'           → { run_id, test_name, class_name, duration_ms }
-- 'TestCaseFailed'           → { run_id, test_name, class_name, failure_message, failure_type, stack_trace }
-- 'TestRunCompleted'         → { run_id, total, passed, failed, errors, skipped, duration_ms }
--
-- 'CoverageReportIngested'   → { run_id, repo_id, commit_sha, source_format, totals }
-- 'FileCoverageRecorded'     → { report_id, file_path, lines_total, lines_covered, branches_total, branches_covered }
-- 'CoverageGapIdentified'    → { report_id, function_id, gap_type, severity, explanation }
--
-- 'MutationRunStarted'       → { session_id, repo_id, commit_sha, tool }
-- 'MutantCreated'            → { run_id, file_path, mutator_name, location, original_code, replacement }
-- 'MutantKilled'             → { mutant_id, killed_by_test_id, duration_ms }
-- 'MutantSurvived'           → { mutant_id, covered_by_test_ids }
-- 'MutationRunCompleted'     → { run_id, totals, mutation_score }
--
-- 'TestRepairProposed'       → { test_id, breaking_commit_sha, repair_type, original_code, repaired_code, is_regression }
-- 'TestRepairAccepted'       → { repair_id }
-- 'TestRepairRejected'       → { repair_id, reason }
--
-- 'PipelineGateEvaluated'    → { repo_id, commit_sha, ci_provider, thresholds, actuals, gate_result }
-- 'SecurityTestMapped'       → { test_id, owasp_test_id, owasp_category }
```

## Materialized Views (Read Side)

```sql
-- ============================================================
-- READ MODELS (Materialized Projections)
-- These are rebuilt from events. They can be dropped and
-- reconstructed at any time.
-- ============================================================

-- Projection: Current state of each organization
CREATE TABLE v_organizations (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL,
    member_count    INTEGER NOT NULL DEFAULT 0,
    repo_count      INTEGER NOT NULL DEFAULT 0,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Current state of each repository
CREATE TABLE v_repositories (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            TEXT NOT NULL,
    url             TEXT,
    vcs_provider    TEXT NOT NULL,
    default_branch  TEXT NOT NULL,
    primary_language TEXT,
    latest_commit_sha TEXT,
    latest_line_coverage NUMERIC(5,2),
    latest_branch_coverage NUMERIC(5,2),
    latest_mutation_score NUMERIC(5,2),
    total_tests     INTEGER NOT NULL DEFAULT 0,
    total_functions INTEGER NOT NULL DEFAULT 0,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_repos_org ON v_repositories(organization_id);

-- Projection: Latest generation session state
CREATE TABLE v_generation_sessions (
    id              UUID PRIMARY KEY,
    repository_id   UUID NOT NULL,
    commit_sha      TEXT NOT NULL,
    strategy        TEXT NOT NULL,
    trigger_source  TEXT NOT NULL,
    status          TEXT NOT NULL,
    llm_model       TEXT,
    tests_generated INTEGER NOT NULL DEFAULT 0,
    tokens_used     INTEGER NOT NULL DEFAULT 0,
    duration_ms     INTEGER,
    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_sessions_repo ON v_generation_sessions(repository_id);
CREATE INDEX idx_v_sessions_status ON v_generation_sessions(status);

-- Projection: Current state of each generated test
CREATE TABLE v_generated_tests (
    id              UUID PRIMARY KEY,
    session_id      UUID NOT NULL,
    repository_id   UUID NOT NULL,
    function_name   TEXT,
    function_qualified_name TEXT,
    test_framework  TEXT NOT NULL,
    test_file_path  TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    test_class_name TEXT,
    test_code       TEXT NOT NULL,
    description     TEXT,
    test_type       TEXT NOT NULL,
    status          TEXT NOT NULL,                  -- generated, validated, accepted, rejected, stale, repaired
    last_run_result TEXT,                           -- passed, failed, error, skipped
    repair_count    INTEGER NOT NULL DEFAULT 0,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_tests_session ON v_generated_tests(session_id);
CREATE INDEX idx_v_tests_repo ON v_generated_tests(repository_id);
CREATE INDEX idx_v_tests_status ON v_generated_tests(status);

-- Projection: Coverage timeline (one row per commit per repo)
-- This is the key view for coverage trend dashboards
CREATE TABLE v_coverage_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL,
    commit_sha      TEXT NOT NULL,
    branch_name     TEXT NOT NULL DEFAULT 'main',
    line_coverage_pct NUMERIC(5,2),
    branch_coverage_pct NUMERIC(5,2),
    mutation_score_pct NUMERIC(5,2),
    total_tests     INTEGER NOT NULL DEFAULT 0,
    total_functions INTEGER NOT NULL DEFAULT 0,
    covered_functions INTEGER NOT NULL DEFAULT 0,
    recorded_at     TIMESTAMPTZ NOT NULL,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, commit_sha)
);

CREATE INDEX idx_v_coverage_timeline ON v_coverage_timeline(repository_id, recorded_at DESC);
CREATE INDEX idx_v_coverage_timeline_branch ON v_coverage_timeline(repository_id, branch_name, recorded_at DESC);

-- Projection: Mutation analysis summary per run
CREATE TABLE v_mutation_summaries (
    id              UUID PRIMARY KEY,
    session_id      UUID NOT NULL,
    repository_id   UUID NOT NULL,
    commit_sha      TEXT NOT NULL,
    tool            TEXT NOT NULL,
    status          TEXT NOT NULL,
    total_mutants   INTEGER NOT NULL DEFAULT 0,
    killed          INTEGER NOT NULL DEFAULT 0,
    survived        INTEGER NOT NULL DEFAULT 0,
    no_coverage     INTEGER NOT NULL DEFAULT 0,
    mutation_score  NUMERIC(5,2),
    duration_ms     INTEGER,
    completed_at    TIMESTAMPTZ,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_mutation_repo ON v_mutation_summaries(repository_id);

-- Projection: Surviving mutants (action items for LLM feedback loop)
CREATE TABLE v_surviving_mutants (
    id              UUID PRIMARY KEY,
    mutation_run_id UUID NOT NULL,
    repository_id   UUID NOT NULL,
    file_path       TEXT NOT NULL,
    mutator_name    TEXT NOT NULL,
    original_code   TEXT,
    replacement     TEXT,
    start_line      INTEGER NOT NULL,
    end_line        INTEGER NOT NULL,
    covered_by_test_ids UUID[],
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_surviving_repo ON v_surviving_mutants(repository_id);
CREATE INDEX idx_v_surviving_run ON v_surviving_mutants(mutation_run_id);

-- Projection: Coverage gaps (action items for generation)
CREATE TABLE v_coverage_gaps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL,
    commit_sha      TEXT NOT NULL,
    function_name   TEXT NOT NULL,
    function_qualified_name TEXT NOT NULL,
    file_path       TEXT NOT NULL,
    gap_type        TEXT NOT NULL,
    severity        TEXT NOT NULL,
    explanation     TEXT NOT NULL,
    suggested_approach TEXT,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    identified_at   TIMESTAMPTZ NOT NULL,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_gaps_repo ON v_coverage_gaps(repository_id);
CREATE INDEX idx_v_gaps_severity ON v_coverage_gaps(severity);

-- Projection: Test repair history
CREATE TABLE v_test_repairs (
    id              UUID PRIMARY KEY,
    generated_test_id UUID NOT NULL,
    session_id      UUID NOT NULL,
    breaking_commit_sha TEXT,
    repair_type     TEXT NOT NULL,
    change_summary  TEXT NOT NULL,
    is_regression   BOOLEAN,
    confidence      NUMERIC(3,2),
    status          TEXT NOT NULL,
    proposed_at     TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_repairs_test ON v_test_repairs(generated_test_id);
CREATE INDEX idx_v_repairs_status ON v_test_repairs(status);

-- Projection: Pipeline gate results
CREATE TABLE v_pipeline_gates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL,
    commit_sha      TEXT NOT NULL,
    ci_provider     TEXT NOT NULL,
    line_coverage_threshold NUMERIC(5,2),
    line_coverage_actual NUMERIC(5,2),
    branch_coverage_threshold NUMERIC(5,2),
    branch_coverage_actual NUMERIC(5,2),
    mutation_score_threshold NUMERIC(5,2),
    mutation_score_actual NUMERIC(5,2),
    gate_result     TEXT NOT NULL,                  -- passed, failed
    evaluated_at    TIMESTAMPTZ NOT NULL,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_gates_repo ON v_pipeline_gates(repository_id, evaluated_at DESC);
```

## Projection Tracking

```sql
-- ============================================================
-- PROJECTION CHECKPOINTS
-- Tracks which events each projection has processed.
-- Used to resume projections after restart.
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,              -- e.g. 'v_repositories', 'v_coverage_timeline'
    last_global_position BIGINT NOT NULL DEFAULT 0,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Identity & Auth (Non-Event-Sourced)

```sql
-- ============================================================
-- IDENTITY (Non-event-sourced; these are operational lookup tables)
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    auth_provider   TEXT NOT NULL,
    auth_provider_id TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
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

CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
```

---

## Example: Appending Events

```sql
-- A generation session lifecycle as events:

-- 1. Session starts
INSERT INTO events (aggregate_id, aggregate_type, event_type, event_version, payload, metadata)
VALUES (
    '550e8400-e29b-41d4-a716-446655440000',
    'generation_session',
    'GenerationSessionStarted',
    1,
    '{
        "repo_id": "a1b2c3d4-...",
        "commit_sha": "abc123def",
        "strategy": "coverage_gap",
        "trigger_source": "ci",
        "llm_model": "claude-opus-4-20250514"
    }'::jsonb,
    '{"user_id": "u-123", "correlation_id": "corr-456", "source": "github_actions"}'::jsonb
);

-- 2. Tests are generated (multiple events)
INSERT INTO events (aggregate_id, aggregate_type, event_type, event_version, payload, metadata)
VALUES (
    '550e8400-e29b-41d4-a716-446655440000',
    'generation_session',
    'TestGenerated',
    2,
    '{
        "test_id": "t-789",
        "function_qualified_name": "auth.service.validate_token",
        "framework": "pytest",
        "test_name": "test_validate_token_expired",
        "test_code": "def test_validate_token_expired():\n    ...",
        "description": "Verifies that expired JWT tokens are rejected with 401",
        "test_type": "unit"
    }'::jsonb,
    '{"user_id": "u-123", "correlation_id": "corr-456"}'::jsonb
);

-- 3. Session completes
INSERT INTO events (aggregate_id, aggregate_type, event_type, event_version, payload, metadata)
VALUES (
    '550e8400-e29b-41d4-a716-446655440000',
    'generation_session',
    'GenerationSessionCompleted',
    5,
    '{
        "tests_generated": 3,
        "duration_ms": 45000,
        "tokens_used": 28500
    }'::jsonb,
    '{"user_id": "u-123", "correlation_id": "corr-456"}'::jsonb
);
```

## Example: Temporal Query (Coverage at a Point in Time)

```sql
-- "What was the coverage of repo X on March 15, 2026?"
-- With event sourcing, this is a projection query:

SELECT
    commit_sha,
    line_coverage_pct,
    branch_coverage_pct,
    mutation_score_pct,
    total_tests,
    recorded_at
FROM v_coverage_timeline
WHERE repository_id = 'a1b2c3d4-...'
  AND recorded_at <= '2026-03-15 23:59:59+00'
ORDER BY recorded_at DESC
LIMIT 1;

-- Or replay from the event store directly:
SELECT
    payload->>'commit_sha' AS commit_sha,
    (payload->'totals'->>'line_coverage_pct')::numeric AS line_coverage,
    created_at AS recorded_at
FROM events
WHERE aggregate_type = 'coverage_analysis'
  AND event_type = 'CoverageReportIngested'
  AND payload->>'repo_id' = 'a1b2c3d4-...'
  AND created_at <= '2026-03-15 23:59:59+00'
ORDER BY created_at DESC
LIMIT 1;
```

## Example: Reconstructing Session State

```sql
-- Replay all events for a generation session to see its full lifecycle:
SELECT
    event_type,
    event_version,
    payload,
    created_at
FROM events
WHERE aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY event_version;

-- Returns:
-- 1: GenerationSessionStarted  → initial config
-- 2: TestGenerated             → first test
-- 3: TestGenerated             → second test
-- 4: TestGenerated             → third test
-- 5: GenerationSessionCompleted → summary
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write Side) | 4 | aggregates, events, event_positions, event_type_catalog |
| Projection Tracking | 1 | projection_checkpoints |
| Read Models (Projections) | 10 | v_organizations, v_repositories, v_generation_sessions, v_generated_tests, v_coverage_timeline, v_mutation_summaries, v_surviving_mutants, v_coverage_gaps, v_test_repairs, v_pipeline_gates |
| Identity (Non-ES) | 2 | users, api_keys |
| **Total** | **17** | Write side: 5 tables; Read side: 12 tables |

---

## Key Design Decisions

1. **Single event store table with JSONB payloads** — all events live in one table partitioned by time. The `event_type` field discriminates, and the `payload` JSONB column carries event-specific data. This avoids dozens of event-type-specific tables while keeping the write path simple (one INSERT per event).

2. **Aggregate-scoped event versioning with optimistic concurrency** — the `UNIQUE (aggregate_id, event_version)` constraint prevents concurrent writes to the same aggregate from creating inconsistencies. The application loads the current version, increments it, and the INSERT fails if another process wrote first.

3. **Global event position for projections** — the `event_positions` table with a global sequence number enables projections to process events in total order across all aggregates, ensuring consistent read models.

4. **Schema versioning on events** — the `schema_version` field on each event allows upcasting old event formats when projections are rebuilt. When the payload schema for `TestGenerated` changes, old events retain `schema_version: 1` and the projection code handles both versions.

5. **Read models are disposable** — every `v_*` table can be dropped and rebuilt from the event store. This makes it safe to add new projections (e.g., a new dashboard) or fix bugs in existing projections without data loss.

6. **Coverage timeline as a first-class projection** — the `v_coverage_timeline` table is the natural result of event sourcing: each `CoverageReportIngested` event adds a row, building a complete historical record without explicit snapshot logic.

7. **Surviving mutants as a projection** — the `v_surviving_mutants` table collects mutants with status `survived` from `MutantSurvived` events, providing the LLM feedback loop with a ready-to-query action list without scanning the full event store.

8. **Non-event-sourced identity tables** — users and API keys are operational lookup tables that don't benefit from event sourcing. They use standard CRUD to avoid unnecessary complexity.

9. **Metadata envelope on every event** — the `metadata` JSONB field captures correlation_id (for tracing a request across events), causation_id (which event caused this one), user_id, and source. This enables end-to-end request tracing and audit attribution without separate audit tables.

10. **Partition-ready event store** — the `events` table is designed for declarative range partitioning on `created_at` (monthly partitions). Old partitions can be archived to cold storage without affecting the active write path, managing storage costs as the event store grows.
