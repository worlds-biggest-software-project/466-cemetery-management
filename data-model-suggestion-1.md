# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL + PostGIS)

> Project: Cemetery Management (466) · Generated: 2026-05-26

---

## Overview

This model uses a fully normalized relational schema on PostgreSQL with the PostGIS extension for geospatial data. Every entity occupies its own table with foreign key relationships, check constraints, and referential integrity enforced at the database level. PostGIS geometry columns store plot boundaries, section polygons, and point-of-interest locations natively, enabling spatial queries (containment, intersection, nearest-neighbour) directly in SQL.

This is the most conventional and broadly understood approach. It aligns directly with the project README's recommendation of "PostgreSQL/PostGIS as the recommended spatial database backend" and suits the domain's emphasis on data integrity, legal compliance, and long-term archival retention.

---

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| RDBMS | PostgreSQL 16+ | Mature, open-source, excellent PostGIS integration, JSONB support for semi-structured data |
| Spatial Extension | PostGIS 3.4+ | Industry-standard spatial extension; OGC-compliant; supports WKT, WKB, GeoJSON import/export |
| Full-Text Search | PostgreSQL tsvector/tsquery + pg_trgm | Built-in fuzzy and phonetic search for genealogy portal |
| Phonetic Search | fuzzystrmatch (soundex, metaphone, dmetaphone) | PostgreSQL extension for phonetic name matching across historic spelling variations |
| Connection Pool | PgBouncer | Connection pooling for the public genealogy portal tier |
| Migrations | Flyway or golang-migrate | Versioned, repeatable schema migrations |
| Backup | pg_dump + WAL archiving + pg_basebackup | Point-in-time recovery for 100+ year archival requirements |

---

## Schema Definition

### Core Location Hierarchy

The cemetery location model follows the industry-standard hierarchy: Organisation > Cemetery > Section > Lot > Space (plot/niche/crypt). Each level is a separate table with a parent foreign key.

```sql
-- ============================================================
-- ORGANISATIONS & CEMETERIES
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL CHECK (org_type IN (
                        'municipal', 'religious', 'private', 'memorial_park', 'veterans', 'other'
                    )),
    tax_id          VARCHAR(50),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    phone           VARCHAR(30),
    email           VARCHAR(255),
    website         VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cemeteries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(20) UNIQUE NOT NULL,  -- Short identifier (e.g., "OLV", "ROSE")
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    -- Geospatial: overall cemetery boundary polygon in WGS 84
    boundary        GEOMETRY(POLYGON, 4326),
    -- Centre point for map initialisation
    centre_point    GEOMETRY(POINT, 4326),
    default_zoom    SMALLINT DEFAULT 18,
    -- Aerial imagery tile URL template (e.g., Mapbox, custom tile server)
    tile_url        VARCHAR(500),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    established_date DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'closed', 'historical')),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cemeteries_organisation ON cemeteries(organisation_id);
CREATE INDEX idx_cemeteries_boundary ON cemeteries USING GIST(boundary);
```

```sql
-- ============================================================
-- SECTIONS
-- ============================================================

CREATE TABLE sections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cemetery_id     UUID NOT NULL REFERENCES cemeteries(id),
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    section_type    VARCHAR(50) NOT NULL CHECK (section_type IN (
                        'ground_burial', 'cremation_garden', 'columbarium',
                        'mausoleum', 'veterans', 'infant_child',
                        'memorial_wall', 'scattering_garden', 'natural_burial', 'other'
                    )),
    -- Multi-faith / cultural configuration
    faith_designation   VARCHAR(100),  -- e.g., 'Catholic', 'Jewish', 'Muslim', 'Non-denominational'
    orientation_rule    VARCHAR(50),   -- e.g., 'east_facing', 'mecca_facing', 'none'
    spacing_rule_cm     INTEGER,       -- Minimum spacing between interments in centimetres
    -- Geospatial: section boundary polygon
    boundary        GEOMETRY(POLYGON, 4326),
    -- Visual styling for map rendering
    fill_colour     VARCHAR(7) DEFAULT '#CCCCCC',  -- Hex colour
    sort_order      INTEGER DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'closed', 'full', 'reserved')),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(cemetery_id, code)
);

CREATE INDEX idx_sections_cemetery ON sections(cemetery_id);
CREATE INDEX idx_sections_boundary ON sections USING GIST(boundary);
CREATE INDEX idx_sections_type ON sections(section_type);
```

```sql
-- ============================================================
-- LOTS (grouping of spaces, typically sold as a unit)
-- ============================================================

CREATE TABLE lots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES sections(id),
    lot_number      VARCHAR(20) NOT NULL,
    -- Geospatial: lot boundary polygon
    boundary        GEOMETRY(POLYGON, 4326),
    total_spaces    INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'available' CHECK (status IN (
                        'available', 'sold', 'reserved', 'unavailable', 'historical'
                    )),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(section_id, lot_number)
);

CREATE INDEX idx_lots_section ON lots(section_id);
CREATE INDEX idx_lots_boundary ON lots USING GIST(boundary);
CREATE INDEX idx_lots_status ON lots(status);
```

```sql
-- ============================================================
-- SPACES (individual burial/niche/crypt positions)
-- ============================================================

CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lot_id          UUID REFERENCES lots(id),          -- NULL for standalone niches/crypts
    section_id      UUID NOT NULL REFERENCES sections(id),
    space_number    VARCHAR(20) NOT NULL,
    space_type      VARCHAR(30) NOT NULL CHECK (space_type IN (
                        'ground_single', 'ground_double', 'ground_infant',
                        'cremation_ground', 'columbarium_niche', 'mausoleum_crypt',
                        'mausoleum_crypt_double', 'ossuary', 'memorial_only', 'other'
                    )),
    -- Niche/crypt-specific: row, column, level within structure
    structure_row       VARCHAR(10),
    structure_column    VARCHAR(10),
    structure_level     VARCHAR(10),
    -- Geospatial: exact position (point for niches, polygon for ground plots)
    location_point  GEOMETRY(POINT, 4326),
    location_polygon GEOMETRY(POLYGON, 4326),
    -- Capacity: how many interments this space can hold
    max_interments  SMALLINT NOT NULL DEFAULT 1,
    current_interments SMALLINT NOT NULL DEFAULT 0,
    -- Pricing
    base_price      NUMERIC(12, 2),
    currency_code   CHAR(3) DEFAULT 'USD',
    -- Status
    status          VARCHAR(20) NOT NULL DEFAULT 'available' CHECK (status IN (
                        'available', 'sold', 'reserved', 'occupied', 'full',
                        'unavailable', 'condemned', 'historical'
                    )),
    -- GPS coordinates for walk-to-grave navigation
    gps_latitude    DOUBLE PRECISION,
    gps_longitude   DOUBLE PRECISION,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(section_id, space_number)
);

CREATE INDEX idx_spaces_lot ON spaces(lot_id);
CREATE INDEX idx_spaces_section ON spaces(section_id);
CREATE INDEX idx_spaces_status ON spaces(status);
CREATE INDEX idx_spaces_type ON spaces(space_type);
CREATE INDEX idx_spaces_location_point ON spaces USING GIST(location_point);
CREATE INDEX idx_spaces_location_polygon ON spaces USING GIST(location_polygon);
```

### People & Contacts

```sql
-- ============================================================
-- CONTACTS (plot owners, next of kin, funeral homes, officiants)
-- ============================================================

CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_type    VARCHAR(30) NOT NULL CHECK (contact_type IN (
                        'individual', 'funeral_home', 'officiant', 'monument_company',
                        'government_agency', 'other_organisation'
                    )),
    -- Individual fields
    first_name      VARCHAR(100),
    middle_name     VARCHAR(100),
    last_name       VARCHAR(100),
    suffix          VARCHAR(20),      -- Jr., Sr., III
    -- Organisation fields
    organisation_name VARCHAR(255),
    -- Contact details
    email           VARCHAR(255),
    phone_primary   VARCHAR(30),
    phone_secondary VARCHAR(30),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) DEFAULT 'US',
    -- For genealogy search / GEDCOM export
    display_name    VARCHAR(300) GENERATED ALWAYS AS (
                        COALESCE(organisation_name,
                            TRIM(COALESCE(first_name, '') || ' ' || COALESCE(middle_name, '') || ' ' || COALESCE(last_name, '') || ' ' || COALESCE(suffix, ''))
                        )
                    ) STORED,
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_type ON contacts(contact_type);
CREATE INDEX idx_contacts_last_name ON contacts(last_name);
CREATE INDEX idx_contacts_display_name ON contacts(display_name);
-- Full-text search index for fuzzy name matching
CREATE INDEX idx_contacts_name_fts ON contacts USING GIN(
    to_tsvector('simple', COALESCE(first_name, '') || ' ' || COALESCE(middle_name, '') || ' ' || COALESCE(last_name, ''))
);
-- Trigram index for phonetic/fuzzy search
CREATE INDEX idx_contacts_name_trgm ON contacts USING GIN(
    (COALESCE(first_name, '') || ' ' || COALESCE(last_name, '')) gin_trgm_ops
);
```

### Deceased & Interment Records

```sql
-- ============================================================
-- DECEASED (the person who is interred)
-- ============================================================

CREATE TABLE deceased (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Name fields (historic records may have incomplete data)
    first_name          VARCHAR(100),
    middle_name         VARCHAR(100),
    last_name           VARCHAR(100),
    maiden_name         VARCHAR(100),
    suffix              VARCHAR(20),
    -- Alternative name spellings for historic fuzzy search
    name_aliases        TEXT[],   -- Array of known alternative spellings
    -- Dates (partial dates allowed for historic records)
    date_of_birth       DATE,
    date_of_birth_approx BOOLEAN DEFAULT false,
    date_of_death       DATE,
    date_of_death_approx BOOLEAN DEFAULT false,
    -- Demographics
    gender              VARCHAR(20) CHECK (gender IN ('male', 'female', 'other', 'unknown')),
    -- VRDR / vital records fields
    ssn_last_four       CHAR(4),
    death_certificate_number VARCHAR(50),
    cause_of_death      VARCHAR(500),
    manner_of_death     VARCHAR(50) CHECK (manner_of_death IN (
                            'natural', 'accident', 'suicide', 'homicide',
                            'pending_investigation', 'could_not_be_determined', 'unknown', NULL
                        )),
    -- Military / veteran status
    is_veteran          BOOLEAN DEFAULT false,
    military_branch     VARCHAR(100),
    military_rank       VARCHAR(100),
    military_service_dates VARCHAR(100),
    -- Biographical
    obituary_text       TEXT,
    biography           TEXT,
    -- Search optimisation
    display_name        VARCHAR(300) GENERATED ALWAYS AS (
                            TRIM(COALESCE(first_name, '') || ' ' || COALESCE(middle_name, '') || ' ' || COALESCE(last_name, ''))
                        ) STORED,
    -- Full-text search vector for genealogy portal
    search_vector       TSVECTOR GENERATED ALWAYS AS (
                            to_tsvector('simple',
                                COALESCE(first_name, '') || ' ' ||
                                COALESCE(middle_name, '') || ' ' ||
                                COALESCE(last_name, '') || ' ' ||
                                COALESCE(maiden_name, '') || ' ' ||
                                COALESCE(array_to_string(name_aliases, ' '), '')
                            )
                        ) STORED,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deceased_last_name ON deceased(last_name);
CREATE INDEX idx_deceased_dates ON deceased(date_of_death, date_of_birth);
CREATE INDEX idx_deceased_search_vector ON deceased USING GIN(search_vector);
CREATE INDEX idx_deceased_name_trgm ON deceased USING GIN(
    (COALESCE(first_name, '') || ' ' || COALESCE(last_name, '')) gin_trgm_ops
);
CREATE INDEX idx_deceased_death_cert ON deceased(death_certificate_number) WHERE death_certificate_number IS NOT NULL;
```

```sql
-- ============================================================
-- INTERMENTS (the act of burying or placing remains in a space)
-- ============================================================

CREATE TABLE interments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id            UUID NOT NULL REFERENCES spaces(id),
    deceased_id         UUID NOT NULL REFERENCES deceased(id),
    -- Interment details
    interment_type      VARCHAR(30) NOT NULL CHECK (interment_type IN (
                            'casket_burial', 'cremation_inurnment', 'cremation_ground_burial',
                            'entombment', 'disinterment', 'reinterment',
                            'scattering', 'green_burial', 'other'
                        )),
    interment_date      DATE NOT NULL,
    interment_time      TIME,
    -- Officiants and funeral home
    officiant_id        UUID REFERENCES contacts(id),
    funeral_home_id     UUID REFERENCES contacts(id),
    -- Permits and legal
    burial_permit_number VARCHAR(50),
    burial_permit_date  DATE,
    -- Physical details
    depth_inches        INTEGER,
    vault_type          VARCHAR(100),
    casket_type         VARCHAR(100),
    -- Scheduling
    scheduled_date      DATE,
    scheduled_time      TIME,
    actual_date         DATE,
    actual_time         TIME,
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'scheduled' CHECK (status IN (
                            'scheduled', 'confirmed', 'completed', 'cancelled', 'pending_permit'
                        )),
    notes               TEXT,
    created_by          UUID,  -- Staff user ID
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_interments_space ON interments(space_id);
CREATE INDEX idx_interments_deceased ON interments(deceased_id);
CREATE INDEX idx_interments_date ON interments(interment_date);
CREATE INDEX idx_interments_status ON interments(status);
CREATE INDEX idx_interments_funeral_home ON interments(funeral_home_id);
```

### Ownership & Deeds

```sql
-- ============================================================
-- OWNERSHIP RIGHTS (who owns or has rights to a space/lot)
-- ============================================================

CREATE TABLE ownership_rights (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Can be tied to a lot or individual space
    lot_id              UUID REFERENCES lots(id),
    space_id            UUID REFERENCES spaces(id),
    owner_contact_id    UUID NOT NULL REFERENCES contacts(id),
    -- Deed details
    deed_number         VARCHAR(50),
    deed_date           DATE,
    deed_type           VARCHAR(30) NOT NULL CHECK (deed_type IN (
                            'perpetual', 'term_lease', 'pre_need', 'at_need',
                            'transfer', 'inheritance', 'donation', 'other'
                        )),
    -- Term lease expiry
    lease_start_date    DATE,
    lease_end_date      DATE,
    -- Transfer history
    transferred_from_id UUID REFERENCES ownership_rights(id),  -- Previous ownership record
    transfer_date       DATE,
    transfer_reason     VARCHAR(100),
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN (
                            'active', 'transferred', 'expired', 'revoked', 'pending'
                        )),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (lot_id IS NOT NULL OR space_id IS NOT NULL)
);

CREATE INDEX idx_ownership_lot ON ownership_rights(lot_id);
CREATE INDEX idx_ownership_space ON ownership_rights(space_id);
CREATE INDEX idx_ownership_owner ON ownership_rights(owner_contact_id);
CREATE INDEX idx_ownership_deed ON ownership_rights(deed_number);
CREATE INDEX idx_ownership_status ON ownership_rights(status);
```

### Financial Management

```sql
-- ============================================================
-- CONTRACTS (sales contracts for spaces/lots)
-- ============================================================

CREATE TABLE contracts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_number     VARCHAR(50) UNIQUE NOT NULL,
    cemetery_id         UUID NOT NULL REFERENCES cemeteries(id),
    buyer_contact_id    UUID NOT NULL REFERENCES contacts(id),
    -- Contract type
    contract_type       VARCHAR(30) NOT NULL CHECK (contract_type IN (
                            'at_need', 'pre_need', 'transfer', 'upgrade', 'other'
                        )),
    contract_date       DATE NOT NULL,
    -- Financials
    total_amount        NUMERIC(12, 2) NOT NULL,
    perpetual_care_amount NUMERIC(12, 2) DEFAULT 0,
    tax_amount          NUMERIC(12, 2) DEFAULT 0,
    discount_amount     NUMERIC(12, 2) DEFAULT 0,
    currency_code       CHAR(3) DEFAULT 'USD',
    -- Payment terms
    payment_method      VARCHAR(30) CHECK (payment_method IN (
                            'full_payment', 'payment_plan', 'pre_need_trust', 'other'
                        )),
    payment_plan_months INTEGER,
    monthly_payment     NUMERIC(12, 2),
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN (
                            'draft', 'pending', 'active', 'completed', 'cancelled', 'defaulted'
                        )),
    signed_date         DATE,
    notes               TEXT,
    created_by          UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE contract_line_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(id) ON DELETE CASCADE,
    space_id            UUID REFERENCES spaces(id),
    lot_id              UUID REFERENCES lots(id),
    description         VARCHAR(500) NOT NULL,
    item_type           VARCHAR(30) NOT NULL CHECK (item_type IN (
                            'plot_purchase', 'niche_purchase', 'crypt_purchase',
                            'opening_closing', 'perpetual_care', 'monument_setting',
                            'marker', 'vault', 'service_fee', 'other'
                        )),
    quantity            INTEGER NOT NULL DEFAULT 1,
    unit_price          NUMERIC(12, 2) NOT NULL,
    total_price         NUMERIC(12, 2) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contract_lines_contract ON contract_line_items(contract_id);
```

```sql
-- ============================================================
-- INVOICES & PAYMENTS
-- ============================================================

CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_number      VARCHAR(50) UNIQUE NOT NULL,
    contract_id         UUID REFERENCES contracts(id),
    contact_id          UUID NOT NULL REFERENCES contacts(id),
    cemetery_id         UUID NOT NULL REFERENCES cemeteries(id),
    invoice_date        DATE NOT NULL,
    due_date            DATE NOT NULL,
    subtotal            NUMERIC(12, 2) NOT NULL,
    tax_amount          NUMERIC(12, 2) DEFAULT 0,
    total_amount        NUMERIC(12, 2) NOT NULL,
    amount_paid         NUMERIC(12, 2) DEFAULT 0,
    balance_due         NUMERIC(12, 2) GENERATED ALWAYS AS (total_amount - amount_paid) STORED,
    currency_code       CHAR(3) DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN (
                            'draft', 'sent', 'paid', 'partial', 'overdue', 'cancelled', 'void'
                        )),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID NOT NULL REFERENCES invoices(id),
    payment_date        DATE NOT NULL,
    amount              NUMERIC(12, 2) NOT NULL,
    payment_method      VARCHAR(30) NOT NULL CHECK (payment_method IN (
                            'cash', 'check', 'credit_card', 'debit_card',
                            'bank_transfer', 'online', 'money_order', 'other'
                        )),
    reference_number    VARCHAR(100),  -- Check number, transaction ID, etc.
    stripe_payment_id   VARCHAR(100),  -- Stripe payment intent ID if applicable
    notes               TEXT,
    created_by          UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_invoices_contract ON invoices(contract_id);
CREATE INDEX idx_invoices_contact ON invoices(contact_id);
CREATE INDEX idx_invoices_status ON invoices(status);
```

```sql
-- ============================================================
-- PERPETUAL CARE FUND
-- ============================================================

CREATE TABLE perpetual_care_funds (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cemetery_id         UUID NOT NULL REFERENCES cemeteries(id),
    fund_name           VARCHAR(255) NOT NULL,
    -- Corpus tracking
    total_corpus        NUMERIC(14, 2) NOT NULL DEFAULT 0,
    -- Investment tracking
    total_market_value  NUMERIC(14, 2) NOT NULL DEFAULT 0,
    last_valuation_date DATE,
    -- Annual income
    annual_income       NUMERIC(12, 2) DEFAULT 0,
    fiscal_year_end     CHAR(5) DEFAULT '12-31',  -- MM-DD
    -- Regulatory
    jurisdiction        VARCHAR(100),
    regulatory_body     VARCHAR(255),
    trustee_name        VARCHAR(255),
    trustee_contact_id  UUID REFERENCES contacts(id),
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE perpetual_care_contributions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id             UUID NOT NULL REFERENCES perpetual_care_funds(id),
    contract_id         UUID REFERENCES contracts(id),
    space_id            UUID REFERENCES spaces(id),
    contribution_date   DATE NOT NULL,
    amount              NUMERIC(12, 2) NOT NULL,
    description         VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE perpetual_care_transactions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fund_id             UUID NOT NULL REFERENCES perpetual_care_funds(id),
    transaction_date    DATE NOT NULL,
    transaction_type    VARCHAR(30) NOT NULL CHECK (transaction_type IN (
                            'contribution', 'investment_income', 'capital_gain',
                            'capital_loss', 'dividend', 'interest',
                            'distribution', 'fee', 'adjustment'
                        )),
    amount              NUMERIC(14, 2) NOT NULL,
    running_balance     NUMERIC(14, 2) NOT NULL,
    description         VARCHAR(500),
    reference_number    VARCHAR(100),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pc_contributions_fund ON perpetual_care_contributions(fund_id);
CREATE INDEX idx_pc_transactions_fund ON perpetual_care_transactions(fund_id);
CREATE INDEX idx_pc_transactions_date ON perpetual_care_transactions(transaction_date);
```

### Operations & Maintenance

```sql
-- ============================================================
-- WORK ORDERS
-- ============================================================

CREATE TABLE work_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cemetery_id         UUID NOT NULL REFERENCES cemeteries(id),
    work_order_number   VARCHAR(50) UNIQUE NOT NULL,
    -- Location reference (may be section-level, lot-level, or space-level)
    section_id          UUID REFERENCES sections(id),
    lot_id              UUID REFERENCES lots(id),
    space_id            UUID REFERENCES spaces(id),
    -- Work order details
    work_type           VARCHAR(50) NOT NULL CHECK (work_type IN (
                            'mowing', 'trimming', 'planting', 'monument_repair',
                            'monument_cleaning', 'monument_setting', 'grave_opening',
                            'grave_closing', 'marker_installation', 'drainage',
                            'road_repair', 'fence_repair', 'tree_removal',
                            'snow_removal', 'seasonal_decoration', 'other'
                        )),
    priority            VARCHAR(10) NOT NULL DEFAULT 'normal' CHECK (priority IN (
                            'urgent', 'high', 'normal', 'low'
                        )),
    title               VARCHAR(255) NOT NULL,
    description         TEXT,
    -- Assignment
    assigned_to         UUID,  -- Staff user ID
    assigned_date       DATE,
    -- Scheduling
    scheduled_date      DATE,
    due_date            DATE,
    completed_date      DATE,
    -- Cost tracking
    estimated_hours     NUMERIC(6, 2),
    actual_hours        NUMERIC(6, 2),
    material_cost       NUMERIC(10, 2),
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'open' CHECK (status IN (
                            'open', 'assigned', 'in_progress', 'completed',
                            'cancelled', 'deferred'
                        )),
    -- AI-suggested priority (from seasonal pattern analysis)
    ai_priority_score   NUMERIC(5, 2),
    ai_priority_reason  VARCHAR(500),
    notes               TEXT,
    created_by          UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_work_orders_cemetery ON work_orders(cemetery_id);
CREATE INDEX idx_work_orders_status ON work_orders(status);
CREATE INDEX idx_work_orders_scheduled ON work_orders(scheduled_date);
CREATE INDEX idx_work_orders_assigned ON work_orders(assigned_to);
```

### Document Management

```sql
-- ============================================================
-- DOCUMENTS (attached to any record)
-- ============================================================

CREATE TABLE documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cemetery_id         UUID NOT NULL REFERENCES cemeteries(id),
    -- Polymorphic attachment: one of these will be set
    deceased_id         UUID REFERENCES deceased(id),
    interment_id        UUID REFERENCES interments(id),
    space_id            UUID REFERENCES spaces(id),
    ownership_id        UUID REFERENCES ownership_rights(id),
    contract_id         UUID REFERENCES contracts(id),
    work_order_id       UUID REFERENCES work_orders(id),
    contact_id          UUID REFERENCES contacts(id),
    -- Document metadata
    document_type       VARCHAR(50) NOT NULL CHECK (document_type IN (
                            'deed', 'burial_permit', 'death_certificate',
                            'contract', 'invoice', 'receipt', 'ledger_page',
                            'photograph', 'monument_photo', 'aerial_image',
                            'survey_map', 'obituary', 'military_record',
                            'genealogy_document', 'memorial_tribute', 'other'
                        )),
    title               VARCHAR(255) NOT NULL,
    description         TEXT,
    -- Storage
    file_name           VARCHAR(255) NOT NULL,
    file_path           VARCHAR(1000) NOT NULL,  -- Object storage key (S3/MinIO)
    file_size_bytes     BIGINT,
    mime_type           VARCHAR(100),
    -- OCR / AI processing status
    ocr_status          VARCHAR(20) DEFAULT 'pending' CHECK (ocr_status IN (
                            'pending', 'processing', 'completed', 'failed', 'not_applicable'
                        )),
    ocr_text            TEXT,  -- Extracted text from OCR
    ocr_confidence      NUMERIC(5, 4),
    -- Dublin Core metadata (ISO 15836)
    dc_creator          VARCHAR(255),
    dc_date             DATE,
    dc_subject          VARCHAR(500),
    -- Access control
    is_public           BOOLEAN NOT NULL DEFAULT false,  -- Visible on genealogy portal?
    -- Audit
    uploaded_by         UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_deceased ON documents(deceased_id);
CREATE INDEX idx_documents_interment ON documents(interment_id);
CREATE INDEX idx_documents_space ON documents(space_id);
CREATE INDEX idx_documents_type ON documents(document_type);
CREATE INDEX idx_documents_ocr_text ON documents USING GIN(to_tsvector('simple', COALESCE(ocr_text, '')));
```

### Public Genealogy Portal

```sql
-- ============================================================
-- MEMORIAL PAGES (public-facing memorial per deceased)
-- ============================================================

CREATE TABLE memorial_pages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deceased_id         UUID NOT NULL REFERENCES deceased(id) UNIQUE,
    -- Public URL slug
    slug                VARCHAR(200) UNIQUE NOT NULL,
    -- Memorial content
    headline            VARCHAR(500),
    tribute_text        TEXT,
    -- Display settings
    is_published        BOOLEAN NOT NULL DEFAULT false,
    allow_public_tributes BOOLEAN NOT NULL DEFAULT true,
    allow_photo_uploads BOOLEAN NOT NULL DEFAULT true,
    -- Moderation
    moderation_status   VARCHAR(20) DEFAULT 'approved',
    -- Analytics
    view_count          BIGINT DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memorial_tributes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    memorial_page_id    UUID NOT NULL REFERENCES memorial_pages(id) ON DELETE CASCADE,
    -- Contributor info
    contributor_name    VARCHAR(200) NOT NULL,
    contributor_email   VARCHAR(255),
    relationship        VARCHAR(100),
    -- Tribute content
    tribute_text        TEXT NOT NULL,
    photo_url           VARCHAR(1000),
    -- Moderation
    moderation_status   VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (moderation_status IN (
                            'pending', 'approved', 'rejected', 'flagged'
                        )),
    moderated_by        UUID,
    moderated_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tributes_memorial ON memorial_tributes(memorial_page_id);
CREATE INDEX idx_tributes_moderation ON memorial_tributes(moderation_status);
```

### Audit Trail

```sql
-- ============================================================
-- AUDIT LOG (immutable append-only trail for legal compliance)
-- ============================================================

CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    -- What changed
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    -- Who and when
    user_id         UUID,
    user_email      VARCHAR(255),
    user_ip         INET,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Change details
    old_values      JSONB,
    new_values      JSONB,
    -- Context
    cemetery_id     UUID,
    description     VARCHAR(500)
);

-- Partition audit log by month for performance on long-term retention
-- (100+ year archival requirement means this table will grow very large)
-- In production, use declarative partitioning:
-- CREATE TABLE audit_log (...) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_audit_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_cemetery ON audit_log(cemetery_id);
```

### GEDCOM Export Support

```sql
-- ============================================================
-- GEDCOM EXPORT MAPPINGS
-- ============================================================

CREATE TABLE gedcom_exports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cemetery_id     UUID NOT NULL REFERENCES cemeteries(id),
    export_date     TIMESTAMPTZ NOT NULL DEFAULT now(),
    gedcom_version  VARCHAR(10) NOT NULL DEFAULT '7.0',
    file_path       VARCHAR(1000),
    record_count    INTEGER,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Family relationship links for GEDCOM FAM records
CREATE TABLE deceased_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deceased_id     UUID NOT NULL REFERENCES deceased(id),
    related_deceased_id UUID NOT NULL REFERENCES deceased(id),
    relationship_type VARCHAR(30) NOT NULL CHECK (relationship_type IN (
                        'spouse', 'parent', 'child', 'sibling', 'other'
                    )),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(deceased_id, related_deceased_id, relationship_type)
);

CREATE INDEX idx_deceased_rel_deceased ON deceased_relationships(deceased_id);
CREATE INDEX idx_deceased_rel_related ON deceased_relationships(related_deceased_id);
```

### Users & Staff

```sql
-- ============================================================
-- USERS & ROLES
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    display_name    VARCHAR(200) NOT NULL,
    password_hash   VARCHAR(255),  -- NULL for OAuth-only users
    auth_provider   VARCHAR(30) DEFAULT 'local',
    auth_provider_id VARCHAR(255),
    role            VARCHAR(30) NOT NULL DEFAULT 'staff' CHECK (role IN (
                        'super_admin', 'cemetery_admin', 'staff', 'grounds_crew',
                        'sales', 'read_only', 'public_portal'
                    )),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Multi-cemetery access control
CREATE TABLE user_cemetery_access (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    cemetery_id     UUID NOT NULL REFERENCES cemeteries(id) ON DELETE CASCADE,
    role_override   VARCHAR(30),  -- Override org-level role for this cemetery
    PRIMARY KEY (user_id, cemetery_id)
);
```

---

## Key Views for Common Queries

```sql
-- Genealogy search view (powers the public portal)
CREATE VIEW v_burial_search AS
SELECT
    d.id AS deceased_id,
    d.first_name,
    d.middle_name,
    d.last_name,
    d.maiden_name,
    d.date_of_birth,
    d.date_of_death,
    d.is_veteran,
    d.display_name,
    d.search_vector,
    i.interment_date,
    i.interment_type,
    s.space_number,
    s.gps_latitude,
    s.gps_longitude,
    s.location_point,
    sec.name AS section_name,
    sec.code AS section_code,
    l.lot_number,
    c.name AS cemetery_name,
    c.id AS cemetery_id,
    mp.slug AS memorial_slug,
    mp.is_published AS has_memorial
FROM deceased d
JOIN interments i ON i.deceased_id = d.id AND i.status = 'completed'
JOIN spaces s ON s.id = i.space_id
JOIN sections sec ON sec.id = s.section_id
LEFT JOIN lots l ON l.id = s.lot_id
JOIN cemeteries c ON c.id = sec.cemetery_id
LEFT JOIN memorial_pages mp ON mp.deceased_id = d.id;

-- Plot availability view (powers the sales workflow)
CREATE VIEW v_space_availability AS
SELECT
    s.id AS space_id,
    s.space_number,
    s.space_type,
    s.status,
    s.base_price,
    s.max_interments,
    s.current_interments,
    s.gps_latitude,
    s.gps_longitude,
    sec.name AS section_name,
    sec.code AS section_code,
    sec.section_type,
    sec.faith_designation,
    l.lot_number,
    c.name AS cemetery_name,
    c.id AS cemetery_id,
    -- Current owner (if sold)
    ow.owner_contact_id,
    ct.display_name AS owner_name
FROM spaces s
JOIN sections sec ON sec.id = s.section_id
LEFT JOIN lots l ON l.id = s.lot_id
JOIN cemeteries c ON c.id = sec.cemetery_id
LEFT JOIN ownership_rights ow ON ow.space_id = s.id AND ow.status = 'active'
LEFT JOIN contacts ct ON ct.id = ow.owner_contact_id;
```

---

## Pros and Cons

### Pros

1. **Data integrity**: Foreign keys, check constraints, and NOT NULL constraints enforce business rules at the database level. A plot cannot reference a nonexistent section; an interment cannot reference a nonexistent deceased record.

2. **Standards alignment**: PostGIS is the industry standard for geospatial data in open-source databases. The OGC compliance (WMS, WFS, GeoJSON) required by the project README is natively supported.

3. **Mature ecosystem**: PostgreSQL has decades of production use, extensive tooling (pgAdmin, DBeaver, DataGrip), and a large talent pool. The learning curve for new developers is minimal.

4. **Built-in search capabilities**: The combination of tsvector full-text search, pg_trgm trigram matching, and fuzzystrmatch phonetic functions provides the fuzzy/phonetic genealogy search the project needs without an external search engine.

5. **Proven for long-term archival**: PostgreSQL's WAL-based replication, point-in-time recovery, and table partitioning (for the audit log) support the 100+ year data retention requirement.

6. **Transaction safety**: ACID transactions ensure that multi-step operations (e.g., selling a plot: create contract, update space status, record payment, generate deed) either fully complete or fully roll back.

7. **PostGIS spatial queries**: Finding available plots near a family's existing burials, calculating section utilisation, or detecting overlapping boundaries are all single SQL queries.

### Cons

1. **Schema rigidity**: Adding new fields (e.g., a new cultural requirement, a new document type) requires ALTER TABLE migrations. For a domain with significant variation across jurisdictions and cemetery types, this can slow iteration.

2. **Join complexity**: Deep hierarchical queries (organisation > cemetery > section > lot > space > interment > deceased) require multi-table joins that can be slow without careful indexing and query optimisation.

3. **Polymorphic document attachment**: The documents table uses nullable foreign keys to attach to multiple entity types. This pattern works but does not enforce "exactly one parent" at the database level.

4. **Audit log growth**: An append-only audit log for 100+ years of operations on a busy cemetery system will grow to billions of rows. Partitioning is essential but adds operational complexity.

5. **No built-in event replay**: Unlike event sourcing, a normalised model stores only current state. Reconstructing "what was the plot status on 15 March 1987?" requires querying the audit log and replaying changes.

6. **Geospatial performance at scale**: Very large cemeteries (50,000+ plots) with complex polygon boundaries can strain spatial indexes. GiST index maintenance adds overhead to bulk imports and migrations.

---

## Migration & Scaling Considerations

### Initial Migration Strategy

1. **Legacy data import**: Build a CSV/Excel import pipeline with a staging table pattern. Raw imported data lands in a `staging_*` table, undergoes validation and deduplication, then is promoted to production tables via INSERT...SELECT.

2. **OCR pipeline integration**: Scanned ledger pages are uploaded as documents with `ocr_status = 'pending'`. An async worker processes them, populates `ocr_text`, and creates draft `deceased` and `interment` records for human review.

3. **GIS data migration**: Import existing GIS data from Shapefile, GeoJSON, or GeoPackage using `ogr2ogr` or PostGIS `ST_GeomFromGeoJSON`. The `shp2pgsql` loader handles bulk Shapefile imports.

### Scaling Strategies

| Growth stage | Strategy |
|-------------|----------|
| Single cemetery (<10K plots) | Single PostgreSQL instance; no replication needed |
| Multi-site (10-50K plots) | Read replicas for the public genealogy portal; primary for admin writes |
| Large operator (50K-500K plots) | Connection pooling (PgBouncer); audit log partitioning by year; materialised views for search |
| Enterprise (500K+ plots) | Citus extension for horizontal sharding by cemetery_id; separate PostGIS instance for tile serving |

### Backup & Archival

- **WAL archiving** to object storage (S3/MinIO) for continuous point-in-time recovery
- **Nightly pg_dump** for human-readable backups
- **Cross-region replication** for disaster recovery (geo-redundant per InStone/Pontem patterns)
- **Annual cold archive** of old audit log partitions to compressed storage

### Performance Optimisation

- **Materialised views** for the genealogy search portal (refresh on a schedule, not on every write)
- **Partial indexes** on `status = 'available'` for the space availability query
- **BRIN indexes** on timestamp columns in the audit log for range scans
- **Connection pooling** via PgBouncer to handle public portal traffic spikes

---

## Summary

The normalised relational model is the safest, most predictable choice for this domain. Cemetery management is fundamentally a record-keeping system with well-defined entities and relationships. The data model directly maps to the industry's conceptual model (cemeteries contain sections, sections contain lots, lots contain spaces, spaces hold interments of deceased persons). PostGIS adds geospatial capability without leaving the PostgreSQL ecosystem. The main trade-off is schema rigidity -- jurisdictional and cultural variations may require frequent migrations -- but this is mitigable with careful use of nullable columns and enum expansion.
