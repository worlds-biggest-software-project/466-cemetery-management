# 466 – Cemetery Management

*Research date: 2026-05-02*

---

## 1. Problem Statement

Cemetery operators – public authorities, religious organisations, and private memorial parks – maintain records spanning decades or centuries, manage physical inventory of plots and niches, process sales and interment scheduling, and respond to genealogical enquiries from families. Paper ledgers and disconnected spreadsheets make it difficult to locate historic burial records, verify plot ownership, track maintenance tasks, or offer families a self-service way to find and memorialise loved ones. A purpose-built system is needed to digitise and centralise these functions.

---

## 2. Market Landscape

Cemetery management software is a specialist niche with a small but clearly defined vendor set. PlotBox is the dominant international platform used by cemeteries, crematories, and funeral homes, offering integrated mapping, records management, and a public genealogy portal. Other solutions include CIMS, Chronicle, Memorial Mapper CMS, CityView (for municipalities), and Everspot (digitisation and mapping focus). GetApp and Capterra list over 22 rated tools as of 2026. The market is driven by digitisation of legacy paper records and growing public demand for online memorials and genealogy access.

Key vendors:
- PlotBox – end-to-end cemetery, crematory, and funeral home platform [plotbox.com]
- Everspot – digital mapping and record digitisation [everspot.io]
- CityView – municipal cemetery management [municipalsoftware.com]
- Chronicle – cloud-based mapping, records, and public access [capterra.com]
- Memorial Mapper CMS – searchable burial database and plot tracking [memorialmapper.com]

---

## 3. Core Features

1. **Interactive cemetery map** – georeferenced digital map of the cemetery with plot, section, and niche inventory visualised, showing sold, available, and reserved spaces in real time.
2. **Plot and niche sales** – sales workflow from availability search through contract generation, payment processing, and deed or certificate issuance.
3. **Burial and interment records** – comprehensive record per interment including deceased details, date, officiant, funeral home, and linked documents (death certificate, permit).
4. **Interment scheduling** – calendar-based scheduling of burials, cremation interments, disinterments, and memorialisation services with conflict detection.
5. **Maintenance management** – work-order creation for groundskeeping, monument repairs, section improvements, and seasonal tasks, with assignment and completion tracking.
6. **Document management** – scanned deeds, burial permits, historic ledger pages, and photos attached to records and searchable by name, section, or date.
7. **Genealogy and public search portal** – publicly accessible portal allowing families and researchers to search burial records, view plot locations on the map, and access memorial pages.
8. **Financial management** – accounts receivable for plot sales, perpetual care fund contributions, pre-need contract tracking, and payment plans.
9. **Reporting and compliance** – statutory burial registers, financial reports, plot inventory summaries, and export for government reporting requirements.
10. **Multi-site management** – single platform managing multiple cemeteries under one organisation, with shared staff access and consolidated reporting.

---

## 4. Technical Considerations

- **Geospatial mapping** – cemetery maps require vector plot boundaries overlaid on aerial imagery; GIS integration (GeoJSON, Shapefile, or proprietary formats) and tile-server performance at high zoom levels are important.
- **Legacy record digitisation** – existing paper ledgers may contain decades of records; bulk import tools, OCR assistance, and data validation workflows are needed for migration.
- **Record permanence and audit trail** – burial records are legal documents; the system must provide immutable audit trails and meet archival retention requirements (often 100+ years).
- **Public portal scalability** – genealogy portals can attract high public traffic around significant dates; the public-facing tier should be independently scalable from the administrative back-end.
- **Pre-need contract management** – pre-sold plots and cremation niches require financial tracking of funds held in trust, with interest accrual and regulatory compliance per jurisdiction.
- **Multi-faith and cultural requirements** – section designations, orientation rules, and interment customs vary by religion and culture; the system must support configurable rules per section.
- **Offline access for grounds staff** – maintenance crew working on-site may need offline access to plot maps and work orders via mobile apps.

---

## 5. Citations

1. PlotBox – "Cemetery Management Software" – https://plotbox.com/cemetery-management-software
2. PlotBox – "Cemetery Records Management Software" – https://plotbox.com/cemetery-records-management-software
3. PlotBox EverAfter – "Find loved ones' resting place" – https://plotbox.com/public-genealogy-search-portal
4. Everspot – "Cemetery Software, Digital Mapping & Digitization" – https://everspot.io/
5. CityView – "Cemetery Management Software for Municipalities" – https://www.municipalsoftware.com/cemetery-management/
6. GetApp – "Best Cemetery Software 2026" – https://www.getapp.com/industries-software/cemetery/
7. Memorial Mapper – "Modern Cemetery Management Software: A Guide to Digitization, Mapping, and Memorials" – https://memorialmapper.com/blog/cemetery-digitization-maping-and-memorials/
8. Capterra – "Best Cemetery Software 2026" – https://www.capterra.com/cemetery-software/
