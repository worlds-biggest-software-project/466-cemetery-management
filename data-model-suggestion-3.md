# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL JSONB)

> Project: Cemetery Management (466) · Generated: 2026-05-26

---

## Overview

This model takes a pragmatic middle path: use normalised relational tables for the stable, well-defined core of the domain (cemeteries, sections, spaces, interments, contracts, payments) and PostgreSQL JSONB columns for data that varies significantly across jurisdictions, cemetery types, cultural traditions, and historical record formats. The result is a single-database architecture on PostgreSQL with PostGIS that combines the referential integrity of a relational model with the schema flexibility of a document store.

The insight driving this approach is that cemetery management has a stable structural core (every cemetery has sections, sections have spaces, spaces hold interments) but enormous variability at the edges:

- A Catholic cemetery section has different cultural rules than a Jewish or Muslim section
- Historic records from the 1800s have completely different fields than modern burial permits
- Military veteran records include service branches and rank while civilian records do not
- Columbarium niches have row/column/level positions while ground plots have polygon boundaries
- Perpetual care fund regulations differ by US state, Canadian province, and country
- Document metadata from OCR extraction is inherently semi-structured

Rather than creating dozens of nullable columns or separate tables for each variant, JSONB columns absorb this variability while the relational structure handles the invariants.

---

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| RDBMS | PostgreSQL 16+ | Native JSONB support with GIN indexing; PostGIS extension for spatial data |
| Spatial | PostGIS 3.4+ | Geometry columns for plot boundaries, section polygons, cemetery boundaries |
| Search | PostgreSQL tsvector + pg_trgm + fuzzystrmatch | Built-in full-text and fuzzy search; avoids external search service |
| Optional Search | OpenSearch (if scale demands it) | Can be added later for high-traffic genealogy portal |
| Cache | Redis | Optional hot cache for map tile data and public portal |
| Migrations | Flyway or golang-migrate | Schema migrations for relational structure; JSONB columns evolve without migrations |
| Backup | pg_dump + WAL archiving | Standard PostgreSQL disaster recovery |

---

## Design Principles

### What Goes in Relational Columns

- **Identity and relationships**: Primary keys, foreign keys, unique constraints
- **Fields used in WHERE clauses**: Status, dates, types -- anything you filter or sort by
- **Fields with referential integrity needs**: Contact IDs, cemetery IDs, space IDs
- **Fields with domain constraints**: Enums (status, type), check constraints (amounts >= 0)
- **Spatial data**: PostGIS geometry columns (indexed with GiST)

### What Goes in JSONB Columns

- **Cultural/jurisdictional variations**: Faith-specific rules, regulatory metadata
- **Historical record fields**: Varying levels of completeness across centuries of data
- **Extension points**: Custom fields per cemetery, user-defined metadata
- **Extracted/computed data**: OCR results, AI-generated content, import metadata
- **Rarely queried details**: Biographical notes, military service details, monument descriptions
- **Configuration**: Per-section rules, pricing tiers, notification templates

### JSONB Validation Strategy

JSONB columns are validated at the application layer using JSON Schema definitions. PostgreSQL 16+ supports `IS JSON` checks, and application-level validators enforce structure before insert. This provides "schema-on-write" without rigidity.

---

## Schema Definition

### Core Location Hierarchy

```sql
-- ============================================================
-- ORGANISATIONS
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL CHECK (org_type IN (
                        'municipal', 'religious', 'private', 'memorial_park', 'veterans', 'other'
                    )),
    -- Core contact info (relational — frequently queried)
    email           VARCHAR(255),
    phone           VARCHAR(30),
    website         VARCHAR(500),
    -- Address and additional details (JSONB — varies by jurisdiction)
    address         JSONB NOT NULL DEFAULT '{}',
    -- Example: { "line1": "123 Main St", "city": "Springfield",
    --            "state": "IL", "postal_code": "62701", "country": "US" }
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example: { "default_currency": "USD", "fiscal_year_end": "12-31",
    --            "tax_rate": 0.07, "perpetual_care_pct": 0.10,
    --            "require_burial_permit": true }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
-- ============================================================
-- CEMETERIES
-- ============================================================

CREATE TABLE cemeteries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(20) UNIQUE NOT NULL,
    -- Geospatial (relational — used in spatial queries)
    boundary        GEOMETRY(POLYGON, 4326),
    centre_point    GEOMETRY(POINT, 4326),
    default_zoom    SMALLINT DEFAULT 18,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    status          VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'closed', 'historical')),
    established_date DATE,
    -- Address and extended metadata (JSONB)
    address         JSONB NOT NULL DEFAULT '{}',
    -- Map configuration (JSONB — varies by mapping provider)
    map_config      JSONB NOT NULL DEFAULT '{}',
    -- Example: { "tile_url": "https://tiles.example.com/{z}/{x}/{y}.png",
    --            "aerial_imagery_url": "...", "max_zoom": 22,
    --            "base_layer": "mapbox", "mapbox_style_id": "..." }
    -- Regulatory/jurisdictional config (JSONB — varies by location)
    regulations     JSONB NOT NULL DEFAULT '{}',
    -- Example: { "jurisdiction": "State of Illinois",
    --            "perpetual_care_required": true, "min_perpetual_care_pct": 0.15,
    --            "burial_permit_authority": "County Clerk",
    --            "record_retention_years": null, -- null = perpetual
    --            "pre_need_trust_requirements": { ... } }
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cemeteries_org ON cemeteries(organisation_id);
CREATE INDEX idx_cemeteries_boundary ON cemeteries USING GIST(boundary);
CREATE INDEX idx_cemeteries_status ON cemeteries(status);
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
    -- Geospatial (relational)
    boundary        GEOMETRY(POLYGON, 4326),
    fill_colour     VARCHAR(7) DEFAULT '#CCCCCC',
    sort_order      INTEGER DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Cultural and faith rules (JSONB — the key flexibility point)
    culture_rules   JSONB NOT NULL DEFAULT '{}',
    -- Example for a Jewish section:
    -- { "faith_designation": "Jewish Orthodox",
    --   "orientation": "east_facing",
    --   "spacing_cm": 60,
    --   "gender_separation": true,
    --   "casket_requirements": "plain_wood_only",
    --   "vault_prohibited": true,
    --   "embalming_prohibited": true,
    --   "no_cremation": true,
    --   "monument_restrictions": { "max_height_cm": 90, "material": "granite_only" },
    --   "burial_timing": "within_24_hours_preferred",
    --   "kohen_restrictions": true,
    --   "notes": "Section reserved for members of Temple Beth Israel" }
    --
    -- Example for a Muslim section:
    -- { "faith_designation": "Muslim",
    --   "orientation": "mecca_facing",
    --   "compass_bearing_deg": 56.5,
    --   "spacing_cm": 50,
    --   "casket_requirements": "no_casket_preferred",
    --   "shroud_burial": true,
    --   "vault_prohibited": true,
    --   "embalming_prohibited": true,
    --   "no_cremation": true,
    --   "gender_separation": true,
    --   "burial_timing": "within_24_hours",
    --   "washing_facility_required": true }
    --
    -- Example for a veterans section:
    -- { "faith_designation": "non_denominational",
    --   "orientation": "none",
    --   "marker_type": "government_headstone",
    --   "va_marker_eligible": true,
    --   "flag_holder_required": true,
    --   "minimum_military_service": "active_duty_or_reserve",
    --   "discharge_requirement": "honorable_or_general" }
    -- Pricing configuration per section (JSONB)
    pricing         JSONB NOT NULL DEFAULT '{}',
    -- Example: { "resident_price": 1500.00, "non_resident_price": 3000.00,
    --            "perpetual_care_fee": 300.00, "opening_closing_fee": 950.00,
    --            "weekend_surcharge": 250.00, "holiday_surcharge": 500.00,
    --            "liner_fee": 200.00, "price_effective_date": "2026-01-01" }
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(cemetery_id, code)
);

CREATE INDEX idx_sections_cemetery ON sections(cemetery_id);
CREATE INDEX idx_sections_boundary ON sections USING GIST(boundary);
CREATE INDEX idx_sections_type ON sections(section_type);
-- GIN index on JSONB for querying by faith or cultural rules
CREATE INDEX idx_sections_culture ON sections USING GIN(culture_rules);
```

```sql
-- ============================================================
-- SPACES (plots, niches, crypts)
-- ============================================================

CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES sections(id),
    lot_id          UUID REFERENCES lots(id),
    space_number    VARCHAR(20) NOT NULL,
    space_type      VARCHAR(30) NOT NULL CHECK (space_type IN (
                        'ground_single', 'ground_double', 'ground_infant',
                        'cremation_ground', 'columbarium_niche', 'mausoleum_crypt',
                        'mausoleum_crypt_double', 'ossuary', 'memorial_only', 'other'
                    )),
    -- Geospatial (relational — used in map rendering and spatial queries)
    location_point  GEOMETRY(POINT, 4326),
    location_polygon GEOMETRY(POLYGON, 4326),
    gps_latitude    DOUBLE PRECISION,
    gps_longitude   DOUBLE PRECISION,
    -- Core inventory fields (relational — filtered and aggregated)
    max_interments  SMALLINT NOT NULL DEFAULT 1,
    current_interments SMALLINT NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'available' CHECK (status IN (
                        'available', 'sold', 'reserved', 'occupied', 'full',
                        'unavailable', 'condemned', 'historical'
                    )),
    base_price      NUMERIC(12, 2),
    currency_code   CHAR(3) DEFAULT 'USD',
    -- Space-type-specific details (JSONB — different fields per type)
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example for columbarium_niche:
    -- { "structure_name": "St. Francis Columbarium",
    --   "row": "C", "column": "7", "level": "3",
    --   "niche_size": "double", "front_type": "bronze",
    --   "inscription_lines": 4, "photo_option": true,
    --   "interior_dimensions_cm": { "width": 30, "height": 30, "depth": 35 } }
    --
    -- Example for mausoleum_crypt:
    -- { "building_name": "Garden Mausoleum",
    --   "wing": "East", "tier": 2, "position": "center",
    --   "crypt_type": "single_casket", "shutter_type": "marble",
    --   "inscription_type": "bronze_plaque", "indoor": true,
    --   "climate_controlled": true }
    --
    -- Example for ground_single:
    -- { "row": "12", "position_in_row": "3",
    --   "monument_allowed": true, "max_monument_height_cm": 120,
    --   "vault_required": true, "liner_accepted": true,
    --   "depth_available_ft": 6, "grass_type": "bermuda" }
    -- Custom fields defined by the cemetery operator
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Example: { "lot_corner_marker": "brass_pin",
    --            "adjacent_to_pathway": true, "tree_shade": "partial" }
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(section_id, space_number)
);

-- Lots table (minimal relational structure)
CREATE TABLE lots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES sections(id),
    lot_number      VARCHAR(20) NOT NULL,
    boundary        GEOMETRY(POLYGON, 4326),
    total_spaces    INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(section_id, lot_number)
);

CREATE INDEX idx_spaces_section ON spaces(section_id);
CREATE INDEX idx_spaces_lot ON spaces(lot_id);
CREATE INDEX idx_spaces_status ON spaces(status);
CREATE INDEX idx_spaces_type ON spaces(space_type);
CREATE INDEX idx_spaces_point ON spaces USING GIST(location_point);
CREATE INDEX idx_spaces_polygon ON spaces USING GIST(location_polygon);
CREATE INDEX idx_spaces_details ON spaces USING GIN(details);
CREATE INDEX idx_lots_section ON lots(section_id);
CREATE INDEX idx_lots_boundary ON lots USING GIST(boundary);
```

### People: Contacts and Deceased

```sql
-- ============================================================
-- CONTACTS
-- ============================================================

CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_type    VARCHAR(30) NOT NULL CHECK (contact_type IN (
                        'individual', 'funeral_home', 'officiant', 'monument_company',
                        'government_agency', 'other_organisation'
                    )),
    -- Core searchable fields (relational)
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    organisation_name VARCHAR(255),
    email           VARCHAR(255),
    phone_primary   VARCHAR(30),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Extended details (JSONB — varies by contact type)
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example for individual:
    -- { "middle_name": "Marie", "suffix": "Jr.",
    --   "phone_secondary": "555-0102",
    --   "address": { "line1": "456 Oak Ave", "city": "Springfield", "state": "IL", "zip": "62701" },
    --   "relationship_to_deceased": "daughter",
    --   "preferred_contact_method": "email",
    --   "communication_preferences": { "email_notifications": true, "sms": false } }
    --
    -- Example for funeral_home:
    -- { "licence_number": "FH-2024-1234",
    --   "director_name": "James Wilson",
    --   "director_phone": "555-0200",
    --   "address": { ... },
    --   "services_offered": ["embalming", "cremation", "transport"],
    --   "mortrak_integration_id": "MRT-5678" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_type ON contacts(contact_type);
CREATE INDEX idx_contacts_last_name ON contacts(last_name);
CREATE INDEX idx_contacts_email ON contacts(email);
CREATE INDEX idx_contacts_org_name ON contacts(organisation_name);
```

```sql
-- ============================================================
-- DECEASED
-- ============================================================

CREATE TABLE deceased (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Core searchable fields (relational — used in genealogy portal queries)
    first_name          VARCHAR(100),
    middle_name         VARCHAR(100),
    last_name           VARCHAR(100),
    maiden_name         VARCHAR(100),
    date_of_birth       DATE,
    date_of_death       DATE,
    is_veteran          BOOLEAN DEFAULT false,
    -- Full-text search (relational computed column)
    search_vector       TSVECTOR GENERATED ALWAYS AS (
                            to_tsvector('simple',
                                COALESCE(first_name, '') || ' ' ||
                                COALESCE(middle_name, '') || ' ' ||
                                COALESCE(last_name, '') || ' ' ||
                                COALESCE(maiden_name, '')
                            )
                        ) STORED,
    -- Extended biographical and vital records data (JSONB)
    bio                 JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- { "suffix": "III",
    --   "name_aliases": ["Jon", "Johnathan", "Johann"],
    --   "date_of_birth_approx": false,
    --   "date_of_death_approx": false,
    --   "gender": "male",
    --   "ssn_last_four": "1234",
    --   "obituary_text": "John was beloved...",
    --   "biography": "Born in County Cork, Ireland...",
    --   "ethnicity": "Irish-American",
    --   "religion": "Catholic",
    --   "birthplace": { "city": "Cork", "country": "IE" },
    --   "last_residence": { "city": "Springfield", "state": "IL" } }
    -- Vital records / death certificate data (JSONB — for VRDR FHIR interop)
    vital_records       JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- { "death_certificate_number": "2024-IL-045678",
    --   "cause_of_death": "Natural causes",
    --   "manner_of_death": "natural",
    --   "certifier_name": "Dr. Sarah Johnson",
    --   "certifier_type": "physician",
    --   "place_of_death": { "facility": "Springfield Memorial Hospital", "type": "hospital" },
    --   "pronouncement_date": "2024-11-01",
    --   "pronouncement_time": "14:30",
    --   "autopsy_performed": false,
    --   "medical_examiner_contacted": false,
    --   "tobacco_contributed": "unknown",
    --   "pregnancy_status": "not_applicable",
    --   "disposition_method": "burial",
    --   "fhir_vrdr_bundle_id": "urn:uuid:abc123..." }
    -- Military service details (JSONB — only for veterans)
    military            JSONB,
    -- Example:
    -- { "branch": "US Army", "rank": "Sergeant",
    --   "service_number": "12345678",
    --   "service_dates": { "start": "1965-06-15", "end": "1969-08-30" },
    --   "wars_served": ["Vietnam"],
    --   "discharge_type": "honorable",
    --   "va_claim_number": "VA-2024-98765",
    --   "headstone_type": "government_upright",
    --   "headstone_inscription": "SGT JOHN DOE / US ARMY / VIETNAM / JUN 15 1945 - NOV 1 2024",
    --   "flag_received_by": "Jane Doe",
    --   "medals": ["Purple Heart", "Bronze Star"] }
    -- OCR/AI-extracted data from historic records (JSONB)
    legacy_record_data  JSONB,
    -- Example:
    -- { "source": "ledger_page_scan",
    --   "source_document_id": "doc-uuid-here",
    --   "ocr_confidence": 0.87,
    --   "original_text": "Jno. Smith, d. March ye 15th, 1887",
    --   "interpreted_as": { "first_name": "John", "last_name": "Smith", "death_date": "1887-03-15" },
    --   "handwriting_quality": "poor",
    --   "verification_status": "unverified",
    --   "verified_by": null,
    --   "verified_at": null }
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deceased_last_name ON deceased(last_name);
CREATE INDEX idx_deceased_dates ON deceased(date_of_death, date_of_birth);
CREATE INDEX idx_deceased_search ON deceased USING GIN(search_vector);
CREATE INDEX idx_deceased_name_trgm ON deceased USING GIN(
    (COALESCE(first_name, '') || ' ' || COALESCE(last_name, '')) gin_trgm_ops
);
CREATE INDEX idx_deceased_veteran ON deceased(is_veteran) WHERE is_veteran = true;
-- GIN index on JSONB for querying name aliases
CREATE INDEX idx_deceased_bio ON deceased USING GIN(bio jsonb_path_ops);
-- Partial index for VRDR integration
CREATE INDEX idx_deceased_vrdr ON deceased USING GIN(vital_records jsonb_path_ops)
    WHERE vital_records != '{}';
```

### Interments

```sql
-- ============================================================
-- INTERMENTS
-- ============================================================

CREATE TABLE interments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id            UUID NOT NULL REFERENCES spaces(id),
    deceased_id         UUID NOT NULL REFERENCES deceased(id),
    -- Core fields (relational — filtered, sorted, and constrained)
    interment_type      VARCHAR(30) NOT NULL CHECK (interment_type IN (
                            'casket_burial', 'cremation_inurnment', 'cremation_ground_burial',
                            'entombment', 'disinterment', 'reinterment',
                            'scattering', 'green_burial', 'other'
                        )),
    interment_date      DATE NOT NULL,
    officiant_id        UUID REFERENCES contacts(id),
    funeral_home_id     UUID REFERENCES contacts(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'scheduled' CHECK (status IN (
                            'scheduled', 'confirmed', 'completed', 'cancelled', 'pending_permit'
                        )),
    -- Extended interment details (JSONB — varies by type and jurisdiction)
    details             JSONB NOT NULL DEFAULT '{}',
    -- Example for casket_burial:
    -- { "scheduled_time": "10:00",
    --   "actual_date": "2024-11-03", "actual_time": "10:15",
    --   "depth_inches": 72, "vault_type": "concrete_sectional",
    --   "casket_type": "steel_20_gauge", "casket_dimensions_in": { "l": 84, "w": 28, "h": 23 },
    --   "outer_burial_container": "concrete_liner",
    --   "burial_permit_number": "BP-2024-5678",
    --   "burial_permit_date": "2024-11-01",
    --   "transit_permit_number": "TP-2024-9012",
    --   "graveside_service": true, "tent_required": true, "chairs_count": 30,
    --   "lowering_device": "standard",
    --   "greens_requested": true, "floral_placement": "on_casket",
    --   "weather_conditions": "clear_45F",
    --   "ground_conditions": "dry",
    --   "crew_members": ["Tom", "Mike", "Dave"],
    --   "equipment_used": ["backhoe_cat_420", "lowering_device_standard"] }
    --
    -- Example for cremation_inurnment (columbarium):
    -- { "scheduled_time": "14:00",
    --   "actual_date": "2024-12-15", "actual_time": "14:00",
    --   "urn_type": "bronze_keepsake",
    --   "urn_dimensions_cm": { "h": 25, "diam": 15 },
    --   "cremation_certificate_number": "CC-2024-3456",
    --   "crematory_name": "Springfield Crematory",
    --   "inurnment_method": "placed_in_niche",
    --   "niche_sealed": true, "seal_type": "bronze_face_plate",
    --   "inscription_text": "In Loving Memory / Jane Doe / 1932-2024" }
    --
    -- Example for green_burial:
    -- { "shroud_type": "organic_cotton",
    --   "no_embalming_verified": true,
    --   "no_vault_verified": true,
    --   "biodegradable_casket": "willow_wicker",
    --   "native_planting": "wildflower_mix",
    --   "gps_tree_marker": true,
    --   "depth_inches": 42,
    --   "natural_burial_certification": "Green Burial Council Gold" }
    -- Scheduling metadata
    scheduling          JSONB NOT NULL DEFAULT '{}',
    -- Example: { "requested_date": "2024-11-03", "requested_time": "10:00",
    --            "confirmed_at": "2024-11-01T14:30:00Z",
    --            "conflict_check_passed": true,
    --            "adjacent_services": [],
    --            "special_equipment_needed": ["tent", "chairs_30"],
    --            "access_route": "Gate 3 → Road B → Section 4" }
    notes               TEXT,
    created_by          UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_interments_space ON interments(space_id);
CREATE INDEX idx_interments_deceased ON interments(deceased_id);
CREATE INDEX idx_interments_date ON interments(interment_date);
CREATE INDEX idx_interments_status ON interments(status);
CREATE INDEX idx_interments_type ON interments(interment_type);
CREATE INDEX idx_interments_funeral_home ON interments(funeral_home_id);
```

### Financial Management

```sql
-- ============================================================
-- OWNERSHIP RIGHTS
-- ============================================================

CREATE TABLE ownership_rights (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lot_id              UUID REFERENCES lots(id),
    space_id            UUID REFERENCES spaces(id),
    owner_contact_id    UUID NOT NULL REFERENCES contacts(id),
    deed_number         VARCHAR(50),
    deed_type           VARCHAR(30) NOT NULL CHECK (deed_type IN (
                            'perpetual', 'term_lease', 'pre_need', 'at_need',
                            'transfer', 'inheritance', 'donation', 'other'
                        )),
    deed_date           DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Extended deed and ownership details (JSONB)
    deed_details        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- { "lease_start_date": "2024-06-01", "lease_end_date": "2049-06-01",
    --   "lease_term_years": 25, "renewal_option": true,
    --   "restrictions": ["no_upright_monument", "cremation_only"],
    --   "transferred_from_deed": "D-2019-0042",
    --   "transfer_reason": "inheritance",
    --   "notarised": true, "notary_name": "Jane Smith",
    --   "witnesses": ["Alice Brown", "Bob Green"],
    --   "recorded_with_county": true, "county_recording_number": "CR-2024-78901",
    --   "interment_rights_holders": [
    --       { "name": "John Smith", "relationship": "self", "rights": 2 },
    --       { "name": "Jane Smith", "relationship": "spouse", "rights": 1 }
    --   ] }
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

```sql
-- ============================================================
-- CONTRACTS
-- ============================================================

CREATE TABLE contracts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_number     VARCHAR(50) UNIQUE NOT NULL,
    cemetery_id         UUID NOT NULL REFERENCES cemeteries(id),
    buyer_contact_id    UUID NOT NULL REFERENCES contacts(id),
    contract_type       VARCHAR(30) NOT NULL CHECK (contract_type IN (
                            'at_need', 'pre_need', 'transfer', 'upgrade', 'other'
                        )),
    contract_date       DATE NOT NULL,
    -- Core financial fields (relational — for aggregation and reporting)
    total_amount        NUMERIC(12, 2) NOT NULL,
    perpetual_care_amount NUMERIC(12, 2) DEFAULT 0,
    tax_amount          NUMERIC(12, 2) DEFAULT 0,
    amount_paid         NUMERIC(12, 2) DEFAULT 0,
    balance_due         NUMERIC(12, 2) GENERATED ALWAYS AS (total_amount - amount_paid) STORED,
    currency_code       CHAR(3) DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    signed_date         DATE,
    -- Line items and payment plan details (JSONB — variable structure)
    line_items          JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   { "item_type": "plot_purchase", "description": "Section A, Lot 12, Space 3",
    --     "space_id": "uuid", "quantity": 1, "unit_price": 2500.00, "total": 2500.00 },
    --   { "item_type": "perpetual_care", "description": "Perpetual care contribution",
    --     "quantity": 1, "unit_price": 375.00, "total": 375.00 },
    --   { "item_type": "opening_closing", "description": "Grave opening and closing",
    --     "quantity": 1, "unit_price": 950.00, "total": 950.00 }
    -- ]
    payment_plan        JSONB,
    -- Example: { "method": "installment", "months": 24, "monthly_amount": 159.38,
    --            "first_payment_date": "2024-07-01", "interest_rate": 0.0,
    --            "autopay_enrolled": true, "autopay_day_of_month": 1 }
    -- Pre-need trust details (JSONB — jurisdiction-specific)
    pre_need_trust      JSONB,
    -- Example: { "trust_account_number": "PN-2024-5678",
    --            "trustee_name": "First National Bank",
    --            "amount_in_trust": 2500.00,
    --            "trust_type": "revocable",
    --            "state_regulations": "Illinois Cemetery Care Act (760 ILCS 100)",
    --            "cancellation_penalty_pct": 0.10 }
    notes               TEXT,
    created_by          UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contracts_cemetery ON contracts(cemetery_id);
CREATE INDEX idx_contracts_buyer ON contracts(buyer_contact_id);
CREATE INDEX idx_contracts_status ON contracts(status);
CREATE INDEX idx_contracts_date ON contracts(contract_date);
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
    total_amount        NUMERIC(12, 2) NOT NULL,
    amount_paid         NUMERIC(12, 2) DEFAULT 0,
    balance_due         NUMERIC(12, 2) GENERATED ALWAYS AS (total_amount - amount_paid) STORED,
    currency_code       CHAR(3) DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    -- Line items (JSONB)
    line_items          JSONB NOT NULL DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID NOT NULL REFERENCES invoices(id),
    payment_date        DATE NOT NULL,
    amount              NUMERIC(12, 2) NOT NULL,
    payment_method      VARCHAR(30) NOT NULL,
    reference_number    VARCHAR(100),
    -- Payment processor details (JSONB — varies by processor)
    processor_data      JSONB,
    -- Example for Stripe: { "payment_intent_id": "pi_abc123",
    --                        "charge_id": "ch_xyz789",
    --                        "receipt_url": "https://pay.stripe.com/...",
    --                        "card_last_four": "4242", "card_brand": "visa" }
    notes               TEXT,
    created_by          UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_contract ON invoices(contract_id);
CREATE INDEX idx_invoices_contact ON invoices(contact_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_date ON payments(payment_date);
```

```sql
-- ============================================================
-- PERPETUAL CARE FUNDS
-- ============================================================

CREATE TABLE perpetual_care_funds (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cemetery_id         UUID NOT NULL REFERENCES cemeteries(id),
    fund_name           VARCHAR(255) NOT NULL,
    total_corpus        NUMERIC(14, 2) NOT NULL DEFAULT 0,
    total_market_value  NUMERIC(14, 2) NOT NULL DEFAULT 0,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Fund configuration and regulatory details (JSONB — varies by jurisdiction)
    fund_config         JSONB NOT NULL DEFAULT '{}',
    -- Example: { "jurisdiction": "State of Illinois",
    --            "governing_statute": "760 ILCS 100/1 et seq.",
    --            "trustee_name": "First National Bank & Trust",
    --            "trustee_contact_id": "uuid",
    --            "fiscal_year_end": "12-31",
    --            "minimum_contribution_pct": 0.15,
    --            "distribution_rules": "income_only",
    --            "investment_policy": "conservative_balanced",
    --            "allowed_investments": ["bonds", "equities", "money_market"],
    --            "required_reporting": ["annual_state_report", "quarterly_trustee_statement"],
    --            "audit_required": true, "audit_frequency": "annual" }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
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
    -- Transaction-specific details (JSONB)
    details             JSONB,
    -- Example for investment_income: { "security": "US Treasury 10yr",
    --                                   "cusip": "912828XY", "coupon_rate": 0.0375 }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pct_fund ON perpetual_care_transactions(fund_id);
CREATE INDEX idx_pct_date ON perpetual_care_transactions(transaction_date);
```

### Operations & Documents

```sql
-- ============================================================
-- WORK ORDERS
-- ============================================================

CREATE TABLE work_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cemetery_id         UUID NOT NULL REFERENCES cemeteries(id),
    work_order_number   VARCHAR(50) UNIQUE NOT NULL,
    section_id          UUID REFERENCES sections(id),
    space_id            UUID REFERENCES spaces(id),
    work_type           VARCHAR(50) NOT NULL,
    priority            VARCHAR(10) NOT NULL DEFAULT 'normal',
    title               VARCHAR(255) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'open',
    assigned_to         UUID,
    scheduled_date      DATE,
    due_date            DATE,
    completed_date      DATE,
    -- Extended work order details (JSONB)
    details             JSONB NOT NULL DEFAULT '{}',
    -- Example: { "description": "Replace broken headstone base",
    --            "estimated_hours": 3.0, "actual_hours": 2.5,
    --            "material_cost": 150.00, "materials_used": ["concrete_mix", "gravel"],
    --            "equipment_used": ["level", "trowel", "mixing_drill"],
    --            "weather_dependent": true,
    --            "before_photo_ids": ["doc-uuid-1"], "after_photo_ids": ["doc-uuid-2"],
    --            "ai_priority_score": 8.5,
    --            "ai_priority_reason": "Monument safety hazard; visitor area" }
    notes               TEXT,
    created_by          UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_cemetery ON work_orders(cemetery_id);
CREATE INDEX idx_wo_status ON work_orders(status);
CREATE INDEX idx_wo_scheduled ON work_orders(scheduled_date);
CREATE INDEX idx_wo_assigned ON work_orders(assigned_to);
```

```sql
-- ============================================================
-- DOCUMENTS
-- ============================================================

CREATE TABLE documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cemetery_id         UUID NOT NULL REFERENCES cemeteries(id),
    -- Polymorphic reference (relational)
    attached_to_type    VARCHAR(30) NOT NULL,  -- 'deceased', 'interment', 'space', etc.
    attached_to_id      UUID NOT NULL,
    document_type       VARCHAR(50) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    file_name           VARCHAR(255) NOT NULL,
    file_path           VARCHAR(1000) NOT NULL,
    file_size_bytes     BIGINT,
    mime_type           VARCHAR(100),
    is_public           BOOLEAN NOT NULL DEFAULT false,
    -- OCR and AI extraction results (JSONB — inherently semi-structured)
    ocr_data            JSONB,
    -- Example: { "status": "completed", "confidence": 0.91,
    --            "full_text": "Record of burial of Mary O'Brien...",
    --            "extracted_entities": [
    --                { "type": "person_name", "value": "Mary O'Brien", "confidence": 0.95 },
    --                { "type": "date", "value": "1923-04-12", "confidence": 0.88, "original": "April ye 12th, 1923" },
    --                { "type": "location", "value": "Section B, Lot 7", "confidence": 0.92 }
    --            ],
    --            "language": "en",
    --            "handwriting_vs_print": "handwriting",
    --            "processing_model": "gpt-4-vision",
    --            "processed_at": "2026-05-15T10:30:00Z" }
    -- Dublin Core metadata (JSONB — optional scholarly metadata)
    dublin_core         JSONB,
    -- Example: { "creator": "Rev. Thomas Murphy", "date": "1923-04-12",
    --            "subject": "Burial record", "format": "image/jpeg",
    --            "rights": "Public domain" }
    uploaded_by         UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docs_attached ON documents(attached_to_type, attached_to_id);
CREATE INDEX idx_docs_type ON documents(document_type);
CREATE INDEX idx_docs_cemetery ON documents(cemetery_id);
CREATE INDEX idx_docs_ocr ON documents USING GIN(ocr_data jsonb_path_ops) WHERE ocr_data IS NOT NULL;
```

### Memorial & Genealogy

```sql
-- ============================================================
-- MEMORIAL PAGES & FAMILY RELATIONSHIPS
-- ============================================================

CREATE TABLE memorial_pages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deceased_id         UUID NOT NULL REFERENCES deceased(id) UNIQUE,
    slug                VARCHAR(200) UNIQUE NOT NULL,
    is_published        BOOLEAN NOT NULL DEFAULT false,
    view_count          BIGINT DEFAULT 0,
    -- Memorial content (JSONB — rich, variable structure)
    content             JSONB NOT NULL DEFAULT '{}',
    -- Example: { "headline": "In Loving Memory of John Doe",
    --            "tribute_text": "A devoted father and grandfather...",
    --            "birth_place": "Cork, Ireland",
    --            "life_highlights": ["Immigrated to US in 1952", "Founded Doe Lumber Co."],
    --            "photo_gallery": [
    --                { "url": "/photos/abc123.jpg", "caption": "Wedding day, 1955", "year": 1955 },
    --                { "url": "/photos/def456.jpg", "caption": "Family reunion, 2010", "year": 2010 }
    --            ],
    --            "allow_public_tributes": true,
    --            "allow_photo_uploads": true }
    -- Moderation settings
    settings            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memorial_tributes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    memorial_page_id    UUID NOT NULL REFERENCES memorial_pages(id) ON DELETE CASCADE,
    contributor_name    VARCHAR(200) NOT NULL,
    contributor_email   VARCHAR(255),
    relationship        VARCHAR(100),
    tribute_text        TEXT NOT NULL,
    photo_url           VARCHAR(1000),
    moderation_status   VARCHAR(20) NOT NULL DEFAULT 'pending',
    moderated_by        UUID,
    moderated_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE deceased_relationships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deceased_id         UUID NOT NULL REFERENCES deceased(id),
    related_deceased_id UUID NOT NULL REFERENCES deceased(id),
    relationship_type   VARCHAR(30) NOT NULL CHECK (relationship_type IN (
                            'spouse', 'parent', 'child', 'sibling', 'other'
                        )),
    -- Relationship details (JSONB)
    details             JSONB,
    -- Example: { "marriage_date": "1955-06-12", "marriage_place": "Springfield, IL" }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(deceased_id, related_deceased_id, relationship_type)
);

CREATE INDEX idx_tributes_memorial ON memorial_tributes(memorial_page_id);
CREATE INDEX idx_tributes_status ON memorial_tributes(moderation_status);
CREATE INDEX idx_rel_deceased ON deceased_relationships(deceased_id);
CREATE INDEX idx_rel_related ON deceased_relationships(related_deceased_id);
```

### Audit Trail

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL,
    user_id         UUID,
    user_email      VARCHAR(255),
    user_ip         INET,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Changes stored as JSONB (natural fit for variable column sets)
    old_values      JSONB,
    new_values      JSONB,
    cemetery_id     UUID,
    description     VARCHAR(500)
) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_audit_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_time ON audit_log(timestamp);
CREATE INDEX idx_audit_user ON audit_log(user_id);
```

---

## JSONB Query Examples

```sql
-- Find all sections designated for Muslim burial
SELECT id, name, code, culture_rules->>'faith_designation' AS faith
FROM sections
WHERE culture_rules->>'faith_designation' = 'Muslim';

-- Find spaces in a columbarium with double-size niches
SELECT id, space_number, details->>'structure_name' AS structure
FROM spaces
WHERE space_type = 'columbarium_niche'
  AND details->>'niche_size' = 'double';

-- Search deceased by name aliases (fuzzy historic name matching)
SELECT id, first_name, last_name, bio->'name_aliases' AS aliases
FROM deceased
WHERE bio->'name_aliases' ? 'Johann'     -- contains alias 'Johann'
   OR last_name ILIKE '%smith%';

-- Find contracts with active payment plans past due
SELECT c.contract_number, c.balance_due,
       c.payment_plan->>'monthly_amount' AS monthly,
       c.payment_plan->>'first_payment_date' AS started
FROM contracts c
WHERE c.status = 'active'
  AND c.balance_due > 0
  AND c.payment_plan IS NOT NULL
  AND (c.payment_plan->>'autopay_enrolled')::boolean = false;

-- Find documents with high-confidence OCR extractions
SELECT id, title, ocr_data->>'confidence' AS confidence,
       ocr_data->'extracted_entities' AS entities
FROM documents
WHERE (ocr_data->>'status') = 'completed'
  AND (ocr_data->>'confidence')::numeric > 0.85;

-- Veteran burial statistics with military details
SELECT d.first_name, d.last_name,
       d.military->>'branch' AS branch,
       d.military->>'rank' AS rank,
       d.military->>'wars_served' AS wars
FROM deceased d
WHERE d.is_veteran = true
  AND d.military->>'branch' IS NOT NULL
ORDER BY d.date_of_death DESC;
```

---

## Pros and Cons

### Pros

1. **Single database engine**: Everything runs on PostgreSQL. No need for a separate document database, search engine, or configuration store. Reduces operational complexity and deployment cost.

2. **Schema flexibility where it matters**: Cultural rules, jurisdictional regulations, space-type-specific details, and historic record variations live in JSONB columns that evolve without schema migrations. Adding a new cultural rule for a Sikh section or a new field for green burial certification requires no ALTER TABLE.

3. **Relational integrity where it matters**: Foreign keys ensure spaces reference valid sections, interments reference valid deceased records, and payments reference valid invoices. The structural core is as safe as a fully normalised model.

4. **Natural fit for OCR output**: AI-extracted data from historic ledger scans is inherently semi-structured (varying confidence levels, different fields per document, evolving extraction models). JSONB accommodates this without schema changes.

5. **Custom fields without custom tables**: Cemetery operators can add their own fields via the `custom_fields` JSONB column on spaces, or extend `details` on any entity, without database modifications or custom migration scripts.

6. **Efficient queries**: GIN indexes on JSONB columns enable fast querying of document properties. Common filters (status, dates, types) remain in relational columns with B-tree indexes for optimal performance.

7. **Incremental complexity**: Start with a nearly traditional relational schema (ignore the JSONB columns). As cultural, jurisdictional, and historical variations emerge, absorb them into JSONB without restructuring.

### Cons

1. **No referential integrity within JSONB**: If `line_items` in a contract contains a `space_id`, the database cannot enforce that the space exists. Application-layer validation must compensate.

2. **Query complexity**: JSONB queries use operators (`->`, `->>`, `?`, `@>`) that are less familiar to many SQL developers. Complex nested JSONB queries can be harder to read and optimise than relational joins.

3. **Schema documentation burden**: JSONB columns are flexible but opaque. Without rigorous documentation (or JSON Schema validation), different parts of the application may write different structures into the same column. The examples in comments above are illustrative but must be enforced in code.

4. **Index limitations**: GIN indexes on JSONB support containment (`@>`) and existence (`?`) operators efficiently, but range queries (e.g., "all contributions over $1000") on JSONB fields are less efficient than on relational numeric columns.

5. **Reporting challenges**: Aggregate queries across JSONB fields (e.g., "total material cost across all work orders") require JSONB extraction in the query, which is slower than summing a relational column.

6. **Migration ambiguity**: When a JSONB field becomes important enough to filter/sort/aggregate on, it should be promoted to a relational column. Deciding when to make this transition requires ongoing architectural judgment.

---

## Migration & Scaling Considerations

### JSONB-to-Relational Promotion

When a JSONB field is used frequently in queries, promote it to a relational column:

```sql
-- Example: promoting "military branch" from JSONB to relational column
ALTER TABLE deceased ADD COLUMN military_branch VARCHAR(100);
UPDATE deceased SET military_branch = military->>'branch' WHERE military->>'branch' IS NOT NULL;
CREATE INDEX idx_deceased_mil_branch ON deceased(military_branch) WHERE military_branch IS NOT NULL;
-- Application code updated to write to both; JSONB field retained for backward compatibility
```

### Legacy Data Import

1. **CSV/Excel imports**: Core fields map to relational columns; remaining columns import as JSONB key-value pairs in the `details` or `custom_fields` column
2. **OCR pipeline**: Uploads create a document record with `ocr_data = NULL`; async processing populates the JSONB `ocr_data` field with extracted text and entities
3. **Shapefile/GeoJSON import**: Standard PostGIS tools (`ogr2ogr`, `shp2pgsql`) for spatial data; non-spatial attributes flow into JSONB `details`

### Scaling

| Stage | Strategy |
|-------|----------|
| Single cemetery | Single PostgreSQL instance; no partitioning needed |
| Multi-site | Read replicas for public portal; primary for writes |
| High traffic | Materialised views for genealogy search; Redis cache for map tiles |
| Enterprise | Audit log partitioning; Citus sharding by cemetery_id if needed |

### Backup & Archival

- Standard PostgreSQL WAL archiving and pg_dump
- JSONB data is included in all standard backup methods
- No special handling needed beyond regular PostgreSQL operations

---

## Summary

The hybrid relational + JSONB model is the pragmatic choice for cemetery management. It provides the referential integrity and query performance of a normalised relational model for the stable core (locations, interments, contracts, payments) while absorbing the domain's extensive variability (cultural rules, historic records, space-type details, jurisdictional regulations) into JSONB columns that evolve without migrations. The single-database architecture on PostgreSQL minimises operational complexity. The primary risk is JSONB schema drift -- mitigated by application-layer JSON Schema validation and a disciplined approach to promoting frequently-queried JSONB fields to relational columns. This model is recommended for teams that want relational safety with document flexibility and want to avoid managing multiple database engines.
