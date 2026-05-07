# Cemetery Management — Feature & Functionality Survey

> Candidate #466 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| PlotBox | SaaS (cloud) | Commercial — subscription | https://plotbox.com |
| Everspot | SaaS (cloud) | Commercial — subscription | https://everspot.io |
| Cemify | SaaS (cloud) | Commercial — from $599/yr | https://www.cemify.com |
| Memorial Mapper CMS | SaaS (cloud) | Freemium (free starter tier) | https://memorialmapper.com/cms |
| Chronicle | SaaS (cloud) | Commercial — subscription | https://chronicle.rip |
| CemSites | SaaS (cloud) | Commercial — subscription | https://www.cemsites.com |
| EVERARK | SaaS (cloud) | Commercial — subscription | https://www.everark.io |
| Pontem Cemetery Data Manager | On-premise / cloud | Commercial — 30+ yr vendor | https://pontem.com |
| InStone (SRS Computing) | On-premise / cloud | Commercial — subscription | https://www.srscomputing.com/instone-new |
| webCemeteries | SaaS (cloud) | Commercial — subscription | https://webcemeteries.com |
| ArcGIS Cemetery Management (Esri) | Platform extension | Commercial — ArcGIS licence | https://www.arcgis.com/apps/solutions/cemetery-management |
| Sunrise CMS | Web app | Open source (MIT) | https://github.com/cityssm/sunrise-cms |
| pg_friedhof | Desktop (QGIS plugin) | Open source | https://pbsgeo.com/en/pg_friedhof-cemetery-management-software/ |
| CityView (municipal) | Platform module | Commercial — municipal | https://www.municipalsoftware.com/cemetery-management/ |

---

## Feature Analysis by Solution

### PlotBox

**Core features**
- Interactive drone-imagery-based cemetery map with real-time inventory status (sold, available, reserved)
- 360-degree immersive virtual tours of mausolea and columbaria with live inventory overlaid
- Contract management: availability search, price lists, online payment, and contract generation
- Burial and interment records linked to plot map; single searchable database
- AI Obituary Assistant: auto-generates personalised obituaries from case data already in the system
- Finance suite: invoicing, receipts, trust fund management, perpetual care tracking
- Work order management: real-time creation, assignment, and completion tracking
- Document management: scanned deeds, permits, and historic records linked to plots
- EverAfter public portal: family search, visit planning, "walk-to-grave" directions
- Funeral home integration (case data import from platforms such as MorTrack)

**Differentiating features**
- AI-powered obituary generation (launched ~2025) is the first AI-native feature in the category
- Verified mapping service: PlotBox technicians photograph and GPS-verify each plot
- 360-degree virtual walkthroughs of indoor columbarium spaces
- MorTrack API integration for direct case-data sync from funeral management software

**UX patterns**
- Integrated map-first interface; clicking a plot reveals full record inline
- Staff and family-facing tiers with separate UI contexts
- Progressive onboarding via professional mapping service removes DIY barrier

**Integration points**
- Payment processing (online credit/debit)
- MorTrack funeral home case management API
- Funeral home software data import
- EverAfter public portal as separate scalable tier

**Known gaps**
- Premium pricing places it out of reach for very small or volunteer-managed cemeteries
- Mapping requires PlotBox professional service; limits DIY adoption
- API access not publicly documented for third-party developers

**Licence / IP notes**
- Proprietary SaaS; no open source components. AI obituary feature is a proprietary add-on.

---

### Everspot

**Core features**
- Digital cemetery mapping: plot availability, ownership, and burial visualisation
- Public burial search portal for visitors and genealogical researchers
- Sales contracts and payment plans (pre-need and at-need)
- Work order management: creation, assignment, and tracking
- Perpetual care and pre-need trust accounting
- Document storage with record linkage to plots
- CRM and contact management
- QuickBooks Online 360-degree integration (bidirectional sync)
- Liability tracking and property management tools
- Automated commission calculations for sales staff

**Differentiating features**
- Built by former cemetery operators — deep domain workflow knowledge
- QuickBooks Online as native accounting integration (full bidirectional sync, not export-only)
- Automated commissions module for sales staff incentive management
- Exposed API for third-party integrations (documented via their support/training materials)

**UX patterns**
- Operations-first design reflecting cemetery manager workflows
- Training offered via documentation, live online, webinars, in person, and video

**Integration points**
- QuickBooks Online (bidirectional)
- Public API (available; documentation via training channels)
- Payment processor integration

**Known gaps**
- Public API documentation not readily available via a developer portal
- Less prominent mapping aesthetics compared to drone-imagery-first competitors

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Cemify

**Core features**
- Interactive digital map converted from paper cemetery maps; colour-coded plot status
- Click-to-edit plot records: burials, ownership, sales, document uploads, photos
- Custom fields for records and plots; detailed filters and search
- Change log and tracking for all record modifications
- Document generation: proof of ownership certificates for sold plots
- Work orders from templates, printable or email-deliverable
- Public burial search portal
- Mobile-accessible (cloud-based, any device)

**Differentiating features**
- Accessible pricing ($599/yr) targeted at small parish and community cemeteries
- DIY map conversion service from paper originals
- Highly praised customer support cited repeatedly in reviews as a differentiator

**UX patterns**
- Simplified interface designed for non-technical cemetery volunteers and administrators
- Map colour-coding provides instant plot-status overview without training

**Integration points**
- Online payment portal
- Email work order delivery

**Known gaps**
- No native accounting or trust fund management module
- No funeral home / case management integration
- Limited API or webhook capability noted in reviews

**Licence / IP notes**
- Proprietary SaaS.

---

### Memorial Mapper CMS

**Core features**
- High-resolution interactive digital map (clickable plots with burial details and ownership)
- Cemetery Management System (CMS) as central hub for interment records, deed info, plot sales, and financial reporting
- Automated online memorial pages per burial (family photo/memory uploads)
- "Walk-to-grave" smartphone GPS navigation
- Record digitisation service: converts paper ledgers to .xlsx/.csv
- Public searchable burial database

**Differentiating features**
- Free Starter Plan (plot tracking, payment portal, memorial page per burial) lowers barrier to entry for small cemeteries
- Memorial pages as first-class citizen: every interment auto-generates a shareable online memorial
- Digitisation service as standalone product (not just software migration)

**UX patterns**
- Consumer-grade public memorial UX (sharing, memories, photos) alongside admin management tools
- Freemium model encourages adoption before upsell

**Integration points**
- Payment portal
- Social sharing for memorial pages

**Known gaps**
- Free tier lacks advanced financial and trust management
- No documented API
- Unclear enterprise pricing for multi-site operators

**Licence / IP notes**
- Proprietary SaaS. Free tier with commercial upgrade path.

---

### Chronicle

**Core features**
- Survey-grade GPS-accurate plot mapping linking each burial record to exact grave location
- Work order creation, assignment, and tracking for grounds maintenance
- Deed and certificate upload and secure storage
- Interactive history and storytelling: burial records linked to genealogical data, photos, and narratives
- Cemetery database with reporting and statistics
- Built-in interment scheduling

**Differentiating features**
- Strongest historical preservation and storytelling focus in the category
- Survey-grade GPS accuracy (not drone imagery) for legally precise plot boundaries
- Community history platform positioning beyond operational software

**UX patterns**
- Map-centric, visually rich interface emphasising historical narrative
- Genealogy-forward public portal

**Integration points**
- Limited: closed API noted in competitive reviews
- Deed and document storage

**Known gaps**
- Closed API limits integration with funeral home or accounting software
- Less focus on financial/trust accounting features
- Limited payment processing capability

**Licence / IP notes**
- Proprietary SaaS. Closed API.

---

### CemSites

**Core features**
- Cloud cemetery management: record keeping, grave mapping, sales, finance management, customer CRM
- Interactive mapping with GPS walk-to-grave navigation
- Sales module with lead capture and marketing tools
- E-commerce: flowers, wreaths, online memorial purchases
- Pre-need trust management and perpetual care fund tracking
- Detailed reporting and analytics (plot occupancy, sales performance)
- Maintenance scheduling for plots
- PCI and HIPAA compliant platform

**Differentiating features**
- E-commerce module (floral sales, memorial products) as revenue generation for cemetery
- HIPAA compliance coverage (beyond typical payment-only certification)
- Lead capture and marketing pipeline for plot sales

**UX patterns**
- Sales and marketing orientation alongside operational management
- Consumer purchase flow for cemetery merchandise

**Integration points**
- Payment processing (PCI-compliant)
- Email and marketing tools

**Known gaps**
- Mixed user reviews: some report poor mapping quality (dot-on-map rather than true overlays)
- Some users report unresponsive support and stability issues
- No documented public API

**Licence / IP notes**
- Proprietary SaaS.

---

### EVERARK

**Core features**
- Plot mapping with GPS coordinates and interactive map
- Digital records management for all burial and ownership data
- Sales automation with performance dashboards for sales teams
- Customer management and order management workflows
- Work order management for cemetery operations
- Financial tools: invoicing, payment processing, trust accounting
- E-commerce site for digital legacy packages and cemetery services
- Secure data management and reporting

**Differentiating features**
- Sales team performance dashboards (booking and revenue KPIs)
- Digital legacy packages as a product category (pre-need memorialisation bundles)
- Dedicated AI cemetery software positioning — AI feature roadmap published

**UX patterns**
- Sales-team-oriented dashboards as primary operational view alongside records management
- Personal workspace concept for individual staff members

**Integration points**
- Payment processing
- E-commerce for legacy packages

**Known gaps**
- Newer entrant; fewer independent user reviews than established competitors
- AI features in roadmap but limited details on delivered capabilities

**Licence / IP notes**
- Proprietary SaaS.

---

### Pontem Cemetery Data Manager

**Core features**
- Data management for burial records, plot tracking, and administrative functions
- Two integrated mapping options including full GIS integration (Esri-based)
- Cloud hosting with web and mobile apps
- Built-in accounting: ledger, accounts payable/receivable, financial reports
- Public online burial search
- Real-time reporting and customisation
- 850+ successful US implementations over 30+ years

**Differentiating features**
- Longest-standing specialist vendor (30+ years) with deep domain expertise
- Native Esri GIS integration (among the most sophisticated mapping options in the category)
- Built-in accounting module with full AP/AR (not reliant on QuickBooks)

**UX patterns**
- Deep customisation orientation suited to large and complex operations
- Enterprise-grade data reliability (SQL clustering, nightly geo-redundant backup)

**Integration points**
- Esri ArcGIS (native GIS integration)
- API: no public documentation found; enterprise integrations likely via professional services

**Known gaps**
- No documented public API or webhook capability
- Interface may feel dated compared to newer SaaS competitors
- Higher complexity may deter smaller cemeteries

**Licence / IP notes**
- Proprietary commercial software.

---

### InStone (SRS Computing)

**Core features**
- Interactive cemetery map using Google Maps base layer plus custom drawn overlays
- Grave status displayed on map; on-the-fly space assignment
- Custom task lists and work order templates for staff assignments
- Printed directions to any grave from any starting point
- SQL-based data storage with geo-redundant clustering and nightly offsite backup
- Funeral home suite integration (SRS PROCession funeral management software)

**Differentiating features**
- Tight integration with SRS's funeral home management software (end-to-end deathcare suite)
- High-availability SQL cluster architecture with automatic failover
- Google Maps base with hand-drawn overlay for low-cost implementation

**UX patterns**
- Task-list and work order focus for operational staff
- Print-to-paper grave directions (useful for on-site visitor assistance)

**Integration points**
- SRS PROCession funeral home software (native integration)
- Google Maps (base layer)

**Known gaps**
- Google Maps base layer less precise than drone imagery or survey-grade GPS
- No public API documented
- Primarily US-focused; limited international case studies

**Licence / IP notes**
- Proprietary commercial software.

---

### webCemeteries

**Core features**
- Digital property inventory and interactive mapping
- Work order creation and tracking
- Integrated payment processing with autopay and invoicing
- CRM and contact history per individual
- Online family portal: burial search, memorial sharing
- Document generation: deeds and contracts
- Role-based user access for staff and vendors
- GPS walk-to-grave navigation
- Digitisation services for paper records and maps

**Differentiating features**
- US-based support team (a differentiator in a market with offshore support providers)
- Autopay capability for payment plans
- Comprehensive role-based access control for multi-staff operations

**UX patterns**
- All-in-one cloud approach with onboarding and training included

**Integration points**
- Payment processing
- No API available (confirmed by Capterra listing)

**Known gaps**
- No API — limits integration with funeral homes, accounting software, or automation platforms
- Fewer advanced financial/trust features than Pontem or CemSites

**Licence / IP notes**
- Proprietary SaaS. No API.

---

### ArcGIS Cemetery Management (Esri)

**Core features**
- Authoritative cemetery inventory built with ArcGIS Pro GIS data tools
- Field verification app (Cemetery Field Map) for staff to walk grounds and update records
- Cemetery Manager App for administrators to manage authoritative data
- Public-facing memorial search and memorial submission by families
- Burial and ownership information linked to georeferenced gravesites
- Integration with broader Esri local government solution stack

**Differentiating features**
- Enterprise GIS platform — highest-fidelity spatial data model in the category
- Part of an Esri Solutions ecosystem (integrates with permits, assets, public works)
- Open OGC standards compliance (WMS, WFS) for data interoperability

**UX patterns**
- GIS-professional setup with configuration tooling; requires ArcGIS expertise to deploy
- Field app for ground staff using Survey123 or Field Maps

**Integration points**
- Full Esri ArcGIS platform APIs (REST, JavaScript SDK)
- OGC WMS/WFS for external data consumers
- ArcGIS Online for public-facing apps

**Known gaps**
- Requires ArcGIS licence (significant cost) and GIS expertise
- Not a standalone product — dependent on broader Esri stack
- Too complex for small or volunteer-run cemeteries

**Licence / IP notes**
- Proprietary Esri licence. Solution templates are available free via ArcGIS Solutions but require paid ArcGIS platform.

---

### Sunrise CMS (Open Source)

**Core features**
- Web-based cemetery records management (completely free, MIT licence)
- Optional map integration (maps not required — reduces complexity)
- Basic burial records database
- Designed for low-cost self-hosted deployment (no expensive server required)
- Maintained by City of Sault Ste. Marie, Ontario via GitHub (cityssm/sunrise-cms)

**Differentiating features**
- Only actively maintained open-source cemetery management system found
- Municipal heritage: built for real operational use by a Canadian city
- Map-optional design makes deployment accessible with minimal data

**UX patterns**
- Minimal, functional interface focused on record keeping over visual mapping
- Self-hosted deployment model for IT-capable organisations

**Integration points**
- Open source — extensible by developers
- No built-in third-party integrations

**Known gaps**
- No financial/trust management module
- No public portal or memorial page features
- No mapping unless custom-built
- Limited feature set compared to commercial offerings

**Licence / IP notes**
- MIT licence. Fully open source.

---

### pg_friedhof (QGIS Plugin)

**Core features**
- Desktop GIS application built on QGIS for cemetery spatial data management
- Comprehensive GIS functionality: spatial queries, burial location management, map printing
- Android and iOS mobile apps for field access
- Integration with PostgreSQL/PostGIS database backend
- European (German-language) origin with international use

**Differentiating features**
- Full GIS capability at open-source cost (QGIS + PostgreSQL stack)
- Mobile apps for both Android and iOS
- Best open-source option for organisations that need sophisticated spatial analysis

**UX patterns**
- GIS-professional desktop interface; requires QGIS knowledge
- Not suited to non-technical administrators

**Integration points**
- PostgreSQL/PostGIS database (standard SQL access)
- QGIS plugin ecosystem

**Known gaps**
- Desktop-first, not cloud-native
- No public portal or family-facing features
- Steep learning curve for non-GIS users
- German documentation primary; English materials limited

**Licence / IP notes**
- Open source (QGIS is GNU GPL). No IP restrictions.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Interactive digital cemetery map with colour-coded plot status (available, sold, reserved, occupied)
- Burial and interment records with full deceased details, dates, section/plot reference
- Plot and niche inventory management with availability tracking
- Document storage linked to individual records (deeds, permits, certificates)
- Public family search portal for locating burials
- Work order management for grounds maintenance
- Sales workflow: availability search, contract generation, payment processing
- Reporting (statutory burial registers, plot inventory, financial summaries)

### Differentiating Features
- AI-driven obituary generation from structured case data (PlotBox)
- Survey-grade GPS accuracy for legally precise plot boundaries (Chronicle)
- Drone/aerial imagery as base map layer (PlotBox)
- 360-degree virtual tour of enclosed spaces such as mausolea (PlotBox)
- Built-in e-commerce for memorial merchandise (CemSites, EVERARK)
- Sales team performance dashboards with revenue KPIs (EVERARK)
- Bidirectional QuickBooks Online sync (Everspot)
- Full Esri GIS enterprise integration (Pontem, ArcGIS Solution)
- Free tier with full feature access for small cemeteries (Memorial Mapper)
- Digital legacy / pre-need memorial packages as e-commerce product (EVERARK)

### Underserved Areas / Opportunities
- **Public API / developer ecosystem**: Most vendors lack documented REST APIs; Everspot is an exception. Cemetery data is siloed, limiting integration with funeral homes, genealogy platforms, and civic records systems.
- **AI-assisted record digitisation**: OCR and AI extraction from scanned historic ledgers is largely a manual professional service; no tool automates this pipeline end-to-end.
- **Offline mobile access for groundskeepers**: Several vendors mention mobile apps but few explicitly support offline-first operation in areas with poor connectivity.
- **Multi-faith section rules**: Configurable interment rules per section (orientation, spacing, cultural practices) are mentioned as a requirement but rarely highlighted as a feature.
- **GEDCOM export for genealogy platforms**: None of the reviewed tools explicitly support GEDCOM 7.0 export to enable families to connect burial records with genealogy trees in FamilySearch, Ancestry, or MyHeritage.
- **Accessible pricing for very small cemeteries**: Open-source options are limited to Sunrise CMS (minimal features) or pg_friedhof (high GIS complexity). The gap between free tools and $600/yr SaaS is largely unaddressed.
- **HL7 VRDR integration**: No tool connects burial records to the HL7 Vital Records Death Reporting FHIR standard for interoperability with public health vital records systems.
- **AI-powered natural language search**: Searching historic records by approximate name spelling, alternative transcriptions, or phonetic matching is not offered by any tool reviewed.

### AI-Augmentation Candidates
- **Legacy record transcription**: AI OCR and named-entity extraction from photographed ledger pages to populate structured burial records
- **Obituary drafting**: AI generation of draft obituaries from structured deceased data (PlotBox has pioneered this)
- **Natural language search**: Fuzzy/phonetic search across burial names for genealogical research (historic spelling variation is a major pain point)
- **Maintenance prioritisation**: AI scheduling of groundskeeping work orders based on seasonal patterns, visitor traffic, and reported issues
- **Plot recommendation**: AI-assisted plot suggestion based on family preferences (section, proximity to existing family plots, pricing tier)
- **Anomaly detection in records**: Flagging duplicate interments, conflicting ownership records, or data migration errors during digitisation

---

## Legal & IP Summary

No patent or copyright concerns were identified for an open-source AI-native cemetery management system. All reviewed commercial software is proprietary SaaS with no open-source components (except Sunrise CMS under MIT and pg_friedhof under GPL). GEDCOM 7.0 is an openly published specification by FamilySearch with no licence restrictions on implementation. GeoJSON, OGC WMS/WFS, and OpenAPI are open standards with no IP restrictions. The EVERARK, PlotBox, and Chronicle brands and product names are trademarks of their respective owners and must not be used. No evidence of patents on any specific cemetery software feature was found.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Interactive digital cemetery map with plot status colour-coding and click-to-view record
- Burial and interment records database with full deceased and plot details
- Plot/niche inventory management with availability search
- Sales workflow: contract generation, PDF deed/certificate output
- Public burial search portal with walk-to-grave directions
- Work order management for grounds maintenance
- Document attachment per record (upload/store deeds, permits, photos)

**Should-have (v1.1)**
- AI-assisted OCR digitisation pipeline for paper ledger import
- Natural language / fuzzy search for genealogical research (phonetic matching)
- Financial module: invoicing, payment plans, perpetual care fund tracking
- Pre-need trust accounting with interest accrual and regulatory reporting
- GEDCOM 7.0 export for genealogy platform interoperability
- REST API with OpenAPI 3.1 specification for funeral home and third-party integration
- Mobile-first offline-capable app for grounds staff work orders

**Nice-to-have (backlog)**
- AI obituary generation from structured case data
- 360-degree virtual tour of enclosed memorial spaces
- E-commerce for floral and memorial product sales
- Multi-site management with consolidated reporting
- Multi-faith/multi-cultural section configuration rules engine
- HL7 VRDR FHIR endpoint for vital records system interoperability
- Sales team performance dashboards with revenue KPIs
