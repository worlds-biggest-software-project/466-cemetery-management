# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Cemetery Management (466) · Generated: 2026-05-26

---

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) as the primary architectural pattern. Every change to cemetery data is captured as an immutable domain event in an append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimised projections (materialised views) serve the different query needs: the public genealogy portal, the administrative dashboard, the map renderer, and the financial reports.

This approach is a natural fit for cemetery management because burial records are permanent legal documents requiring an immutable audit trail that spans 100+ years. The project README explicitly calls for "immutable audit trail architecture for burial records." Event sourcing provides this as a first-class architectural property rather than a bolt-on audit log table.

---

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Event Store | PostgreSQL with append-only event tables | Familiar, transactional, and supports JSONB event payloads natively |
| Message Broker | NATS JetStream or Apache Kafka | Durable event streaming for projecting to read models and triggering async workflows |
| Read Model DB | PostgreSQL (separate instance or schema) | Denormalised projections optimised for each query pattern |
| Spatial Read Model | PostgreSQL + PostGIS | Spatial projections for map rendering and GIS queries |
| Search Read Model | OpenSearch / Elasticsearch | Full-text, fuzzy, and phonetic search for the genealogy portal |
| Cache | Redis | Hot cache for frequently accessed projections (plot availability map) |
| API Layer | Node.js / TypeScript or Go | Command handlers (writes) and query handlers (reads) separated |
| Migrations | Flyway (for projection schemas) | Event store schema is append-only; projections are rebuilt from events |

---

## Core Concepts

### Event Sourcing Fundamentals

In traditional CRUD, the database stores the latest state of each entity. In event sourcing, the database stores the sequence of state-changing events. Current state is computed by replaying the events in order.

```
Traditional:  Plot #A-107 → { status: "occupied", owner: "Smith", interments: 2 }

Event-sourced: Plot #A-107 →
  1. PlotCreated       { section: "A", number: "107", type: "ground_single" }
  2. PlotPriceSet       { price: 2500.00, currency: "USD" }
  3. PlotSold           { buyer: "John Smith", deed: "D-2019-0042" }
  4. IntermentScheduled { deceased: "Mary Smith", date: "2019-06-15" }
  5. IntermentCompleted { actual_date: "2019-06-15", officiant: "Rev. Brown" }
  6. IntermentScheduled { deceased: "John Smith", date: "2024-11-03" }
  7. IntermentCompleted { actual_date: "2024-11-03", officiant: "Rev. Chen" }
```

To know the current state of Plot A-107, replay events 1-7. To know the state on 2020-01-01, replay events 1-5 only. This time-travel capability is invaluable for legal disputes, historical research, and regulatory audits.

### CQRS Separation

Commands (writes) go through command handlers that validate business rules and emit events. Queries (reads) go through read-optimised projections that are built asynchronously from the event stream.

```
┌──────────────┐         ┌──────────────┐
│  Command API │────────▶│  Event Store │
│  (writes)    │         │  (PostgreSQL)│
└──────────────┘         └──────┬───────┘
                                │ event stream
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
            ┌──────────┐ ┌──────────┐ ┌──────────┐
            │ Map Read │ │ Search   │ │ Finance  │
            │ Model    │ │ Read     │ │ Read     │
            │ (PostGIS)│ │ Model    │ │ Model    │
            │          │ │(OpenSrch)│ │ (PG)     │
            └──────────┘ └──────────┘ └──────────┘
                    │           │           │
                    ▼           ▼           ▼
            ┌──────────┐ ┌──────────┐ ┌──────────┐
            │ Map UI   │ │ Genealogy│ │ Finance  │
            │ (admin)  │ │ Portal   │ │ Reports  │
            └──────────┘ └──────────┘ └──────────┘
```

---

## Event Store Schema

```sql
-- ============================================================
-- EVENT STORE (append-only, immutable)
-- ============================================================

-- Core event table: every state change in the system is an event
CREATE TABLE events (
    -- Global sequential position (used for ordered replay)
    global_position     BIGSERIAL NOT NULL,
    -- Event identity
    event_id            UUID NOT NULL DEFAULT gen_random_uuid(),
    -- Aggregate identity: which entity this event belongs to
    aggregate_type      VARCHAR(100) NOT NULL,  -- e.g., 'Cemetery', 'Space', 'Interment', 'Contract'
    aggregate_id        UUID NOT NULL,
    -- Version within the aggregate (optimistic concurrency control)
    aggregate_version   INTEGER NOT NULL,
    -- Event type and payload
    event_type          VARCHAR(200) NOT NULL,  -- e.g., 'SpaceSold', 'IntermentCompleted'
    event_data          JSONB NOT NULL,          -- Event-specific payload
    -- Metadata
    metadata            JSONB NOT NULL DEFAULT '{}',  -- user_id, ip_address, correlation_id, causation_id
    -- Timestamp
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Constraints
    PRIMARY KEY (global_position),
    UNIQUE (aggregate_type, aggregate_id, aggregate_version)
);

-- Indexes for efficient replay
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, aggregate_version);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);
CREATE INDEX idx_events_global_position ON events(global_position);

-- Ensure the events table is append-only (no UPDATE or DELETE)
-- Enforced via database role permissions:
-- REVOKE UPDATE, DELETE ON events FROM app_role;
-- GRANT INSERT, SELECT ON events TO app_role;

-- ============================================================
-- SNAPSHOTS (optional: cached aggregate state for performance)
-- ============================================================

CREATE TABLE snapshots (
    aggregate_type      VARCHAR(100) NOT NULL,
    aggregate_id        UUID NOT NULL,
    aggregate_version   INTEGER NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);

-- ============================================================
-- PROJECTION CHECKPOINTS (track which events each projector has processed)
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(100) PRIMARY KEY,
    last_global_position BIGINT NOT NULL DEFAULT 0,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Domain Events

### Cemetery & Location Events

```typescript
// TypeScript event type definitions

// --- Cemetery Aggregate ---
interface CemeteryCreated {
    type: 'CemeteryCreated';
    data: {
        cemetery_id: string;
        organisation_id: string;
        name: string;
        code: string;
        address: Address;
        boundary_geojson: GeoJSON.Polygon;
        centre_point: { lat: number; lng: number };
        timezone: string;
        established_date?: string;
    };
}

interface CemeteryBoundaryUpdated {
    type: 'CemeteryBoundaryUpdated';
    data: {
        cemetery_id: string;
        old_boundary_geojson: GeoJSON.Polygon;
        new_boundary_geojson: GeoJSON.Polygon;
        reason: string;
    };
}

// --- Section Aggregate ---
interface SectionCreated {
    type: 'SectionCreated';
    data: {
        section_id: string;
        cemetery_id: string;
        name: string;
        code: string;
        section_type: SectionType;
        faith_designation?: string;
        orientation_rule?: string;
        spacing_rule_cm?: number;
        boundary_geojson: GeoJSON.Polygon;
        fill_colour: string;
    };
}

interface SectionCultureRulesUpdated {
    type: 'SectionCultureRulesUpdated';
    data: {
        section_id: string;
        faith_designation: string;
        orientation_rule: string;
        spacing_rule_cm: number;
        cultural_notes: string;
    };
}

interface SectionClosed {
    type: 'SectionClosed';
    data: {
        section_id: string;
        reason: string;
        closed_date: string;
    };
}
```

### Space (Plot/Niche/Crypt) Events

```typescript
// --- Space Aggregate ---
interface SpaceCreated {
    type: 'SpaceCreated';
    data: {
        space_id: string;
        section_id: string;
        lot_id?: string;
        space_number: string;
        space_type: SpaceType;
        max_interments: number;
        location_geojson: GeoJSON.Point | GeoJSON.Polygon;
        gps_coordinates: { lat: number; lng: number };
        structure_position?: {
            row: string;
            column: string;
            level: string;
        };
    };
}

interface SpacePriceSet {
    type: 'SpacePriceSet';
    data: {
        space_id: string;
        base_price: number;
        currency: string;
        effective_date: string;
        previous_price?: number;
    };
}

interface SpaceReserved {
    type: 'SpaceReserved';
    data: {
        space_id: string;
        reserved_for_contact_id: string;
        reserved_until?: string;
        reason: string;
    };
}

interface SpaceSold {
    type: 'SpaceSold';
    data: {
        space_id: string;
        buyer_contact_id: string;
        contract_id: string;
        deed_number: string;
        deed_type: DeedType;
        sale_price: number;
        perpetual_care_amount: number;
        sale_date: string;
    };
}

interface SpaceOwnershipTransferred {
    type: 'SpaceOwnershipTransferred';
    data: {
        space_id: string;
        from_contact_id: string;
        to_contact_id: string;
        transfer_reason: string;
        new_deed_number: string;
        transfer_date: string;
    };
}

interface SpaceStatusChanged {
    type: 'SpaceStatusChanged';
    data: {
        space_id: string;
        old_status: SpaceStatus;
        new_status: SpaceStatus;
        reason: string;
    };
}
```

### Interment Events

```typescript
// --- Interment Aggregate ---
interface DeceasedRecorded {
    type: 'DeceasedRecorded';
    data: {
        deceased_id: string;
        first_name?: string;
        middle_name?: string;
        last_name?: string;
        maiden_name?: string;
        name_aliases: string[];
        date_of_birth?: string;
        date_of_birth_approx: boolean;
        date_of_death?: string;
        date_of_death_approx: boolean;
        gender?: string;
        death_certificate_number?: string;
        cause_of_death?: string;
        manner_of_death?: string;
        is_veteran: boolean;
        military_details?: MilitaryDetails;
    };
}

interface DeceasedRecordCorrected {
    type: 'DeceasedRecordCorrected';
    data: {
        deceased_id: string;
        corrections: Record<string, { old_value: any; new_value: any }>;
        correction_reason: string;
        correction_authority?: string;  // Who authorised the correction
    };
}

interface IntermentScheduled {
    type: 'IntermentScheduled';
    data: {
        interment_id: string;
        space_id: string;
        deceased_id: string;
        interment_type: IntermentType;
        scheduled_date: string;
        scheduled_time?: string;
        officiant_id?: string;
        funeral_home_id?: string;
        burial_permit_number?: string;
        vault_type?: string;
        casket_type?: string;
    };
}

interface IntermentRescheduled {
    type: 'IntermentRescheduled';
    data: {
        interment_id: string;
        old_date: string;
        new_date: string;
        new_time?: string;
        reason: string;
    };
}

interface IntermentCompleted {
    type: 'IntermentCompleted';
    data: {
        interment_id: string;
        actual_date: string;
        actual_time?: string;
        depth_inches?: number;
        completed_by: string;
        notes?: string;
    };
}

interface IntermentCancelled {
    type: 'IntermentCancelled';
    data: {
        interment_id: string;
        reason: string;
        cancelled_by: string;
    };
}

interface DisintermentAuthorised {
    type: 'DisintermentAuthorised';
    data: {
        interment_id: string;
        authorisation_number: string;
        authorised_by: string;
        reason: string;
        destination_space_id?: string;  // If reinterring within same cemetery
        destination_description?: string;  // If transferring elsewhere
    };
}

interface DisintermentCompleted {
    type: 'DisintermentCompleted';
    data: {
        interment_id: string;
        completed_date: string;
        completed_by: string;
    };
}
```

### Financial Events

```typescript
// --- Contract Aggregate ---
interface ContractCreated {
    type: 'ContractCreated';
    data: {
        contract_id: string;
        contract_number: string;
        cemetery_id: string;
        buyer_contact_id: string;
        contract_type: ContractType;
        line_items: ContractLineItem[];
        total_amount: number;
        perpetual_care_amount: number;
        tax_amount: number;
        payment_method: PaymentMethod;
        payment_plan_months?: number;
        monthly_payment?: number;
        currency: string;
    };
}

interface ContractSigned {
    type: 'ContractSigned';
    data: {
        contract_id: string;
        signed_date: string;
        signature_method: 'wet_ink' | 'electronic';
    };
}

interface PaymentReceived {
    type: 'PaymentReceived';
    data: {
        payment_id: string;
        contract_id?: string;
        invoice_id?: string;
        amount: number;
        payment_method: string;
        reference_number?: string;
        stripe_payment_id?: string;
        payment_date: string;
    };
}

interface InvoiceIssued {
    type: 'InvoiceIssued';
    data: {
        invoice_id: string;
        invoice_number: string;
        contract_id?: string;
        contact_id: string;
        cemetery_id: string;
        line_items: InvoiceLineItem[];
        subtotal: number;
        tax: number;
        total: number;
        due_date: string;
    };
}

// --- Perpetual Care Fund Aggregate ---
interface PerpetualCareContribution {
    type: 'PerpetualCareContribution';
    data: {
        fund_id: string;
        contract_id?: string;
        space_id?: string;
        amount: number;
        contribution_date: string;
    };
}

interface PerpetualCareInvestmentIncome {
    type: 'PerpetualCareInvestmentIncome';
    data: {
        fund_id: string;
        income_type: 'dividend' | 'interest' | 'capital_gain';
        amount: number;
        source: string;
        transaction_date: string;
    };
}

interface PerpetualCareDistribution {
    type: 'PerpetualCareDistribution';
    data: {
        fund_id: string;
        amount: number;
        purpose: string;
        approved_by: string;
        distribution_date: string;
    };
}
```

### Operations Events

```typescript
// --- Work Order Aggregate ---
interface WorkOrderCreated {
    type: 'WorkOrderCreated';
    data: {
        work_order_id: string;
        cemetery_id: string;
        work_order_number: string;
        work_type: WorkType;
        title: string;
        description?: string;
        priority: Priority;
        section_id?: string;
        lot_id?: string;
        space_id?: string;
        due_date?: string;
        ai_priority_score?: number;
        ai_priority_reason?: string;
    };
}

interface WorkOrderAssigned {
    type: 'WorkOrderAssigned';
    data: {
        work_order_id: string;
        assigned_to: string;
        assigned_date: string;
    };
}

interface WorkOrderCompleted {
    type: 'WorkOrderCompleted';
    data: {
        work_order_id: string;
        completed_date: string;
        actual_hours: number;
        material_cost?: number;
        notes?: string;
    };
}

// --- Document Aggregate ---
interface DocumentUploaded {
    type: 'DocumentUploaded';
    data: {
        document_id: string;
        cemetery_id: string;
        attached_to: { type: string; id: string };
        document_type: DocumentType;
        title: string;
        file_name: string;
        file_path: string;
        file_size: number;
        mime_type: string;
        is_public: boolean;
    };
}

interface DocumentOCRCompleted {
    type: 'DocumentOCRCompleted';
    data: {
        document_id: string;
        ocr_text: string;
        ocr_confidence: number;
        extracted_entities?: ExtractedEntity[];
    };
}

// --- Memorial Page Aggregate ---
interface MemorialPagePublished {
    type: 'MemorialPagePublished';
    data: {
        memorial_id: string;
        deceased_id: string;
        slug: string;
        headline?: string;
        tribute_text?: string;
    };
}

interface MemorialTributeSubmitted {
    type: 'MemorialTributeSubmitted';
    data: {
        tribute_id: string;
        memorial_id: string;
        contributor_name: string;
        contributor_email?: string;
        relationship?: string;
        tribute_text: string;
        photo_url?: string;
    };
}

interface MemorialTributeModerated {
    type: 'MemorialTributeModerated';
    data: {
        tribute_id: string;
        memorial_id: string;
        decision: 'approved' | 'rejected';
        moderated_by: string;
        reason?: string;
    };
}
```

---

## Read Model Projections

Each projection is a separate database table (or index) that is built and kept up-to-date by a projector process consuming the event stream.

### Projection 1: Map Read Model (PostGIS)

```sql
-- ============================================================
-- MAP PROJECTION: Spatial data for cemetery map rendering
-- ============================================================

-- Projected from: CemeteryCreated, CemeteryBoundaryUpdated, SectionCreated,
--                 SpaceCreated, SpaceStatusChanged, SpaceSold, IntermentCompleted

CREATE TABLE map_cemeteries (
    cemetery_id     UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    boundary        GEOMETRY(POLYGON, 4326),
    centre_point    GEOMETRY(POINT, 4326),
    default_zoom    SMALLINT,
    tile_url        VARCHAR(500),
    status          VARCHAR(20),
    last_event_position BIGINT NOT NULL  -- Track which event built this
);

CREATE TABLE map_sections (
    section_id      UUID PRIMARY KEY,
    cemetery_id     UUID NOT NULL,
    name            VARCHAR(100),
    code            VARCHAR(20),
    section_type    VARCHAR(50),
    faith_designation VARCHAR(100),
    boundary        GEOMETRY(POLYGON, 4326),
    fill_colour     VARCHAR(7),
    status          VARCHAR(20),
    total_spaces    INTEGER DEFAULT 0,
    available_spaces INTEGER DEFAULT 0,
    occupied_spaces INTEGER DEFAULT 0,
    last_event_position BIGINT NOT NULL
);

CREATE TABLE map_spaces (
    space_id        UUID PRIMARY KEY,
    section_id      UUID NOT NULL,
    lot_id          UUID,
    space_number    VARCHAR(20),
    space_type      VARCHAR(30),
    status          VARCHAR(20),  -- 'available', 'sold', 'reserved', 'occupied', 'full'
    location_point  GEOMETRY(POINT, 4326),
    location_polygon GEOMETRY(POLYGON, 4326),
    gps_latitude    DOUBLE PRECISION,
    gps_longitude   DOUBLE PRECISION,
    -- Denormalised for map popup display
    owner_name      VARCHAR(300),
    interment_count INTEGER DEFAULT 0,
    latest_interment_name VARCHAR(300),
    latest_interment_date DATE,
    last_event_position BIGINT NOT NULL
);

CREATE INDEX idx_map_sections_boundary ON map_sections USING GIST(boundary);
CREATE INDEX idx_map_spaces_point ON map_spaces USING GIST(location_point);
CREATE INDEX idx_map_spaces_polygon ON map_spaces USING GIST(location_polygon);
CREATE INDEX idx_map_spaces_status ON map_spaces(status);
```

### Projection 2: Genealogy Search Read Model

```sql
-- ============================================================
-- SEARCH PROJECTION: Denormalised for fast burial search queries
-- ============================================================

-- Projected from: DeceasedRecorded, DeceasedRecordCorrected, IntermentCompleted,
--                 MemorialPagePublished, SpaceCreated, SectionCreated

CREATE TABLE search_burials (
    deceased_id         UUID PRIMARY KEY,
    -- Name fields (all variants for search)
    first_name          VARCHAR(100),
    middle_name         VARCHAR(100),
    last_name           VARCHAR(100),
    maiden_name         VARCHAR(100),
    display_name        VARCHAR(300),
    name_aliases        TEXT[],
    -- Phonetic codes (pre-computed for fast phonetic search)
    last_name_soundex   VARCHAR(10),
    last_name_metaphone VARCHAR(20),
    last_name_dmetaphone VARCHAR(20),
    first_name_soundex  VARCHAR(10),
    first_name_metaphone VARCHAR(20),
    -- Dates
    date_of_birth       DATE,
    date_of_death       DATE,
    interment_date      DATE,
    -- Location (denormalised)
    cemetery_id         UUID,
    cemetery_name       VARCHAR(255),
    section_name        VARCHAR(100),
    section_code        VARCHAR(20),
    lot_number          VARCHAR(20),
    space_number        VARCHAR(20),
    -- GPS for walk-to-grave
    gps_latitude        DOUBLE PRECISION,
    gps_longitude       DOUBLE PRECISION,
    -- Memorial
    memorial_slug       VARCHAR(200),
    has_memorial        BOOLEAN DEFAULT false,
    has_photo           BOOLEAN DEFAULT false,
    -- Veterans
    is_veteran          BOOLEAN DEFAULT false,
    military_branch     VARCHAR(100),
    -- Search vectors
    search_vector       TSVECTOR,
    last_event_position BIGINT NOT NULL
);

CREATE INDEX idx_search_last_name ON search_burials(last_name);
CREATE INDEX idx_search_dates ON search_burials(date_of_death, date_of_birth);
CREATE INDEX idx_search_cemetery ON search_burials(cemetery_id);
CREATE INDEX idx_search_vector ON search_burials USING GIN(search_vector);
CREATE INDEX idx_search_name_trgm ON search_burials USING GIN(
    (COALESCE(first_name, '') || ' ' || COALESCE(last_name, '')) gin_trgm_ops
);
CREATE INDEX idx_search_soundex ON search_burials(last_name_soundex);
CREATE INDEX idx_search_metaphone ON search_burials(last_name_metaphone);
```

For even more powerful fuzzy search, the genealogy search can also be projected into OpenSearch/Elasticsearch:

```json
{
    "mappings": {
        "properties": {
            "deceased_id":      { "type": "keyword" },
            "first_name":       { "type": "text", "analyzer": "name_analyzer" },
            "last_name":        { "type": "text", "analyzer": "name_analyzer" },
            "maiden_name":      { "type": "text", "analyzer": "name_analyzer" },
            "display_name":     { "type": "text", "analyzer": "name_analyzer" },
            "name_aliases":     { "type": "text", "analyzer": "name_analyzer" },
            "date_of_birth":    { "type": "date", "format": "yyyy-MM-dd" },
            "date_of_death":    { "type": "date", "format": "yyyy-MM-dd" },
            "interment_date":   { "type": "date", "format": "yyyy-MM-dd" },
            "cemetery_name":    { "type": "keyword" },
            "section_name":     { "type": "keyword" },
            "space_number":     { "type": "keyword" },
            "is_veteran":       { "type": "boolean" },
            "memorial_slug":    { "type": "keyword" },
            "location":         { "type": "geo_point" },
            "suggest":          { "type": "completion" }
        }
    },
    "settings": {
        "analysis": {
            "analyzer": {
                "name_analyzer": {
                    "tokenizer": "standard",
                    "filter": ["lowercase", "asciifolding", "phonetic_filter", "name_synonym"]
                }
            },
            "filter": {
                "phonetic_filter": {
                    "type": "phonetic",
                    "encoder": "double_metaphone",
                    "replace": false
                }
            }
        }
    }
}
```

### Projection 3: Financial Read Model

```sql
-- ============================================================
-- FINANCE PROJECTION: Contract and payment status
-- ============================================================

-- Projected from: ContractCreated, ContractSigned, PaymentReceived,
--                 InvoiceIssued, PerpetualCareContribution, etc.

CREATE TABLE finance_contracts (
    contract_id         UUID PRIMARY KEY,
    contract_number     VARCHAR(50) NOT NULL,
    cemetery_id         UUID NOT NULL,
    buyer_name          VARCHAR(300),
    buyer_contact_id    UUID,
    contract_type       VARCHAR(30),
    contract_date       DATE,
    signed_date         DATE,
    total_amount        NUMERIC(12, 2),
    perpetual_care_amount NUMERIC(12, 2),
    total_paid          NUMERIC(12, 2) DEFAULT 0,
    balance_due         NUMERIC(12, 2),
    payment_method      VARCHAR(30),
    status              VARCHAR(20),
    last_payment_date   DATE,
    next_payment_due    DATE,
    last_event_position BIGINT NOT NULL
);

CREATE TABLE finance_payments (
    payment_id          UUID PRIMARY KEY,
    contract_id         UUID,
    invoice_id          UUID,
    amount              NUMERIC(12, 2),
    payment_method      VARCHAR(30),
    payment_date        DATE,
    reference_number    VARCHAR(100),
    last_event_position BIGINT NOT NULL
);

CREATE TABLE finance_perpetual_care (
    fund_id             UUID PRIMARY KEY,
    cemetery_id         UUID NOT NULL,
    fund_name           VARCHAR(255),
    total_corpus        NUMERIC(14, 2) DEFAULT 0,
    total_income        NUMERIC(14, 2) DEFAULT 0,
    total_distributions NUMERIC(14, 2) DEFAULT 0,
    current_balance     NUMERIC(14, 2) DEFAULT 0,
    contribution_count  INTEGER DEFAULT 0,
    last_event_position BIGINT NOT NULL
);

CREATE INDEX idx_finance_contracts_cemetery ON finance_contracts(cemetery_id);
CREATE INDEX idx_finance_contracts_status ON finance_contracts(status);
CREATE INDEX idx_finance_payments_contract ON finance_payments(contract_id);
```

### Projection 4: Operations Read Model

```sql
-- ============================================================
-- OPERATIONS PROJECTION: Work orders, scheduling, inventory
-- ============================================================

CREATE TABLE ops_work_orders (
    work_order_id       UUID PRIMARY KEY,
    cemetery_id         UUID NOT NULL,
    work_order_number   VARCHAR(50),
    work_type           VARCHAR(50),
    title               VARCHAR(255),
    priority            VARCHAR(10),
    status              VARCHAR(20),
    assigned_to_name    VARCHAR(200),
    assigned_to_id      UUID,
    scheduled_date      DATE,
    due_date            DATE,
    completed_date      DATE,
    section_name        VARCHAR(100),
    space_number        VARCHAR(20),
    actual_hours        NUMERIC(6, 2),
    material_cost       NUMERIC(10, 2),
    last_event_position BIGINT NOT NULL
);

CREATE TABLE ops_interment_schedule (
    interment_id        UUID PRIMARY KEY,
    space_id            UUID NOT NULL,
    deceased_name       VARCHAR(300),
    interment_type      VARCHAR(30),
    scheduled_date      DATE,
    scheduled_time      TIME,
    cemetery_id         UUID,
    section_name        VARCHAR(100),
    space_number        VARCHAR(20),
    funeral_home_name   VARCHAR(255),
    officiant_name      VARCHAR(200),
    status              VARCHAR(20),
    last_event_position BIGINT NOT NULL
);

CREATE TABLE ops_inventory_summary (
    cemetery_id         UUID NOT NULL,
    section_id          UUID NOT NULL,
    section_name        VARCHAR(100),
    section_type        VARCHAR(50),
    total_spaces        INTEGER DEFAULT 0,
    available           INTEGER DEFAULT 0,
    sold                INTEGER DEFAULT 0,
    reserved            INTEGER DEFAULT 0,
    occupied            INTEGER DEFAULT 0,
    full                INTEGER DEFAULT 0,
    unavailable         INTEGER DEFAULT 0,
    last_event_position BIGINT NOT NULL,
    PRIMARY KEY (cemetery_id, section_id)
);

CREATE INDEX idx_ops_wo_cemetery ON ops_work_orders(cemetery_id);
CREATE INDEX idx_ops_wo_status ON ops_work_orders(status);
CREATE INDEX idx_ops_schedule_date ON ops_interment_schedule(scheduled_date);
```

---

## Command Handlers (Write Side)

```typescript
// Example: Command handler for selling a space

interface SellSpaceCommand {
    space_id: string;
    buyer_contact_id: string;
    contract_details: {
        total_amount: number;
        perpetual_care_amount: number;
        payment_method: PaymentMethod;
        payment_plan_months?: number;
    };
    deed_type: DeedType;
}

async function handleSellSpace(cmd: SellSpaceCommand, eventStore: EventStore): Promise<void> {
    // 1. Load aggregate from event stream
    const spaceEvents = await eventStore.loadEvents('Space', cmd.space_id);
    const spaceState = replaySpaceEvents(spaceEvents);

    // 2. Validate business rules
    if (spaceState.status !== 'available' && spaceState.status !== 'reserved') {
        throw new BusinessRuleViolation(`Space ${cmd.space_id} is not available for sale (status: ${spaceState.status})`);
    }

    // 3. Generate new events
    const contractId = generateUUID();
    const contractNumber = await generateContractNumber(spaceState.cemetery_id);
    const deedNumber = await generateDeedNumber(spaceState.cemetery_id);

    const events: DomainEvent[] = [
        {
            type: 'ContractCreated',
            aggregate_type: 'Contract',
            aggregate_id: contractId,
            data: {
                contract_id: contractId,
                contract_number: contractNumber,
                cemetery_id: spaceState.cemetery_id,
                buyer_contact_id: cmd.buyer_contact_id,
                contract_type: 'at_need',
                line_items: [{
                    space_id: cmd.space_id,
                    item_type: 'plot_purchase',
                    unit_price: cmd.contract_details.total_amount,
                    quantity: 1
                }],
                ...cmd.contract_details
            }
        },
        {
            type: 'SpaceSold',
            aggregate_type: 'Space',
            aggregate_id: cmd.space_id,
            data: {
                space_id: cmd.space_id,
                buyer_contact_id: cmd.buyer_contact_id,
                contract_id: contractId,
                deed_number: deedNumber,
                deed_type: cmd.deed_type,
                sale_price: cmd.contract_details.total_amount,
                perpetual_care_amount: cmd.contract_details.perpetual_care_amount,
                sale_date: new Date().toISOString().split('T')[0]
            }
        }
    ];

    // 4. Append events atomically
    await eventStore.appendEvents(events, spaceState.version);
    // Projectors will pick up these events asynchronously and update read models
}
```

---

## Event Store Partitioning Strategy

```sql
-- For 100+ year retention, partition the event store by year
-- Each partition can be individually compressed, archived, or moved to cold storage

CREATE TABLE events (
    global_position     BIGSERIAL,
    event_id            UUID NOT NULL DEFAULT gen_random_uuid(),
    aggregate_type      VARCHAR(100) NOT NULL,
    aggregate_id        UUID NOT NULL,
    aggregate_version   INTEGER NOT NULL,
    event_type          VARCHAR(200) NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_type, aggregate_id, aggregate_version)
) PARTITION BY RANGE (created_at);

-- Create partitions per year (automate via cron or pg_partman)
CREATE TABLE events_2024 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE events_2025 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE events_2026 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
-- ... (automated creation of future partitions)
```

---

## Pros and Cons

### Pros

1. **Perfect audit trail by design**: Every state change is an immutable event. There is no separate audit log to maintain -- the event store *is* the audit trail. This directly satisfies the project's requirement for "immutable audit trail architecture for burial records."

2. **Time-travel queries**: Reconstructing the state of any entity at any point in time is trivial -- replay events up to that timestamp. This is invaluable for legal disputes ("who owned this plot on 3 March 2005?") and regulatory audits.

3. **Temporal debugging**: When a data discrepancy is found (e.g., two families both believe they own the same plot), the complete event history reveals exactly when and how the conflict arose.

4. **Independent read model scaling**: The public genealogy portal can have its own read model (OpenSearch) scaled independently from the admin write path. High traffic on the public portal does not affect administrative operations.

5. **Flexible projections**: New query requirements (e.g., a new report for regulatory compliance) can be added by creating a new projector that replays the existing event stream, without altering the event store or any existing projection.

6. **Natural fit for the OCR pipeline**: Scanned ledger digitisation naturally produces events (DocumentUploaded, DocumentOCRCompleted, DeceasedRecorded). The event-sourced model handles this async, multi-step workflow elegantly.

7. **GDPR compliance advantage**: "Right to erasure" for living contacts can be implemented via crypto-shredding: encrypt PII in event payloads with per-contact keys; delete the key to make the data unreadable without altering the event stream.

### Cons

1. **Complexity**: Event sourcing is significantly more complex than CRUD. Developers must understand aggregates, event replay, projections, eventual consistency, and idempotency. The talent pool is smaller than for traditional PostgreSQL development.

2. **Eventual consistency**: Read models are updated asynchronously. After a space is sold, the map might show it as "available" for a brief moment until the projection catches up. For cemetery operations (where transactions happen at a pace of minutes, not milliseconds), this is typically acceptable, but it must be designed for.

3. **Event schema evolution**: Over 100+ years, event schemas will inevitably change. Events stored in 2026 must still be readable in 2126. This requires careful versioning strategies (upcasting, event adapters) that add ongoing maintenance burden.

4. **Projection rebuild time**: If a projection is corrupted or a new one is added, it must be rebuilt by replaying the entire event history. For a large cemetery with decades of history, this could take hours.

5. **Increased storage**: Storing every event rather than just current state uses significantly more disk space. A single plot that has been created, priced, sold, and had two interments stores 7+ events versus one row in a traditional model.

6. **Operational complexity**: Running the event store, message broker (NATS/Kafka), multiple read model databases (PostgreSQL, PostGIS, OpenSearch), and projection processes requires more operational expertise and monitoring than a single PostgreSQL instance.

7. **Overkill for simple queries**: Looking up "what is the current owner of plot A-107?" requires either a projection lookup (eventual consistency) or event replay (expensive for simple queries). Snapshots mitigate this but add another moving part.

---

## Migration & Scaling Considerations

### Migrating from Legacy Data

1. **Import as "historical" events**: Legacy records from CSV/spreadsheet imports are converted into synthetic events with backdated timestamps. E.g., a burial from 1955 becomes a `DeceasedRecorded` event with `created_at = '1955-03-15'` and a `IntermentCompleted` event.

2. **Bulk import batching**: Large imports (50,000+ records) should batch events and rebuild projections once after the full import, rather than processing projections in real-time during import.

3. **Data quality events**: Where legacy data is ambiguous, create `DeceasedRecordCorrected` events with metadata indicating "imported from legacy ledger, unverified."

### Scaling Strategies

| Growth stage | Strategy |
|-------------|----------|
| Single cemetery | Single PostgreSQL event store + single read model DB; no message broker needed (in-process projections) |
| Multi-site (5-20 cemeteries) | Add NATS JetStream for async projections; separate read model instances for portal vs. admin |
| Large operator | Shard event store by aggregate_type; dedicated OpenSearch cluster for genealogy search |
| Enterprise | Kafka for event streaming; per-cemetery event store partitions; global search projection across all cemeteries |

### Long-Term Archival

- **Event store partitions** by year enable cold storage of old partitions (e.g., events from 1950-2000 on compressed, archival-tier storage)
- **Snapshots** reduce replay time: snapshot each aggregate every N events, so replays start from the snapshot rather than event #1
- **Event schema versioning** via upcasters: when reading old events, an upcaster transforms them to the current schema version before processing

### Disaster Recovery

- **Event store is the source of truth**: If all projections are lost, they can be rebuilt by replaying the event stream
- **WAL archiving + streaming replication** for the event store PostgreSQL instance
- **Projection checkpoints** enable resumable projection rebuilds after failures

---

## Summary

Event sourcing is the architecturally purest solution for cemetery management's core requirement: an immutable, legally defensible audit trail spanning 100+ years. Every change is permanently recorded, temporal queries are trivial, and read models can be independently optimised for each use case (map, search, finance, operations). The trade-off is substantially higher architectural and operational complexity. This model is best suited if the project team has event sourcing experience and expects to operate at a scale (multi-site, high genealogy portal traffic) that justifies the infrastructure investment. For a small single-cemetery deployment, the complexity may be disproportionate to the benefit.
