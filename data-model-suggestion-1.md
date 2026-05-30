# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Scientific Peer Review Manager · Generated: 2026-05-25

## Summary

A fully normalized relational schema in PostgreSQL, modeled after the proven entity architecture of systems like Open Journal Systems (OJS) and Editorial Manager. Every concept — journals, manuscripts, users, roles, reviews, decisions, files — is represented as a dedicated table with strict foreign keys and referential integrity. This is the most conservative and well-understood approach, prioritizing data consistency, auditability, and standards compliance.

---

## Key Entities and Relationships

### Entity-Relationship Overview

```
Journal 1──* Manuscript *──1 User (Submitting Author)
Manuscript 1──* ManuscriptVersion
Manuscript 1──* ManuscriptAuthor *──1 User
Manuscript *──* SubjectArea (via ManuscriptSubject)
Manuscript 1──* ReviewRound
ReviewRound 1──* ReviewAssignment *──1 User (Reviewer)
ReviewAssignment 1──1 ReviewReport
ReviewRound 1──* EditorialDecision *──1 User (Editor)
Manuscript 1──* ManuscriptFile
User 1──* UserRole (per Journal)
User 1──* ReviewerExpertise *──1 SubjectArea
User 1──* ConflictOfInterest *──1 User
Journal 1──* ReviewForm 1──* ReviewFormElement
```

### Core Schema

```sql
-- ============================================================
-- IDENTITY & ACCESS
-- ============================================================

CREATE TABLE users (
    user_id         BIGSERIAL PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    given_name      VARCHAR(255) NOT NULL,
    family_name     VARCHAR(255) NOT NULL,
    orcid_id        VARCHAR(19),          -- ISO 27729 / ISNI format
    ror_id          VARCHAR(30),          -- Research Organization Registry
    affiliation     TEXT,
    country_code    CHAR(2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at   TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE journals (
    journal_id      BIGSERIAL PRIMARY KEY,
    title           VARCHAR(500) NOT NULL,
    issn_print      VARCHAR(9),
    issn_online     VARCHAR(9),
    publisher_name  VARCHAR(500),
    description     TEXT,
    review_model    VARCHAR(50) NOT NULL DEFAULT 'double-anonymous',
        -- ANSI/NISO Z39.106-2023 peer review terminology
        -- Values: 'open', 'single-anonymous', 'double-anonymous'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_roles (
    user_role_id    BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(user_id),
    journal_id      BIGINT NOT NULL REFERENCES journals(journal_id),
    role            VARCHAR(30) NOT NULL,
        -- Values: 'author', 'reviewer', 'editor', 'managing_editor',
        --         'editor_in_chief', 'admin'
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, journal_id, role)
);

-- ============================================================
-- SUBJECT TAXONOMY
-- ============================================================

CREATE TABLE subject_areas (
    subject_id      BIGSERIAL PRIMARY KEY,
    parent_id       BIGINT REFERENCES subject_areas(subject_id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) UNIQUE
);

CREATE TABLE reviewer_expertise (
    user_id         BIGINT NOT NULL REFERENCES users(user_id),
    subject_id      BIGINT NOT NULL REFERENCES subject_areas(subject_id),
    self_rated      BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (user_id, subject_id)
);

-- ============================================================
-- MANUSCRIPTS & SUBMISSIONS
-- ============================================================

CREATE TABLE manuscripts (
    manuscript_id       BIGSERIAL PRIMARY KEY,
    journal_id          BIGINT NOT NULL REFERENCES journals(journal_id),
    submitting_user_id  BIGINT NOT NULL REFERENCES users(user_id),
    manuscript_number   VARCHAR(50) NOT NULL UNIQUE,  -- e.g. "SPRM-2026-00142"
    title               TEXT NOT NULL,
    abstract            TEXT,
    article_type        VARCHAR(100),     -- e.g. 'original-research', 'review', 'registered-report'
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
        -- Values: 'draft', 'submitted', 'with_editor', 'under_review',
        --         'revision_requested', 'revised', 'accepted',
        --         'rejected', 'desk_rejected', 'withdrawn'
    submitted_at        TIMESTAMPTZ,
    accepted_at         TIMESTAMPTZ,
    rejected_at         TIMESTAMPTZ,
    doi                 VARCHAR(255),     -- CrossRef DOI once assigned
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE manuscript_versions (
    version_id      BIGSERIAL PRIMARY KEY,
    manuscript_id   BIGINT NOT NULL REFERENCES manuscripts(manuscript_id),
    version_number  SMALLINT NOT NULL DEFAULT 1,
    title           TEXT NOT NULL,
    abstract        TEXT,
    cover_letter    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(manuscript_id, version_number)
);

CREATE TABLE manuscript_authors (
    manuscript_author_id BIGSERIAL PRIMARY KEY,
    manuscript_id   BIGINT NOT NULL REFERENCES manuscripts(manuscript_id),
    user_id         BIGINT REFERENCES users(user_id),  -- NULL if external
    given_name      VARCHAR(255) NOT NULL,
    family_name     VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    orcid_id        VARCHAR(19),
    affiliation     TEXT,
    author_order    SMALLINT NOT NULL,
    is_corresponding BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE TABLE manuscript_subjects (
    manuscript_id   BIGINT NOT NULL REFERENCES manuscripts(manuscript_id),
    subject_id      BIGINT NOT NULL REFERENCES subject_areas(subject_id),
    PRIMARY KEY (manuscript_id, subject_id)
);

CREATE TABLE manuscript_files (
    file_id         BIGSERIAL PRIMARY KEY,
    manuscript_id   BIGINT NOT NULL REFERENCES manuscripts(manuscript_id),
    version_id      BIGINT REFERENCES manuscript_versions(version_id),
    file_type       VARCHAR(50) NOT NULL,
        -- Values: 'manuscript', 'figure', 'table', 'supplement',
        --         'cover_letter', 'response_to_reviewers', 'revision_diff'
    file_name       VARCHAR(500) NOT NULL,
    mime_type       VARCHAR(100),
    file_size_bytes BIGINT,
    storage_path    VARCHAR(1000) NOT NULL,
    checksum_sha256 VARCHAR(64),
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    uploaded_by     BIGINT NOT NULL REFERENCES users(user_id)
);

-- ============================================================
-- PEER REVIEW WORKFLOW
-- ============================================================

CREATE TABLE review_rounds (
    round_id        BIGSERIAL PRIMARY KEY,
    manuscript_id   BIGINT NOT NULL REFERENCES manuscripts(manuscript_id),
    version_id      BIGINT NOT NULL REFERENCES manuscript_versions(version_id),
    round_number    SMALLINT NOT NULL DEFAULT 1,
    handling_editor BIGINT NOT NULL REFERENCES users(user_id),
    opened_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    closed_at       TIMESTAMPTZ,
    UNIQUE(manuscript_id, round_number)
);

CREATE TABLE review_forms (
    form_id         BIGSERIAL PRIMARY KEY,
    journal_id      BIGINT NOT NULL REFERENCES journals(journal_id),
    title           VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE review_form_elements (
    element_id      BIGSERIAL PRIMARY KEY,
    form_id         BIGINT NOT NULL REFERENCES review_forms(form_id),
    element_type    VARCHAR(30) NOT NULL,
        -- Values: 'textarea', 'rating_scale', 'radio', 'checkbox', 'dropdown'
    label           TEXT NOT NULL,
    description     TEXT,
    is_required     BOOLEAN NOT NULL DEFAULT TRUE,
    display_order   SMALLINT NOT NULL,
    options         TEXT  -- semicolon-delimited for radio/dropdown
);

CREATE TABLE review_assignments (
    assignment_id   BIGSERIAL PRIMARY KEY,
    round_id        BIGINT NOT NULL REFERENCES review_rounds(round_id),
    reviewer_id     BIGINT NOT NULL REFERENCES users(user_id),
    form_id         BIGINT REFERENCES review_forms(form_id),
    status          VARCHAR(30) NOT NULL DEFAULT 'invited',
        -- Values: 'invited', 'accepted', 'declined', 'completed',
        --         'cancelled', 'overdue'
    recommendation  VARCHAR(30),
        -- Values: 'accept', 'minor_revision', 'major_revision',
        --         'reject', 'reject_resubmit'
    invited_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    responded_at    TIMESTAMPTZ,
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    quality_score   SMALLINT,  -- editor-assigned 1-5 quality rating
    UNIQUE(round_id, reviewer_id)
);

CREATE TABLE review_responses (
    response_id     BIGSERIAL PRIMARY KEY,
    assignment_id   BIGINT NOT NULL REFERENCES review_assignments(assignment_id),
    element_id      BIGINT NOT NULL REFERENCES review_form_elements(element_id),
    response_text   TEXT,
    response_rating SMALLINT,
    UNIQUE(assignment_id, element_id)
);

CREATE TABLE review_comments (
    comment_id      BIGSERIAL PRIMARY KEY,
    assignment_id   BIGINT NOT NULL REFERENCES review_assignments(assignment_id),
    comment_type    VARCHAR(30) NOT NULL DEFAULT 'for_author',
        -- Values: 'for_author', 'for_editor'
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- EDITORIAL DECISIONS
-- ============================================================

CREATE TABLE editorial_decisions (
    decision_id     BIGSERIAL PRIMARY KEY,
    round_id        BIGINT NOT NULL REFERENCES review_rounds(round_id),
    editor_id       BIGINT NOT NULL REFERENCES users(user_id),
    decision        VARCHAR(30) NOT NULL,
        -- Values: 'accept', 'minor_revision', 'major_revision',
        --         'reject', 'desk_reject'
    rationale       TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE decision_letters (
    letter_id       BIGSERIAL PRIMARY KEY,
    decision_id     BIGINT NOT NULL REFERENCES editorial_decisions(decision_id),
    subject         VARCHAR(500),
    body            TEXT NOT NULL,
    sent_at         TIMESTAMPTZ
);

-- ============================================================
-- CONFLICT OF INTEREST & INTEGRITY
-- ============================================================

CREATE TABLE conflicts_of_interest (
    conflict_id     BIGSERIAL PRIMARY KEY,
    user_a_id       BIGINT NOT NULL REFERENCES users(user_id),
    user_b_id       BIGINT NOT NULL REFERENCES users(user_id),
    conflict_type   VARCHAR(50) NOT NULL,
        -- Values: 'co_author', 'institutional', 'personal',
        --         'financial', 'advisor_student', 'declared'
    detected_by     VARCHAR(30) NOT NULL DEFAULT 'system',
        -- Values: 'system', 'self_declared', 'editor'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (user_a_id < user_b_id)  -- prevent duplicates
);

CREATE TABLE integrity_checks (
    check_id        BIGSERIAL PRIMARY KEY,
    manuscript_id   BIGINT NOT NULL REFERENCES manuscripts(manuscript_id),
    check_type      VARCHAR(50) NOT NULL,
        -- Values: 'plagiarism', 'image_integrity', 'statistical_reporting'
    provider        VARCHAR(100),   -- e.g. 'ithenticate', 'proofig'
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    similarity_pct  DECIMAL(5,2),
    report_url      VARCHAR(1000),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- AUDIT & NOTIFICATIONS
-- ============================================================

CREATE TABLE audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    manuscript_id   BIGINT REFERENCES manuscripts(manuscript_id),
    actor_id        BIGINT REFERENCES users(user_id),
    action          VARCHAR(100) NOT NULL,
    detail          TEXT,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE notifications (
    notification_id BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(user_id),
    notification_type VARCHAR(100) NOT NULL,
    manuscript_id   BIGINT REFERENCES manuscripts(manuscript_id),
    subject         VARCHAR(500),
    body            TEXT,
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    email_sent      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- KEY INDEXES
-- ============================================================

CREATE INDEX idx_manuscripts_journal ON manuscripts(journal_id);
CREATE INDEX idx_manuscripts_status ON manuscripts(status);
CREATE INDEX idx_manuscripts_submitter ON manuscripts(submitting_user_id);
CREATE INDEX idx_review_assignments_reviewer ON review_assignments(reviewer_id);
CREATE INDEX idx_review_assignments_status ON review_assignments(status);
CREATE INDEX idx_audit_log_manuscript ON audit_log(manuscript_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);
CREATE INDEX idx_notifications_user ON notifications(user_id, is_read);
```

---

## Pros and Cons

### Pros

- **Proven in this domain.** OJS, Editorial Manager, and ScholarOne all use relational schemas. The entity relationships of scholarly publishing are well-understood and map naturally to normalized tables.
- **Strong referential integrity.** Foreign keys enforce that every review assignment references a valid reviewer and review round, every decision references a valid editor, and every file references a valid manuscript. This prevents orphaned data and workflow corruption.
- **Excellent auditability.** The dedicated `audit_log` table combined with strict schema constraints makes GDPR compliance, COPE ethics audits, and institutional data requests straightforward.
- **Standards alignment.** Columns like `orcid_id`, `doi`, `review_model`, and `article_type` directly encode values from ORCID (ISO 27729), CrossRef, and ANSI/NISO Z39.106-2023, making API integration and JATS XML export simple.
- **Mature tooling.** PostgreSQL has decades of production-grade tooling for backups, replication, monitoring, migration (Flyway, Alembic), and ORMs in every major language.
- **Query flexibility.** Complex queries such as "find all reviewers with expertise in subject X who have no conflict with manuscript Y's authors and have completed fewer than 3 reviews this month" are natural joins across well-indexed tables.

### Cons

- **Schema rigidity.** Adding new review form types, new manuscript metadata fields, or new workflow states requires ALTER TABLE migrations. In a multi-journal platform where each journal may have different metadata requirements, this creates ongoing migration burden.
- **Review form flexibility is limited.** The `review_form_elements` + `review_responses` pattern (Entity-Attribute-Value) handles custom forms but produces complex queries and can degrade performance at scale.
- **Workflow state is implicit.** The manuscript `status` column and the latest `editorial_decisions` row together define the current workflow state, but reconstructing the full history of state transitions requires querying multiple tables and ordering by timestamp. There is no explicit event log.
- **Scalability ceiling.** For very large publishers (100,000+ manuscripts/year across hundreds of journals), the normalized joins across `manuscripts`, `review_rounds`, `review_assignments`, and `review_responses` can become expensive. Read replicas and materialized views can help but add operational complexity.
- **AI feature integration is bolted on.** Reviewer matching scores, AI pre-screening results, and NLP-derived metadata do not fit naturally into the normalized relational model and tend to accumulate in loosely-typed columns or side tables.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ |
| **ORM** | Prisma (TypeScript), SQLAlchemy (Python), or Diesel (Rust) |
| **Migration** | Flyway or Alembic with version-controlled SQL scripts |
| **Search** | pg_trgm + full-text search for manuscript/reviewer search; optional Elasticsearch for AI features |
| **File Storage** | S3-compatible object storage with paths recorded in `manuscript_files` |
| **Caching** | Redis for session management, notification queues, and frequently-accessed reviewer metrics |
| **API** | REST with OpenAPI 3.1 specification; GraphQL optional |

---

## Migration and Scaling Considerations

- **Schema migrations** should use versioned SQL files managed by Flyway or Alembic, with CI/CD checks that prevent backward-incompatible changes in production.
- **Multi-journal isolation** can be achieved via a `journal_id` column on all tenant-scoped tables (shared schema multi-tenancy) or via PostgreSQL schemas (one schema per journal). Shared schema is simpler for cross-journal analytics.
- **Read replicas** should be deployed for reporting, reviewer analytics, and AI feature queries to keep the primary database responsive for editorial workflow writes.
- **Partitioning** the `audit_log` and `notifications` tables by date (using PostgreSQL declarative partitioning) prevents these high-volume tables from degrading query performance.
- **Data retention** policies should be implemented for GDPR compliance, with configurable retention periods for audit logs, reviewer identities, and manuscript files.
- **Export pipeline** for JATS XML and CrossRef deposit XML should read from the normalized tables and transform to XML at the application layer, using the structured column values to populate metadata elements.
