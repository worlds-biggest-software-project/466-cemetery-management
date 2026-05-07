# Standards & API Reference

> Project: Cemetery Management · Generated: 2026-05-07

---

## Industry Standards & Specifications

### ISO Standards

**ISO 19128:2005 — Geographic information: Web map server interface**
- URL: https://www.iso.org/standard/32546.html
- Equivalent to OGC WMS 1.3. Defines the protocol for serving geo-registered map images over HTTP. Relevant to serving the cemetery's tile-based map layer to browser and mobile clients from a standards-compliant tile server.

**ISO 19142:2010 — Geographic information: Web Feature Service (WFS)**
- URL: https://www.iso.org/standard/42136.html
- Equivalent to OGC WFS 2.0. Specifies transactional and query access to geographic features (vector geometries + attributes) independent of underlying data store. Relevant for exposing cemetery plot boundaries, section polygons, and associated attributes as queryable geospatial features.

**ISO 19115 — Geographic information: Metadata**
- URL: https://www.iso.org/standard/53798.html
- Standard schema for describing geospatial datasets. Relevant for documenting the cemetery's GIS data layers, coordinate reference system, accuracy, and provenance — particularly for data published to municipal GIS portals.

**ISO 15836 — Dublin Core Metadata Element Set**
- URL: https://www.iso.org/standard/71339.html
- Lightweight metadata standard for describing digital resources. Relevant for tagging scanned burial documents, historic ledger images, and memorial pages with discoverable metadata (title, date, creator, subject).

**ISO/IEC 27001 — Information security management systems**
- URL: https://www.iso.org/standard/82875.html
- Framework for information security management. Relevant because burial records are permanent legal documents with long retention obligations; data breach or loss would be irreversible. Guides security policies for the application.

**ISO/IEC 27018 — Protection of personally identifiable information (PII) in public clouds**
- URL: https://www.iso.org/standard/76559.html
- Code of practice for PII protection in cloud environments. Relevant because cemetery systems store sensitive personal data (deceased identities, family contacts, financial records) in a cloud SaaS architecture.

---

### W3C & IETF Standards

**RFC 7946 — The GeoJSON Format**
- URL: https://datatracker.ietf.org/doc/html/rfc7946
- Defines the GeoJSON encoding for geographic data structures (points, polygons, feature collections). The de facto format for exchanging cemetery plot boundaries, section polygons, and gravesite coordinates between the application, mapping libraries, and external GIS tools.

**RFC 9110 — HTTP Semantics**
- URL: https://datatracker.ietf.org/doc/html/rfc9110
- Defines HTTP method semantics, status codes, and caching headers. Foundational for designing the REST API used by cemetery management clients (web, mobile, third-party integrations).

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header and link relations for hypermedia APIs. Relevant for pagination and HATEOAS patterns in the cemetery management REST API.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorization framework for delegated access. Required for the public genealogy portal (families authenticating to manage memorial pages) and for third-party integrations (funeral home software connecting to the cemetery API).

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Defines the compact token format used in OAuth 2.0 and OpenID Connect access tokens. Relevant for stateless authentication across the web app, mobile app, and REST API.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0 providing authentication (not just authorisation). Relevant for single sign-on for cemetery staff, and for social login on the public family portal.

**W3C WCAG 2.2 — Web Content Accessibility Guidelines**
- URL: https://www.w3.org/TR/WCAG22/
- Accessibility guidelines for web content. Cemetery public portals serve elderly family members and genealogical researchers with diverse accessibility needs; WCAG 2.2 Level AA compliance is important.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry-standard machine-readable format for describing REST APIs (YAML/JSON). The cemetery management REST API should be documented using OAS 3.1 to enable auto-generated client SDKs, interactive documentation, and validation.

**GEDCOM 7.0 — FamilySearch Genealogical Data Communication**
- URL: https://gedcom.io/specifications/FamilySearchGEDCOMv7.html
- The de facto standard for exchanging genealogical data between applications. A cemetery management system should support GEDCOM 7.0 export so burial records can be imported into FamilySearch, Ancestry, MyHeritage, and other genealogy platforms. GEDCOM encodes birth, death, relationship, and place data in a structured format.

**OGC GeoPackage (GPKG)**
- URL: https://www.geopackage.org/
- An open, standards-based, platform-independent container for geospatial data based on SQLite. Relevant for exporting cemetery map data in a portable format compatible with desktop GIS tools (QGIS, ArcGIS) for offline use or data migration.

**HL7 FHIR US VRDR — Vital Records Death Reporting Implementation Guide**
- URL: https://hl7.org/fhir/us/vrdr/
- FHIR-based specification for exchanging death certificate data between jurisdictional vital records offices, funeral directors, and the CDC/NCHS. Cemetery systems that record interments for official reporting purposes may need to consume or produce VRDR-compliant records for integration with state vital records systems.

**vCard (RFC 6350)**
- URL: https://datatracker.ietf.org/doc/html/rfc6350
- Standard format for electronic business cards. Relevant for exporting contact records (plot owners, next of kin) in a portable format compatible with contact management software.

---

### Security & Authentication Standards

**OAuth 2.0 with PKCE (RFC 7636)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Proof Key for Code Exchange extension to OAuth 2.0. Required for secure authorisation in public (SPA and mobile) clients where a client secret cannot be safely stored. The recommended flow for the cemetery's public family portal and mobile groundskeeper app.

**PCI DSS 4.0.1 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/standards/
- Global security standard for any system that stores, processes, or transmits cardholder data. As of March 2025, v4.0.1 requirements are mandatory. Cemetery management systems accepting plot sale payments online must comply with PCI DSS (or delegate card handling entirely to a compliant payment processor).

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- OWASP's enumeration of the top ten API security risks. Particularly relevant for the cemetery's REST API and public portal, where sensitive personal and financial data is exposed.

**NIST SP 800-63B — Digital Identity Guidelines: Authentication**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- NIST guidance on authentication assurance levels, password policies, and MFA requirements. Relevant for staff authentication to the cemetery administration interface where legal burial records are managed.

**GDPR (EU) 2016/679 / UK GDPR — General Data Protection Regulation**
- URL: https://gdpr.eu/
- European and UK data protection law. Relevant for any cemetery system operating in or serving residents of the EU or UK: deceased personal data, family contact data, and financial records must be handled with appropriate retention, consent, and subject access provisions. Some member states treat deceased persons' data under GDPR for limited periods after death.

---

### Geospatial & Mapping Standards

**OGC Web Map Tile Service (WMTS)**
- URL: https://www.ogc.org/standards/wmts/
- Standard for serving pre-rendered map tiles. Relevant for efficient delivery of the cemetery base map layer at high zoom levels, enabling smooth pan-and-zoom performance in the browser and mobile clients.

**OGC Web Feature Service (WFS) 2.0**
- URL: https://www.ogc.org/standards/wfs/
- Standard for querying and transacting geographic features. Relevant for exposing cemetery plot polygon data to GIS desktop tools (QGIS, ArcGIS) and to data consumers such as municipal GIS portals.

**EPSG:4326 / WGS 84 Coordinate Reference System**
- URL: https://epsg.io/4326
- The global GPS coordinate reference system (latitude/longitude). The default CRS for GeoJSON and the basis for plot GPS coordinates. All cemetery geospatial data should be stored and exchanged in WGS 84 unless a local projected CRS is required for precision legal surveys.

---

## Similar Products — Developer Documentation & APIs

### Esri ArcGIS Cemetery Management Solution

- **Description:** A local-government GIS solution that enables authoritative cemetery inventory, field verification, and public memorial search on top of the full ArcGIS platform. The highest-fidelity geospatial cemetery data model available.
- **API Documentation:** https://doc.arcgis.com/en/arcgis-solutions/latest/reference/introduction-to-cemetery-management.htm
- **SDKs/Libraries:** ArcGIS Maps SDK for JavaScript (https://developers.arcgis.com/javascript/latest/), ArcGIS Maps SDK for Swift/Kotlin, ArcGIS REST API
- **Developer Guide:** https://developers.arcgis.com/
- **Standards:** OGC WMS (ISO 19128), OGC WFS (ISO 19142), GeoJSON, REST/JSON
- **Authentication:** OAuth 2.0 via ArcGIS Identity (ArcGIS Online / Enterprise Portal)

---

### Mapbox Maps & Tiling Service

- **Description:** A location platform providing vector tile maps, custom data hosting, geocoding, navigation, and a JavaScript/mobile SDK. Used by several cemetery mapping tools as the tile server and mapping library backend. Mapbox GL JS renders high-performance interactive maps in-browser.
- **API Documentation:** https://docs.mapbox.com/api/
- **SDKs/Libraries:** Mapbox GL JS (web), Mapbox Maps SDK for iOS/Android/React Native
- **Developer Guide:** https://docs.mapbox.com/
- **Standards:** Vector Tile Specification (Mapbox), GeoJSON (RFC 7946), WMTS, REST/JSON
- **Authentication:** API access tokens (per-token scoping); no OAuth flow for server-side use

---

### Google Maps Platform

- **Description:** Google's mapping APIs providing satellite/street imagery base maps, geocoding, directions, and the JavaScript Maps API with custom data layer support. Commonly used by cemetery software (e.g., InStone) as the base map layer.
- **API Documentation:** https://developers.google.com/maps/documentation
- **SDKs/Libraries:** Maps JavaScript API, Maps SDK for Android/iOS, Maps Datasets API
- **Developer Guide:** https://developers.google.com/maps/get-started
- **Standards:** REST/JSON, GeoJSON for Datasets API
- **Authentication:** API Key (per-key; IP/referrer restrictions recommended); OAuth 2.0 for user-specific services

---

### Everspot Public API

- **Description:** All-in-one cemetery management SaaS with a documented API enabling third-party developers to integrate with Everspot data. One of the few cemetery-specific vendors with a publicly accessible API.
- **API Documentation:** Via Everspot support and training channels (https://everspot.io/); no public developer portal found as of research date
- **SDKs/Libraries:** Not publicly documented; likely REST/JSON
- **Developer Guide:** Available via Everspot onboarding training (documentation, webinars, live support)
- **Standards:** REST/JSON (assumed; not publicly specified)
- **Authentication:** Not publicly documented

---

### FamilySearch GEDCOM API & Developer Platform

- **Description:** FamilySearch is the world's largest genealogy platform, operated by The Church of Jesus Christ of Latter-day Saints. Its platform API enables reading and writing family trees, sources, and memories. GEDCOM export/import is the bridge for connecting cemetery burial records to genealogical trees.
- **API Documentation:** https://www.familysearch.org/developers/
- **SDKs/Libraries:** FamilySearch JavaScript SDK (https://github.com/FamilySearch/fs-js-lite), community SDKs for Python and Java
- **Developer Guide:** https://www.familysearch.org/developers/docs/guides
- **Standards:** GEDCOM 7.0, REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0 (FamilySearch Identity)

---

### Stripe Payments API

- **Description:** The dominant payment processing API for SaaS applications. Cemetery management systems use Stripe (or equivalent) for processing plot sale payments, payment plan instalments, and pre-need contract deposits online.
- **API Documentation:** https://docs.stripe.com/api
- **SDKs/Libraries:** stripe-node, stripe-python, stripe-php, stripe-java, stripe-go, stripe-dotnet, stripe-ios, stripe-android
- **Developer Guide:** https://docs.stripe.com/
- **Standards:** REST/JSON, webhooks (HTTP POST callbacks), OpenAPI spec published
- **Authentication:** API Key (publishable + secret keys); webhook signature verification via HMAC-SHA256

---

### QuickBooks Online API (Intuit Developer)

- **Description:** The accounting API for QuickBooks Online, used by Everspot and others for bidirectional sync of cemetery financial data (invoices, payments, accounts receivable) with the operator's accounting software.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/api/accounting/most-commonly-used/account
- **SDKs/Libraries:** intuit-oauth (Node.js), QuickBooks PHP/Python/Java/Ruby SDKs
- **Developer Guide:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **Standards:** REST/JSON, OAuth 2.0, OpenID Connect
- **Authentication:** OAuth 2.0 (60-minute token expiry, refresh token flow); tiered API pricing from $0–$4,500/month (as of 2025)

---

### Sunrise CMS (Open Source Reference Implementation)

- **Description:** A fully free, open-source web-based cemetery records management system maintained by the City of Sault Ste. Marie, Ontario. Provides the most accessible open-source reference implementation for burial record data models and UI patterns. Useful as a base or reference for an AI-native open-source alternative.
- **API Documentation:** https://github.com/cityssm/sunrise-cms (source code; no REST API in current form)
- **SDKs/Libraries:** Node.js / TypeScript stack (examination of repository)
- **Developer Guide:** GitHub README at https://github.com/cityssm/sunrise-cms
- **Standards:** None currently; data model is bespoke SQL
- **Authentication:** Session-based (no OAuth); local user accounts

---

### HL7 FHIR VRDR — Vital Records Death Reporting

- **Description:** The HL7 FHIR Implementation Guide for US death reporting, enabling bidirectional exchange of mortality data between jurisdictional vital records offices and the CDC/NCHS. Cemetery systems accepting interment records that feed into official death registration may need to produce or consume VRDR bundles.
- **API Documentation:** https://hl7.org/fhir/us/vrdr/
- **SDKs/Libraries:** HAPI FHIR (Java), firely-net-sdk (.NET), fhir.js (JavaScript)
- **Developer Guide:** https://build.fhir.org/ig/HL7/vrdr/
- **Standards:** HL7 FHIR R4, REST/JSON, SMART on FHIR (OAuth 2.0)
- **Authentication:** SMART on FHIR (OAuth 2.0 + OpenID Connect profile for healthcare)

---

## Notes

- **No category-wide API standard exists.** Unlike funeral home case management (where NFDA has facilitated some data-sharing conventions), cemetery management has no industry-wide data interchange standard. An open-source system publishing an OpenAPI 3.1 specification with GEDCOM 7.0 export could become a de facto standard for the sector.
- **GEDCOM 7.0 adoption is still maturing.** GEDCOM 5.5.1 remains the format most genealogy platforms consume; a cemetery system should support both 5.5.1 and 7.0 export to maximise compatibility.
- **HL7 VRDR is US-specific.** Other jurisdictions (UK, Australia, EU member states) have their own death registration data flows; international deployments would require jurisdiction-specific adapters.
- **Geospatial precision varies by use case.** Consumer mapping (Google Maps, Mapbox) is sufficient for walk-to-grave navigation; legal plot boundary surveys require a projected local CRS (e.g., EPSG:27700 for Great Britain) and survey-grade GPS capture rather than consumer-grade location services.
- **PCI DSS scope reduction.** Cemetery management systems should use a hosted payment field solution (Stripe.js, Stripe Elements) to avoid PCI scope for cardholder data — the application never handles raw card numbers.
