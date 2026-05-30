# Data Model Suggestion 4: Graph Database (Neo4j) + PostgreSQL/PostGIS Polyglot

> Project: Cemetery Management (466) · Generated: 2026-05-26

---

## Overview

This model uses a polyglot persistence architecture centred on a graph database (Neo4j) for the relationship-rich core of cemetery management, with PostgreSQL/PostGIS retained for geospatial data and financial transactions. The insight is that cemetery data is fundamentally a web of relationships: families are related to each other, deceased individuals are related to spaces, spaces are related to sections, ownership chains transfer across generations, and genealogy is explicitly a graph traversal problem.

A graph database treats relationships as first-class citizens rather than join-table afterthoughts. This makes the genealogy portal, ownership chain tracing, family plot clustering, and relationship-based search natural and performant -- queries that would require multiple recursive CTEs or self-joins in a relational model become single Cypher traversals.

The polyglot approach acknowledges that not everything is best served by a graph: geospatial polygon operations are best in PostGIS, financial transactions need ACID guarantees and numeric precision in PostgreSQL, and full-text search may benefit from a dedicated engine. The architecture uses each storage technology for what it does best.

---

## Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Graph Database | Neo4j 5+ (Community or Enterprise) | Core entity graph: people, spaces, relationships, ownership chains, genealogy |
| Relational + Spatial | PostgreSQL 16+ with PostGIS 3.4+ | Geospatial data (plot boundaries, section polygons), financial transactions, audit log |
| Full-Text Search | Neo4j full-text indexes + PostgreSQL pg_trgm | Name search with fuzzy/phonetic matching |
| Sync Layer | Debezium CDC or application-level dual-write | Keep Neo4j and PostgreSQL in sync |
| Cache | Redis | Map tile cache, session store |
| API Layer | Node.js/TypeScript or Go | GraphQL for graph queries, REST for spatial/financial |
| Migrations | Neo4j Migrations (Liquigraph/Neo4j-Migrations) + Flyway (PostgreSQL) | Schema evolution for both databases |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                     API Gateway                      │
├──────────────┬──────────────┬───────────────────────┤
│  GraphQL API │   REST API   │   Public Portal API   │
│  (graph ops) │  (spatial +  │  (genealogy search)   │
│              │   financial) │                       │
└──────┬───────┴──────┬───────┴───────────┬───────────┘
       │              │                   │
       ▼              ▼                   ▼
┌──────────────┐ ┌──────────────┐  ┌──────────────┐
│   Neo4j      │ │  PostgreSQL  │  │    Redis     │
│   Graph DB   │ │  + PostGIS   │  │    Cache     │
│              │ │              │  │              │
│ • Entities   │ │ • Geometries │  │ • Map tiles  │
│ • Relations  │ │ • Financials │  │ • Sessions   │
│ • Genealogy  │ │ • Audit log  │  │ • Search     │
│ • Ownership  │ │ • Documents  │  │   results    │
└──────────────┘ └──────────────┘  └──────────────┘
       │              │
       └──────┬───────┘
              │
      ┌───────▼───────┐
      │  Sync Layer   │
      │  (Debezium /  │
      │   dual-write) │
      └───────────────┘
```

---

## Neo4j Graph Schema

### Node Types (Labels)

```cypher
// ============================================================
// ORGANISATION & CEMETERY NODES
// ============================================================

// Organisation
CREATE CONSTRAINT org_id IF NOT EXISTS
FOR (o:Organisation) REQUIRE o.id IS UNIQUE;

// Properties: id, name, org_type, email, phone, website, address (map),
//             settings (map), created_at, updated_at

// Cemetery
CREATE CONSTRAINT cemetery_id IF NOT EXISTS
FOR (c:Cemetery) REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT cemetery_code IF NOT EXISTS
FOR (c:Cemetery) REQUIRE c.code IS UNIQUE;

// Properties: id, name, code, timezone, status, established_date,
//             centre_lat, centre_lng, default_zoom, tile_url,
//             address (map), map_config (map), regulations (map),
//             created_at, updated_at
// Note: boundary polygon stored in PostGIS, not Neo4j


// ============================================================
// LOCATION HIERARCHY NODES
// ============================================================

// Section
CREATE CONSTRAINT section_id IF NOT EXISTS
FOR (s:Section) REQUIRE s.id IS UNIQUE;

// Properties: id, name, code, section_type, fill_colour, sort_order, status,
//             faith_designation, orientation_rule, spacing_rule_cm,
//             culture_rules (map), pricing (map), created_at, updated_at
// Note: boundary polygon stored in PostGIS

// Lot
CREATE CONSTRAINT lot_id IF NOT EXISTS
FOR (l:Lot) REQUIRE l.id IS UNIQUE;

// Properties: id, lot_number, total_spaces, status, created_at, updated_at

// Space (Plot, Niche, Crypt)
CREATE CONSTRAINT space_id IF NOT EXISTS
FOR (sp:Space) REQUIRE sp.id IS UNIQUE;

// Properties: id, space_number, space_type, max_interments, current_interments,
//             status, base_price, currency_code, gps_latitude, gps_longitude,
//             details (map), created_at, updated_at
// Note: location polygon stored in PostGIS; point coordinates duplicated here for convenience


// ============================================================
// PEOPLE NODES
// ============================================================

// Contact (living person or organisation)
CREATE CONSTRAINT contact_id IF NOT EXISTS
FOR (ct:Contact) REQUIRE ct.id IS UNIQUE;

// Properties: id, contact_type, first_name, last_name, organisation_name,
//             email, phone_primary, is_active, profile (map),
//             created_at, updated_at

// Deceased
CREATE CONSTRAINT deceased_id IF NOT EXISTS
FOR (d:Deceased) REQUIRE d.id IS UNIQUE;

// Properties: id, first_name, middle_name, last_name, maiden_name,
//             date_of_birth, date_of_death, date_of_birth_approx,
//             date_of_death_approx, gender, is_veteran,
//             display_name, name_aliases (list),
//             bio (map), vital_records (map), military (map),
//             legacy_record_data (map), created_at, updated_at

// Full-text index for genealogy search
CREATE FULLTEXT INDEX deceased_name_search IF NOT EXISTS
FOR (d:Deceased) ON EACH [d.first_name, d.middle_name, d.last_name, d.maiden_name, d.display_name];


// ============================================================
// OPERATIONAL NODES
// ============================================================

// Interment
CREATE CONSTRAINT interment_id IF NOT EXISTS
FOR (i:Interment) REQUIRE i.id IS UNIQUE;

// Properties: id, interment_type, interment_date, interment_time,
//             status, burial_permit_number, details (map),
//             scheduling (map), created_at, updated_at

// WorkOrder
CREATE CONSTRAINT work_order_id IF NOT EXISTS
FOR (wo:WorkOrder) REQUIRE wo.id IS UNIQUE;

// Properties: id, work_order_number, work_type, priority, title,
//             description, status, scheduled_date, due_date, completed_date,
//             details (map), created_at, updated_at

// Document
CREATE CONSTRAINT document_id IF NOT EXISTS
FOR (doc:Document) REQUIRE doc.id IS UNIQUE;

// Properties: id, document_type, title, file_name, file_path,
//             file_size_bytes, mime_type, is_public,
//             ocr_data (map), dublin_core (map), created_at

// MemorialPage
CREATE CONSTRAINT memorial_id IF NOT EXISTS
FOR (mp:MemorialPage) REQUIRE mp.id IS UNIQUE;

// Properties: id, slug, is_published, view_count,
//             content (map), settings (map), created_at, updated_at

// MemorialTribute
CREATE CONSTRAINT tribute_id IF NOT EXISTS
FOR (mt:MemorialTribute) REQUIRE mt.id IS UNIQUE;

// Properties: id, contributor_name, contributor_email, relationship,
//             tribute_text, photo_url, moderation_status,
//             moderated_by, moderated_at, created_at


// ============================================================
// FINANCIAL NODES
// (Minimal — core financial data lives in PostgreSQL)
// ============================================================

// Contract
CREATE CONSTRAINT contract_id IF NOT EXISTS
FOR (c:Contract) REQUIRE c.id IS UNIQUE;

// Properties: id, contract_number, contract_type, contract_date,
//             total_amount, status, signed_date, created_at
// Note: Detailed financials (line items, payments) in PostgreSQL

// Deed
CREATE CONSTRAINT deed_id IF NOT EXISTS
FOR (deed:Deed) REQUIRE deed.id IS UNIQUE;

// Properties: id, deed_number, deed_type, deed_date, status,
//             deed_details (map), created_at
```

### Relationship Types

This is where the graph model truly shines. Relationships are the core of cemetery data.

```cypher
// ============================================================
// ORGANISATIONAL RELATIONSHIPS
// ============================================================

// Organisation -[OPERATES]-> Cemetery
(:Organisation)-[:OPERATES {since: date}]->(:Cemetery)

// Cemetery -[HAS_SECTION]-> Section
(:Cemetery)-[:HAS_SECTION]->(:Section)

// Section -[HAS_LOT]-> Lot
(:Section)-[:HAS_LOT]->(:Lot)

// Section -[HAS_SPACE]-> Space (for spaces not in lots)
(:Section)-[:HAS_SPACE]->(:Space)

// Lot -[CONTAINS]-> Space
(:Lot)-[:CONTAINS]->(:Space)


// ============================================================
// INTERMENT RELATIONSHIPS (the core cemetery operation)
// ============================================================

// Interment connects deceased to space
(:Deceased)-[:INTERRED_IN {interment_id: uuid}]->(:Space)
// Alternative direction: (:Space)-[:HOLDS]->(:Deceased)

// Interment event node for rich metadata
(:Interment)-[:OF_DECEASED]->(:Deceased)
(:Interment)-[:IN_SPACE]->(:Space)
(:Interment)-[:OFFICIATED_BY]->(:Contact)
(:Interment)-[:HANDLED_BY]->(:Contact)  // Funeral home


// ============================================================
// FAMILY & GENEALOGY RELATIONSHIPS (the graph advantage)
// ============================================================

// Family relationships between deceased (and living contacts)
(:Deceased)-[:SPOUSE_OF {marriage_date: date, marriage_place: string}]->(:Deceased)
(:Deceased)-[:PARENT_OF]->(:Deceased)
(:Deceased)-[:CHILD_OF]->(:Deceased)
(:Deceased)-[:SIBLING_OF]->(:Deceased)

// Cross-cemetery family connections (critical for multi-site)
// Same relationship types work across cemeteries because relationships
// are between Deceased nodes, not between spaces

// Living family members connected to deceased
(:Contact)-[:NEXT_OF_KIN {relationship: string}]->(:Deceased)
(:Contact)-[:FAMILY_MEMBER {relationship: string}]->(:Deceased)


// ============================================================
// OWNERSHIP RELATIONSHIPS (chain of title over decades)
// ============================================================

// Current and historical ownership
(:Contact)-[:OWNS {
    deed_number: string,
    deed_type: string,
    deed_date: date,
    status: string,      // 'active', 'transferred', 'expired'
    acquired_via: string  // 'purchase', 'inheritance', 'transfer', 'donation'
}]->(:Space)

(:Contact)-[:OWNS]->(:Lot)

// Ownership transfer chain (traversable history)
(:Contact)-[:TRANSFERRED_TO {
    transfer_date: date,
    reason: string,
    deed_number: string
}]->(:Contact)
// With context: this transfer was for Space X
// Modeled as a transfer chain: Contact A -> Contact B -> Contact C
// Each link preserves the deed that governed the transfer


// ============================================================
// DEED RELATIONSHIPS
// ============================================================

(:Deed)-[:COVERS]->(:Space)
(:Deed)-[:COVERS]->(:Lot)
(:Deed)-[:ISSUED_TO]->(:Contact)
(:Deed)-[:SUPERSEDES]->(:Deed)  // Chain of deeds for ownership transfers


// ============================================================
// CONTRACT & FINANCIAL RELATIONSHIPS
// ============================================================

(:Contract)-[:PURCHASED_BY]->(:Contact)
(:Contract)-[:FOR_SPACE]->(:Space)
(:Contract)-[:FOR_LOT]->(:Lot)
(:Contract)-[:GENERATED_DEED]->(:Deed)


// ============================================================
// DOCUMENT RELATIONSHIPS
// ============================================================

(:Document)-[:ATTACHED_TO]->(:Deceased)
(:Document)-[:ATTACHED_TO]->(:Interment)
(:Document)-[:ATTACHED_TO]->(:Space)
(:Document)-[:ATTACHED_TO]->(:Contract)
(:Document)-[:ATTACHED_TO]->(:WorkOrder)
// Multiple documents can attach to multiple entities naturally


// ============================================================
// MEMORIAL RELATIONSHIPS
// ============================================================

(:MemorialPage)-[:MEMORIALISES]->(:Deceased)
(:MemorialTribute)-[:TRIBUTE_ON]->(:MemorialPage)


// ============================================================
// WORK ORDER RELATIONSHIPS
// ============================================================

(:WorkOrder)-[:AT_SECTION]->(:Section)
(:WorkOrder)-[:AT_SPACE]->(:Space)
(:WorkOrder)-[:ASSIGNED_TO]->(:User)
(:WorkOrder)-[:FOR_CEMETERY]->(:Cemetery)
```

---

## PostgreSQL Schema (Companion)

The PostgreSQL database handles what Neo4j does not do well: spatial geometry, financial transactions with ACID guarantees, and immutable audit logging.

```sql
-- ============================================================
-- GEOSPATIAL DATA (PostGIS)
-- ============================================================

CREATE TABLE geo_cemeteries (
    cemetery_id     UUID PRIMARY KEY,  -- Matches Neo4j Cemetery.id
    boundary        GEOMETRY(POLYGON, 4326),
    centre_point    GEOMETRY(POINT, 4326),
    aerial_imagery  JSONB,  -- Tile server configuration
    last_synced     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE geo_sections (
    section_id      UUID PRIMARY KEY,  -- Matches Neo4j Section.id
    cemetery_id     UUID NOT NULL REFERENCES geo_cemeteries(cemetery_id),
    boundary        GEOMETRY(POLYGON, 4326),
    fill_colour     VARCHAR(7),
    last_synced     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE geo_spaces (
    space_id        UUID PRIMARY KEY,  -- Matches Neo4j Space.id
    section_id      UUID NOT NULL REFERENCES geo_sections(section_id),
    location_point  GEOMETRY(POINT, 4326),
    location_polygon GEOMETRY(POLYGON, 4326),
    status          VARCHAR(20),  -- Denormalised from Neo4j for spatial+status queries
    space_type      VARCHAR(30),
    last_synced     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_geo_cemeteries_boundary ON geo_cemeteries USING GIST(boundary);
CREATE INDEX idx_geo_sections_boundary ON geo_sections USING GIST(boundary);
CREATE INDEX idx_geo_spaces_point ON geo_spaces USING GIST(location_point);
CREATE INDEX idx_geo_spaces_polygon ON geo_spaces USING GIST(location_polygon);
CREATE INDEX idx_geo_spaces_status ON geo_spaces(status);


-- ============================================================
-- FINANCIAL TRANSACTIONS (ACID)
-- ============================================================

CREATE TABLE fin_invoices (
    id              UUID PRIMARY KEY,
    invoice_number  VARCHAR(50) UNIQUE NOT NULL,
    contract_id     UUID,       -- References Neo4j Contract.id
    contact_id      UUID,       -- References Neo4j Contact.id
    cemetery_id     UUID,       -- References Neo4j Cemetery.id
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    subtotal        NUMERIC(12, 2) NOT NULL,
    tax_amount      NUMERIC(12, 2) DEFAULT 0,
    total_amount    NUMERIC(12, 2) NOT NULL,
    amount_paid     NUMERIC(12, 2) DEFAULT 0,
    balance_due     NUMERIC(12, 2) GENERATED ALWAYS AS (total_amount - amount_paid) STORED,
    currency_code   CHAR(3) DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    line_items      JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE fin_payments (
    id              UUID PRIMARY KEY,
    invoice_id      UUID NOT NULL REFERENCES fin_invoices(id),
    payment_date    DATE NOT NULL,
    amount          NUMERIC(12, 2) NOT NULL,
    payment_method  VARCHAR(30) NOT NULL,
    reference_number VARCHAR(100),
    processor_data  JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE fin_perpetual_care_funds (
    id              UUID PRIMARY KEY,
    cemetery_id     UUID NOT NULL,
    fund_name       VARCHAR(255) NOT NULL,
    total_corpus    NUMERIC(14, 2) NOT NULL DEFAULT 0,
    total_market_value NUMERIC(14, 2) NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    fund_config     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE fin_perpetual_care_transactions (
    id              UUID PRIMARY KEY,
    fund_id         UUID NOT NULL REFERENCES fin_perpetual_care_funds(id),
    transaction_date DATE NOT NULL,
    transaction_type VARCHAR(30) NOT NULL,
    amount          NUMERIC(14, 2) NOT NULL,
    running_balance NUMERIC(14, 2) NOT NULL,
    description     VARCHAR(500),
    details         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fin_invoices_contract ON fin_invoices(contract_id);
CREATE INDEX idx_fin_invoices_status ON fin_invoices(status);
CREATE INDEX idx_fin_payments_invoice ON fin_payments(invoice_id);
CREATE INDEX idx_fin_pct_fund ON fin_perpetual_care_transactions(fund_id);
CREATE INDEX idx_fin_pct_date ON fin_perpetual_care_transactions(transaction_date);


-- ============================================================
-- AUDIT LOG (immutable, append-only)
-- ============================================================

CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    source_system   VARCHAR(20) NOT NULL,  -- 'neo4j' or 'postgresql'
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    user_id         UUID,
    user_email      VARCHAR(255),
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    old_values      JSONB,
    new_values      JSONB,
    cemetery_id     UUID,
    description     VARCHAR(500)
) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(timestamp);


-- ============================================================
-- DOCUMENT STORAGE METADATA
-- ============================================================

CREATE TABLE doc_files (
    id              UUID PRIMARY KEY,  -- Matches Neo4j Document.id
    file_path       VARCHAR(1000) NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    -- Full OCR text for PostgreSQL full-text search
    ocr_text        TEXT,
    ocr_search      TSVECTOR GENERATED ALWAYS AS (
                        to_tsvector('simple', COALESCE(ocr_text, ''))
                    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_ocr_search ON doc_files USING GIN(ocr_search);
```

---

## Key Graph Queries (Cypher)

### Genealogy Portal: Family Network Discovery

```cypher
// Find all family members buried in any cemetery managed by this organisation
// Starting from a single deceased person, traverse family relationships
MATCH (start:Deceased {id: $deceased_id})
MATCH path = (start)-[:SPOUSE_OF|PARENT_OF|CHILD_OF|SIBLING_OF*1..5]-(relative:Deceased)
MATCH (relative)-[:INTERRED_IN]->(space:Space)<-[:HAS_SPACE|CONTAINS]-(container)
MATCH (container)-[:HAS_LOT|HAS_SPACE*0..1]->(section:Section)<-[:HAS_SECTION]-(cemetery:Cemetery)
RETURN relative.display_name AS name,
       relative.date_of_birth AS birth,
       relative.date_of_death AS death,
       cemetery.name AS cemetery,
       section.name AS section,
       space.space_number AS space,
       [r IN relationships(path) | type(r)] AS relationship_path,
       length(path) AS degrees_of_separation
ORDER BY degrees_of_separation, relative.date_of_death DESC
```

This single query would require a recursive CTE with multiple self-joins in SQL. In a graph, it is a natural traversal.

### Ownership Chain Tracing

```cypher
// Trace the complete ownership history of a space
MATCH (space:Space {id: $space_id})
MATCH (owner:Contact)-[owns:OWNS]->(space)
OPTIONAL MATCH chain = (owner)<-[:TRANSFERRED_TO*]-(previous_owners:Contact)
RETURN owner.first_name + ' ' + owner.last_name AS current_owner,
       owns.deed_number AS deed,
       owns.deed_date AS acquired,
       owns.status AS status,
       [n IN nodes(chain) | n.first_name + ' ' + n.last_name] AS previous_owners,
       [r IN relationships(chain) | {
           from: startNode(r).last_name,
           to: endNode(r).last_name,
           date: r.transfer_date,
           reason: r.reason
       }] AS transfer_history
ORDER BY owns.deed_date DESC
```

### Family Plot Clustering

```cypher
// Find families with multiple interments for targeted sales outreach
// (e.g., a family with 3 people buried may want to purchase adjacent plots)
MATCH (d1:Deceased)-[:INTERRED_IN]->(s1:Space)<-[:HAS_SPACE|CONTAINS]-(container1)
MATCH (container1)-[:HAS_LOT|HAS_SPACE*0..1]->(sec:Section)<-[:HAS_SECTION]-(cem:Cemetery {id: $cemetery_id})
MATCH (d1)-[:SPOUSE_OF|PARENT_OF|CHILD_OF|SIBLING_OF*1..3]-(d2:Deceased)
MATCH (d2)-[:INTERRED_IN]->(s2:Space)
WITH cem, d1, collect(DISTINCT d2) AS family_members,
     collect(DISTINCT s1) + collect(DISTINCT s2) AS family_spaces
WHERE size(family_members) >= 2
RETURN d1.last_name AS family_name,
       size(family_members) + 1 AS family_size,
       [m IN family_members | m.display_name] AS members,
       [s IN family_spaces | s.space_number] AS spaces_used
ORDER BY family_size DESC
```

### Genealogy Search with Fuzzy Matching

```cypher
// Full-text search with fuzzy matching for the public portal
CALL db.index.fulltext.queryNodes('deceased_name_search', $search_term + '~')
YIELD node AS deceased, score
MATCH (deceased)-[:INTERRED_IN]->(space:Space)
MATCH (space)<-[:HAS_SPACE|CONTAINS]-(container)
MATCH (container)-[:HAS_LOT|HAS_SPACE*0..1]->(section:Section)<-[:HAS_SECTION]-(cemetery:Cemetery)
OPTIONAL MATCH (deceased)<-[:MEMORIALISES]-(memorial:MemorialPage {is_published: true})
RETURN deceased.display_name AS name,
       deceased.date_of_birth AS birth,
       deceased.date_of_death AS death,
       deceased.is_veteran AS veteran,
       cemetery.name AS cemetery,
       section.name AS section,
       space.space_number AS space,
       space.gps_latitude AS lat,
       space.gps_longitude AS lng,
       memorial.slug AS memorial_url,
       score
ORDER BY score DESC
LIMIT 50
```

### Interment Scheduling Conflict Detection

```cypher
// Check for scheduling conflicts at a space and adjacent spaces
MATCH (space:Space {id: $space_id})<-[:IN_SPACE]-(existing:Interment)
WHERE existing.status IN ['scheduled', 'confirmed']
  AND existing.interment_date = $requested_date
RETURN existing.id AS conflicting_interment,
       existing.interment_date AS date,
       existing.details.scheduled_time AS time,
       'same_space' AS conflict_type

UNION

// Also check adjacent spaces in the same lot
MATCH (space:Space {id: $space_id})<-[:CONTAINS]-(lot:Lot)-[:CONTAINS]->(adjacent:Space)
MATCH (adjacent)<-[:IN_SPACE]-(existing:Interment)
WHERE existing.status IN ['scheduled', 'confirmed']
  AND existing.interment_date = $requested_date
  AND adjacent.id <> $space_id
RETURN existing.id AS conflicting_interment,
       existing.interment_date AS date,
       existing.details.scheduled_time AS time,
       'adjacent_space' AS conflict_type
```

### GEDCOM Export (Family Tree Extraction)

```cypher
// Extract a family tree rooted at a deceased person for GEDCOM export
// This query naturally produces the INDI and FAM records needed for GEDCOM 7.0
MATCH (root:Deceased {id: $deceased_id})
MATCH path = (root)-[:SPOUSE_OF|PARENT_OF|CHILD_OF*1..10]-(relative:Deceased)
WITH collect(DISTINCT relative) + [root] AS all_individuals
UNWIND all_individuals AS person
OPTIONAL MATCH (person)-[spouse:SPOUSE_OF]-(partner:Deceased)
  WHERE partner IN all_individuals
OPTIONAL MATCH (person)-[:CHILD_OF]->(parent:Deceased)
  WHERE parent IN all_individuals
OPTIONAL MATCH (person)-[:INTERRED_IN]->(space:Space)
OPTIONAL MATCH (space)<-[:HAS_SPACE|CONTAINS]-(container)-[:HAS_LOT|HAS_SPACE*0..1]->(section:Section)<-[:HAS_SECTION]-(cemetery:Cemetery)
RETURN person.id AS indi_id,
       person.first_name AS given_name,
       person.last_name AS surname,
       person.maiden_name AS maiden,
       person.date_of_birth AS birth_date,
       person.date_of_death AS death_date,
       person.gender AS sex,
       person.bio AS bio,
       collect(DISTINCT {partner_id: partner.id, marriage_date: spouse.marriage_date}) AS marriages,
       collect(DISTINCT parent.id) AS parent_ids,
       cemetery.name AS burial_place,
       space.space_number AS burial_plot
```

---

## Data Synchronisation Strategy

Since the system uses both Neo4j and PostgreSQL, keeping them in sync is critical.

### Option A: Application-Level Dual Write

```typescript
// Transactional dual-write pattern
async function sellSpace(cmd: SellSpaceCommand): Promise<void> {
    const neo4jSession = neo4jDriver.session();
    const pgClient = await pgPool.connect();

    try {
        // 1. Start transactions in both databases
        const neo4jTx = neo4jSession.beginTransaction();
        await pgClient.query('BEGIN');

        // 2. Update Neo4j graph
        await neo4jTx.run(`
            MATCH (space:Space {id: $spaceId})
            MATCH (buyer:Contact {id: $buyerId})
            SET space.status = 'sold'
            CREATE (buyer)-[:OWNS {
                deed_number: $deedNumber,
                deed_type: $deedType,
                deed_date: date($saleDate),
                status: 'active',
                acquired_via: 'purchase'
            }]->(space)
            CREATE (contract:Contract {
                id: $contractId,
                contract_number: $contractNumber,
                contract_type: 'at_need',
                total_amount: $totalAmount,
                status: 'active',
                created_at: datetime()
            })
            CREATE (contract)-[:PURCHASED_BY]->(buyer)
            CREATE (contract)-[:FOR_SPACE]->(space)
        `, cmd);

        // 3. Update PostgreSQL (spatial status + financials)
        await pgClient.query(
            'UPDATE geo_spaces SET status = $1, last_synced = now() WHERE space_id = $2',
            ['sold', cmd.spaceId]
        );
        await pgClient.query(
            `INSERT INTO fin_invoices (id, invoice_number, contract_id, contact_id, cemetery_id,
                                       invoice_date, due_date, total_amount, status, line_items)
             VALUES ($1, $2, $3, $4, $5, $6, $7, $8, 'sent', $9)`,
            [cmd.invoiceId, cmd.invoiceNumber, cmd.contractId, cmd.buyerId,
             cmd.cemeteryId, cmd.saleDate, cmd.dueDate, cmd.totalAmount, cmd.lineItemsJson]
        );

        // 4. Audit log in PostgreSQL
        await pgClient.query(
            `INSERT INTO audit_log (source_system, entity_type, entity_id, action, user_id, cemetery_id, new_values)
             VALUES ('neo4j', 'Space', $1, 'SOLD', $2, $3, $4)`,
            [cmd.spaceId, cmd.userId, cmd.cemeteryId, JSON.stringify(cmd)]
        );

        // 5. Commit both transactions
        await neo4jTx.commit();
        await pgClient.query('COMMIT');
    } catch (error) {
        // Rollback both on failure
        await neo4jTx.rollback();
        await pgClient.query('ROLLBACK');
        throw error;
    } finally {
        await neo4jSession.close();
        pgClient.release();
    }
}
```

### Option B: Change Data Capture (CDC) with Debezium

For systems where dual-write complexity is unacceptable, use Debezium to capture changes from PostgreSQL and sync to Neo4j (or vice versa) via a message queue.

```
PostgreSQL (writes) → Debezium CDC → Kafka/NATS → Neo4j Sync Consumer
Neo4j (writes) → Neo4j Change Streams → Kafka/NATS → PostgreSQL Sync Consumer
```

This provides eventual consistency between the two databases with at-least-once delivery guarantees.

---

## Pros and Cons

### Pros

1. **Genealogy is a graph problem**: Family trees, relationship traversals, and degrees-of-separation queries are the natural language of graph databases. Finding all relatives of a deceased person buried across multiple cemeteries is a single Cypher query, not a recursive CTE.

2. **Ownership chains are naturally traversable**: Plot ownership transfers across generations (A bought it in 1950, left it to B in 1975, B transferred to C in 2010) form a chain that is elegant in a graph and awkward in a relational self-join.

3. **Flexible schema for relationships**: Adding a new relationship type (e.g., `:GODPARENT_OF`, `:BUSINESS_PARTNER_OF`) requires no schema migration -- just create edges with the new type. This is important for the diverse cultural and religious relationship conventions a cemetery system must accommodate.

4. **GEDCOM export is natural**: GEDCOM's data model (INDI records linked by FAM records) maps directly to graph nodes and edges. Extracting a family tree for GEDCOM export is a single traversal query.

5. **Multi-cemetery family connections**: When families have members buried across multiple cemeteries managed by the same organisation, graph traversals naturally cross cemetery boundaries. In a relational model, this requires cross-schema joins or denormalisation.

6. **Visual data exploration**: Neo4j's browser and Bloom tools provide visual graph exploration that is invaluable for genealogical researchers and for debugging data quality issues in historic records.

7. **PostGIS retained for spatial**: By keeping geometry data in PostGIS, the system gets the full power of OGC-compliant spatial operations (ST_Contains, ST_Intersects, nearest-neighbour) without forcing them into a graph model where they do not belong.

### Cons

1. **Polyglot complexity**: Operating two database engines (Neo4j + PostgreSQL) doubles the operational burden: two backup strategies, two monitoring systems, two upgrade paths, two sets of expertise needed.

2. **Data synchronisation risk**: Keeping Neo4j and PostgreSQL in sync is the single hardest technical challenge. Dual-write has failure modes (one succeeds, one fails); CDC has eventual consistency delays. Either approach requires careful engineering.

3. **No native ACID across databases**: A space sale that updates Neo4j and creates a PostgreSQL invoice cannot be in a single transaction. The application must implement compensating transactions or accept eventual consistency.

4. **Smaller talent pool**: Neo4j/Cypher expertise is far less common than SQL/PostgreSQL expertise. Hiring, onboarding, and community support are all more constrained.

5. **Financial data limitations**: Graph databases are not designed for numeric precision, aggregation, and financial reporting. Storing invoices, payments, and perpetual care fund balances in Neo4j would be risky. Hence the PostgreSQL companion -- but that creates the sync problem.

6. **Licensing considerations**: Neo4j Community Edition is open source (GPLv3), but lacks clustering, role-based access control, and some enterprise features. Neo4j Enterprise requires a commercial licence. For a self-hosted open-source project, this creates a tension.

7. **Overkill for simple operations**: Most day-to-day cemetery operations (record a burial, create a work order, process a payment) do not benefit from graph traversals. The graph advantage is concentrated in genealogy and ownership queries.

8. **Geospatial gap**: Neo4j has limited native spatial support (point indexes only). Polygon operations (plot boundary containment, section area calculation, overlapping detection) must go to PostGIS, splitting the query path.

---

## Migration & Scaling Considerations

### Initial Data Migration

1. **Import into Neo4j**: Use `neo4j-admin database import` for bulk loading from CSV. Map cemetery hierarchy and interment records to nodes, then create relationships in a second pass.

2. **Import into PostGIS**: Use `ogr2ogr` or `shp2pgsql` for spatial data. Standard CSV import for financial data.

3. **Cross-reference IDs**: All entities use the same UUID primary key in both Neo4j and PostgreSQL. This is the synchronisation key.

4. **Legacy relationship inference**: For historic records where explicit family relationships are not recorded, use surname matching and shared lot ownership to infer probable relationships. Store these as `:PROBABLE_FAMILY` edges with a confidence score.

### Scaling Strategies

| Growth stage | Neo4j | PostgreSQL |
|-------------|-------|------------|
| Single cemetery | Single Neo4j Community instance | Single PostgreSQL instance |
| Multi-site (5-20) | Neo4j Community with more RAM | PostgreSQL read replicas for portal |
| Large operator | Neo4j Enterprise with read replicas | PostgreSQL with Citus or partitioning |
| Enterprise | Neo4j Aura (managed) or clustered | Managed PostgreSQL (RDS/Cloud SQL) |

### Backup & Recovery

- **Neo4j**: `neo4j-admin database dump` for offline backups; online backup (Enterprise only) for hot backups
- **PostgreSQL**: Standard WAL archiving + pg_dump
- **Cross-database consistency**: After restore, run a reconciliation job to verify UUID cross-references between Neo4j and PostgreSQL

### When to Choose This Model

Choose the graph + polyglot approach if:

- The genealogy portal is a primary product differentiator and must support multi-generational family exploration
- The system will manage many cemeteries where families are scattered across sites
- Ownership chain tracing and deed lineage are legally critical operations
- The team has Neo4j experience or is willing to invest in learning it
- The organisation can afford the operational overhead of two databases

Choose a simpler model (Suggestion 1 or 3) if:

- The genealogy portal is a secondary feature
- Most cemeteries are small single-site operations
- The team's expertise is primarily in SQL/PostgreSQL
- Operational simplicity is a priority over query elegance

---

## Summary

The graph database approach is the most architecturally specialised of the four suggestions. It excels precisely where cemetery management has its most distinctive requirements: multi-generational family relationships, century-spanning ownership chains, and genealogical search across interconnected records. Neo4j makes these queries natural and performant, while PostgreSQL/PostGIS handles what graphs do not do well (spatial geometry, financial transactions, full-text search). The trade-off is significant operational complexity from running two database engines and keeping them synchronised. This model is best suited for organisations where the genealogy portal and family relationship features are the primary value proposition, and where the team has the expertise to manage a polyglot architecture.
