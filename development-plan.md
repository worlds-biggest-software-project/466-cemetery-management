# Cemetery Management — Phased Development Plan

> Project: 466-cemetery-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` documents into a concrete, phased implementation roadmap for an AI-native, open-source, self-hostable cemetery management platform.

**Core value proposition:** Centralise plot inventory, interment records, sales/financials, grounds maintenance, and a public genealogy portal for cemetery operators (municipal, religious, private memorial parks) who today rely on paper ledgers and spreadsheets — while using AI to solve legacy-record digitisation (OCR + entity extraction), fuzzy/phonetic genealogy search, and obituary drafting that incumbents either lock behind premium pricing or do not offer at all.

**Primary differentiators (the AI-native + open advantage):**
1. **OCR + named-entity-extraction pipeline** that converts photographed paper ledger pages into structured, reviewable burial records — replacing an expensive manual professional service.
2. **Natural-language / phonetic / fuzzy genealogy search** over historic names with spelling variations — unmatched by any reviewed tool.
3. **A fully documented public REST API (OpenAPI 3.1) + GEDCOM 7.0 export** — closing the most consistent gap across incumbents (most have no public API).
4. **Accessible to small operators** — self-hostable, no per-plot licence, fills the gap between minimal free tools (Sunrise CMS) and $600+/yr SaaS.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The differentiators are AI-heavy (OCR, NER, embeddings, LLM obituary drafting). Python has the strongest ecosystem for these (Tesseract/PaddleOCR bindings, spaCy, the OpenAI/Anthropic SDKs, `rapidfuzz`, `jellyfish` for phonetics) while remaining excellent for REST APIs. |
| API framework | **FastAPI** | Native async (needed for long-running AI calls and webhook handling), automatic **OpenAPI 3.1** generation (a stated project requirement — closes the incumbents' API gap), Pydantic v2 validation, and first-class dependency injection for auth/RBAC. |
| Database | **PostgreSQL 16 + PostGIS 3.4** | Directly matches the README's recommended backend. Hosts the normalised core schema (data-model-suggestion-1) plus JSONB columns for cultural/jurisdictional variability (data-model-suggestion-3). PostGIS gives OGC-compliant spatial queries; `pg_trgm`, `fuzzystrmatch`, and `tsvector` give fuzzy/phonetic search without a separate search engine. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** (with GeoAlchemy2) | Mature async ORM; GeoAlchemy2 maps PostGIS geometry types; Alembic provides the versioned, repeatable migrations required for 100+ year archival schemas. |
| Async task queue | **Celery + Redis** | OCR/NER, GEDCOM export, bulk imports, and obituary generation are long-running and must not block API requests. Redis doubles as the broker, result backend, and rate-limit/cache store. |
| Object storage | **S3-compatible (MinIO self-hosted / AWS S3 cloud)** | Scanned deeds, ledger pages, permits, and memorial photos are large binary blobs that must not live in Postgres. `boto3` works against both MinIO (self-host) and S3 (cloud). |
| OCR | **PaddleOCR** (primary) with **Tesseract** fallback | PaddleOCR handles degraded historic handwriting/print better than Tesseract alone; both are open-source, satisfying the open-source mandate and avoiding per-page cloud OCR fees. A cloud OCR provider is pluggable behind an interface. |
| Entity extraction / obituary drafting | **Pluggable LLM provider** (Anthropic Claude default; OpenAI + local Ollama adapters) | NER from OCR text and obituary drafting need an LLM. A provider interface keeps the system open-source-friendly (local Ollama) while allowing hosted models. |
| Phonetic / fuzzy search | **PostgreSQL `fuzzystrmatch` (dmetaphone, soundex) + `pg_trgm` + `jellyfish`** | Handles historic spelling variation in SQL; `jellyfish` (Python) supplies Jaro-Winkler ranking in the application layer for result ordering. |
| Frontend (admin) | **React 18 + TypeScript + Vite**, **MapLibre GL JS**, **TanStack Query** | Map-first admin UI. **MapLibre GL JS** (open-source fork of Mapbox GL) renders vector tiles with no per-load licence fee — critical for an open project. TanStack Query manages server state against the REST API. |
| Frontend (public portal) | **Next.js 14 (App Router) + TypeScript** | The public genealogy portal must be independently scalable and SEO-friendly (memorial pages should be indexable). Server-side rendering + ISR suits high-traffic, read-heavy, cacheable memorial pages. Deployed as a separate tier per the README. |
| Map tile serving | **TileServer GL / pg_tileserv** over PostGIS | Serves OGC-aligned vector tiles and WFS-style feature queries from PostGIS directly. WGS 84 (EPSG:4326) default CRS per standards.md. |
| Auth | **OAuth 2.0 + OIDC, JWT (RS256)**, **PKCE** for SPA/mobile/public clients | Per standards.md (RFC 6749, 7519, 7636, OIDC Core). Staff use OIDC SSO; public-portal families and the mobile app use Authorization Code + PKCE. |
| Payments | **Stripe (Stripe Elements / hosted fields)** | Per standards.md — using hosted fields keeps the platform out of PCI DSS cardholder-data scope. Webhook signatures verified via HMAC-SHA256. A `PaymentProvider` interface allows alternatives. |
| Containerisation | **Docker + docker-compose** | Self-hosting is a core deployment mode; one `docker-compose up` must bring up Postgres/PostGIS, Redis, MinIO, API, workers, tile server, and both frontends. |
| Testing | **pytest + pytest-asyncio + testcontainers** (backend); **Vitest + Playwright** (frontend) | `testcontainers` spins up real PostGIS/Redis for integration tests; Playwright covers e2e map and portal flows. |
| Code quality | **Ruff** (lint+format), **mypy** (types), **pre-commit**; **ESLint + Prettier + tsc** (frontend) | Standard, fast tooling for each ecosystem. |
| Package management | **uv** (Python), **pnpm** (frontend) | Fast, reproducible installs. |
| Offline mobile | **PWA (the public portal codebase) + service worker + IndexedDB** | A progressive web app satisfies the offline-first groundskeeper requirement without maintaining native apps. Work orders and plot maps cache locally and sync when connectivity returns. |

### Standards alignment (from standards.md)

- **OpenAPI 3.1** — auto-generated from FastAPI; the public API contract.
- **RFC 7946 GeoJSON** + **OGC WMS/WFS/WMTS**, **EPSG:4326 (WGS 84)** — geospatial interchange and tile/feature serving.
- **GEDCOM 7.0** (with **5.5.1** fallback) — genealogy export.
- **OAuth 2.0 / OIDC / JWT / PKCE** — authentication & authorisation.
- **PCI DSS 4.0.1** — satisfied by Stripe hosted fields (no raw card data touches the app).
- **OWASP API Security Top 10**, **NIST SP 800-63B**, **ISO/IEC 27001/27018**, **GDPR/UK GDPR** — security and privacy posture.
- **W3C WCAG 2.2 AA** — public portal accessibility (elderly/genealogy users).
- **ISO 15836 Dublin Core** — document metadata.
- **OGC GeoPackage** — portable GIS export.
- **HL7 FHIR VRDR** — vital-records interoperability (backlog phase).

### Project Structure

```
cemetery-management/
├── docker-compose.yml              # postgres/postgis, redis, minio, api, worker, tileserver, web-admin, web-portal
├── docker-compose.prod.yml
├── .env.example
├── README.md
├── backend/
│   ├── pyproject.toml              # uv-managed; FastAPI, SQLAlchemy, Alembic, Celery, GeoAlchemy2, pydantic
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── migrations/                 # Alembic versioned migrations (+ PostGIS/extension bootstrap)
│   ├── src/
│   │   └── cemetery/
│   │       ├── main.py             # FastAPI app factory, router mounting, OpenAPI metadata
│   │       ├── config.py           # Pydantic Settings (env-driven)
│   │       ├── db.py               # async engine, session factory
│   │       ├── deps.py             # FastAPI dependencies (db session, current_user, RBAC)
│   │       ├── models/             # SQLAlchemy ORM models (one module per aggregate)
│   │       │   ├── location.py     # organisation, cemetery, section, lot, space
│   │       │   ├── people.py       # contact, deceased, deceased_relationship
│   │       │   ├── interment.py    # interment
│   │       │   ├── ownership.py    # ownership_rights
│   │       │   ├── finance.py      # contract, line_item, invoice, payment, perpetual_care_*
│   │       │   ├── operations.py   # work_order
│   │       │   ├── document.py     # document
│   │       │   ├── memorial.py     # memorial_page, memorial_tribute
│   │       │   ├── audit.py        # audit_log
│   │       │   └── auth.py         # user, user_cemetery_access
│   │       ├── schemas/            # Pydantic request/response models (mirrors models/)
│   │       ├── api/                # FastAPI routers (one per resource)
│   │       │   └── v1/
│   │       ├── services/           # business logic (sales workflow, availability, scheduling)
│   │       ├── geo/                # GeoJSON (de)serialisation, PostGIS helpers, WFS endpoints
│   │       ├── search/             # fuzzy/phonetic genealogy search
│   │       ├── ai/                 # provider interfaces + adapters
│   │       │   ├── ocr.py          # OCRProvider (PaddleOCR/Tesseract/cloud)
│   │       │   ├── extract.py      # NER / record extraction from OCR text
│   │       │   ├── obituary.py     # obituary drafting
│   │       │   └── llm.py          # LLMProvider interface (Anthropic/OpenAI/Ollama)
│   │       ├── exports/            # GEDCOM, GeoPackage, CSV, PDF (deeds/certificates)
│   │       ├── integrations/       # stripe, payment provider interface
│   │       ├── tasks/              # Celery tasks (ocr, import, export, obituary)
│   │       └── audit/              # audit middleware + triggers
│   └── tests/
│       ├── unit/
│       ├── integration/            # testcontainers (PostGIS, Redis, MinIO)
│       ├── e2e/
│       └── fixtures/               # sample ledger images, GeoJSON, GEDCOM, CSV
├── web-admin/                      # React + Vite + MapLibre admin SPA
│   ├── package.json
│   └── src/
├── web-portal/                     # Next.js public genealogy portal (PWA)
│   ├── package.json
│   └── src/
└── docs/
    ├── api/                        # generated OpenAPI artefacts
    └── deployment.md
```

The structure is grouped by concern (models, schemas, api, services, ai, exports, integrations) — not by phase — so each phase adds files without restructuring.

---

## Phase 1: Foundation & Data Model

### Purpose
Establish the project skeleton, configuration, database with PostGIS, the full core schema as Alembic migrations, and the ORM/Pydantic layer. After this phase the database can be created from scratch, all core entities exist, and the test harness (with real PostGIS via testcontainers) runs green. Nothing is user-facing yet, but every later phase builds directly on these models.

### Tasks

#### 1.1 — Project scaffold, config, and containerised dev environment

**What:** Create the `backend/` package, `docker-compose.yml`, env-driven config, and the FastAPI app factory exposing a health check.

**Design:**
- `docker-compose.yml` services: `postgres` (image `postgis/postgis:16-3.4`), `redis:7`, `minio`, `api`, `worker`, plus volumes for Postgres data and MinIO.
- `config.py` using Pydantic `BaseSettings`:
```python
class Settings(BaseSettings):
    database_url: str            # postgresql+asyncpg://...
    redis_url: str = "redis://redis:6379/0"
    s3_endpoint: str             # http://minio:9000
    s3_bucket: str = "cemetery-documents"
    s3_access_key: str
    s3_secret_key: str
    jwt_private_key_path: str
    jwt_public_key_path: str
    jwt_issuer: str = "cemetery-management"
    jwt_access_ttl_seconds: int = 900
    llm_provider: Literal["anthropic", "openai", "ollama"] = "anthropic"
    ocr_provider: Literal["paddle", "tesseract", "cloud"] = "paddle"
    stripe_secret_key: str | None = None
    stripe_webhook_secret: str | None = None
    environment: Literal["dev", "test", "prod"] = "dev"
    model_config = SettingsConfigDict(env_file=".env", env_prefix="CEM_")
```
- `main.py` exposes `GET /healthz` → `{"status": "ok", "db": "ok"|"down"}` (db check via `SELECT 1`).
- App factory mounts `/api/v1` router (empty for now) and sets OpenAPI metadata (title, version, contact, licence).

**Testing:**
- `Unit: Settings loads from env vars → all fields populated, defaults applied for unset optionals.`
- `Unit: Settings missing required CEM_DATABASE_URL → ValidationError naming database_url.`
- `Integration (testcontainers): GET /healthz with live Postgres → 200, db: "ok".`
- `Integration: GET /healthz with Postgres stopped → 200, db: "down".`

#### 1.2 — PostGIS bootstrap & core location hierarchy migration

**What:** First Alembic migration enabling extensions and creating the location hierarchy (organisations, cemeteries, sections, lots, spaces).

**Design:**
- Migration `0001_extensions` runs: `CREATE EXTENSION IF NOT EXISTS postgis; pg_trgm; fuzzystrmatch; "uuid-ossp"|pgcrypto;`.
- Migration `0002_location` creates `organisations`, `cemeteries`, `sections`, `lots`, `spaces` per **data-model-suggestion-1** (UUID PKs, `GEOMETRY(POLYGON,4326)` boundaries, GiST spatial indexes, status CHECK constraints).
- **Hybrid extension (from data-model-suggestion-3):** add a `cultural_rules JSONB` column to `sections` and an `attributes JSONB` column to `spaces` for jurisdiction/faith-specific fields, alongside the typed columns (`faith_designation`, `orientation_rule`, `spacing_rule_cm`).
- Status enums enforced by CHECK constraints exactly as in suggestion-1 (e.g. `spaces.status IN ('available','sold','reserved','occupied','full','unavailable','condemned','historical')`).

**Testing:**
- `Integration (testcontainers): run migration upgrade head → all five tables + extensions exist (query pg_extension, information_schema).`
- `Integration: insert space with status 'banana' → IntegrityError (CHECK violation).`
- `Integration: insert section with boundary GeoJSON polygon → ST_Area returns >0; GiST index used (EXPLAIN).`
- `Integration: migration downgrade → upgrade round-trips cleanly.`

#### 1.3 — People, interment, ownership, finance, operations, document, memorial, audit, user migrations

**What:** Remaining migrations creating all other core tables and the two query views.

**Design:**
- Create `contacts`, `deceased` (incl. `name_aliases TEXT[]`, generated `display_name`, generated `search_vector TSVECTOR`, trigram + GIN indexes), `deceased_relationships`, `interments`, `ownership_rights`, `contracts`, `contract_line_items`, `invoices`, `payments`, `perpetual_care_funds/contributions/transactions`, `work_orders` (incl. `ai_priority_score`, `ai_priority_reason`), `documents` (polymorphic nullable FKs, `ocr_status`, `ocr_text`, Dublin Core columns, `is_public`), `memorial_pages`, `memorial_tributes`, `audit_log` (BIGSERIAL, declaratively **partitioned by RANGE(timestamp)** monthly), `users`, `user_cemetery_access`, `gedcom_exports` — all per suggestion-1.
- Create views `v_burial_search` and `v_space_availability` (suggestion-1).
- Add a `documents` CHECK enforcing at least one parent FK is non-null.

**Testing:**
- `Integration: full upgrade head from empty DB → every table + view present.`
- `Integration: insert deceased('Jon','','Smith') → display_name = 'Jon Smith', search_vector populated.`
- `Integration: insert ownership_rights with neither lot_id nor space_id → CHECK violation.`
- `Integration: query v_burial_search after seeding deceased+completed interment+space → one row with section_name and gps coords.`
- `Integration: audit_log partition for current month auto-resolves on insert.`

#### 1.4 — ORM models & Pydantic schemas

**What:** SQLAlchemy 2.0 (async) models and Pydantic v2 schemas mirroring the migrations.

**Design:**
- One ORM module per aggregate (see structure). Use `Mapped[...]` typing; GeoAlchemy2 `Geometry('POLYGON', srid=4326)` for spatial columns.
- Pydantic schemas split into `*Create`, `*Update`, `*Read`. Geometry fields (de)serialise to/from GeoJSON dicts via a custom `GeoJSONGeometry` type validated against RFC 7946 structure.
- A shared `AuditMixin` (created_at/updated_at) and a base `ReadModel(orm_mode/from_attributes=True)`.

**Testing:**
- `Unit: SpaceCreate validates GeoJSON Polygon → ok; malformed ring (not closed) → ValidationError.`
- `Unit: DeceasedRead serialises ORM row including computed display_name.`
- `Integration: round-trip — persist Cemetery ORM with boundary, reload, GeoJSON output equals input geometry.`

---

## Phase 2: Authentication, RBAC & Audit Trail

### Purpose
Add identity, role-based access control, and the immutable audit trail before any data-mutating endpoints exist — so every write from Phase 3 onward is authorised and logged. This satisfies the legal-record-permanence and security requirements (ISO 27001, NIST 800-63B, OWASP API Security) and is a hard dependency for all subsequent API work.

### Tasks

#### 2.1 — OAuth2/OIDC + JWT authentication

**What:** Local credential login, JWT issuance/verification, and OIDC-ready token structure.

**Design:**
- `POST /api/v1/auth/login` (email+password) → `{access_token, refresh_token, token_type:"bearer", expires_in}`. Passwords hashed with Argon2id (per NIST 800-63B).
- JWTs signed RS256 with claims: `sub` (user id), `email`, `role`, `cemeteries` (list of accessible cemetery ids), `iss`, `exp`, `iat`, `jti`.
- `POST /api/v1/auth/refresh`, `POST /api/v1/auth/logout` (refresh-token revocation list in Redis).
- `get_current_user` FastAPI dependency validates the bearer JWT, loads the user, attaches role + cemetery scope.
- PKCE Authorization-Code endpoints stubbed here, completed for public/mobile clients in Phase 7.

**Testing:**
- `Unit: Argon2id hash + verify round-trip; wrong password → False.`
- `Integration: login valid creds → 200 + tokens; JWT verifies against public key, contains role + cemeteries.`
- `Integration: login bad password → 401, generic message (no user enumeration).`
- `Integration: request protected route with expired JWT → 401.`
- `Integration: refresh after logout (jti revoked) → 401.`

#### 2.2 — RBAC enforcement & cemetery scoping

**What:** Role + per-cemetery authorisation dependencies applied to routes.

**Design:**
- Roles per suggestion-1 `users.role`: `super_admin, cemetery_admin, staff, grounds_crew, sales, read_only, public_portal`.
- `require_role(*roles)` and `require_cemetery_access(cemetery_id, write=True)` dependencies. `grounds_crew` may read maps + read/write only work orders; `sales` may write contracts/invoices; `read_only` GET-only; `public_portal` restricted to portal endpoints.
- Authorisation matrix documented in `docs/rbac.md` and enforced centrally so it is testable as a unit.

**Testing:**
- `Unit: authz_matrix(grounds_crew, "POST /interments") → denied; (cemetery_admin, same) → allowed.`
- `Integration: staff scoped to cemetery A requesting cemetery B resource → 403.`
- `Integration: read_only user POST → 403; GET → 200.`

#### 2.3 — Immutable audit trail

**What:** Append-only audit logging of every create/update/delete on core tables.

**Design:**
- Two-layer: (a) PostgreSQL row-level triggers writing old/new JSONB to `audit_log` (guarantees capture even for direct SQL); (b) an app-layer middleware enriching entries with `user_id`, `user_email`, `user_ip`, `cemetery_id` via a request-scoped context var.
- `audit_log` is INSERT-only: revoke UPDATE/DELETE from the application DB role.
- `GET /api/v1/audit?table=&record_id=&from=&to=` (super_admin/cemetery_admin) returns the change history with pagination (RFC 8288 `Link` headers).

**Testing:**
- `Integration: update a deceased record via API → audit_log row with action UPDATE, old_values+new_values diff, correct user_id+ip.`
- `Integration: attempt UPDATE on audit_log as app role → permission denied.`
- `Integration: GET /audit filtered by record_id → ordered history; pagination Link header present.`

---

## Phase 3: Inventory & Records Core (the heart of the product)

### Purpose
Deliver the central record-keeping engine: managing the cemetery/section/lot/space inventory, deceased records, and interments, with real-time availability. This is the operational core every cemetery needs day one and the foundation the map, sales, and portal all read from. After this phase a cemetery's entire inventory and burial history can be entered and queried via the API.

### Tasks

#### 3.1 — Location hierarchy CRUD + GeoJSON

**What:** CRUD endpoints for organisations, cemeteries, sections, lots, spaces with geometry I/O.

**Design:**
- REST resources under `/api/v1`: `organisations`, `cemeteries`, `cemeteries/{id}/sections`, `sections/{id}/lots`, `lots/{id}/spaces` (+ flat `spaces/{id}`).
- Geometry accepted/returned as GeoJSON (RFC 7946). On write, validate ring closure and SRID; store as PostGIS geometry.
- `GET /cemeteries/{id}/map?bbox=&status=` returns a GeoJSON `FeatureCollection` of sections+spaces with `properties.status` and `properties.fill_colour` for client colour-coding (available/sold/reserved/occupied).
- Cascading status: marking a section `full` is computed, not stored arbitrarily — derived from child space statuses via a service function.

**Testing:**
- `Unit: section status roll-up — all child spaces occupied → section reported 'full'.`
- `Integration: POST cemetery with boundary GeoJSON → 201; GET returns identical geometry.`
- `Integration: GET /map?status=available → FeatureCollection containing only available spaces.`
- `Integration: POST space with polygon outside parent section boundary → 422 (ST_Contains check).`

#### 3.2 — Deceased & interment records with availability enforcement

**What:** Manage deceased persons and the interments that place them in spaces, enforcing capacity.

**Design:**
- `deceased` CRUD; supports partial/approximate dates (`date_of_*_approx`) and `name_aliases[]` for historic records.
- `interments` CRUD. On create: verify `space.current_interments < space.max_interments`; in one transaction increment `current_interments`, set `space.status` to `occupied`/`full`, create the interment. State machine: `scheduled → confirmed → completed → cancelled` (+ `pending_permit`).
- Disinterment/reinterment decrement/transfer counts and write audit entries.
- `GET /spaces/{id}/interments`, `GET /deceased/{id}` (with linked interment + location).

**Testing:**
- `Unit: interment state machine — completed → scheduled transition rejected.`
- `Integration: create interment in single-capacity space already occupied → 409 conflict, no row written.`
- `Integration: create interment → space.current_interments incremented and status flips to 'occupied' atomically.`
- `Integration: cancel interment → count decremented, status reverts, audit entries for both writes.`

#### 3.3 — Availability search service

**What:** Query for purchasable spaces by criteria, backing both sales and the admin map.

**Design:**
- `GET /api/v1/availability?cemetery_id=&section_type=&faith=&space_type=&max_price=&near_space_id=&limit=` reading `v_space_availability`.
- `near_space_id` triggers a PostGIS `ORDER BY location_point <-> (SELECT location_point ...)` nearest-neighbour sort (for "near an existing family plot").
- Returns spaces with price, section, faith designation, and GPS coordinates.

**Testing:**
- `Integration: availability filtered faith='Jewish' → only spaces in Jewish-designated sections.`
- `Integration: near_space_id=X → results ordered by ascending distance from X.`
- `Integration: max_price filter excludes higher-priced spaces.`

---

## Phase 4: Document Management & Object Storage

### Purpose
Enable upload, storage, retrieval, and metadata tagging of scanned deeds, permits, certificates, ledger pages, and photos — attached to any record. This is a table-stakes feature and a prerequisite for the AI OCR digitisation pipeline (Phase 8), which consumes uploaded ledger images.

### Tasks

#### 4.1 — Document upload & retrieval via S3-compatible storage

**What:** Store binary documents in MinIO/S3 with metadata rows in Postgres, attachable to any entity.

**Design:**
- `POST /api/v1/documents` (multipart): file + JSON metadata (`document_type`, polymorphic parent ref e.g. `deceased_id`, `is_public`, Dublin Core `dc_creator/dc_date/dc_subject` per ISO 15836). File streamed to S3 at key `{cemetery_id}/{document_type}/{uuid}/{filename}`; row inserted with `file_path`, `file_size_bytes`, `mime_type`, `ocr_status='pending'` for images/PDFs else `not_applicable`.
- `GET /api/v1/documents/{id}/content` → time-limited S3 presigned URL (302 redirect). Public docs (`is_public=true`) served via the portal; private docs require auth + cemetery scope.
- Allowed MIME types whitelisted; max size configurable (default 50 MB).

**Testing:**
- `Integration (testcontainers MinIO): upload PDF → 201, object exists in bucket, row has correct size+mime, ocr_status='pending'.`
- `Integration: upload disallowed type (.exe) → 415.`
- `Integration: GET content for private doc without auth → 401; with wrong-cemetery user → 403.`
- `Integration: presigned URL expires (mock clock) → access denied after TTL.`

#### 4.2 — Document search & listing

**What:** List and full-text search documents by parent record, type, date, and OCR text.

**Design:**
- `GET /api/v1/documents?deceased_id=&type=&q=` — `q` runs against the `to_tsvector(ocr_text)` GIN index plus filename.
- Returns metadata + presigned thumbnail URL where applicable.

**Testing:**
- `Integration: search q matching a phrase in ocr_text → document returned, ranked.`
- `Integration: list by deceased_id → only that record's documents.`

---

## Phase 5: Sales, Financial & Deed Generation

### Purpose
Implement the revenue workflow: ownership records, sales contracts, invoicing, payment capture (Stripe), perpetual-care fund tracking, and PDF deed/certificate generation. This turns the records core into a system a cemetery can run its business on and produces the legal ownership documents incumbents charge for.

### Tasks

#### 5.1 — Ownership & sales contract workflow

**What:** Create ownership rights and sales contracts linking buyers to spaces/lots.

**Design:**
- `POST /api/v1/contracts` with line items (`plot_purchase`, `opening_closing`, `perpetual_care`, etc.). Service computes totals (subtotal + tax + perpetual_care − discount). State machine: `draft → pending → active → completed` (+ `cancelled`, `defaulted`).
- On contract `active`: in one transaction create `ownership_rights` (deed_type from contract_type), set space `status='sold'`, and create the initial `invoice`.
- Transfers create a new `ownership_rights` row referencing `transferred_from_id`, preserving the chain (suggestion-1) — never mutate the prior record (audit integrity).

**Testing:**
- `Unit: contract total = sum(line totals) + tax − discount.`
- `Integration: activate contract → ownership row created, space status 'sold', invoice generated, all atomic.`
- `Integration: transfer ownership → new row links transferred_from_id; old row status 'transferred', not deleted.`

#### 5.2 — Invoicing, payments & Stripe integration

**What:** Issue invoices, record payments (manual + Stripe), support payment plans.

**Design:**
- `invoices` with generated `balance_due`. `POST /api/v1/invoices/{id}/payments` records cash/cheque manually or creates a Stripe PaymentIntent (hosted fields client-side; server never sees PAN → PCI scope avoided).
- `POST /api/v1/webhooks/stripe` verifies `Stripe-Signature` (HMAC-SHA256), and on `payment_intent.succeeded` records a `payment`, updates `invoice.amount_paid`/`status`, enqueues nothing blocking. Idempotent on Stripe event id.
- Payment plans: `payment_plan_months` + `monthly_payment` on contract; a scheduled Celery beat task flags overdue invoices.
- `PaymentProvider` interface so Stripe is swappable.

**Testing:**
- `Integration: record cash payment → amount_paid increases, status → 'partial' then 'paid', balance_due recomputed.`
- `Integration (mocked Stripe): valid webhook signature, payment_intent.succeeded → payment recorded, invoice paid.`
- `Integration: webhook with bad signature → 400, no payment recorded.`
- `Integration: duplicate webhook (same event id) → second call is a no-op.`

#### 5.3 — Perpetual care fund tracking

**What:** Track trust corpus, contributions, and transactions per fund.

**Design:**
- On contract activation, the `perpetual_care` line item creates a `perpetual_care_contributions` row and a `contribution` transaction; `perpetual_care_transactions` maintains a `running_balance`. Endpoints for transactions (income, distribution, fee) and a fund statement report.

**Testing:**
- `Unit: running_balance after [contribution 1000, interest 50, fee 10] = 1040.`
- `Integration: contract with perpetual_care line → contribution + transaction created, fund corpus updated.`

#### 5.4 — PDF deed & certificate generation

**What:** Generate ownership deeds and interment certificates as PDFs.

**Design:**
- Jinja2 HTML templates rendered to PDF (WeasyPrint). `GET /api/v1/ownership/{id}/deed.pdf`, `GET /api/v1/interments/{id}/certificate.pdf`. Generated PDFs stored as `documents` (`document_type='deed'`) and audit-logged. Templates configurable per organisation (logo, registrar signature block).

**Testing:**
- `Integration: generate deed PDF → valid PDF (magic bytes), contains owner name + space ref + deed number, stored as document.`
- `Unit: template renders with missing optional fields (no middle name) without error.`

---

## Phase 6: Operations & Maintenance (Work Orders)

### Purpose
Provide grounds-maintenance work-order management — creation, assignment, scheduling with conflict detection, and completion tracking — including the data fields AI prioritisation (Phase 8) will populate. Can be developed in parallel with Phase 5 once Phase 3 is complete.

### Tasks

#### 6.1 — Work order CRUD, assignment & scheduling

**What:** Manage work orders against sections/lots/spaces with assignment and a schedule view.

**Design:**
- `work_orders` CRUD per suggestion-1 (`work_type`, `priority`, `assigned_to`, `scheduled_date`, `status: open→assigned→in_progress→completed`/`cancelled`/`deferred`).
- `GET /api/v1/work-orders?status=&assigned_to=&from=&to=&cemetery_id=` and a `GET /api/v1/schedule?from=&to=` calendar feed combining work orders and scheduled interments, with **conflict detection** (two interments in the same space/time, or grave-opening overlapping an interment).
- `grounds_crew` role: list/update own assigned orders only.

**Testing:**
- `Unit: conflict detector flags two interments same space overlapping times; non-overlapping → no conflict.`
- `Integration: assign work order to grounds_crew user → appears in their filtered list; other crew cannot update it (403).`
- `Integration: complete work order → completed_date set, status 'completed', audit entry.`

---

## Phase 7: Public Genealogy Portal & Search

### Purpose
Build the public-facing, independently scalable tier: fuzzy/phonetic burial search, memorial pages with moderated family tributes, walk-to-grave navigation, and PKCE auth for families. This delivers a headline differentiator (search incumbents lack) and the consumer-facing value. Depends on Phase 3 (records) and Phase 4 (public documents); the search service (7.1) can begin once Phase 3 lands.

### Tasks

#### 7.1 — Fuzzy / phonetic genealogy search

**What:** Public search over completed interments tolerant of historic spelling variation.

**Design:**
- `GET /api/v1/public/search?name=&cemetery_id=&death_year=&fuzzy=true` over `v_burial_search`.
- Ranking pipeline: (1) `tsvector` full-text match; (2) `pg_trgm` similarity for typo tolerance; (3) `dmetaphone`/`soundex` (`fuzzystrmatch`) for phonetic equivalence ("Smith"/"Smyth", "Catherine"/"Katharine"); (4) application-layer Jaro-Winkler (`jellyfish`) re-rank. Also matches `name_aliases[]`.
- Optional natural-language mode: an LLM normalises free-text queries ("graves of the Müller family who died in the 1890s") into structured filters before SQL.
- Returns name, dates, cemetery/section, GPS, and `memorial_slug` if published. Rate-limited (Redis) and cached.

**Testing:**
- `Unit: dmetaphone('Smith') == dmetaphone('Smyth'); search 'Smyth' returns 'Smith' record.`
- `Integration: search 'Katharine' returns 'Catherine' interment via phonetic match, ranked below exact matches.`
- `Integration: name in name_aliases array matched.`
- `Integration: rate limit exceeded → 429.`
- `E2E (Playwright): portal search box → results list with walk-to-grave link.`

#### 7.2 — Memorial pages & moderated tributes

**What:** Public memorial page per deceased with family photo/tribute submission and moderation.

**Design:**
- Next.js routes: `/memorial/[slug]` (SSR + ISR; SEO meta; WCAG 2.2 AA). Shows tribute text, public documents/photos, map location.
- `POST /api/v1/public/memorials/{slug}/tributes` (PKCE-authenticated or captcha-gated anonymous) → tribute with `moderation_status='pending'`. Staff moderation queue `GET /api/v1/memorials/tributes?status=pending`, `PATCH .../{id}` to approve/reject. Photo uploads reuse Phase 4 with `is_public=true`.
- `view_count` incremented asynchronously.

**Testing:**
- `Integration: submit tribute → status 'pending', not visible publicly until approved.`
- `Integration: approve tribute → appears in public memorial response.`
- `Integration: published memorial page → 200 SSR with deceased name in HTML title; unpublished → 404.`
- `E2E: family uploads photo + tribute → appears after staff approval.`

#### 7.3 — Walk-to-grave navigation & PKCE auth

**What:** GPS directions to a grave and Authorization-Code + PKCE login for public/mobile clients.

**Design:**
- `GET /api/v1/public/spaces/{id}/navigation` → space GPS coords + GeoJSON route hint from cemetery entrance (nearest path node if path network exists, else straight-line bearing + distance).
- Complete PKCE flow (`/authorize`, `/token` with `code_verifier`) for the portal and the groundskeeper PWA, plus optional social login via OIDC.

**Testing:**
- `Integration: navigation for space with GPS → returns lat/lng + distance from entrance.`
- `Integration: PKCE flow with valid verifier → tokens; mismatched verifier → 400.`

---

## Phase 8: AI-Native Pipeline (Digitisation, Obituary, Prioritisation)

### Purpose
Deliver the AI differentiators: OCR + entity-extraction of historic ledger pages into reviewable draft records, AI obituary drafting, and AI work-order prioritisation. These features run as async Celery tasks and are the project's core competitive advantage. Depends on Phase 4 (documents), Phase 3 (records), Phase 6 (work orders).

### Tasks

#### 8.1 — OCR + entity-extraction digitisation pipeline

**What:** Convert uploaded ledger-page images into structured draft `deceased`/`interment` records for human review.

**Design:**
- `OCRProvider` interface: `extract_text(image_bytes) -> OcrResult(text, confidence, blocks)`. Adapters: PaddleOCR (default), Tesseract, cloud (pluggable).
- Celery task `process_document_ocr(document_id)` triggered on upload of image/PDF ledger pages: sets `ocr_status='processing'`, runs OCR, stores `ocr_text`+`ocr_confidence`, then calls `extract.py` which prompts the `LLMProvider` to return structured records.
- Extraction prompt (structured JSON output) template:
```
System: You extract burial-register entries from OCR'd historic cemetery ledger text.
Return JSON array; each entry: {first_name, last_name, maiden_name?, date_of_birth?,
date_of_death?, section?, lot?, space?, interment_date?, notes?, confidence (0-1)}.
Preserve original spelling. Use null for missing fields. Flag uncertainty in confidence.
User: <ocr_text>
```
- Extracted entries land in a **review queue** as draft `deceased`+`interment` rows with a `needs_review` flag and `ocr_confidence`; they are NOT auto-published. `GET /api/v1/imports/{batch}/review`, `POST .../approve` promotes to live records (audit-logged with source document link).
- Anomaly detection: flag likely duplicate deceased (trigram similarity > threshold on name+death year) during review.

**Testing:**
- `Integration (fixture ledger image, mocked OCR+LLM): upload → ocr_status 'completed', ocr_text stored, N draft records in review queue, none live.`
- `Integration: OCR provider raises → ocr_status 'failed', error captured, no partial records.`
- `Integration: approve reviewed entry → live deceased+interment created, linked to source document, audit entry.`
- `Unit: duplicate detector flags 'John Smith' d.1901 vs existing 'Jon Smith' d.1901.`
- `Unit: extraction parser handles LLM returning fewer fields (nulls) without error.`

#### 8.2 — AI obituary drafting

**What:** Generate a draft obituary from a deceased record's structured data.

**Design:**
- `POST /api/v1/deceased/{id}/obituary/draft` builds a prompt from name, dates, biography, military service, relationships; calls `LLMProvider`; returns draft text. Staff edit and save to `deceased.obituary_text` (never auto-published). Prompt is template-driven and tone-configurable.

**Testing:**
- `Integration (mocked LLM): draft for deceased with military service → draft includes service detail; saved only on explicit PATCH.`
- `Unit: prompt builder omits sections for missing data (no birth date → no born phrase).`

#### 8.3 — AI maintenance prioritisation

**What:** Score open work orders by suggested priority from seasonal/contextual signals.

**Design:**
- Celery beat task `score_work_orders()` computes `ai_priority_score` + `ai_priority_reason` per open order using rules + optional LLM reasoning over season, work_type, days-open, and upcoming interments nearby. Advisory only — never auto-reassigns. Surfaced as a sortable column in the admin UI.

**Testing:**
- `Unit: snow_removal in winter scores higher than in summer; reason string explains why.`
- `Integration: scoring task populates ai_priority_score for all open orders, leaves closed ones untouched.`

---

## Phase 9: Interoperability — API, GEDCOM & Geospatial Exports

### Purpose
Ship the documented public API surface and the export/interchange formats that make this platform the open hub incumbents are not: OpenAPI 3.1 spec + API keys/webhooks for third parties, GEDCOM 7.0/5.5.1 export for genealogy platforms, and GeoPackage/WFS for GIS tools. Depends on Phases 3, 5, 7.

### Tasks

#### 9.1 — Public API hardening: keys, rate limits, OpenAPI 3.1, pagination

**What:** Production-grade third-party API access.

**Design:**
- API-key issuance (`POST /api/v1/api-keys`, scoped, hashed at rest) for funeral-home/accounting integrations, in addition to OAuth.
- Consistent pagination + `Link` headers (RFC 8288), Redis rate limiting per key, OWASP API Top 10 controls (object-level authz checks, input validation, no excessive data exposure).
- The FastAPI-generated **OpenAPI 3.1** doc published at `/openapi.json` and committed to `docs/api/` in CI; Swagger UI at `/docs`.

**Testing:**
- `Integration: request with revoked API key → 401; valid scoped key → 200 within scope, 403 outside scope.`
- `Integration: list endpoint returns Link rel="next"; page 2 returns remaining items.`
- `CI: generated openapi.json validates against OpenAPI 3.1 schema.`

#### 9.2 — GEDCOM 7.0 / 5.5.1 export

**What:** Export burial records (with relationships) as GEDCOM.

**Design:**
- `POST /api/v1/cemeteries/{id}/exports/gedcom?version=7.0|5.5.1` enqueues a Celery task that emits INDI records (deceased: NAME, BIRT/DEAT with dates+place, SEX), FAM records from `deceased_relationships`, and SOUR records referencing the cemetery; writes file to S3; records a `gedcom_exports` row. Per standards.md note, support both 7.0 and 5.5.1.

**Testing:**
- `Unit: GEDCOM writer emits well-formed INDI with NAME 'John /Smith/' and DEAT date; validates against a GEDCOM 7.0 grammar checker.`
- `Integration: export for cemetery with 3 related deceased → FAM record links them; downloadable file produced.`
- `Unit: 5.5.1 mode emits 5.5.1-compatible tags.`

#### 9.3 — Geospatial export (GeoPackage) & WFS feature access

**What:** Export/serve plot geometry for desktop GIS.

**Design:**
- `GET /api/v1/cemeteries/{id}/export.gpkg` produces an OGC GeoPackage (via GDAL/`ogr2ogr` against PostGIS) of sections/lots/spaces with attributes. A read-only WFS-style endpoint (`/api/v1/wfs`) returns GeoJSON feature collections by layer for QGIS/ArcGIS consumers (EPSG:4326).

**Testing:**
- `Integration: export.gpkg → valid GeoPackage openable by GDAL, layer 'spaces' has expected feature count.`
- `Integration: WFS spaces layer → GeoJSON FeatureCollection with valid geometries.`

---

## Phase 10: Admin Web UI & Offline Groundskeeper PWA

### Purpose
Provide the map-first admin SPA (the daily interface for cemetery staff) and the offline-capable groundskeeper PWA. This makes the platform usable by non-technical operators — the adoption barrier that limits Sunrise CMS and pg_friedhof. Depends on the relevant API phases per screen.

### Tasks

#### 10.1 — Map-first admin SPA

**What:** React + MapLibre admin interface over the API.

**Design:**
- MapLibre GL JS renders the cemetery vector tiles (from the tile server) with spaces colour-coded by status; clicking a space opens its record inline (interments, ownership, documents) — the incumbent "map-first" UX pattern. Screens: inventory map, deceased/interment forms, sales/contract workflow, invoices/payments, documents (with OCR review queue), work-order board, tribute moderation, audit log viewer. TanStack Query against the REST API; auth via OIDC.

**Testing:**
- `E2E (Playwright): log in → map renders → click available space → record panel opens.`
- `E2E: create interment via form → space recolours to occupied on map.`
- `Component (Vitest): status legend maps each status to its fill colour.`

#### 10.2 — Offline-first groundskeeper PWA

**What:** Installable PWA for grounds crew to view maps and manage work orders offline.

**Design:**
- Service worker caches the map tiles, assigned work orders, and space records for the staff member's cemetery in IndexedDB. Work-order status changes made offline queue locally and sync (with conflict resolution: last-write-wins on status, audit entry on reconnect). Walk-to-grave navigation works offline using cached GPS coords.

**Testing:**
- `E2E (Playwright offline mode): load PWA, go offline, mark work order complete → queued; go online → synced, server reflects completion.`
- `E2E offline: open cached space → GPS navigation renders without network.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Model        ─── required by everything
    │
Phase 2: Auth, RBAC & Audit             ─── requires P1
    │
Phase 3: Inventory & Records Core       ─── requires P2   (the heart of the product)
    ├── Phase 4: Documents & Storage     ─── requires P3
    │       │
    ├── Phase 5: Sales & Financials      ─── requires P3  ┐ can be built
    ├── Phase 6: Operations/Work Orders  ─── requires P3  ┘ in parallel
    │
    ├── Phase 7: Public Portal & Search  ─── requires P3 (search) + P4 (public docs)
    │
Phase 8: AI Pipeline                     ─── requires P3, P4, P6
    │
Phase 9: Interoperability (API/GEDCOM/GIS) ─ requires P3, P5, P7
    │
Phase 10: Admin UI & Groundskeeper PWA   ─── each screen requires its backing phase
```

**Parallelism opportunities:**
- After **Phase 3**: Phases **4, 5, 6** can be developed concurrently (independent subsystems sharing only the records core).
- The **Phase 7.1 search service** can start as soon as Phase 3 lands, in parallel with Phases 4–6.
- **Phase 8 tasks (8.1, 8.2, 8.3)** are mutually independent once their data dependencies exist.
- **Phase 9 exports (9.2 GEDCOM, 9.3 GIS)** are independent of each other.
- **Phase 10** frontend screens can be built incrementally against each completed backend phase rather than waiting for all backend work.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass; integration tests run against real PostGIS/Redis/MinIO via testcontainers.
3. `ruff check` and `ruff format --check` pass (backend); `eslint` + `prettier --check` + `tsc --noEmit` pass (frontend).
4. `mypy` type-checking passes with no new errors (backend).
5. `docker compose build` succeeds and `docker compose up` brings the affected services up healthy.
6. The phase's feature works end-to-end against a seeded sample cemetery.
7. New configuration options are documented in `.env.example` and `docs/deployment.md`.
8. New/changed API endpoints appear correctly in the auto-generated OpenAPI 3.1 spec, and the committed `docs/api/openapi.json` is regenerated.
9. New Alembic migrations exist, upgrade/downgrade cleanly on an empty database, and are committed.
10. Audit logging covers every new data-mutating endpoint (verified by an integration test).
11. Security checks for the phase pass (authz on every new route; no secrets in code; OWASP API Top 10 review for new endpoints).
12. Public-facing UI added in the phase meets WCAG 2.2 AA on its primary flows.
```
