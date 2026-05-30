# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Scientific Peer Review Manager · Generated: 2026-05-25

## Summary

An event-sourced architecture with Command Query Responsibility Segregation (CQRS) where every state change in the manuscript lifecycle is captured as an immutable event in an append-only event store. The write side records domain events (e.g. `ManuscriptSubmitted`, `ReviewerInvited`, `DecisionMade`), while one or more read-side projections materialize optimized views for querying. This approach treats the audit trail as the primary source of truth — a natural fit for peer review, where the complete history of editorial decisions, reviewer actions, and manuscript states is essential for COPE compliance, dispute resolution, and institutional auditing.

---

## Key Entities and Relationships

### Domain Aggregates

The system is organized around **aggregates** — consistency boundaries that process commands and emit events:

```
ManuscriptAggregate
  ├── ManuscriptSubmitted
  ├── ManuscriptAssignedToEditor
  ├── ReviewRoundOpened
  ├── ReviewerInvited
  ├── ReviewerResponded (accepted/declined)
  ├── ReviewSubmitted
  ├── RevisionRequested
  ├── RevisionSubmitted
  ├── EditorialDecisionMade
  ├── ManuscriptAccepted
  ├── ManuscriptRejected
  ├── ManuscriptWithdrawn
  └── DOIAssigned

UserAggregate
  ├── UserRegistered
  ├── UserProfileUpdated
  ├── ORCIDLinked
  ├── RoleAssigned
  ├── ExpertiseUpdated
  └── ConflictDeclared

JournalAggregate
  ├── JournalCreated
  ├── ReviewFormConfigured
  ├── ReviewPolicyUpdated
  └── SubjectTaxonomyUpdated

IntegrityCheckAggregate
  ├── PlagiarismCheckRequested
  ├── PlagiarismCheckCompleted
  ├── ImageIntegrityCheckRequested
  └── ImageIntegrityCheckCompleted
```

### Event Store Schema

```sql
-- ============================================================
-- WRITE SIDE: EVENT STORE
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,
        -- Values: 'Manuscript', 'User', 'Journal', 'IntegrityCheck'
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
        -- Values: 'ManuscriptSubmitted', 'ReviewerInvited',
        --         'ReviewSubmitted', 'EditorialDecisionMade', etc.
    event_version   INTEGER NOT NULL,     -- per-aggregate sequence number
    event_data      JSONB NOT NULL,       -- full event payload
    metadata        JSONB,                -- actor_id, ip_address, correlation_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(aggregate_type, aggregate_id, event_version)
);

-- Optimized for aggregate replay
CREATE INDEX idx_event_store_aggregate
    ON event_store(aggregate_type, aggregate_id, event_version);

-- Optimized for global event ordering (projections)
CREATE INDEX idx_event_store_created
    ON event_store(created_at);

-- Optimized for event type filtering
CREATE INDEX idx_event_store_type
    ON event_store(event_type);

-- ============================================================
-- SNAPSHOTS (optional optimization)
-- ============================================================

CREATE TABLE aggregate_snapshots (
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,
    state_data      JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);
```

### Example Event Payloads

```json
// ManuscriptSubmitted
{
  "event_type": "ManuscriptSubmitted",
  "aggregate_id": "a1b2c3d4-...",
  "event_data": {
    "journal_id": "j1k2l3m4-...",
    "manuscript_number": "SPRM-2026-00142",
    "title": "Machine Learning Approaches to Protein Folding",
    "abstract": "We present a novel...",
    "article_type": "original-research",
    "submitting_author": {
      "user_id": "u5v6w7x8-...",
      "orcid_id": "0000-0002-1234-5678",
      "given_name": "Jane",
      "family_name": "Smith"
    },
    "co_authors": [
      {"given_name": "John", "family_name": "Doe", "orcid_id": "0000-0003-9876-5432"}
    ],
    "subject_areas": ["machine-learning", "structural-biology"],
    "files": [
      {"file_id": "f9g0h1i2-...", "file_type": "manuscript", "file_name": "main.pdf"}
    ]
  },
  "metadata": {
    "actor_id": "u5v6w7x8-...",
    "ip_address": "203.0.113.42",
    "timestamp": "2026-05-25T10:30:00Z"
  }
}

// ReviewSubmitted
{
  "event_type": "ReviewSubmitted",
  "aggregate_id": "a1b2c3d4-...",
  "event_data": {
    "round_number": 1,
    "reviewer_id": "r3s4t5u6-...",
    "recommendation": "major_revision",
    "comments_for_author": "The methodology section needs...",
    "comments_for_editor": "I have concerns about...",
    "form_responses": [
      {"element_id": "e1f2g3h4-...", "label": "Novelty", "rating": 3},
      {"element_id": "e5f6g7h8-...", "label": "Methodology", "rating": 2}
    ],
    "quality_flags": {
      "word_count": 1247,
      "avg_sentence_length": 18.3,
      "boilerplate_score": 0.12
    }
  }
}

// EditorialDecisionMade
{
  "event_type": "EditorialDecisionMade",
  "aggregate_id": "a1b2c3d4-...",
  "event_data": {
    "round_number": 1,
    "editor_id": "ed1ed2ed3-...",
    "decision": "major_revision",
    "rationale": "Both reviewers agree that...",
    "decision_letter": {
      "subject": "Decision on SPRM-2026-00142",
      "body": "Dear Dr. Smith,\n\nThank you for submitting..."
    }
  }
}
```

### Read-Side Projections

```sql
-- ============================================================
-- READ SIDE: PROJECTIONS (materialized from events)
-- ============================================================

-- Current manuscript state (rebuilt from events)
CREATE TABLE manuscripts_view (
    manuscript_id       UUID PRIMARY KEY,
    journal_id          UUID NOT NULL,
    manuscript_number   VARCHAR(50) NOT NULL UNIQUE,
    title               TEXT NOT NULL,
    abstract            TEXT,
    status              VARCHAR(50) NOT NULL,
    current_round       SMALLINT NOT NULL DEFAULT 0,
    submitting_user_id  UUID NOT NULL,
    handling_editor_id  UUID,
    submitted_at        TIMESTAMPTZ,
    last_decision       VARCHAR(30),
    last_decision_at    TIMESTAMPTZ,
    doi                 VARCHAR(255),
    updated_at          TIMESTAMPTZ NOT NULL
);

-- Denormalized reviewer dashboard view
CREATE TABLE reviewer_dashboard_view (
    assignment_id       UUID PRIMARY KEY,
    reviewer_id         UUID NOT NULL,
    manuscript_id       UUID NOT NULL,
    manuscript_number   VARCHAR(50) NOT NULL,
    manuscript_title    TEXT NOT NULL,
    journal_title       VARCHAR(500),
    round_number        SMALLINT NOT NULL,
    status              VARCHAR(30) NOT NULL,
    invited_at          TIMESTAMPTZ,
    due_date            DATE,
    completed_at        TIMESTAMPTZ
);

-- Editor overview: manuscripts needing attention
CREATE TABLE editor_queue_view (
    manuscript_id       UUID NOT NULL,
    journal_id          UUID NOT NULL,
    manuscript_number   VARCHAR(50) NOT NULL,
    title               TEXT NOT NULL,
    status              VARCHAR(50) NOT NULL,
    handling_editor_id  UUID,
    days_since_submission INTEGER,
    reviews_completed   SMALLINT DEFAULT 0,
    reviews_pending     SMALLINT DEFAULT 0,
    needs_attention     BOOLEAN DEFAULT FALSE,
    priority_score      DECIMAL(5,2)
);

-- Reviewer performance metrics (aggregated from events)
CREATE TABLE reviewer_metrics_view (
    reviewer_id         UUID PRIMARY KEY,
    total_invitations   INTEGER DEFAULT 0,
    total_accepted      INTEGER DEFAULT 0,
    total_completed     INTEGER DEFAULT 0,
    total_declined      INTEGER DEFAULT 0,
    avg_days_to_respond DECIMAL(5,1),
    avg_days_to_complete DECIMAL(5,1),
    avg_quality_score   DECIMAL(3,1),
    last_review_at      TIMESTAMPTZ,
    active_assignments  SMALLINT DEFAULT 0
);

-- Manuscript timeline (ordered event projection)
CREATE TABLE manuscript_timeline_view (
    timeline_entry_id   BIGSERIAL PRIMARY KEY,
    manuscript_id       UUID NOT NULL,
    event_type          VARCHAR(100) NOT NULL,
    actor_name          VARCHAR(500),
    summary             TEXT NOT NULL,
    detail              JSONB,
    occurred_at         TIMESTAMPTZ NOT NULL
);

-- Full-text search projection
CREATE TABLE manuscript_search_view (
    manuscript_id       UUID PRIMARY KEY,
    manuscript_number   VARCHAR(50) NOT NULL,
    title               TEXT NOT NULL,
    abstract            TEXT,
    author_names        TEXT,   -- concatenated for search
    subject_areas       TEXT,   -- concatenated for search
    journal_title       VARCHAR(500),
    status              VARCHAR(50),
    search_vector       TSVECTOR GENERATED ALWAYS AS (
        setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(abstract, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(author_names, '')), 'C')
    ) STORED
);

CREATE INDEX idx_manuscript_search ON manuscript_search_view USING GIN(search_vector);
```

---

## Pros and Cons

### Pros

- **Complete audit trail by design.** Every state change is an immutable event. COPE-mandated audit trails, GDPR data subject access requests, and institutional dispute investigations can be served directly from the event store without reconstructing history from mutable table snapshots. This is the single strongest advantage for a peer review system.
- **Temporal queries are trivial.** "What was the state of manuscript SPRM-2026-00142 on March 15th?" is answered by replaying events up to that timestamp. This is essential for editorial disputes and appeals processes.
- **Natural workflow modeling.** The manuscript lifecycle (submitted -> with_editor -> under_review -> revision_requested -> revised -> accepted/rejected) maps directly to a sequence of domain events. Each event carries the full context of who did what and why.
- **Read-side optimization.** Each projection is purpose-built for a specific view (reviewer dashboard, editor queue, search). No need for complex joins across normalized tables — the projection rebuilder denormalizes data at write time.
- **AI feature integration.** AI-generated events (e.g. `ReviewerMatchScored`, `PlagiarismCheckCompleted`, `BoilerplateReviewDetected`) fit naturally into the event stream. AI features become first-class participants in the manuscript lifecycle rather than bolted-on side tables.
- **Schema evolution without migration.** New event types can be added without altering the event store schema. New projections can be built from historical events at any time. A new "reviewer workload analytics" view can be projected from existing `ReviewerInvited`, `ReviewerResponded`, and `ReviewSubmitted` events without any backfill migration.

### Cons

- **Operational complexity.** Running an event-sourced system requires infrastructure for event handlers, projection rebuilders, idempotent processors, and eventual consistency monitoring. This is significantly more complex than a traditional CRUD application.
- **Eventual consistency on read side.** Projections may lag behind the event store by milliseconds to seconds. An editor who submits a decision may not immediately see the updated manuscript status in their dashboard. This can be mitigated with synchronous projections for critical views, but adds complexity.
- **Event schema evolution is hard.** Once events are stored, their schema is immutable. If the `ManuscriptSubmitted` event needs a new required field, all consumers must handle both the old and new event shapes. Upcasting strategies and event versioning are necessary from day one.
- **Debugging is harder.** Bugs in projection logic can silently corrupt read-side views. Diagnosing "why does the editor queue show 3 pending reviews when there should be 2?" requires tracing through the event stream and the projection handler.
- **Storage growth.** The event store grows indefinitely. A large publisher processing 50,000 manuscripts/year with ~20 events per manuscript generates ~1M events/year. After 10 years, the event store contains 10M+ events. Snapshots mitigate replay performance but add their own maintenance burden.
- **Team skill requirements.** Event sourcing and CQRS are less widely understood than CRUD. Hiring developers and training editorial staff on a conceptually unfamiliar architecture increases onboarding cost.
- **Testing complexity.** Unit testing requires given-when-then event-based test fixtures. Integration testing requires verifying that projections correctly materialize from event sequences. This is a different testing paradigm from standard repository-based testing.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Event Store** | PostgreSQL with `event_store` table (see schema above); alternatively EventStoreDB for dedicated event streaming |
| **Message Bus** | Apache Kafka or NATS JetStream for event distribution to projection handlers |
| **Projection Engine** | Custom async workers consuming from Kafka/NATS; or Marten (.NET) / Axon (Java) frameworks |
| **Read Database** | PostgreSQL for relational projections; Redis for hot caches (editor queue, active assignments) |
| **Search Projection** | Elasticsearch or Meilisearch for full-text manuscript and reviewer search |
| **Application Framework** | Axon Framework (Java/Kotlin), Marten (C#/.NET), or custom TypeScript with pg + Kafka |
| **File Storage** | S3-compatible object store; file upload events reference storage paths |
| **API** | REST + WebSocket for real-time projection updates; OpenAPI 3.1 for REST surface |

---

## Migration and Scaling Considerations

- **Starting simple.** Begin with PostgreSQL as both event store and read database. Move to dedicated event infrastructure (EventStoreDB, Kafka) only when throughput demands it. A peer review system handling <100K manuscripts/year can comfortably use PostgreSQL for everything.
- **Event versioning strategy.** Adopt semantic event versioning from day one (e.g. `ManuscriptSubmitted.v1`, `ManuscriptSubmitted.v2`). Implement upcasters that transform old event versions to the latest shape during replay.
- **Projection rebuild pipeline.** Every projection must be rebuildable from scratch by replaying the entire event stream. This enables adding new projections at any time and recovering from projection corruption. Build this pipeline before going to production.
- **Snapshot cadence.** For the `ManuscriptAggregate`, take snapshots after every 50 events to keep replay times under 100ms. Most manuscripts will have 10-30 events total, so snapshots are primarily needed for long-running manuscripts with multiple revision rounds.
- **GDPR and event immutability.** GDPR right to erasure conflicts with event immutability. The recommended pattern is **crypto-shredding**: encrypt PII fields in events with a per-user key, and delete the key when erasure is requested. The events remain in the store but the PII becomes irrecoverable.
- **Multi-journal scaling.** Partition the event store by journal or use separate Kafka topics per journal. Projections can consume events from multiple journals to build cross-journal analytics views.
- **Migrating from CRUD.** If starting with Suggestion 1 (normalized relational) and later migrating to event sourcing, the migration path is: (1) add event emission alongside CRUD writes (dual-write), (2) build projections from the event stream, (3) verify projection consistency against the CRUD database, (4) switch reads to projections, (5) remove CRUD writes.
