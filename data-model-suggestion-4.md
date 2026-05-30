# Data Model Suggestion 4: Graph-Relational Hybrid (PostgreSQL + Neo4j)

> Project: Scientific Peer Review Manager · Generated: 2026-05-25

## Summary

A dual-database architecture that uses PostgreSQL for transactional workflow data (manuscripts, review rounds, decisions, files) and Neo4j as a dedicated graph database for the academic knowledge graph — modeling researchers, publications, institutions, expertise areas, co-authorship networks, and reviewer-manuscript affinity. The graph layer powers the AI-native features that differentiate this platform: semantic reviewer matching, conflict-of-interest detection, workload optimization, and research community discovery. This approach acknowledges that the core editorial workflow is fundamentally transactional and well-served by a relational model, while the reviewer matching and network analysis problems are fundamentally graph problems that become exponentially harder to solve with SQL joins.

---

## Key Entities and Relationships

### Architecture Overview

```
┌──────────────────────────────────────────────────┐
│              Application Layer                    │
│   (Manuscript workflow, editorial decisions,      │
│    file management, notifications)                │
├────────────────────┬─────────────────────────────┤
│                    │                              │
│  PostgreSQL        │  Neo4j Knowledge Graph       │
│  (Transactional)   │  (Analytical / AI)           │
│                    │                              │
│  - manuscripts     │  - (:Researcher)             │
│  - review_rounds   │  - (:Publication)            │
│  - assignments     │  - (:Institution)            │
│  - decisions       │  - (:Topic)                  │
│  - files           │  - (:Journal)                │
│  - users           │  - (:Manuscript)             │
│  - audit_log       │  - [:CO_AUTHORED]            │
│                    │  - [:PUBLISHED_IN]           │
│                    │  - [:AFFILIATED_WITH]        │
│                    │  - [:HAS_EXPERTISE]          │
│                    │  - [:REVIEWED]               │
│                    │  - [:CITES]                  │
│                    │  - [:CONFLICTS_WITH]         │
│                    │  - [:SIMILAR_TO]             │
└────────────────────┴─────────────────────────────┘
```

### PostgreSQL Schema (Transactional Layer)

The PostgreSQL schema handles the core editorial workflow. It is similar to the relational core of Suggestion 3, with JSONB for variable data, but delegates all reviewer matching and network analysis to Neo4j.

```sql
-- ============================================================
-- POSTGRESQL: TRANSACTIONAL WORKFLOW
-- ============================================================

CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    given_name      VARCHAR(255) NOT NULL,
    family_name     VARCHAR(255) NOT NULL,
    orcid_id        VARCHAR(19),
    affiliation     TEXT,
    graph_node_id   VARCHAR(50),   -- Reference to Neo4j Researcher node
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE journals (
    journal_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           VARCHAR(500) NOT NULL,
    issn_print      VARCHAR(9),
    issn_online     VARCHAR(9),
    review_model    VARCHAR(50) NOT NULL DEFAULT 'double-anonymous',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

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
    graph_node_id       VARCHAR(50),   -- Reference to Neo4j Manuscript node
    submitted_at        TIMESTAMPTZ,
    doi                 VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE manuscript_versions (
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manuscript_id   UUID NOT NULL REFERENCES manuscripts(manuscript_id),
    version_number  SMALLINT NOT NULL DEFAULT 1,
    title           TEXT NOT NULL,
    abstract        TEXT,
    cover_letter    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(manuscript_id, version_number)
);

CREATE TABLE manuscript_authors (
    manuscript_author_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manuscript_id   UUID NOT NULL REFERENCES manuscripts(manuscript_id),
    user_id         UUID REFERENCES users(user_id),
    given_name      VARCHAR(255) NOT NULL,
    family_name     VARCHAR(255) NOT NULL,
    orcid_id        VARCHAR(19),
    affiliation     TEXT,
    author_order    SMALLINT NOT NULL,
    is_corresponding BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE TABLE manuscript_files (
    file_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manuscript_id   UUID NOT NULL REFERENCES manuscripts(manuscript_id),
    version_id      UUID REFERENCES manuscript_versions(version_id),
    file_type       VARCHAR(50) NOT NULL,
    file_name       VARCHAR(500) NOT NULL,
    storage_path    VARCHAR(1000) NOT NULL,
    mime_type       VARCHAR(100),
    file_size_bytes BIGINT,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    uploaded_by     UUID NOT NULL REFERENCES users(user_id)
);

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

CREATE TABLE review_assignments (
    assignment_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    round_id        UUID NOT NULL REFERENCES review_rounds(round_id),
    reviewer_id     UUID NOT NULL REFERENCES users(user_id),
    status          VARCHAR(30) NOT NULL DEFAULT 'invited',
    recommendation  VARCHAR(30),
    review_data     JSONB DEFAULT '{}',
    match_context   JSONB DEFAULT '{}',
        -- Graph-derived match information:
        -- {
        --   "graph_match_score": 0.89,
        --   "expertise_path": ["cryo-EM", "structural-biology", "protein-folding"],
        --   "shared_references_count": 7,
        --   "co_author_distance": 4,
        --   "institution_overlap": false,
        --   "workload_score": 0.3,
        --   "composite_score": 0.82
        -- }
    invited_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    responded_at    TIMESTAMPTZ,
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    UNIQUE(round_id, reviewer_id)
);

CREATE TABLE editorial_decisions (
    decision_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    round_id        UUID NOT NULL REFERENCES review_rounds(round_id),
    editor_id       UUID NOT NULL REFERENCES users(user_id),
    decision        VARCHAR(30) NOT NULL,
    rationale       TEXT,
    decision_letter JSONB NOT NULL,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE integrity_checks (
    check_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manuscript_id   UUID NOT NULL REFERENCES manuscripts(manuscript_id),
    check_type      VARCHAR(50) NOT NULL,
    provider        VARCHAR(100),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    results         JSONB NOT NULL DEFAULT '{}',
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    manuscript_id   UUID REFERENCES manuscripts(manuscript_id),
    actor_id        UUID REFERENCES users(user_id),
    action          VARCHAR(100) NOT NULL,
    detail          JSONB DEFAULT '{}',
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Neo4j Knowledge Graph Schema

The graph database models the scholarly network — researchers, their publications, topics, institutions, and the relationships between them. This is the engine behind reviewer matching, conflict detection, and research community analytics.

```cypher
// ============================================================
// NEO4J: KNOWLEDGE GRAPH NODES
// ============================================================

// Researcher node — synced from PostgreSQL users + enriched from ORCID/publications
// Properties:
//   user_id: UUID (FK to PostgreSQL users.user_id)
//   orcid_id: String
//   given_name: String
//   family_name: String
//   affiliation: String
//   h_index: Integer
//   publications_count: Integer
//   embedding: List<Float>  -- semantic embedding of research profile
//   availability_score: Float  -- current review workload (0=overloaded, 1=available)
//   last_synced_at: DateTime

CREATE CONSTRAINT researcher_user_id FOR (r:Researcher) REQUIRE r.user_id IS UNIQUE;
CREATE CONSTRAINT researcher_orcid FOR (r:Researcher) REQUIRE r.orcid_id IS UNIQUE;

// Publication node — external publications from ORCID, CrossRef, Semantic Scholar
// Properties:
//   doi: String
//   title: String
//   abstract: String
//   published_year: Integer
//   venue: String
//   citation_count: Integer
//   embedding: List<Float>  -- semantic embedding of content

CREATE CONSTRAINT publication_doi FOR (p:Publication) REQUIRE p.doi IS UNIQUE;

// Manuscript node — submitted manuscripts (mirrored from PostgreSQL)
// Properties:
//   manuscript_id: UUID (FK to PostgreSQL manuscripts.manuscript_id)
//   manuscript_number: String
//   title: String
//   abstract: String
//   article_type: String
//   embedding: List<Float>  -- semantic embedding for matching

CREATE CONSTRAINT manuscript_id FOR (m:Manuscript) REQUIRE m.manuscript_id IS UNIQUE;

// Topic node — hierarchical subject taxonomy
// Properties:
//   code: String (e.g. "cs.AI", "q-bio.BM")
//   name: String
//   level: Integer (0=root, 1=domain, 2=subfield, 3=specialty)
//   embedding: List<Float>

CREATE CONSTRAINT topic_code FOR (t:Topic) REQUIRE t.code IS UNIQUE;

// Institution node — from ROR (Research Organization Registry)
// Properties:
//   ror_id: String
//   name: String
//   country_code: String
//   type: String  -- "university", "research_institute", "company"

CREATE CONSTRAINT institution_ror FOR (i:Institution) REQUIRE i.ror_id IS UNIQUE;

// Journal node — journals in the system
// Properties:
//   journal_id: UUID (FK to PostgreSQL journals.journal_id)
//   title: String
//   issn: String

CREATE CONSTRAINT journal_id FOR (j:Journal) REQUIRE j.journal_id IS UNIQUE;

// ============================================================
// NEO4J: RELATIONSHIPS
// ============================================================

// Co-authorship (weighted by recency and frequency)
// (:Researcher)-[:CO_AUTHORED {count: 3, most_recent: 2025, dois: [...]}]->(:Researcher)

// Publication authorship
// (:Researcher)-[:AUTHORED {position: 1, is_corresponding: true}]->(:Publication)

// Publication venue
// (:Publication)-[:PUBLISHED_IN {year: 2024}]->(:Journal)

// Citation network
// (:Publication)-[:CITES]->(:Publication)

// Researcher expertise (derived from publications)
// (:Researcher)-[:HAS_EXPERTISE {score: 0.92, publication_count: 12, last_year: 2026}]->(:Topic)

// Publication topics
// (:Publication)-[:COVERS_TOPIC {relevance: 0.85}]->(:Topic)

// Manuscript topics (for submitted manuscripts)
// (:Manuscript)-[:COVERS_TOPIC {relevance: 0.90}]->(:Topic)

// Institutional affiliation
// (:Researcher)-[:AFFILIATED_WITH {current: true, since: 2019}]->(:Institution)

// Conflict of interest (explicit and detected)
// (:Researcher)-[:CONFLICTS_WITH {
//     type: "co_author",
//     evidence: "3 co-authored papers since 2022",
//     detected_at: datetime(),
//     source: "system"
// }]->(:Researcher)

// Review history
// (:Researcher)-[:REVIEWED {
//     round_id: "uuid",
//     recommendation: "minor_revision",
//     completed_at: datetime(),
//     quality_score: 4
// }]->(:Manuscript)

// Semantic similarity between manuscripts and publications
// (:Manuscript)-[:SIMILAR_TO {score: 0.87, method: "embedding_cosine"}]->(:Publication)

// Topic hierarchy
// (:Topic)-[:SUBTOPIC_OF]->(:Topic)
```

### Key Graph Queries

```cypher
// ============================================================
// REVIEWER MATCHING: Find qualified reviewers for a manuscript
// ============================================================

// 1. Find researchers with expertise matching the manuscript's topics,
//    excluding authors, conflicted researchers, and overloaded reviewers
MATCH (m:Manuscript {manuscript_id: $manuscript_id})-[:COVERS_TOPIC]->(t:Topic)
MATCH (r:Researcher)-[e:HAS_EXPERTISE]->(t)
WHERE NOT (r)-[:CONFLICTS_WITH]-(:Researcher)-[:AUTHORED]->
      (:Publication)<-[:SIMILAR_TO]-(m)
  AND NOT EXISTS {
    MATCH (r)-[:CO_AUTHORED]-(:Researcher {user_id: $author_user_ids})
  }
  AND r.availability_score > 0.3
WITH r, m, AVG(e.score) AS expertise_score
// 2. Boost score by semantic similarity to manuscript
OPTIONAL MATCH (r)-[:AUTHORED]->(p:Publication)-[s:SIMILAR_TO]->(m)
WITH r, expertise_score,
     COALESCE(AVG(s.score), 0) AS semantic_score,
     COUNT(p) AS relevant_publications
// 3. Penalize by current workload
WITH r, expertise_score, semantic_score, relevant_publications,
     (expertise_score * 0.4 + semantic_score * 0.4 +
      r.availability_score * 0.2) AS composite_score
RETURN r.user_id, r.given_name, r.family_name, r.orcid_id,
       composite_score, expertise_score, semantic_score,
       relevant_publications, r.availability_score
ORDER BY composite_score DESC
LIMIT 20;


// ============================================================
// CONFLICT DETECTION: Check all conflicts for a manuscript
// ============================================================

// Find all potential conflicts between a candidate reviewer and
// the manuscript's authors
MATCH (candidate:Researcher {user_id: $reviewer_id})
MATCH (m:Manuscript {manuscript_id: $manuscript_id})
          <-[:SIMILAR_TO]-(:Publication)<-[:AUTHORED]-(author:Researcher)

// Direct conflicts
OPTIONAL MATCH (candidate)-[c:CONFLICTS_WITH]-(author)
// Co-authorship within 3 years
OPTIONAL MATCH (candidate)-[co:CO_AUTHORED]-(author)
  WHERE co.most_recent >= date().year - 3
// Same institution
OPTIONAL MATCH (candidate)-[:AFFILIATED_WITH {current: true}]->(i:Institution)
              <-[:AFFILIATED_WITH {current: true}]-(author)
// Advisor-student (via co-authorship pattern)
OPTIONAL MATCH path = (candidate)-[:CO_AUTHORED*1..2]-(author)

RETURN candidate.user_id,
       COLLECT(DISTINCT {type: 'declared', detail: c.type}) AS declared_conflicts,
       COLLECT(DISTINCT {type: 'co_author', detail: co.most_recent}) AS coauthor_conflicts,
       COLLECT(DISTINCT {type: 'institutional', detail: i.name}) AS institutional_conflicts;


// ============================================================
// RESEARCH COMMUNITY: Discover reviewer networks by topic
// ============================================================

MATCH (t:Topic {code: $topic_code})<-[:HAS_EXPERTISE]-(r:Researcher)
WITH r, t
MATCH (r)-[:CO_AUTHORED]-(collaborator:Researcher)
WHERE (collaborator)-[:HAS_EXPERTISE]->(t)
RETURN r.user_id, r.given_name, r.family_name,
       COLLECT(DISTINCT collaborator.family_name) AS collaborators,
       SIZE([(r)-[:AUTHORED]->(p) WHERE (p)-[:COVERS_TOPIC]->(t) | p]) AS topic_publications
ORDER BY topic_publications DESC
LIMIT 50;


// ============================================================
// WORKLOAD BALANCING: Reviewer invitation optimization
// ============================================================

MATCH (r:Researcher)-[:HAS_EXPERTISE]->(t:Topic {code: $topic_code})
WHERE r.availability_score > 0.2
WITH r
OPTIONAL MATCH (r)-[rev:REVIEWED]->(m:Manuscript)
  WHERE rev.completed_at > datetime() - duration('P6M')
WITH r, COUNT(rev) AS reviews_last_6m,
     AVG(rev.quality_score) AS avg_quality
RETURN r.user_id, r.given_name, r.family_name,
       r.availability_score, reviews_last_6m, avg_quality,
       // Predict acceptance probability based on historical pattern
       CASE
         WHEN reviews_last_6m < 3 THEN 0.8
         WHEN reviews_last_6m < 6 THEN 0.5
         ELSE 0.2
       END AS predicted_accept_probability
ORDER BY predicted_accept_probability DESC, avg_quality DESC;
```

### Data Synchronization

```
┌─────────────┐    Events     ┌──────────────────┐
│ PostgreSQL  │──────────────>│  Sync Service    │
│ (writes)    │  (CDC / WAL)  │  (transformer)   │
│             │               │                  │
│ - User      │               │  Enriches with:  │
│   created   │               │  - ORCID API     │
│ - Manuscript│               │  - CrossRef API  │
│   submitted │               │  - Semantic      │
│ - Review    │               │    Scholar API   │
│   completed │               │  - Embedding     │
│             │               │    service       │
└─────────────┘               └────────┬─────────┘
                                       │
                                       │ Cypher writes
                                       ▼
                              ┌──────────────────┐
                              │     Neo4j        │
                              │  Knowledge Graph │
                              │                  │
                              │  - Researcher    │
                              │    nodes         │
                              │  - Publication   │
                              │    nodes         │
                              │  - Relationships │
                              │  - Embeddings    │
                              └──────────────────┘
```

Synchronization rules:
1. When a user registers or links ORCID, the sync service creates/updates a `Researcher` node and fetches their publication history via the ORCID Member API and CrossRef.
2. When a manuscript is submitted, the sync service creates a `Manuscript` node, computes its semantic embedding, and creates `COVERS_TOPIC` and `SIMILAR_TO` relationships.
3. When a review is completed, the sync service creates a `REVIEWED` relationship and updates the reviewer's `availability_score`.
4. A nightly batch job refreshes publication data from CrossRef/Semantic Scholar and recomputes co-authorship relationships and conflict-of-interest edges.

---

## Pros and Cons

### Pros

- **Reviewer matching is a graph problem.** Finding a qualified reviewer who has expertise in topic X, has no co-authorship with any of the manuscript's authors within 3 years, is not at the same institution, has published in related venues, is not overloaded, and has a good review quality history — this query is natural and efficient in a graph database. In SQL, it requires 6+ table joins with subqueries and becomes increasingly expensive as the researcher network grows.
- **Conflict detection at scale.** Detecting indirect conflicts (e.g. "the reviewer's PhD advisor is a co-author on this paper") requires traversing co-authorship and institutional networks. Graph traversals handle this in milliseconds; SQL recursive CTEs become impractical beyond 2-3 hops.
- **Knowledge graph enrichment.** The Neo4j graph can be continuously enriched from external sources (ORCID, CrossRef, Semantic Scholar, OpenAlex) without affecting the transactional schema. New publication data, citation links, and institutional affiliations expand the graph organically.
- **Semantic embeddings are native.** Neo4j's vector index support (since Neo4j 5.11) enables storing and querying semantic embeddings directly on nodes. Finding "publications most similar to this manuscript" by cosine similarity over embeddings is a built-in operation.
- **Network analytics.** Graph algorithms (PageRank, community detection, betweenness centrality) can identify influential reviewers, research communities, and emerging topic clusters. This powers features like "international reviewer network building" from the project backlog.
- **Clean separation of concerns.** The PostgreSQL layer handles ACID transactional workflows (submit, assign, decide) with strong consistency. The Neo4j layer handles analytical queries (match, discover, analyze) where eventual consistency is acceptable. Each database is used for what it does best.

### Cons

- **Operational complexity.** Running two database engines doubles the deployment, backup, monitoring, and upgrade burden. Self-hosted deployments (a key target market for smaller journals) must manage both PostgreSQL and Neo4j instances.
- **Data synchronization overhead.** Keeping the PostgreSQL and Neo4j layers in sync requires a reliable change data capture (CDC) pipeline. If the sync service fails silently, the graph becomes stale and reviewer matching degrades. Building and monitoring this sync pipeline is non-trivial.
- **Consistency trade-offs.** The graph is eventually consistent with the transactional layer. A reviewer who just completed a review in PostgreSQL may still show as "available" in Neo4j until the sync processes. This lag must be managed at the application layer.
- **Neo4j licensing.** Neo4j Community Edition is open source (GPLv3), but some enterprise features (clustering, role-based access control, advanced monitoring) require Neo4j Enterprise Edition, which is commercially licensed. This affects the open-source positioning of the platform.
- **Team skill diversity.** The team must be proficient in both SQL and Cypher, and understand both relational and graph data modeling. This narrows the hiring pool and increases onboarding time.
- **Graph cold-start problem.** When the system launches, the knowledge graph is empty. Reviewer matching quality depends on having a rich graph of researchers, publications, and relationships. Building this graph requires either bulk-importing data from external sources (CrossRef, OpenAlex) or waiting for organic growth as researchers register and link their ORCID profiles.
- **Cost at scale.** Neo4j's performance depends heavily on having the working set in memory. For a graph with millions of publications and researcher nodes, this requires significant RAM (32GB+ for production graphs). This is more expensive than the equivalent PostgreSQL deployment.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Transactional DB** | PostgreSQL 16+ |
| **Graph DB** | Neo4j 5.x Community Edition (or AuraDB cloud for managed deployment) |
| **Sync Pipeline** | Debezium (PostgreSQL CDC) -> Kafka -> Neo4j Sink Connector; or custom sync service |
| **Embedding Service** | OpenAI text-embedding-3-small or open-source (nomic-embed-text) for manuscript/publication embeddings |
| **External Data** | OpenAlex API (free, comprehensive) for publication data enrichment; ORCID Member API for researcher profiles |
| **Application Framework** | TypeScript/Python backend with `pg` + `neo4j-driver` libraries |
| **API** | REST for workflow operations (PostgreSQL); GraphQL for graph queries (Neo4j) |
| **Search** | Neo4j full-text indexes for researcher/publication search; PostgreSQL tsvector for manuscript search |

---

## Migration and Scaling Considerations

- **Start with PostgreSQL only.** The graph layer is additive. Build the core workflow (Suggestion 1 or 3) first, then add Neo4j when AI reviewer matching becomes the priority. Store `graph_node_id` columns on users and manuscripts from day one to make the connection seamless.
- **Bootstrap the graph from OpenAlex.** OpenAlex (https://openalex.org/) provides a free, open dataset of 250M+ scholarly works, 100M+ authors, and their relationships. Bulk-import relevant slices of this dataset to bootstrap the knowledge graph before the first user registers. This solves the cold-start problem.
- **Embedding pipeline.** Build an asynchronous embedding pipeline that processes new manuscripts and publications through a text embedding model (e.g. `text-embedding-3-small`) and stores the resulting vectors on Neo4j nodes. Use Neo4j's vector similarity index for nearest-neighbor queries.
- **Graph partitioning by discipline.** For very large graphs (10M+ nodes), partition the graph by broad discipline (life sciences, physical sciences, social sciences) using Neo4j fabric or separate database instances. Most reviewer matching queries operate within a single discipline.
- **Fallback for self-hosted.** For smaller journals that cannot run Neo4j, provide a fallback reviewer matching algorithm that uses PostgreSQL full-text search and keyword matching. The graph layer enhances but should not be required for basic operation.
- **GDPR in the graph.** Researcher nodes contain PII (name, affiliation, ORCID). GDPR erasure requests must delete or anonymize both the PostgreSQL user record and the Neo4j Researcher node and its relationships. The sync service must handle bidirectional deletion.
- **Neo4j Community vs Enterprise.** Start with Community Edition. If clustering and advanced RBAC become necessary, evaluate AuraDB (managed cloud) as an alternative to self-hosting Enterprise Edition. AuraDB pricing is based on graph size and compute, making it predictable for budgeting.
