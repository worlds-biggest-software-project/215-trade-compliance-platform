# Trade Compliance Platform — Feature & Functionality Survey

> Candidate #215 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Descartes Visual Compliance | SaaS | Commercial / Custom pricing | https://www.visualcompliance.com/ |
| KYG Trade | SaaS | Commercial / Custom pricing | https://www.kygtrade.com/ |
| ONESOURCE Global Trade (Thomson Reuters) | SaaS | Commercial / Custom pricing | https://www.thomsonreuters.com/en/products/onesource.html |
| eCustoms | SaaS | Commercial / Custom pricing | https://ecustoms.com/ |
| MIC-CUST Export Control Management | SaaS / On-premise | Commercial / Custom pricing | https://www.mic-cust.com/ |
| AEB Export Controls | SaaS / On-premise | Commercial / Custom pricing | https://www.aeb.com/en/export-controls/ |
| e2open (Amber Road) Global Trade | SaaS | Commercial / Custom pricing | https://www.e2open.com/global-trade/ |
| SAP Global Trade Services (GTS) | ERP module | SAP licensing | https://www.sap.com/products/financial-management/global-trade-management.html |
| CargoWise (WiseTech) | SaaS | Commercial / Subscription (Value Packs) | https://www.cargowise.com/ |
| OpenSanctions | Open-source / API | Open-source (various); commercial API tier | https://www.opensanctions.org/ |

---

## Feature Analysis by Solution

### Descartes Visual Compliance

**Core features**
- Denied party screening (DPS) against 750+ global watchlists including OFAC SDN, BIS DPL, and EU Consolidated List
- Real-time and batch screening modes
- OFAC 50% rule / sanctioned ownership screening for indirect beneficiaries
- ECCN and USML export classification
- Export licence management and consumption tracking
- AES/EEI electronic filing for US exports
- Daily dynamic rescreening with alerts and notifications
- Centralized audit trail and compliance reporting

**Differentiating features**
- AI Assist (2025): AI-powered false-positive reduction — automatically clears low-risk hits below configurable risk thresholds, dramatically reducing manual review queue
- Configurable risk levels per geography, department, or regulatory regime
- G2 2026 Best Denied Party Screening Software designation
- Broad list coverage (750+) is one of the widest in the market

**UX patterns**
- Web-based portal with dashboard view of screening queues and outstanding reviews
- Results presented as match/no-match with confidence scoring
- Audit trail surfaced inline with each screening result
- Configurable alert thresholds reduce noise for compliance officers

**Integration points**
- SAP ERP (ECC and S/4HANA) via certified add-on on SAP Business Technology Platform
- Oracle ERP integration
- Salesforce CRM integration with dynamic screening
- REST APIs available via Descartes API portal (api.descartes.com/apis)
- Sine Workflows, Sign In Enterprise, and other workplace management tools

**Known gaps**
- Complexity of initial setup and configuration for smaller organisations
- API maturity described as adequate but not developer-first in design
- Limited native support for EU-specific compliance workflows (historically US-export-focused)

**Licence / IP notes**
- Proprietary SaaS; no published open-source components. The AI Assist feature is a proprietary capability. List database content is licensed from government and commercial sources.

---

### KYG Trade

**Core features**
- AI-powered ECCN classification using configurable decision trees (ChatECN™)
- AI-powered HS/HTS classification using product descriptions, BOMs, specs, and images (ChatHTS™)
- Restricted party screening (RPS) against global lists
- Free Trade Agreement (FTA) qualification and duty optimisation
- First-sale duty savings calculation
- Forced labour / UFLPA compliance module
- Export licence determination and management
- Full audit trails for all classification decisions

**Differentiating features**
- Kay AI assistant (passed the October 2025 US Customs Brokers License Exam — CBLE — a first for AI in trade compliance)
- 70%+ reduction in classification time reported
- Continuous regulatory tracking: CCL, USML, Wassenaar Agreement monitored in real time
- Microservices architecture with JSON API-first design
- ISO 27001 certified cloud

**UX patterns**
- AI-assisted workflows with rationale and explainability for each classification decision
- Progressive disclosure: simpler classification view for routine goods, detailed mode for complex ones
- Collaborative workflows supporting multi-stakeholder sign-off
- WCO Explanatory Notes and CROSS rulings surfaced inline during classification

**Integration points**
- JSON API library integrating with ERP (SAP, Oracle), PLM, TMS, and SCM systems
- Pre-built connectors for major GTM, ERP, and eCommerce platforms
- One-time integration fee model (connectors are optional)
- Partner ecosystem of certified consultants for enterprise deployments

**Known gaps**
- Newer entrant (third year in Gartner's 2025 Market Guide) — reference base growing but smaller than incumbents
- Pricing not publicly disclosed; likely prohibitive for true SMBs
- FTA optimisation capability less mature than legacy GTM suites

**Licence / IP notes**
- Proprietary SaaS. ChatECN™ and ChatHTS™ are proprietary trademarks. No open-source components published.

---

### ONESOURCE Global Trade (Thomson Reuters)

**Core features**
- Import management: HS classification, duty calculation, drawback, and FTA qualification
- Export management: ECCN classification, export licence management, denied party screening
- Foreign Trade Zone (FTZ) management
- Customs documentation and filing
- Regulatory content across 220+ countries and 500+ FTA rules of origin
- SAP S/4HANA and Oracle ERP integration
- Global Knowledge database updated by worldwide network of trade experts

**Differentiating features**
- Deepest regulatory content library of any trade compliance platform (220+ countries, updated daily)
- Proven end-to-end GTM from quote to delivery across import, export, and FTZ
- Strong brand trust with large enterprises; over 15 years in the market

**UX patterns**
- Complex enterprise UI optimised for trained compliance officers
- Workflow-driven screens for import/export transactions
- Dashboard for licence consumption, pending filings, and open DPS hits
- Minimal progressive disclosure — designed for expert users

**Integration points**
- SAP ERP (ECC and S/4HANA) certified integration
- Oracle E-Business Suite integration
- REST and SOAP API (REST API available; SOAP being deprecated; REST currently limited to US/Canada)
- AWS Marketplace available

**Known gaps**
- REST API maturity criticised — limited to US/Canada for REST; SOAP deprecated but REST not yet feature-parity
- High implementation cost and long deployment timelines
- UI considered dated and complex by non-specialist users
- Expensive for mid-market; not viable for SMBs

**Licence / IP notes**
- Proprietary SaaS by Thomson Reuters. Regulatory content is licensed. No open-source components.

---

### eCustoms

**Core features**
- Denied party, restricted, blocked, and unverified party screening
- Sanctioned ownership screening (OFAC 50% rule)
- ECCN and USML (ITAR) classification
- Export licence management
- Export documentation management (AES/EEI electronic filing)
- Integration with SAP, Oracle, legacy, and eCommerce systems
- Deemed export and hand-carried export analysis

**Differentiating features**
- Deep ITAR/USML expertise alongside EAR/ECCN — dual-regulation coverage in one tool
- On-demand or ERP-integrated deployment options (more flexible than pure SaaS plays)
- Part of the Descartes ecosystem (acquired) — benefits from Descartes' data infrastructure

**UX patterns**
- Traditional enterprise compliance portal UI
- Screen-by-screen workflow for classification and licence determination
- Results-driven dashboard for compliance officers

**Integration points**
- SAP integration
- Oracle integration
- Legacy system and eCommerce platform connectors
- API available (Descartes-backed infrastructure)

**Known gaps**
- UI dated; minimal AI-native features as of last public information
- Limited public developer API documentation
- Smaller community and documentation base than larger GTM suites

**Licence / IP notes**
- Proprietary SaaS (part of Descartes Systems Group).

---

### MIC-CUST Export Control Management

**Core features**
- Export control classification (ECCN, dual-use, military list) across multiple jurisdictions
- Embargo and sanctions checks
- Denied party screening (MIC DPS)
- End-use assessment and end-user certificate management
- Export licence management and transaction workflow
- AI-assisted customs tariff and export control classification (2025 feature)
- Automated trade document processing

**Differentiating features**
- Strong multi-jurisdiction support — particularly deep EU dual-use regulation coverage
- Modular design: MIC ECM integrates with MIC DPS, MIC CCS ECC, and customs modules
- Direct SAP integration with automatic transaction blocking for non-compliant shipments

**UX patterns**
- Traditional enterprise workflow UI; 2025 rebrand improved visual design
- Compliance chain overview screen providing end-to-end visibility from embargo to licensing
- ERP-embedded experience: checks run inside SAP without leaving the host system

**Integration points**
- SAP ECC and S/4HANA (primary integration; transactions blocked directly in ERP)
- REST and SOAP API for non-SAP integrations
- Modular integration with other MIC products

**Known gaps**
- UI historically described as dated compared to newer entrants
- Less brand recognition outside Europe
- Limited US-specific content depth compared to US-focused vendors

**Licence / IP notes**
- Proprietary SaaS / on-premise hybrid. No open-source components.

---

### AEB Export Controls

**Core features**
- Automated export control checks: embargo, sanctions, and licence requirement determination
- End-use verification for critical goods
- Licence management (flexible definition by goods attributes, origin, destination, end-use)
- Financial sanctions value-restriction compliance
- Manual ad-hoc checks, transaction-level checks, and continuous automated checks
- Data service providing ongoing regulatory change monitoring

**Differentiating features**
- REST API (v2) available with full developer documentation at trade-compliance.docs.developers.aeb.com
- "Try it out" interactive API documentation for testing
- Salesforce AppExchange listing for CRM-embedded screening
- SAP ECC and S/4HANA certified add-on

**UX patterns**
- Enterprise compliance portal with workflow-driven transaction screening
- Risk Assessment module for forward-looking exposure analysis
- Inline ERP experience via SAP add-on

**Integration points**
- SAP ECC and S/4HANA via certified add-on
- REST API (v2) and SOAP API for other ERP systems
- Salesforce AppExchange integration
- AEB Customs Management (separate module) for end-to-end customs + compliance

**Known gaps**
- Primarily European market focus; US regulatory depth less prominent
- Less AI-native marketing than newer competitors (AI features announced in 2025 but less detail publicly available)
- Smaller ecosystem and partner network than Descartes or Thomson Reuters

**Licence / IP notes**
- Proprietary SaaS / on-premise. Developer documentation is publicly accessible.

---

### e2open (Amber Road) Global Trade

**Core features**
- Global trade compliance screening with AI-enhanced due diligence
- Automated product classification replacing manual processes
- Unstructured document processing — extracts and structures transactional data from trade documents
- FTA management and duty optimisation
- Global regulatory content database (170 countries)
- Supply chain partner network for collaboration (suppliers, carriers, brokers, forwarders)
- Tariff management and classification

**Differentiating features**
- AI transliteration of non-Western names for enhanced DPS coverage (2025)
- Up to 90% reduction in manual effort for classification reported
- Millions of dollars in duty savings cited by clients
- Deep supply chain network integration (connected to broad carrier/broker/forwarder ecosystem)
- WiseTech Global acquisition (2025) brings CargoWise logistics integration

**UX patterns**
- Enterprise GTM platform with dashboard-driven compliance and logistics management
- Workflow automation for classification, screening, and documentation
- Collaborative portal for supply chain partner data exchange

**Integration points**
- Deep ERP integrations (SAP, Oracle, and others)
- Carrier Marketplace API for logistics partners
- REST API (limited public documentation)

**Known gaps**
- Corporate turbulence: e2open acquired by WiseTech in 2025 — platform roadmap uncertainty during transition
- REST API documentation not publicly detailed
- Product complexity from absorbing multiple acquisitions (Amber Road, GT Nexus heritage)

**Licence / IP notes**
- Proprietary SaaS (now part of WiseTech Global). Amber Road Global Knowledge database is a licensed content asset.

---

### SAP Global Trade Services (GTS)

**Core features**
- Embargo and sanctioned-party screening executed within SAP in real time
- Export licence determination and consumption tracking
- ECCN, dual-use, and military list classification across multiple jurisdictions
- HTS/HS classification across 190+ countries
- FTZ management
- Customs management and electronic filing
- AI/ML classification predictions with confidence scores (S/4HANA edition)

**Differentiating features**
- Seamless native integration within SAP ERP — no middleware required
- Real-time compliance blocking directly in SAP transaction flows
- AI-powered classification in the S/4HANA edition with learning from historical data
- Single system of record for all compliance, trade, and ERP data

**UX patterns**
- Embedded in SAP UI; compliance users work within familiar SAP Fiori or SAPGUI screens
- No separate login or portal — compliance checks are inline
- Reporting via SAP Analytics Cloud or standard SAP reports

**Integration points**
- Native SAP ECC and S/4HANA integration (primary use case)
- Certified third-party GTM content providers (CustomsInfo, Thomson Reuters) for tariff data
- SAP BTP (Business Technology Platform) for extensions

**Known gaps**
- Requires SAP ecosystem — not viable as standalone for non-SAP organisations
- SAP GTS v11 mainstream support ended December 2025; migration to S/4HANA edition required
- High total cost of ownership; requires specialist SAP GTS consultants
- Innovation pace slower than dedicated trade compliance vendors

**Licence / IP notes**
- Proprietary SAP module. Requires SAP licensing agreements.

---

### CargoWise ComplianceWise (WiseTech Global)

**Core features**
- AI Classification Assistant for HS and commodity code classification
- AI-Assisted Document Ingestion: extracts and validates trade document data
- Trade and regulatory compliance checks (ComplianceWise)
- Customs notifications and declarations (US, EU, Asia-Pacific)
- ACAS (Air Cargo Advance Screening) compliance
- T2L declarations and regional customs compliance

**Differentiating features**
- Integrated within CargoWise logistics platform — compliance and logistics in one system
- AI Classification Assistant included in CargoWise Value Packs (Dec 2025)
- WiseTech acquisition of e2open creates opportunity for deepened trade compliance capability
- Strong Asia-Pacific and Australia/NZ coverage alongside US and EU

**UX patterns**
- Logistics-first platform with compliance embedded into freight workflows
- Single-screen visibility for shipment and compliance status
- Value Pack subscription model simplifies access to AI features

**Integration points**
- Tightly integrated with CargoWise freight and logistics modules
- APIs for customs authority connections (direct filing)
- Limited public third-party ERP integration documentation

**Known gaps**
- ComplianceWise less mature for export control (ECCN/DPS) than pure-play compliance vendors
- AI features only available from December 2025 in Value Pack model
- Less suited for non-logistics compliance workflows (e.g., deemed exports, dual-use screening outside freight)

**Licence / IP notes**
- Proprietary SaaS (WiseTech Global). No open-source components.

---

### OpenSanctions

**Core features**
- Aggregated open-source database of sanctions lists, watchlists, and politically exposed persons (PEPs)
- Hundreds of source lists aggregated and deduplicated
- FollowtheMoney (FtM) entity-relationship data model
- Yente entity-matching API for self-hosted screening applications
- Batch screening and single-entity search via API
- Data pipeline tooling: zavod (pipeline), rigour (normalisation), nomenklatura (data integration)

**Differentiating features**
- Only open-source sanctions dataset with commercial-grade coverage
- Investigators, journalists, and compliance teams all served from one dataset
- Self-hostable (Yente API) — no dependency on commercial data vendors
- Transparent data lineage and provenance for every entity
- Community-maintained with thousands of handcrafted data patches

**UX patterns**
- Developer-first: API and data download as primary interfaces
- Web search portal for ad-hoc lookups
- No enterprise compliance workflow UI (by design)

**Integration points**
- REST API (opensanctions.org/api/)
- Bulk data downloads (CSV, JSON, nested JSON)
- Yente self-hosted API (Docker / Kubernetes deployable)
- Graph database export compatible with Neo4j, etc.

**Known gaps**
- No export classification (ECCN/HS) capability — sanctions/PEP data only
- No licence management, documentation, or customs filing features
- Commercial API tier required for high-volume production use
- No built-in compliance workflow, audit trail UI, or case management

**Licence / IP notes**
- Data: Creative Commons Attribution (CC BY) for non-commercial use; commercial licence for production use. Software components: MIT / Apache 2.0 licensed. Fully open-source technology stack.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Denied/restricted party screening against major government watchlists (OFAC, BIS, EU, UN)
- ECCN and HS/HTS code classification
- Export licence management (determination, application tracking, consumption monitoring)
- Audit trail for all compliance decisions
- ERP integration (SAP and/or Oracle as minimum)
- Batch and real-time screening modes
- Role-based access control for compliance officers, managers, and auditors

### Differentiating Features
- AI-powered false-positive reduction in DPS (Descartes AI Assist)
- Explainable AI classification with cited regulatory sources (KYG Trade, e2open)
- Continuous real-time regulatory monitoring with change alerts (KYG Trade)
- Native logistics-compliance integration (CargoWise)
- Self-hostable open-source sanctions data (OpenSanctions)
- Multi-tier supply chain counterparty screening (e2open)
- Developer-first REST API with interactive documentation (AEB)

### Underserved Areas / Opportunities
- **SMB-accessible pricing and UX**: All enterprise platforms price out SMBs; no modern SaaS with transparent, affordable tiers exists
- **EU CBAM compliance**: A new category (definitive from January 2026) with minimal dedicated software support; most vendors have not yet productised CBAM workflows
- **Forced labour / UFLPA compliance**: Growing regulatory requirement; few platforms have mature, dedicated modules (KYG Trade is an early mover)
- **Supply chain counterparty graph analysis**: Multi-tier indirect sanctions exposure is largely manual; graph-based automated discovery is nascent
- **Natural-language regulatory change digests**: Teams still rely on email newsletters and manual review; no platform delivers AI-summarised BIS/OFAC/EU regulatory update digests inline
- **Open-source classification data**: ECCN and HS classification logic is locked in proprietary systems; an open, auditable classification knowledge base does not exist
- **Carbon/ESG compliance integration**: CBAM requires embedded-emissions data; no mainstream trade compliance platform integrates ESG and trade compliance workflows

### AI-Augmentation Candidates
- ECCN and HS classification from product descriptions, specs, and images (partially addressed by KYG and e2open; large opportunity for open-source model)
- False-positive triage in denied party screening (partially addressed by Descartes AI Assist)
- Automated regulatory change impact analysis (BIS, OFAC, EU dual-use rule updates)
- Natural-language licence determination ("Given product X, destination Y, and end-user Z, is a licence required?")
- Multi-tier supply chain graph traversal to surface indirect sanctions exposure
- Unstructured trade document ingestion and data extraction (partially addressed by e2open)

---

## Legal & IP Summary

No patent concerns were identified during research. The core compliance workflows — screening, classification, licence management — are industry-standard business processes and are not known to be subject to software patents. Vendor-specific trademarks include KYG Trade's ChatECN™ and ChatHTS™. The FollowtheMoney data model used by OpenSanctions is open-source (MIT). Regulatory data (OFAC lists, BIS Entity List, EU Consolidated List) is government-published and freely available; aggregating and repackaging it in a commercial product is standard practice. OpenSanctions' database is CC BY for non-commercial use; commercial use requires a paid licence. No copyright, patent, or licence compatibility concerns were identified that would impede building an open-source AI-native trade compliance platform.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Denied party / restricted party screening against OFAC SDN, BIS Entity List, BIS Denied Persons List, EU Consolidated List, and UN sanctions list
- ECCN classification assistant with AI-assisted reasoning and explainable output
- HS/HTS code classification for import duty calculation
- Export licence determination (does this product + destination + end-user require a licence?)
- Audit trail and compliance reporting
- REST API with OpenAPI 3.x specification

**Should-have (v1.1)**
- Continuous re-screening (automated alerts when existing counterparties appear on new lists)
- Regulatory change digest (AI-summarised BIS/OFAC/EU update briefings)
- Licence management (track issued licences, consumption against value/quantity limits, expiry alerts)
- SAP and/or Oracle ERP integration via certified connector
- FTA qualification for common US, EU, and UK trade agreements

**Nice-to-have (backlog)**
- EU CBAM compliance module (embedded emissions reporting, certificate management)
- Forced labour / UFLPA supply chain diligence module
- Multi-tier supply chain counterparty graph analysis
- AES/EEI electronic filing for US exports
- FTZ management
- Native CargoWise / logistics platform integration
