# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Scientific Peer Review Manager · Generated: 2026-05-25

## Summary

A pragmatic hybrid architecture that uses PostgreSQL as a single database engine, combining normalized relational tables for core entities with stable schemas (users, journals, manuscripts, review rounds) and JSONB columns for semi-structured or variable data (review form responses, AI analysis results, manuscript metadata extensions, integration payloads). This approach captures the referential integrity benefits of a relational model while accommodating the schema variability inherent in a multi-journal peer review platform — where each journal may have different metadata requirements, review forms, and workflow configurations. PostgreSQL's native JSONB support with GIN indexing, JSONPath queries, and schema validation via CHECK constraints makes it possible to get "the best of both worlds" without introducing a separate document database.

---

## Key Entities and Relationships

### Design Principles

1. **Relational where relationships matter.** Users, journals, manuscripts, review rounds, and assignments are normalized tables with foreign keys.
2. **JSONB where schema varies.** Review form definitions, review responses, AI scoring results, manuscript metadata extensions, and integration payloads use JSONB columns.
3. **Typed JSONB with validation.** JSONB columns include CHECK constraints or application-layer JSON Schema validation to prevent garbage data while allowing schema evolution.
4. **Single database engine.** No separate MongoDB or Elasticsearch instance for the core data model. PostgreSQL handles relational queries, document queries, and full-text search.

### Core Schema

```sql
-- ============================================================
-- IDENTITY & ACCESS (relational — stable schema)
-- ============================================================

CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    given_name      VARCHAR(255) NOT NULL,
    family_name     VARCHAR(255) NOT NULL,
    orcid_id        VARCHAR(19),
    affiliation     TEXT,
    country_code    CHAR(2),
    profile         JSONB NOT NULL DEFAULT '{}',
        -- Flexible profile data: research interests, bio, website,
        -- social links, notification preferences, language preferences
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE journals (
    journal_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           VARCHAR(500) NOT NULL,
    issn_print      VARCHAR(9),
    issn_online     VARCHAR(9),
    publisher_name  VARCHAR(500),
    review_model    VARCHAR(50) NOT NULL DEFAULT 'double-anonymous',
    config          JSONB NOT NULL DEFAULT '{}',
        -- Journal-specific configuration:
        -- {
        --   "submission_guidelines": "...",
        --   "article_types": ["original-research", "review", "case-report"],
        --   "required_metadata_fields": ["funding_statement", "data_availability"],
        --   "max_reviewers_per_round": 3,
        --   "default_review_deadline_days": 21,
        --   "desk_rejection_enabled": true,
        --   "open_review_options": { "open_identities": false, "open_reports": false },
        --   "integrity_checks": ["plagiarism", "image_integrity"],
        --   "email_templates": { "invitation": "...", "reminder": "...", "decision": "..." }
        -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_roles (
    user_role_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id),
    journal_id      UUID NOT NULL REFERENCES journals(journal_id),
    role            VARCHAR(30) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
        -- Fine-grained permissions as JSON array:
        -- ["assign_reviewers", "make_decisions", "view_all_manuscripts"]
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, journal_id, role)
);

-- ============================================================
-- REVIEWER EXPERTISE (relational + JSONB)
-- ============================================================

CREATE TABLE subject_areas (
    subject_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES subject_areas(subject_id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) UNIQUE
);

CREATE TABLE reviewer_profiles (
    user_id         UUID PRIMARY KEY REFERENCES users(user_id),
    expertise_areas UUID[] NOT NULL DEFAULT '{}',  -- array of subject_area IDs
    expertise_detail JSONB NOT NULL DEFAULT '{}',
        -- Rich expertise data from ORCID, publications, and self-declaration:
        -- {
        --   "publications_count": 47,
        --   "h_index": 12,
        --   "recent_topics": ["protein folding", "cryo-EM", "molecular dynamics"],
        --   "keyword_embeddings_version": "2026-05-01",
        --   "availability": { "max_active_reviews": 3, "unavailable_until": null },
        --   "historical_stats": {
        --     "invitations_received": 34,
        --     "invitations_accepted": 22,
        --     "avg_days_to_complete": 18.5,
        --     "avg_quality_score": 4.2
        --   }
        -- }
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- MANUSCRIPTS (relational core + JSONB metadata)
-- ============================================================

CREATE TABLE manuscripts (
    manuscript_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_id          UUID NOT NULL REFERENCES journals(journal_id),
    submitting_user_id  UUID NOT NULL REFERENCES users(user_id),
    manuscript_number   VARCHAR(50) NOT NULL UNIQUE,
    title               TEXT NOT NULL,
    abstract            TEXT,
    article_type        VARCHAR(100),
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    metadata            JSONB NOT NULL DEFAULT '{}',
        -- Journal-specific metadata (varies by journal config):
        -- {
        --   "keywords": ["protein folding", "deep learning"],
        --   "funding_statement": "Supported by NIH grant R01-...",
        --   "data_availability": "Data deposited at zenodo.org/...",
        --   "ethics_approval": "IRB #2026-0042",
        --   "competing_interests": "None declared",
        --   "word_count": 8420,
        --   "figure_count": 7,
        --   "table_count": 3,
        --   "registered_report_stage": null,
        --   "preprint_doi": "10.1101/2026.05.01.12345"
        -- }
    ai_analysis         JSONB NOT NULL DEFAULT '{}',
        -- AI pre-screening results (populated asynchronously):
        -- {
        --   "scope_fit_score": 0.87,
        --   "methodology_completeness": 0.72,
        --   "statistical_reporting_flags": ["missing_effect_sizes"],
        --   "suggested_subject_areas": ["cs.LG", "q-bio.BM"],
        --   "desk_rejection_risk": 0.15,
        --   "language_quality_score": 0.94,
        --   "analyzed_at": "2026-05-25T11:00:00Z",
        --   "model_version": "sprm-screen-v3"
        -- }
    submitted_at        TIMESTAMPTZ,
    doi                 VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- GIN index for querying JSONB metadata
CREATE INDEX idx_manuscripts_metadata ON manuscripts USING GIN(metadata);
CREATE INDEX idx_manuscripts_ai ON manuscripts USING GIN(ai_analysis);

CREATE TABLE manuscript_authors (
    manuscript_author_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manuscript_id   UUID NOT NULL REFERENCES manuscripts(manuscript_id),
    user_id         UUID REFERENCES users(user_id),
    given_name      VARCHAR(255) NOT NULL,
    family_name     VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    orcid_id        VARCHAR(19),
    affiliation     TEXT,
    author_order    SMALLINT NOT NULL,
    is_corresponding BOOLEAN NOT NULL DEFAULT FALSE,
    contribution    JSONB DEFAULT '{}',
        -- CRediT (Contributor Roles Taxonomy):
        -- {
        --   "conceptualization": true,
        --   "methodology": true,
        --   "writing_original_draft": true,
        --   "writing_review_editing": false
        -- }
    UNIQUE(manuscript_id, author_order)
);

CREATE TABLE manuscript_versions (
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manuscript_id   UUID NOT NULL REFERENCES manuscripts(manuscript_id),
    version_number  SMALLINT NOT NULL DEFAULT 1,
    title           TEXT NOT NULL,
    abstract        TEXT,
    cover_letter    TEXT,
    changes_summary TEXT,       -- author's description of revisions
    metadata_diff   JSONB,     -- what metadata changed from previous version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(manuscript_id, version_number)
);

CREATE TABLE manuscript_files (
    file_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manuscript_id   UUID NOT NULL REFERENCES manuscripts(manuscript_id),
    version_id      UUID REFERENCES manuscript_versions(version_id),
    file_type       VARCHAR(50) NOT NULL,
    file_name       VARCHAR(500) NOT NULL,
    mime_type       VARCHAR(100),
    file_size_bytes BIGINT,
    storage_path    VARCHAR(1000) NOT NULL,
    checksum_sha256 VARCHAR(64),
    file_metadata   JSONB NOT NULL DEFAULT '{}',
        -- Extracted file metadata:
        -- {
        --   "page_count": 24,
        --   "latex_detected": true,
        --   "figures_detected": ["fig1.png", "fig2.tif"],
        --   "reference_count": 42
        -- }
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    uploaded_by     UUID NOT NULL REFERENCES users(user_id)
);

-- ============================================================
-- PEER REVIEW (relational structure + JSONB content)
-- ============================================================

CREATE TABLE review_rounds (
    round_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manuscript_id   UUID NOT NULL REFERENCES manuscripts(manuscript_id),
    version_id      UUID NOT NULL REFERENCES manuscript_versions(version_id),
    round_number    SMALLINT NOT NULL DEFAULT 1,
    handling_editor UUID NOT NULL REFERENCES users(user_id),
    opened_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    closed_at       TIMESTAMPTZ,
    UNIQUE(manuscript_id, round_number)
);

-- Review forms stored as JSONB schema (no EAV pattern needed)
CREATE TABLE review_forms (
    form_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_id      UUID NOT NULL REFERENCES journals(journal_id),
    title           VARCHAR(255) NOT NULL,
    form_schema     JSONB NOT NULL,
        -- JSON Schema defining the review form structure:
        -- {
        --   "sections": [
        --     {
        --       "title": "Overall Assessment",
        --       "fields": [
        --         {
        --           "id": "novelty",
        --           "type": "rating",
        --           "label": "Novelty and Significance",
        --           "min": 1, "max": 5,
        --           "required": true
        --         },
        --         {
        --           "id": "methodology",
        --           "type": "rating",
        --           "label": "Methodological Rigor",
        --           "min": 1, "max": 5,
        --           "required": true
        --         },
        --         {
        --           "id": "comments_to_author",
        --           "type": "textarea",
        --           "label": "Comments to Authors",
        --           "required": true,
        --           "min_words": 50
        --         }
        --       ]
        --     }
        --   ]
        -- }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE review_assignments (
    assignment_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    round_id        UUID NOT NULL REFERENCES review_rounds(round_id),
    reviewer_id     UUID NOT NULL REFERENCES users(user_id),
    form_id         UUID REFERENCES review_forms(form_id),
    status          VARCHAR(30) NOT NULL DEFAULT 'invited',
    recommendation  VARCHAR(30),
    invited_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    responded_at    TIMESTAMPTZ,
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    match_context   JSONB DEFAULT '{}',
        -- How was this reviewer selected?
        -- {
        --   "match_method": "ai_semantic",
        --   "match_score": 0.89,
        --   "matched_topics": ["protein folding", "molecular dynamics"],
        --   "shared_references": 7,
        --   "conflict_check_passed": true,
        --   "workload_at_invitation": 2,
        --   "predicted_accept_probability": 0.72
        -- }
    review_data     JSONB DEFAULT '{}',
        -- The actual review content (responses to form_schema):
        -- {
        --   "novelty": 4,
        --   "methodology": 3,
        --   "presentation": 4,
        --   "comments_to_author": "This manuscript presents...",
        --   "comments_to_editor": "I recommend...",
        --   "attachment_file_ids": ["f1a2b3c4-..."]
        -- }
    quality_metrics JSONB DEFAULT '{}',
        -- AI-assessed review quality:
        -- {
        --   "word_count": 1247,
        --   "specificity_score": 0.78,
        --   "constructiveness_score": 0.85,
        --   "boilerplate_score": 0.12,
        --   "assessed_at": "2026-05-25T14:00:00Z"
        -- }
    UNIQUE(round_id, reviewer_id)
);

-- ============================================================
-- EDITORIAL DECISIONS (relational + JSONB)
-- ============================================================

CREATE TABLE editorial_decisions (
    decision_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    round_id        UUID NOT NULL REFERENCES review_rounds(round_id),
    editor_id       UUID NOT NULL REFERENCES users(user_id),
    decision        VARCHAR(30) NOT NULL,
    rationale       TEXT,
    ai_synthesis    JSONB DEFAULT '{}',
        -- AI-generated synthesis of reviewer reports:
        -- {
        --   "consensus_summary": "Both reviewers agree the topic is novel but...",
        --   "disagreement_points": [
        --     { "dimension": "methodology", "reviewer_1": "adequate", "reviewer_2": "insufficient" }
        --   ],
        --   "recommended_action": "major_revision",
        --   "confidence": 0.82,
        --   "key_revision_items": [
        --     "Expand sample size justification",
        --     "Add sensitivity analysis for parameter X"
        --   ]
        -- }
    decision_letter JSONB NOT NULL,
        -- {
        --   "subject": "Decision on SPRM-2026-00142",
        --   "body": "Dear Dr. Smith,...",
        --   "sent_at": "2026-05-25T16:00:00Z"
        -- }
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- INTEGRITY CHECKS
-- ============================================================

CREATE TABLE integrity_checks (
    check_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manuscript_id   UUID NOT NULL REFERENCES manuscripts(manuscript_id),
    check_type      VARCHAR(50) NOT NULL,
    provider        VARCHAR(100),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    results         JSONB NOT NULL DEFAULT '{}',
        -- Provider-specific results (schema varies by provider):
        -- For iThenticate:
        -- {
        --   "similarity_pct": 12.3,
        --   "internet_sources_pct": 3.1,
        --   "publication_sources_pct": 9.2,
        --   "top_matches": [
        --     { "source": "doi:10.1234/...", "overlap_pct": 4.2, "word_count": 187 }
        --   ],
        --   "report_url": "https://..."
        -- }
        -- For Proofig:
        -- {
        --   "images_analyzed": 7,
        --   "flags": [
        --     { "figure": "fig3a", "issue": "potential_duplication", "confidence": 0.91 }
        --   ]
        -- }
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_integrity_results ON integrity_checks USING GIN(results);

-- ============================================================
-- CONFLICT OF INTEREST
-- ============================================================

CREATE TABLE conflicts_of_interest (
    conflict_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_a_id       UUID NOT NULL REFERENCES users(user_id),
    user_b_id       UUID NOT NULL REFERENCES users(user_id),
    conflict_type   VARCHAR(50) NOT NULL,
    detected_by     VARCHAR(30) NOT NULL DEFAULT 'system',
    evidence        JSONB DEFAULT '{}',
        -- {
        --   "co_authored_papers": ["doi:10.1234/...", "doi:10.5678/..."],
        --   "shared_institution": "MIT",
        --   "detection_method": "co_authorship_graph",
        --   "confidence": 0.95
        -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (user_a_id < user_b_id)
);

-- ============================================================
-- AUDIT & ACTIVITY LOG
-- ============================================================

CREATE TABLE activity_log (
    log_id          BIGSERIAL PRIMARY KEY,
    manuscript_id   UUID REFERENCES manuscripts(manuscript_id),
    actor_id        UUID REFERENCES users(user_id),
    action          VARCHAR(100) NOT NULL,
    detail          JSONB DEFAULT '{}',
        -- Action-specific context without rigid column requirements
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_activity_log_manuscript ON activity_log(manuscript_id, created_at);
CREATE INDEX idx_activity_log_actor ON activity_log(actor_id, created_at);

-- ============================================================
-- NOTIFICATIONS
-- ============================================================

CREATE TABLE notifications (
    notification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id),
    notification_type VARCHAR(100) NOT NULL,
    manuscript_id   UUID REFERENCES manuscripts(manuscript_id),
    content         JSONB NOT NULL,
        -- {
        --   "subject": "New review assignment",
        --   "body": "You have been invited to review...",
        --   "action_url": "/manuscripts/SPRM-2026-00142/review",
        --   "priority": "normal"
        -- }
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    email_sent_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- FULL-TEXT SEARCH
-- ============================================================

ALTER TABLE manuscripts ADD COLUMN search_vector TSVECTOR
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(abstract, '')), 'B')
    ) STORED;

CREATE INDEX idx_manuscripts_search ON manuscripts USING GIN(search_vector);
```

---

## Pros and Cons

### Pros

- **Single database engine.** PostgreSQL handles relational queries, document queries, full-text search, and JSONB indexing. No need to deploy, synchronize, or maintain a separate MongoDB or Elasticsearch instance. This dramatically reduces operational complexity for self-hosted deployments — a key requirement for the target market.
- **Schema flexibility where it matters.** Review forms, AI analysis results, integration payloads, and journal-specific metadata extensions all vary by journal and evolve over time. JSONB columns accommodate this variation without ALTER TABLE migrations, while the relational skeleton ensures referential integrity for core entities.
- **Review forms without EAV.** The traditional approach (Suggestion 1) uses an Entity-Attribute-Value pattern for custom review forms, which produces complex queries. Here, the review form schema is a JSONB document, and review responses are stored as a JSONB object on the assignment. Querying "average novelty score across all reviews for journal X" is a simple JSONB path query.
- **AI features are first-class.** The `ai_analysis` column on manuscripts, `match_context` on review assignments, and `quality_metrics` on reviews store AI-generated data naturally. When the AI model changes, the schema of these JSONB objects evolves without migration — old and new formats coexist.
- **Queryable documents.** PostgreSQL GIN indexes on JSONB columns enable efficient queries like `WHERE ai_analysis->>'scope_fit_score' > '0.8'` or `WHERE metadata @> '{"article_type": "registered-report"}'`. JSONPath expressions handle complex nested queries.
- **Standards compliance is straightforward.** Core fields like `orcid_id`, `doi`, `review_model`, and `manuscript_number` are typed columns with validation. JATS XML export reads from a mix of relational columns and JSONB metadata. The ANSI/NISO Z39.106-2023 review terminology maps to the `review_model` column.
- **Practical migration path.** This approach can start as Suggestion 1 (pure relational) and incrementally add JSONB columns as flexibility needs emerge. Or it can start with JSONB from day one and add relational constraints as schemas stabilize.

### Cons

- **Discipline required.** Without enforcement, JSONB columns can become dumping grounds for unstructured data. Application-layer JSON Schema validation or PostgreSQL CHECK constraints are needed to maintain data quality. Teams must resist the temptation to put everything in JSONB.
- **Partial indexing limitations.** While GIN indexes on JSONB are powerful, they are less efficient than B-tree indexes on typed columns for range queries and sorting. Queries like "all manuscripts submitted between date X and Y with AI scope score above 0.8" may need composite indexes that mix relational columns and JSONB paths.
- **ORM friction.** Many ORMs (Prisma, ActiveRecord) have limited or awkward support for JSONB query operators. Application code may need raw SQL or query builder patterns for complex JSONB queries, reducing the benefit of ORM abstraction.
- **Reporting complexity.** Business intelligence tools and reporting dashboards may struggle with JSONB columns. Materializing JSONB data into flat reporting tables (or using PostgreSQL views with JSONB extraction) adds a reporting layer.
- **No schema-level documentation.** A relational column named `similarity_pct DECIMAL(5,2)` is self-documenting. A JSONB field nested three levels deep in a `results` column requires external documentation or JSON Schema files to understand. This increases onboarding friction for new developers.
- **GDPR data discovery.** When PII can live in both typed columns and JSONB objects, GDPR data subject access requests and right-to-erasure processing must search both. This requires careful cataloging of where PII appears in JSONB structures.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ (JSONB, GIN indexes, JSONPath, generated columns) |
| **ORM** | Prisma with `@db.JsonB` types, or SQLAlchemy with `JSONB` column type |
| **Schema Validation** | Application-layer JSON Schema validation (Ajv for TypeScript, jsonschema for Python) |
| **Migration** | Flyway or Alembic for relational schema; JSONB schema versions tracked in application code |
| **Full-text Search** | PostgreSQL built-in `tsvector` + GIN indexes for MVP; optional Meilisearch for advanced faceted search |
| **File Storage** | S3-compatible object store |
| **Caching** | Redis for session state, notification queues, and reviewer availability lookups |
| **API** | REST with OpenAPI 3.1; JSONB columns map naturally to JSON API responses |

---

## Migration and Scaling Considerations

- **JSONB schema versioning.** Include a `_schema_version` key in JSONB objects that are likely to evolve (e.g. `ai_analysis`, `form_schema`, `results`). Application code reads the version and applies appropriate parsing logic. This avoids the hard event-schema-evolution problem of Suggestion 2 while maintaining backward compatibility.
- **Progressive structuring.** Start with JSONB for uncertain fields. Once a field's schema stabilizes and it becomes heavily queried, promote it to a typed relational column. Example: if `ai_analysis->>'scope_fit_score'` becomes a critical filter, add a `scope_fit_score DECIMAL(4,3)` column with a trigger that populates it from the JSONB on write.
- **JSONB size monitoring.** Monitor average JSONB column sizes. If `review_data` grows beyond ~10KB per row (large review forms with attachments), consider extracting attachment references to a separate relational table.
- **Multi-journal partitioning.** PostgreSQL declarative partitioning by `journal_id` can partition large tables (manuscripts, review_assignments, activity_log) for publishers managing hundreds of journals.
- **Backup and point-in-time recovery.** Standard PostgreSQL WAL-based backup and PITR works for both relational and JSONB data. No separate backup strategy needed for document data.
- **Self-hosted deployment.** A single PostgreSQL instance with JSONB handles the full data model. This simplifies self-hosted deployment for smaller journals and societies — a key competitive advantage over systems that require Elasticsearch, MongoDB, or Kafka alongside a relational database.
