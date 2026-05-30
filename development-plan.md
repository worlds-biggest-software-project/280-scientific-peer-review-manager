# Scientific Peer Review Manager — Phased Development Plan

> Project: 280-scientific-peer-review-manager · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased implementation. The data model follows **Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** as the primary architecture — chosen because it keeps self-hosted deployment to a single database engine while accommodating variable review forms, journal config, and AI result blobs without constant migrations. **Suggestion 4 (Neo4j knowledge graph)** is incorporated as an additive, optional layer in Phase 8 for AI semantic reviewer matching, with a PostgreSQL-only fallback so the graph is never required for basic operation.

---

## Core Requirements (synthesis)

**What it does.** An AI-native, open-source peer review and editorial workflow platform for scholarly journals. It manages manuscript submission and tracking, reviewer invitation and review collection, editorial decisions and revisions, integrity checking (plagiarism/image), and standards-compliant identity and metadata (ORCID, CrossRef/DOI, JATS). The differentiating layer is AI: semantic reviewer matching, automated desk-rejection scoring, review quality monitoring, and reviewer-report synthesis for editors.

**Who uses it.** Authors (submit, revise, track), Reviewers (accept/decline, submit reviews), Editors / Handling Editors (assign reviewers, decide), Managing Editors / Editors-in-Chief (oversee queues, configure journal), Research Integrity Officers (monitor integrity flags), and Admins (multi-journal/tenant configuration).

**Key differentiators.** Open-source and self-hostable on a single PostgreSQL engine; AI reviewer matching and pre-screening built in (not a paid add-on); integrity checking integrated end-to-end; standards-first (ORCID, CrossRef peer-review deposit, JATS4R, ANSI/NISO Z39.106 review terminology).

**Deployment model.** Self-hosted (Docker Compose, single Postgres) and cloud/SaaS multi-tenant. Shared-schema multi-tenancy via `journal_id` on tenant-scoped tables.

**Integration surface.** ORCID Member API (OAuth 2.0), CrossRef REST + deposit, ROR, Turnitin Core API (iThenticate 2.0), Proofig (image integrity), OpenAlex/Semantic Scholar (publication enrichment), SMTP/email, LLM provider (embeddings + chat), optional Neo4j.

**Standards compliance.** ORCID/ISO 27729 (incl. ISO/IEC 7064 MOD 11-2 checksum), Dublin Core (ISO 15836), JATS 1.4 + JATS4R peer-review tagging, ANSI/NISO Z39.106-2023 review terminology, OAI-PMH v2.0, OAuth 2.0 (RFC 6749) + OpenID Connect, RFC 7807 problem details, OpenAPI 3.1, GDPR, OWASP Top 10, CrossRef peer-review deposit schema.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | Project is API + web-UI heavy with many REST integrations (ORCID, CrossRef, Turnitin); a single language across backend and frontend reduces context-switching and shares DTO/validation types. AI work is API-call-driven (no local model training), so Python's ML edge is not needed. |
| API framework | Fastify 5 + `@fastify/swagger` | High throughput, first-class JSON Schema validation that doubles as the OpenAPI 3.1 source, and native RFC 7807 error shaping via a custom error handler. |
| Schema validation / types | Zod + `zod-to-json-schema` | Single source of truth for request/response DTOs, JSONB blob shapes, and OpenAPI generation; runtime validation guards JSONB columns against garbage data (the key risk of the hybrid model). |
| Database | PostgreSQL 16 (JSONB, GIN, generated `tsvector`, `pgvector` extension) | Implements Suggestion 3 in a single engine: relational integrity for core entities, JSONB for variable data, built-in full-text search, and `pgvector` for embedding-based reviewer matching without requiring Neo4j. |
| ORM / query | Drizzle ORM | Typed schema-as-code, first-class JSONB and SQL escape hatch (raw SQL for JSONPath/GIN queries the ORM cannot express), generates migrations from the schema. |
| Migrations | drizzle-kit (versioned SQL) | Version-controlled SQL migrations; CI gate rejects unsafe down-incompatible changes. |
| Task queue | BullMQ on Redis | Async workloads (integrity-check polling, email send, embedding generation, AI scoring, OpenAlex enrichment) need retries, backoff, and scheduling. Redis is already present for sessions/cache. |
| Cache / sessions | Redis 7 | Session store, rate limiting, reviewer-availability lookups, queue backend. |
| Frontend | Next.js 15 (App Router) + React 19 + shadcn/ui + Tailwind | Server-rendered editor/reviewer/author dashboards with role-gated routes; talks to the Fastify API. shadcn for accessible components (OWASP/a11y baseline). |
| PDF annotation | PDF.js + a thin annotation overlay persisted as JSONB | In-browser PDF rendering with editor notes/highlights stored as structured annotations linked to a file + version. |
| AI / LLM | Vercel AI SDK over a configurable provider (OpenAI-compatible) + `text-embedding-3-small` | Provider-agnostic chat + embeddings for pre-screening, review-quality scoring, decision synthesis, and reviewer-match embeddings. Model name + base URL are env-configurable for self-hosters using local models. |
| Object storage | S3-compatible (MinIO for self-host, AWS S3 for cloud) | Manuscript files referenced by `storage_path` in `manuscript_files`; MinIO keeps self-hosting single-stack. |
| Auth | Lucia-style session + OAuth 2.0/OIDC for ORCID & institutional SSO | Email/password with Argon2id for local accounts; OAuth for authenticated ORCID iD collection and SSO. |
| Email | Nodemailer (SMTP) + MJML templates | Notification system; templates rendered per-journal from `journals.config.email_templates`. |
| Graph (optional) | Neo4j 5 Community, additive in Phase 8 | Enhances reviewer matching/conflict traversal; PostgreSQL+pgvector fallback ensures it is never required. |
| Testing | Vitest (unit/integration) + Playwright (E2E) + Testcontainers (real Postgres/Redis) | Fast unit runs, real-dependency integration via ephemeral containers, browser E2E for workflows. |
| Code quality | ESLint + Prettier + `tsc --noEmit` (strict) | Lint, format, and full type checking gate every phase. |
| Package manager | pnpm (workspaces) | Monorepo: `apps/api`, `apps/web`, `packages/shared`, `packages/db`. |
| Containerisation | Docker + docker-compose | One-command self-hosted stack: api, web, postgres, redis, minio (+ optional neo4j). |

### Project Structure

```
scientific-peer-review-manager/
├── package.json                     # pnpm workspace root
├── pnpm-workspace.yaml
├── docker-compose.yml               # postgres, redis, minio, api, web (+ optional neo4j)
├── Dockerfile.api
├── Dockerfile.web
├── .env.example
├── openapi.json                     # generated artifact (CI-checked)
├── packages/
│   ├── shared/                      # cross-cutting types, Zod schemas, controlled vocabularies
│   │   └── src/
│   │       ├── schemas/             # Zod DTOs + JSONB blob schemas (review_form, ai_analysis, ...)
│   │       ├── vocab/               # NISO Z39.106 review models, decision/status enums, CRediT roles
│   │       └── orcid/               # ISO 7064 MOD 11-2 checksum, ORCID id format
│   └── db/                          # Drizzle schema + migrations + seed
│       └── src/
│           ├── schema.ts            # all tables
│           ├── migrations/
│           └── seed.ts
├── apps/
│   ├── api/
│   │   └── src/
│   │       ├── server.ts            # Fastify bootstrap, swagger, error handler (RFC 7807)
│   │       ├── plugins/             # auth, db, redis, rate-limit, audit
│   │       ├── routes/              # versioned under /api/v1/*
│   │       │   ├── manuscripts/
│   │       │   ├── reviews/
│   │       │   ├── decisions/
│   │       │   ├── users/
│   │       │   ├── journals/
│   │       │   ├── integrity/
│   │       │   ├── matching/
│   │       │   └── oai/             # OAI-PMH v2.0 endpoint
│   │       ├── services/            # business logic (workflow state machine, matching, AI)
│   │       │   ├── workflow/
│   │       │   ├── matching/
│   │       │   ├── ai/
│   │       │   ├── integrity/
│   │       │   ├── export/          # JATS, CrossRef deposit, Dublin Core
│   │       │   └── notifications/
│   │       ├── integrations/        # orcid, crossref, ror, turnitin, proofig, openalex, email
│   │       ├── workers/             # BullMQ processors
│   │       └── lib/                 # storage (S3), embeddings, audit
│   └── web/
│       └── src/
│           ├── app/                 # Next.js App Router (role-gated route groups)
│           │   ├── (author)/
│           │   ├── (reviewer)/
│           │   ├── (editor)/
│           │   └── (admin)/
│           ├── components/
│           └── lib/api-client.ts    # typed client generated from openapi.json
└── tests/
    ├── fixtures/                    # sample PDFs, JATS XML, ORCID/CrossRef API responses
    ├── integration/
    └── e2e/
```

Grouping is by concern (routes / services / integrations / workers), not by phase, so later phases add files without restructuring.

---

## Phase 1: Foundation — Monorepo, Database, Auth, API Skeleton

### Purpose
Establish the runnable skeleton: pnpm monorepo, Dockerized Postgres/Redis/MinIO, the Drizzle schema for core entities, a Fastify server with OpenAPI 3.1 generation and RFC 7807 errors, session-based auth with role checks, and the audit-log plumbing. After this phase, a developer can register, log in, and the system records every action. Nothing journal-specific yet, but every later phase plugs into these primitives.

### Tasks

#### 1.1 — Monorepo & container stack

**What**: Scaffold the pnpm workspace and a docker-compose stack that boots api + web + postgres + redis + minio.

**Design**:
- `pnpm-workspace.yaml` includes `apps/*` and `packages/*`.
- `docker-compose.yml` services: `postgres` (16, with `pgvector` image `pgvector/pgvector:pg16`), `redis` (7), `minio`, `api`, `web`. Healthchecks gate `api` on `postgres` + `redis`.
- `.env.example` keys: `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET`, `SESSION_SECRET`, `BASE_URL`, `LLM_BASE_URL`, `LLM_API_KEY`, `LLM_CHAT_MODEL`, `LLM_EMBED_MODEL`, `ORCID_CLIENT_ID`, `ORCID_CLIENT_SECRET`, `CROSSREF_DEPOSIT_USER`, `TURNITIN_BASE_URL`, `TURNITIN_API_KEY`, `SMTP_URL`, `NEO4J_URL` (optional).
- Config loader in `packages/shared/src/config.ts`: Zod-validated `env` object; process exits with a listed-missing-keys message on invalid config.

**Testing**:
- Unit: `config` parse with all required keys present → typed config object.
- Unit: missing `DATABASE_URL` → throws with message naming `DATABASE_URL`.
- Integration: `docker compose up` → `GET /health` returns `{status:"ok", db:true, redis:true}`.

#### 1.2 — Core database schema (relational + JSONB)

**What**: Implement the Drizzle schema for identity, journals, manuscripts, files, review workflow, decisions, integrity, audit, and notifications per Suggestion 3.

**Design**: Tables (UUID PKs, `gen_random_uuid()`): `users`, `journals`, `user_roles`, `subject_areas`, `reviewer_profiles`, `manuscripts`, `manuscript_authors`, `manuscript_versions`, `manuscript_files`, `review_rounds`, `review_forms`, `review_assignments`, `editorial_decisions`, `integrity_checks`, `conflicts_of_interest`, `activity_log`, `notifications`. Adopt the exact column set and JSONB shapes from `data-model-suggestion-3.md`. Add from day one (per Suggestion 4 migration note) nullable `graph_node_id VARCHAR(50)` on `users` and `manuscripts`, and a `vector(1536)` embedding column on `manuscripts` (pgvector) plus `manuscripts.search_vector` generated `tsvector` with GIN index.

JSONB blob shapes are defined as Zod schemas in `packages/shared/src/schemas` and validated at the service layer before write:
```ts
// journal config
export const JournalConfig = z.object({
  article_types: z.array(z.string()).default(['original-research']),
  required_metadata_fields: z.array(z.string()).default([]),
  max_reviewers_per_round: z.number().int().default(3),
  default_review_deadline_days: z.number().int().default(21),
  desk_rejection_enabled: z.boolean().default(false),
  review_model: z.enum(['open','single-anonymous','double-anonymous']).default('double-anonymous'),
  open_review_options: z.object({ open_identities: z.boolean(), open_reports: z.boolean() }).default({open_identities:false, open_reports:false}),
  integrity_checks: z.array(z.enum(['plagiarism','image_integrity'])).default([]),
  email_templates: z.record(z.string()).default({}),
});
```
Controlled vocabularies in `packages/shared/src/vocab`: manuscript `status`, decision/recommendation enums, and NISO Z39.106 `review_model` values.

**Testing**:
- Integration (Testcontainers Postgres): run migrations → all tables and indexes exist (`pg_indexes` query).
- Unit: `JournalConfig.parse({})` → all defaults applied.
- Unit: `JournalConfig.parse({review_model:'invalid'})` → ZodError.
- Integration: insert manuscript with valid `metadata` JSONB and query `WHERE metadata @> '{"article_type":"review"}'` via GIN index returns the row.

#### 1.3 — Auth, roles, and session middleware

**What**: Email/password auth (Argon2id), Redis-backed sessions, and per-journal role authorization.

**Design**:
- `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `POST /api/v1/auth/logout`, `GET /api/v1/auth/me`.
- Session cookie (HttpOnly, SameSite=Lax, Secure in prod) → Redis session record `{userId, createdAt}`.
- Authorization plugin: `requireRole(journalIdParam, roles[])` reads `user_roles` for the user+journal. Roles: `author`, `reviewer`, `editor`, `managing_editor`, `editor_in_chief`, `admin`. Returns RFC 7807 `403` on failure.
- Reviewers/authors are implicit (any authenticated user may submit or be invited); editorial roles are explicit grants.

**Testing**:
- Integration: register → login → `me` returns user without `password_hash`.
- Integration: wrong password → 401 RFC 7807 body `{type, title, status:401}`.
- Unit: `requireRole` with editor role present → passes; absent → 403.
- Security: password stored as Argon2id hash, never returned in any response (assert across endpoints).

#### 1.4 — RFC 7807 errors, OpenAPI 3.1, and audit log

**What**: Global error handler emitting `application/problem+json`, OpenAPI generation from Zod schemas, and an audit hook recording mutating actions.

**Design**:
- Fastify `setErrorHandler` maps thrown `AppError(status, title, detail)` and ZodErrors → RFC 7807 JSON.
- `@fastify/swagger` produces `/openapi.json` (OAS 3.1); CI step regenerates and `git diff --exit-code`s it.
- `auditPlugin`: an `onResponse` hook on mutating routes writes `activity_log {manuscript_id?, actor_id, action, detail JSONB, ip_address}`.

**Testing**:
- Unit: ZodError → 422 problem+json with `errors` array of field paths.
- Integration: any mutating call writes exactly one `activity_log` row with correct `actor_id` and `ip_address`.
- Integration: `/openapi.json` validates against the OAS 3.1 meta-schema.

---

## Phase 2: Journals, Users, and Standards-Compliant Identity

### Purpose
Make the platform multi-journal and standards-aware. Journals can be created and configured; users can link an ORCID iD (validated with the ISO/IEC 7064 checksum) and resolve their institution via ROR. This phase delivers the identity backbone that reviewer matching, conflict detection, and metadata export all depend on.

### Tasks

#### 2.1 — Journal CRUD and configuration

**What**: Endpoints to create/read/update journals and assign editorial roles.

**Design**:
- `POST /api/v1/journals` (admin), `GET /api/v1/journals/:id`, `PATCH /api/v1/journals/:id` (editor_in_chief|admin), `POST /api/v1/journals/:id/roles` (assign role to user).
- `config` validated by `JournalConfig` (Phase 1.2) on write.
- Subject taxonomy: `POST /api/v1/journals/:id/subjects` seeds `subject_areas` (hierarchical via `parent_id`).

**Testing**:
- Integration: create journal with partial config → defaults filled; read back equals merged config.
- Integration: non-admin create → 403.
- Unit: invalid `max_reviewers_per_round: -1` → ZodError.

#### 2.2 — ORCID format & ISO/IEC 7064 checksum

**What**: Validate and normalize ORCID iDs.

**Design**: `packages/shared/src/orcid/checksum.ts`:
```ts
export function isValidOrcid(id: string): boolean // format 0000-0000-0000-000X, MOD 11-2 check digit
export function normalizeOrcid(input: string): string // strip URL prefix, hyphenate
```
MOD 11-2 (ISO/IEC 7064): iterate first 15 digits, `total = (total + digit) * 2`, `remainder = total % 11`, `result = (12 - remainder) % 11`, check digit = `result === 10 ? 'X' : String(result)`.

**Testing**:
- Unit: `isValidOrcid('0000-0002-1825-0097')` → true (known-valid public ORCID).
- Unit: corrupted check digit → false.
- Unit: `normalizeOrcid('https://orcid.org/0000-0002-1825-0097')` → `'0000-0002-1825-0097'`.

#### 2.3 — ORCID OAuth iD collection & institution (ROR)

**What**: OAuth 2.0 `/authenticate` flow to collect an authenticated ORCID iD; ROR lookup for affiliation.

**Design**:
- `GET /api/v1/auth/orcid/start` → redirect to ORCID authorize URL (scope `/authenticate`).
- `GET /api/v1/auth/orcid/callback` → exchange code, store `orcid_id` on user, write `ORCIDLinked` to audit.
- `integrations/ror.ts`: `searchRor(query)` and `resolveRor(rorId)` against `https://api.ror.org`; populate `users.affiliation` and the canonical ROR id in `reviewer_profiles.expertise_detail` or a `ror_id` field.
- All external HTTP wrapped in a retry-with-backoff client; ORCID/ROR mocked in tests via fixtures.

**Testing**:
- Integration (mocked ORCID): callback with valid code → user.orcid_id set, validated by `isValidOrcid`.
- Integration (mocked ORCID): callback with mismatched `state` → 400, no write.
- Integration (mocked ROR): `searchRor('MIT')` returns canonical ROR id and name.

---

## Phase 3: Manuscript Submission & File Handling

### Purpose
Deliver the heart of the author experience and the first end-to-end workflow: an author creates a submission, adds authors and metadata, uploads files to object storage, and submits. This is the entry point for everything downstream (review, integrity, decisions).

### Tasks

#### 3.1 — Submission lifecycle & manuscript numbers

**What**: Create draft manuscripts, attach authors/metadata, and submit.

**Design**:
- `POST /api/v1/journals/:jid/manuscripts` (draft), `PATCH /manuscripts/:id`, `POST /manuscripts/:id/authors`, `POST /manuscripts/:id/submit`.
- `manuscript_number` generated `{JOURNAL_PREFIX}-{YYYY}-{seq:05d}` via a per-journal Postgres sequence to avoid race conditions.
- Submit validates: required files present, required metadata fields from `journal.config.required_metadata_fields` present, ≥1 corresponding author. On submit, status `draft → submitted`, set `submitted_at`, write audit + notification, enqueue integrity + AI pre-screen jobs (consumed in later phases).
- `manuscript_authors.contribution` accepts CRediT taxonomy JSONB.

**Testing**:
- Integration: full create → add authors → upload → submit happy path; status transitions and `submitted_at` set.
- Integration: submit missing required metadata field → 422 listing the missing field.
- Integration: concurrent submits to same journal → unique sequential manuscript numbers (no duplicates).

#### 3.2 — File upload to object storage

**What**: Upload manuscript/figure/supplement files with checksum and metadata extraction.

**Design**:
- `POST /manuscripts/:id/files` (multipart) → stream to S3 at `journals/{jid}/manuscripts/{mid}/v{n}/{uuid}-{filename}`; compute `checksum_sha256`; insert `manuscript_files` row with `file_type`, `mime_type`, `file_size_bytes`.
- Async worker extracts `file_metadata` JSONB (page_count, reference_count, latex_detected) for PDFs.
- `GET /manuscripts/:id/files/:fid` → signed URL, gated by role/ownership (manuscript confidentiality, OWASP IDOR).

**Testing**:
- Integration (MinIO via Testcontainers): upload PDF → row created, checksum matches recomputed SHA-256.
- Integration: download by unauthorized user → 403 (IDOR guard).
- Unit: `file_type` not in allowed enum → 422.

#### 3.3 — Versions & author revision management

**What**: Support revision rounds where authors submit new versions with a response-to-reviewers file.

**Design**:
- `POST /manuscripts/:id/versions` creates `manuscript_versions` (`version_number` incremented), copying carry-over metadata and recording `changes_summary` + `metadata_diff` JSONB.
- Allowed only when status is `revision_requested`; transitions `revision_requested → revised`.
- New files attach to the new `version_id`.

**Testing**:
- Integration: revision when status=`revision_requested` → v2 created, status `revised`.
- Integration: revision when status=`under_review` → 409 conflict (illegal transition).

---

## Phase 4: Workflow Engine, Reviewer Invitations & Review Collection

### Purpose
Implement the editorial workflow state machine, reviewer invitation/acceptance tracking, customizable review forms (as JSONB schema — no EAV), and structured review submission. After this phase the platform supports a complete manual peer-review cycle from editor assignment through reviewer reports.

### Tasks

#### 4.1 — Manuscript workflow state machine

**What**: Centralize all status transitions behind an explicit, audited state machine.

**Design**: `services/workflow/stateMachine.ts`:
```ts
type Status = 'draft'|'submitted'|'with_editor'|'under_review'
  |'revision_requested'|'revised'|'accepted'|'rejected'|'desk_rejected'|'withdrawn';
const TRANSITIONS: Record<Status, Status[]> = {
  draft: ['submitted','withdrawn'],
  submitted: ['with_editor','desk_rejected','withdrawn'],
  with_editor: ['under_review','desk_rejected','rejected','accepted','withdrawn'],
  under_review: ['revision_requested','accepted','rejected','withdrawn'],
  revision_requested: ['revised','withdrawn'],
  revised: ['under_review','accepted','rejected','withdrawn'],
  accepted: [], rejected: [], desk_rejected: [], withdrawn: [],
};
function transition(m: Manuscript, to: Status, actorId: string): void; // throws IllegalTransition (409)
```
Every transition writes an `activity_log` row, making state history reconstructable (addresses Suggestion 1's "implicit state" weakness without full event sourcing).

**Testing**:
- Unit: every legal transition allowed; every illegal transition throws `IllegalTransition`.
- Unit: terminal states accept no transitions.
- Integration: editor assignment moves `submitted → with_editor` and logs it.

#### 4.2 — Editor assignment & review rounds

**What**: Assign a handling editor and open a review round.

**Design**:
- `POST /manuscripts/:id/assign-editor` (managing_editor|editor_in_chief) → sets handling editor, `submitted → with_editor`.
- `POST /manuscripts/:id/rounds` → creates `review_rounds` row for the current version; opening the first round transitions `with_editor → under_review`.

**Testing**:
- Integration: assign editor then open round → round_number=1, status `under_review`.
- Integration: open second round only allowed from `revised`.

#### 4.3 — Customizable review forms (JSONB schema)

**What**: Editors define review forms as JSON Schema documents.

**Design**: `review_forms.form_schema` per Suggestion 3 (`sections[].fields[]` with `type: rating|textarea|radio|dropdown|checkbox`, `min/max`, `required`, `min_words`). A validator compiles `form_schema` and validates `review_assignments.review_data` against it on submission.

**Testing**:
- Unit: valid form_schema parses; submission matching schema validates.
- Unit: rating outside `min..max` → validation error.
- Unit: textarea below `min_words` → validation error.

#### 4.4 — Reviewer invitation & response

**What**: Invite reviewers, track accept/decline/overdue, enforce conflict and workload limits.

**Design**:
- `POST /rounds/:rid/invitations` (handling editor) → creates `review_assignments` (`status='invited'`), sets `due_date = now + journal.default_review_deadline_days`, records `match_context` if AI-suggested (Phase 8), enqueues invitation email.
- `POST /assignments/:aid/respond {accept:boolean}` → `invited → accepted|declined`.
- A scheduled BullMQ job flips past-due `accepted` assignments to `overdue` and enqueues reminders.
- Guard: cannot invite a user with an open `conflicts_of_interest` row against any manuscript author (basic check now; graph-enhanced in Phase 8).

**Testing**:
- Integration: invite → reviewer accepts → status `accepted`, invitation email enqueued.
- Integration: invite a conflicted user → 409.
- Integration: overdue sweep flips status and enqueues reminder (fake timers).

#### 4.5 — Review submission

**What**: Reviewers submit structured reviews and recommendations.

**Design**:
- `POST /assignments/:aid/review` → validate `review_data` against `form_schema`; set `recommendation`, `status='completed'`, `completed_at`. Separate `comments_to_author` vs `comments_to_editor` honored by anonymity rules from `journal.config.review_model`.
- Editors can rate review quality (`quality_metrics` editor-assigned plus AI in Phase 7).

**Testing**:
- Integration: submit valid review → completed, recommendation persisted.
- Integration: author-facing view in double-anonymous journal omits reviewer identity and `comments_to_editor`.
- Integration: submit before accepting → 409.

---

## Phase 5: Editorial Decisions, Notifications & PDF Annotation

### Purpose
Close the editorial loop: editors synthesize reviews into a decision and decision letter, the system notifies all parties, and editors can annotate manuscript PDFs. After this phase a journal can run real peer review end-to-end with humans only (AI is layered on next).

### Tasks

#### 5.1 — Editorial decision & decision letter

**What**: Record a decision per round and generate/send a decision letter.

**Design**:
- `POST /rounds/:rid/decision {decision, rationale, decision_letter}` (handling editor) → insert `editorial_decisions`; map decision to status via the state machine (`accept→accepted`, `minor/major_revision→revision_requested`, `reject→rejected`, `desk_reject→desk_rejected`); close the round; enqueue decision email. `decision_letter` is JSONB `{subject, body, sent_at}`.
- Decision letter body templated from `journal.config.email_templates.decision` with token substitution.

**Testing**:
- Integration: major_revision decision → status `revision_requested`, round closed, email enqueued.
- Integration: accept decision → `accepted`, terminal; further decisions on round → 409.

#### 5.2 — Notification & email system

**What**: In-app notifications plus SMTP email, rendered per-journal.

**Design**:
- `notifications` rows created by workflow events; `GET /notifications` (mark read).
- `services/notifications` enqueues email jobs; worker renders MJML template selected by `notification_type`, falling back to a default template. `email_sent_at` recorded.
- Notification types: `submission_received`, `editor_assigned`, `review_invitation`, `review_reminder`, `review_completed`, `decision_made`, `revision_requested`.

**Testing**:
- Integration: submit manuscript → `submission_received` notification row + email job enqueued.
- Integration (mock SMTP): worker sends email, sets `email_sent_at`.
- Unit: unknown `notification_type` → default template used (no throw).

#### 5.3 — PDF annotation & editor notes

**What**: In-browser PDF viewing with persisted annotations and private editor notes.

**Design**:
- Frontend renders manuscript PDF via PDF.js with an overlay capturing highlights/comments.
- `POST /files/:fid/annotations` stores annotations JSONB `{page, rect, type, body, author_id, created_at}` in a `file_annotations` table (file_id, version_id, JSONB array). Visibility scoped to editors (and reviewers if journal allows).

**Testing**:
- Integration: create annotation → retrievable; visible to editor, hidden from author.
- E2E (Playwright): open PDF, add highlight, reload → annotation persists.

---

## Phase 6: Integrity Checking (Plagiarism & Image)

### Purpose
Integrate end-to-end integrity checking — the feature competitors fragment across separate licenses. On submission, manuscripts are screened via Turnitin Core API (iThenticate 2.0) for similarity and Proofig for image integrity, with results surfaced to editors before reviewer assignment.

### Tasks

#### 6.1 — Integrity check orchestration

**What**: Provider-agnostic integrity-check pipeline driven by `journal.config.integrity_checks`.

**Design**:
- On submit (Phase 3.1), for each enabled check type create `integrity_checks` row (`status='pending'`) and enqueue a job.
- `integrations/turnitin.ts`: submit document → poll for similarity report → store normalized `results` JSONB (`similarity_pct`, source breakdown, `top_matches`, `report_url`), set `status='completed'`.
- `integrations/proofig.ts`: submit figures → store `results.flags[]`.
- State machine: a configurable threshold (`journal.config`) can auto-flag `desk_rejection` candidacy but never auto-rejects without editor action.

**Testing**:
- Integration (mocked Turnitin): submit → poll → completed with `similarity_pct` populated.
- Integration: provider failure → `status='failed'`, job retried with backoff, editor notified.
- Unit: results normalizer maps provider payload → canonical `results` shape.

#### 6.2 — Editor integrity dashboard surface

**What**: Expose integrity results in the editor manuscript view.

**Design**: `GET /manuscripts/:id/integrity` returns all checks with normalized results; web view shows similarity % with a journal-configurable warning threshold and image flags.

**Testing**:
- Integration: returns both plagiarism and image checks for a manuscript.
- E2E: editor view renders similarity badge above threshold in warning color.

---

## Phase 7: AI Pre-Screening, Review-Quality Monitoring & Decision Synthesis

### Purpose
Deliver the AI-native differentiators that do not require the graph: desk-rejection/scope pre-screening at submission, review-quality flagging before reports reach authors, and synthesis of multiple reviews into a structured editor summary. All AI output lands in JSONB columns (`ai_analysis`, `quality_metrics`, `ai_synthesis`) per Suggestion 3.

### Tasks

#### 7.1 — Manuscript pre-screening & desk-rejection scoring

**What**: Async AI analysis of submitted manuscripts for scope fit, methodology completeness, and statistical reporting.

**Design**:
- Worker (enqueued on submit) extracts manuscript text, calls the LLM with a structured-output prompt, and writes `manuscripts.ai_analysis`:
```json
{ "scope_fit_score": 0.0, "methodology_completeness": 0.0,
  "statistical_reporting_flags": [], "suggested_subject_areas": [],
  "desk_rejection_risk": 0.0, "language_quality_score": 0.0,
  "analyzed_at": "", "model_version": "", "_schema_version": 1 }
```
- Prompt template (system): instructs the model to act as a managing editor screening for journal fit against `journal.config` scope text, returning strict JSON matching a Zod schema; response validated with `zod` (reject + retry on parse failure).
- `GET /manuscripts/:id` surfaces `ai_analysis` to editors only; `desk_rejection_risk` shown as an advisory score (never auto-rejects).

**Testing**:
- Integration (mocked LLM): submit → `ai_analysis` populated and schema-valid.
- Unit: malformed model JSON → parse retry, then `status` flag without corrupting the column.
- Integration: `ai_analysis` not exposed to author role.

#### 7.2 — Review-quality monitoring

**What**: Flag low-quality/boilerplate reviews before authors see them.

**Design**: On review submission, enqueue a quality job computing `quality_metrics` JSONB (`word_count`, `specificity_score`, `constructiveness_score`, `boilerplate_score`, `assessed_at`). Mix of cheap heuristics (word count, sentence length) and an LLM rubric call. Editor view flags reviews below journal-configurable thresholds; editor decides whether to return to reviewer.

**Testing**:
- Integration (mocked LLM): submit review → `quality_metrics` populated.
- Unit: boilerplate sample → high `boilerplate_score`; substantive sample → low.
- Integration: flagged review surfaces a warning to the editor only.

#### 7.3 — Reviewer-report synthesis for editors

**What**: Synthesize all completed reviews in a round into a structured consensus/disagreement summary.

**Design**: `POST /rounds/:rid/synthesize` (editor) → LLM call over all `review_data` → writes `editorial_decisions.ai_synthesis` shape (`consensus_summary`, `disagreement_points[]` across novelty/methodology/presentation/significance, `recommended_action`, `confidence`, `key_revision_items[]`). Used as a drafting aid for the decision; editor edits before finalizing.

**Testing**:
- Integration (mocked LLM): two conflicting reviews → `disagreement_points` includes the conflicting dimension.
- Unit: synthesis JSON validated against Zod schema; invalid → retry.
- Integration: synthesis available only after ≥1 completed review in the round.

---

## Phase 8: AI Reviewer Matching, Conflict Detection & Workload Optimization

### Purpose
Implement the flagship differentiator — semantic reviewer matching — using `pgvector` embeddings by default and an optional Neo4j knowledge graph (Suggestion 4) for richer co-authorship/conflict traversal. Includes conflict-of-interest detection and reviewer workload-aware ranking. The system enriches researcher profiles from OpenAlex/ORCID to solve the cold-start problem.

### Tasks

#### 8.1 — Embeddings & publication enrichment

**What**: Build researcher and manuscript embeddings and enrich reviewer profiles from external sources.

**Design**:
- On manuscript submit, embed title+abstract → `manuscripts.embedding` (pgvector).
- On ORCID link, enqueue enrichment: fetch publications via OpenAlex/CrossRef, embed the researcher's recent-work profile → store in `reviewer_profiles.expertise_detail` (`publications_count`, `h_index`, `recent_topics`, embedding ref) and a `reviewer_profiles.embedding vector(1536)` column.
- Nightly BullMQ job refreshes enrichment and recomputes `availability` from open assignment counts.

**Testing**:
- Integration (mocked OpenAlex): ORCID link → profile enriched with publication count and topics.
- Integration: manuscript embedding stored; nearest-neighbor query returns it.

#### 8.2 — pgvector semantic matching & ranking

**What**: Recommend reviewers by composite score without requiring Neo4j.

**Design**: `POST /manuscripts/:id/reviewer-suggestions` →
1. Candidate set: reviewers with `reviewer_profiles.embedding` near the manuscript embedding (`<=>` cosine, pgvector), filtered to subject overlap.
2. Exclude authors and conflicted users (8.3).
3. Composite score `0.4*semantic + 0.4*expertise + 0.2*availability` (mirrors Suggestion 4's weighting), with `predicted_accept_probability` from historical accept rate and current load.
4. Returns ranked list with explainable `match_context` per candidate (stored on `review_assignments.match_context` when invited).

**Testing**:
- Integration: suggestions ranked by composite score descending; authors excluded.
- Integration: overloaded reviewer (availability low) ranked below an available equal-expertise reviewer.
- Unit: composite-score function deterministic for fixed inputs.

#### 8.3 — Conflict-of-interest detection

**What**: Detect declared, co-authorship, and institutional conflicts.

**Design**:
- Baseline (Postgres): co-authorship from shared `manuscript_authors`/enriched co-author lists, shared `ror_id`, and declared `conflicts_of_interest` rows; populate `conflicts_of_interest.evidence` JSONB.
- Optional graph: if `NEO4J_URL` set, run the Cypher conflict query from Suggestion 4 (multi-hop co-author/advisor-student traversal) for richer detection; results merged with baseline.
- Matching (8.2) consumes detected conflicts to exclude candidates.

**Testing**:
- Integration: candidate co-authored a paper with an author within 3 years → flagged with evidence.
- Integration: same-institution candidate → institutional conflict flagged.
- Integration (optional, skipped if no Neo4j): 2-hop advisor-student conflict detected via graph.

#### 8.4 — Optional Neo4j sync layer

**What**: Additive graph sync with a hard fallback.

**Design**: When `NEO4J_URL` is configured, a sync worker mirrors `Researcher`/`Manuscript` nodes and `CO_AUTHORED`/`AFFILIATED_WITH`/`HAS_EXPERTISE`/`REVIEWED` edges on the relevant events (Suggestion 4 sync rules). `graph_node_id` columns (added in Phase 1.2) link rows to nodes. All matching/conflict code checks for graph availability and falls back to pgvector+SQL otherwise.

**Testing**:
- Integration (optional, Testcontainers Neo4j): submit + ORCID link → nodes and edges created.
- Integration: with Neo4j disabled, matching and conflict detection still function (fallback path).

---

## Phase 9: Standards Export — JATS, CrossRef Deposit, OAI-PMH, DOI

### Purpose
Make the platform interoperable with the scholarly ecosystem: export manuscripts and peer-review materials as JATS 1.4 (JATS4R peer-review tagging), deposit DOIs and peer-review records to CrossRef, and expose harvestable metadata via OAI-PMH with Dublin Core. This is what lets publishers actually adopt the system.

### Tasks

#### 9.1 — JATS 1.4 + JATS4R export

**What**: Export a manuscript and its review materials as JATS XML.

**Design**: `GET /manuscripts/:id/export/jats` → build `<article>` with `<front>` metadata (title, contributors with ORCID, affiliations with ROR, abstract, Dublin Core-aligned fields) and, per JATS4R peer-review guidance, `<sub-article article-type="referee-report">`, `decision-letter`, and `author-comment` elements from review/decision data. Output validated against the JATS 1.4 schema.

**Testing**:
- Fixture-based: export sample manuscript → XML validates against JATS 1.4 XSD.
- Unit: contributor with ORCID → `<contrib-id contrib-id-type="orcid">` present.
- Unit: completed review → `referee-report` sub-article present.

#### 9.2 — CrossRef DOI assignment & peer-review deposit

**What**: Register article DOIs and deposit peer-review records.

**Design**:
- On `accepted`, enqueue CrossRef deposit job: build CrossRef deposit XML, POST to member deposit API, store returned `doi` on `manuscripts`.
- Optional peer-review deposit (when journal runs open review): deposit referee reports/decision letters linked by DOI relationship (CrossRef peer-reviews schema).

**Testing**:
- Integration (mocked CrossRef): accept → deposit submitted, `doi` stored.
- Unit: deposit XML matches CrossRef schema for required fields.
- Integration: deposit failure → retried, editor notified, status surfaced.

#### 9.3 — OAI-PMH v2.0 endpoint

**What**: Expose published metadata for harvesting.

**Design**: `GET /oai?verb=...` implementing `Identify`, `ListRecords`, `ListMetadataFormats`, `GetRecord` with `oai_dc` (Dublin Core, ISO 15836) mandatory and `jats` optional. Only `accepted`/published manuscripts exposed.

**Testing**:
- Integration: `verb=Identify` → valid OAI-PMH XML.
- Integration: `verb=ListRecords&metadataPrefix=oai_dc` → Dublin Core records for accepted manuscripts only.
- Integration: unsupported verb → OAI `badVerb` error response.

---

## Phase 10: Dashboards, Analytics, GDPR & Hardening

### Purpose
Complete the product for production: role-specific dashboards, reviewer performance analytics, GDPR data-subject tooling, and a security/OWASP hardening pass. After this phase the system is deployable for a real journal.

### Tasks

#### 10.1 — Role dashboards & queues

**What**: Author, reviewer, editor, and managing-editor dashboards.

**Design**: Server-rendered Next.js views backed by query endpoints: editor queue (`days_since_submission`, reviews completed/pending, attention flag, priority score), reviewer dashboard (assignments + due dates), author tracking (status timeline reconstructed from `activity_log`). These mirror the read-projections of Suggestion 2 but are computed as SQL views/queries rather than maintained projections.

**Testing**:
- Integration: editor queue orders by priority and flags overdue-review manuscripts.
- E2E: author sees accurate status timeline after a decision.

#### 10.2 — Reviewer performance metrics & analytics

**What**: Aggregate reviewer metrics and journal analytics.

**Design**: `GET /journals/:id/analytics/reviewers` → per-reviewer `total_invitations/accepted/completed/declined`, `avg_days_to_respond/complete`, `avg_quality_score`, `active_assignments`, computed from `review_assignments` + `quality_metrics`. Journal-level: acceptance rate, time-to-first-decision, desk-rejection rate.

**Testing**:
- Integration: metrics match a seeded fixture of assignments/reviews.
- Unit: avg-days calculations handle null `completed_at` (excluded).

#### 10.3 — GDPR data subject tooling

**What**: Data export (portability) and erasure/anonymization.

**Design**:
- `GET /users/:id/gdpr-export` (self|admin) → JSON bundle of all personal data across typed columns and JSONB (PII catalog drives the traversal).
- `POST /users/:id/gdpr-erase` → anonymize `users` PII, pseudonymize authorship/review records while preserving editorial integrity (COPE audit needs), redact PII from JSONB and `activity_log`. Documents the legal-basis tension with double-anonymous review noted in standards.md.

**Testing**:
- Integration: export includes PII from `users`, `manuscript_authors`, and JSONB `metadata`.
- Integration: erase removes/pseudonymizes name + email everywhere; manuscript records remain referentially intact.

#### 10.4 — Security & OWASP hardening

**What**: Apply the OWASP Top 10 baseline.

**Design**: Rate limiting (`@fastify/rate-limit`), strict ownership checks on all file/manuscript routes (IDOR), input validation via Zod everywhere, parameterized queries only, security headers (`@fastify/helmet`), session fixation protection, audit coverage assertion, and dependency scanning in CI. Confidentiality: reviewers/authors can never read another submission's files.

**Testing**:
- Integration: IDOR attempt on every `/:id` resource → 403.
- Integration: rate limit exceeded → 429 RFC 7807.
- Security: automated check that no endpoint returns `password_hash` or another tenant's data.
- E2E: double-anonymous flow never leaks reviewer identity to author across the full cycle.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, auth, OpenAPI, audit)   ─── required by everything
    │
Phase 2: Journals & Standards Identity (ORCID, ROR)        ─── requires 1
    │
Phase 3: Submission & Files                                ─── requires 2
    │
Phase 4: Workflow, Invitations, Reviews                    ─── requires 3
    │
Phase 5: Decisions, Notifications, PDF Annotation          ─── requires 4
    │
    ├── Phase 6: Integrity Checking          ─── requires 3 (can parallel with 5)
    ├── Phase 7: AI Screening/Quality/Synthesis ─── requires 4,5 (7.1 needs only 3)
    └── Phase 9: Standards Export (JATS/CrossRef/OAI) ─── requires 5 (can parallel with 6/7)
         │
Phase 8: AI Reviewer Matching & Conflicts    ─── requires 2,4 (+ optional Neo4j)
    │
Phase 10: Dashboards, Analytics, GDPR, Hardening ─── requires all prior
```

Parallelism opportunities:
- After Phase 5, **Phases 6, 7, and 9** can be developed concurrently (independent service areas).
- **Phase 8** depends only on Phases 2 and 4 and may start in parallel with Phase 5 once the schema and review flow exist.
- Task **7.1** (manuscript pre-screening) needs only Phase 3 and can begin early.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`pnpm test`), including Testcontainers-backed real-Postgres/Redis tests.
3. ESLint and Prettier pass with zero errors (`pnpm lint`).
4. Type checking passes with strict mode (`pnpm tsc --noEmit`).
5. `docker compose build` succeeds and the stack boots with healthy `api`/`web`.
6. The phase's primary capability works end-to-end (demonstrated by an integration or Playwright E2E test).
7. New configuration keys are documented in `.env.example` and the JSONB blob shapes have Zod schemas in `packages/shared`.
8. New/changed API endpoints appear in the regenerated `openapi.json` (CI `git diff --exit-code` passes).
9. Database migrations are generated by drizzle-kit, checked in, and run cleanly forward on an empty database.
10. New external integrations have both a mocked-integration test and a documented (optionally skipped) real-dependency test.
11. Every mutating endpoint writes an `activity_log` entry (audit coverage assertion passes).
```
