# Cemetery Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for digitising cemetery operations -- from plot sales and burial records to maintenance scheduling and public genealogy search.

Cemetery operators -- public authorities, religious organisations, and private memorial parks -- manage physical inventory spanning decades or centuries using paper ledgers and disconnected spreadsheets. This project delivers a modern, self-hostable system that centralises plot management, interment records, financial tracking, and a public-facing genealogy portal, while using AI to solve longstanding problems like historic record digitisation and fuzzy name search.

---

## Why Cemetery Management?

- **Most tools lack APIs.** The majority of incumbent vendors (PlotBox, Chronicle, CemSites, webCemeteries, Pontem) offer no documented public API, locking cemetery data into silos and preventing integration with funeral homes, genealogy platforms, or civic records systems.
- **Premium pricing excludes small operators.** PlotBox and Pontem target large operations with enterprise pricing. The gap between free but minimal tools (Sunrise CMS) and $600+/yr SaaS is largely unaddressed, leaving parish and community cemeteries underserved.
- **Legacy record digitisation is manual and expensive.** Converting decades of paper ledgers into structured data is offered as a costly professional service by vendors like Memorial Mapper and PlotBox, with no end-to-end automated pipeline available.
- **Genealogical search is primitive.** Historic burial records contain spelling variations, transliterations, and errors that none of the reviewed tools handle with phonetic or fuzzy matching -- a major pain point for family researchers.
- **No open-source alternative covers the full workflow.** Sunrise CMS (MIT) offers basic records management without mapping, financials, or a public portal. pg_friedhof provides GIS capability but requires desktop QGIS expertise. Neither approaches the feature breadth needed for day-to-day cemetery operations.

---

## Key Features

### Interactive Cemetery Mapping

- Georeferenced digital map with colour-coded plot status (available, sold, reserved, occupied)
- Click-to-view record from any plot, section, or niche on the map
- GIS support via GeoJSON and OGC WMS/WFS open standards
- Walk-to-grave GPS navigation for visitors and staff

### Burial Records & Inventory

- Comprehensive interment records: deceased details, dates, officiants, funeral home, linked documents
- Plot and niche inventory management with real-time availability tracking
- Immutable audit trails for legal compliance and archival retention (100+ year records)
- Document attachment per record: scanned deeds, burial permits, historic ledger pages, photos

### Sales & Financial Management

- Sales workflow from availability search through contract generation, payment processing, and deed issuance
- Invoicing, payment plans, and perpetual care fund tracking with interest accrual
- Pre-need trust accounting with regulatory compliance per jurisdiction
- PDF deed and certificate generation

### Operations & Maintenance

- Work order creation, assignment, and completion tracking for grounds maintenance
- Calendar-based interment scheduling with conflict detection
- Mobile-first, offline-capable interface for grounds staff in low-connectivity areas
- Multi-faith and multi-cultural section configuration (orientation, spacing, cultural practices)

### Public Genealogy Portal

- Publicly searchable burial database for families and genealogical researchers
- AI-powered natural language and fuzzy/phonetic search across historic names
- Memorial pages with family-uploaded photos and memories
- GEDCOM 7.0 export for interoperability with FamilySearch, Ancestry, and MyHeritage

---

## AI-Native Advantage

AI capabilities address problems that incumbents have not solved. An OCR and named-entity extraction pipeline automates digitisation of photographed ledger pages into structured burial records, replacing expensive manual professional services. Natural language search with phonetic matching handles the historic spelling variations, alternative transcriptions, and data entry errors that make genealogical research difficult in every existing tool. AI-assisted maintenance scheduling prioritises groundskeeping work orders based on seasonal patterns and visitor traffic. Draft obituary generation from structured case data -- pioneered by PlotBox but locked behind proprietary licensing -- becomes available as an open feature.

---

## Tech Stack & Deployment

- **Deployment**: Self-hosted, cloud, or hybrid. The public genealogy portal is designed as an independently scalable tier, separated from the administrative back-end.
- **Geospatial**: Vector plot boundaries over aerial imagery using GeoJSON and OGC WMS/WFS standards. Tile-server architecture for performant high-zoom map rendering.
- **APIs**: REST API with OpenAPI 3.1 specification for funeral home, accounting, and third-party integration -- addressing the most consistent gap across incumbents.
- **Data**: Immutable audit trail architecture for burial records (legal documents with archival retention requirements). PostgreSQL/PostGIS as the recommended spatial database backend.
- **Mobile**: Offline-first progressive web app for grounds staff work orders and plot verification.
- **Standards**: GEDCOM 7.0 for genealogy export; HL7 VRDR FHIR endpoint planned for vital records system interoperability.

---

## Market Context

Cemetery management software is a specialist niche with over 22 rated tools listed on GetApp and Capterra as of 2026. The market is driven by digitisation of legacy paper records and growing public demand for online memorials and genealogy access. Incumbent pricing ranges from Cemify's $599/yr (small cemeteries) to enterprise-tier quotes from PlotBox and Pontem, with Memorial Mapper offering a free starter tier. Primary buyers are municipal governments, religious organisations, private memorial park operators, and funeral home chains managing cemetery operations alongside their core business.

---

## Project Status

> This project is in the **research and specification phase**.
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
