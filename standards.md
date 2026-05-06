# Standards & API Reference

> Project: Trade Compliance Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### Regulatory Frameworks

**Export Administration Regulations (EAR) — 15 CFR Parts 730–774**
- URL: https://www.ecfr.gov/current/title-15/subtitle-B/chapter-VII/subchapter-C
- Administered by the US Bureau of Industry and Security (BIS). Governs the export and re-export of dual-use commercial goods via the Commerce Control List (CCL) and Export Control Classification Numbers (ECCNs). Central to any trade compliance platform serving US exporters.

**International Traffic in Arms Regulations (ITAR) — 22 CFR Parts 120–130**
- URL: https://www.ecfr.gov/current/title-22/chapter-I/subchapter-M
- Administered by the US Directorate of Defense Trade Controls (DDTC). Controls export of defence articles and services on the United States Munitions List (USML). Any platform handling aerospace, defence, or dual-use goods must support USML classification.

**EU Dual-Use Regulation (2021/821)**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32021R0821
- European Union regulation governing the export, brokering, transit, and transfer of dual-use items. Substantially equivalent to EAR but with distinct classification lists and licensing authorities per EU member state. Critical for any platform serving European exporters.

**OFAC Sanctions Programmes**
- URL: https://ofac.treasury.gov/sanctions-programs-and-country-information
- The US Office of Foreign Assets Control administers comprehensive sanctions programmes. The Specially Designated Nationals (SDN) List and Consolidated List are the primary denied-party data sources. OFAC's Advanced Sanctions List Standard provides XML data in a machine-readable schema.

**UK Export Control Order 2008 / Strategic Export Controls**
- URL: https://www.gov.uk/guidance/export-controls-military-goods-software-and-technology
- UK post-Brexit export control regime. Governs controlled military and dual-use goods; relevant for platforms serving UK exporters.

**EU Carbon Border Adjustment Mechanism (CBAM) — Regulation 2023/956**
- URL: https://taxation-customs.ec.europa.eu/carbon-border-adjustment-mechanism_en
- Definitive CBAM regime in effect from 1 January 2026. Importers of steel, aluminium, cement, fertilisers, electricity, and hydrogen from non-EU countries must obtain Authorised CBAM Declarant status and manage certificate purchases. Emerging compliance requirement with minimal existing software support.

**US Uyghur Forced Labor Prevention Act (UFLPA)**
- URL: https://uflpa.cbp.gov/
- Effective June 2022. Creates a rebuttable presumption that goods manufactured wholly or partly in Xinjiang are produced with forced labour and are barred from US import. Requires supply chain due diligence documentation and traceability.

---

### Classification Standards

**Harmonized System (HS) — World Customs Organization**
- URL: https://www.wcoomd.org/en/topics/nomenclature/instrument-and-tools/hs_convention.aspx
- The international product nomenclature administered by the WCO covering 98% of world trade. Six-digit HS codes form the foundation of all customs duty calculation, statistical reporting, and rules of origin determination. Updated every five years; HS 2022 is the current edition.

**WCO Trade Tools**
- URL: https://www.wcotradetools.org/en
- Official WCO platform for HS classification lookup, explanatory notes, and classification opinions. Essential reference for building classification assistance features.

**Commerce Control List (CCL) / ECCN System**
- URL: https://www.bis.doc.gov/index.php/regulations/commerce-control-list-overview
- The BIS CCL contains all dual-use goods subject to EAR, classified using five-character alphanumeric ECCNs. Platform classification engines must be able to navigate CCL commodity groups and control parameters.

**United States Munitions List (USML)**
- URL: https://www.ecfr.gov/current/title-22/chapter-I/subchapter-M/part-121
- The ITAR list of defence articles and services. Platforms supporting ITAR-regulated industries must maintain USML category mappings.

---

### Data Interchange Standards

**WCO Data Model (WCO DM) — Version 4**
- URL: https://www.wcoomd.org/en/topics/facilitation/instrument-and-tools/tools/data-model.aspx
- The internationally standardised data model for cross-border trade information exchange. Maps to UN/TDED and leverages UN/CEFACT and ISO standards. Used as the foundation for single-window systems globally. Version 4 adds support for emerging technologies and enhanced data elements.

**UN/EDIFACT — CUSCAR (Customs Cargo Report Message)**
- URL: https://www.unece.org/trade/untdid/welcome.html
- UN/EDIFACT D01C CUSCAR message defines the electronic customs cargo report format. Used for EDI-based customs filing and cargo reporting. Legacy but still required for many government customs system integrations.

**UN/CEFACT Cross-Industry Invoice / Trade Document Standards**
- URL: https://unece.org/trade/uncefact
- UN/CEFACT produces XML and JSON schemas for trade documents including the Cross-Industry Invoice (CII), used in structured electronic invoicing. Relevant for electronic trade documentation features.

**ISO 20022 — Financial Messaging**
- URL: https://www.iso20022.org/
- International standard for financial messaging adopted by SWIFT for cross-border payments. The November 2025 SWIFT coexistence period end means all CBPR+ traffic is on ISO 20022. Relevant for trade finance integration (letters of credit, payment reconciliation).

**OFAC Advanced Sanctions List Standard (XML)**
- URL: https://ofac.treasury.gov/sdn-list-data-formats-data-schemas/frequently-asked-questions-on-advanced-sanctions-list-standard
- OFAC's machine-readable XML schema for the SDN and Consolidated Lists. Supports non-Western name part decomposition and government-issued identifier labelling. The primary data format for ingesting US sanctions lists programmatically.

---

### Security, Authentication & Compliance Standards

**ISO 27001:2022 — Information Security Management**
- URL: https://www.iso.org/standard/82875.html
- International standard for information security management systems. KYG Trade holds ISO 27001 certification. Required for enterprise procurement of trade compliance SaaS.

**ISO 28000:2022 — Supply Chain Security Management**
- URL: https://www.iso.org/standard/79612.html
- Specifies requirements for security management systems across the supply chain. Aligned with the WCO SAFE Framework for Authorised Economic Operator (AEO) recognition.

**WCO SAFE Framework of Standards**
- URL: https://www.wcoomd.org/en/topics/facilitation/instrument-and-tools/frameworks-of-standards/safe_package.aspx
- World Customs Organization framework establishing standards for trusted-trader programmes (AEO/C-TPAT). Platforms supporting AEO compliance workflows should map to this framework.

**OAuth 2.0 / OpenID Connect**
- RFC 6749 (OAuth 2.0): https://datatracker.ietf.org/doc/html/rfc6749
- RFC 8414 (OpenID Connect Discovery): https://datatracker.ietf.org/doc/html/rfc8414
- Standard authentication and authorisation protocols for SaaS API access. Required for enterprise SSO integration and secure API key management.

**OpenAPI Specification 3.x**
- URL: https://spec.openapis.org/oas/latest.html
- Standard for describing REST APIs. An AI-native trade compliance platform should publish an OpenAPI 3.x specification to enable developer adoption and ecosystem integration.

**NIST Cybersecurity Framework 2.0**
- URL: https://www.nist.gov/cyberframework
- Relevant for platforms serving US government contractors and ITAR-regulated organisations who must demonstrate cybersecurity maturity (aligned with CMMC requirements).

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- An open protocol for connecting AI models to external tools and data sources. An AI-native trade compliance platform should consider publishing MCP server endpoints for: (a) ECCN/HS classification queries, (b) denied party screening checks, and (c) regulatory change lookups. This would allow AI assistants and agents to invoke trade compliance checks programmatically.

---

## Similar Products — Developer Documentation & APIs

### Descartes Visual Compliance

- **Description:** Leading denied party screening and export compliance SaaS. AI Assist (2025) reduces false positives automatically.
- **API Documentation:** https://api.descartes.com/apis
- **SDKs/Libraries:** No public SDKs; REST API with API key authentication
- **Developer Guide:** https://www.descartes.com/solutions/global-trade-intelligence/denied-party-screening (integration overview)
- **Standards:** REST/JSON; SAP-certified connector via SAP BTP
- **Authentication:** API key

---

### AEB Export Controls

- **Description:** European export control software with strong multi-jurisdiction support and a publicly documented REST API (v2).
- **API Documentation:** https://trade-compliance.docs.developers.aeb.com/docs/getting-started
- **SDKs/Libraries:** No public SDKs; REST (v2) and SOAP (legacy) APIs
- **Developer Guide:** https://trade-compliance.docs.developers.aeb.com/docs/the-first-call-to-export-controls
- **Standards:** REST/JSON (v2); OpenAPI-documented endpoint with interactive "Try it out"; SOAP (legacy)
- **Authentication:** Username/password for test environment; enterprise credentials for production

---

### KYG Trade

- **Description:** AI-native export classification and trade compliance SaaS. ISO 27001 certified; microservices architecture; JSON API-first design.
- **API Documentation:** https://www.kygtrade.com/platform (API described but detailed public docs require account)
- **SDKs/Libraries:** JSON API library with ERP/PLM/TMS connectors
- **Developer Guide:** https://www.kygtrade.com/platform/tools
- **Standards:** REST/JSON; microservices architecture; ISO 27001 certified
- **Authentication:** Enterprise credentials; API key

---

### OpenSanctions

- **Description:** Open-source aggregated sanctions, watchlist, and PEP database. The reference implementation for open sanctions data with a commercially licensable API.
- **API Documentation:** https://www.opensanctions.org/api/ and https://www.opensanctions.org/docs/api/
- **SDKs/Libraries:** Yente (Python, self-hosted, MIT licence): https://github.com/opensanctions/yente; zavod (data pipeline toolkit): https://github.com/opensanctions/zavod
- **Developer Guide:** https://www.opensanctions.org/docs/
- **Standards:** REST/JSON; FollowtheMoney (FtM) entity-relationship data model; OpenAPI compatible
- **Authentication:** API key for hosted API; no auth required for self-hosted Yente

---

### OFAC Sanctions List Service (US Treasury)

- **Description:** Official US government source for OFAC sanctions data. Provides the SDN List and Consolidated List in multiple formats including the Advanced Sanctions List Standard XML.
- **API Documentation:** https://ofac.treasury.gov/sanctions-list-service
- **SDKs/Libraries:** Community-maintained Go library: https://github.com/cardonator/ofac; third-party wrappers available via OFAC API (https://www.ofac-api.com/)
- **Developer Guide:** https://ofac.treasury.gov/sdn-list-data-formats-data-schemas/frequently-asked-questions-on-advanced-sanctions-list-standard
- **Standards:** XML (Advanced Standard); CSV and fixed-width legacy formats; no REST API from OFAC directly (third-party commercial wrappers exist)
- **Authentication:** Public download — no authentication required for data files

---

### classification.tools

- **Description:** AI-powered customs classification API providing HS codes, ECCN, DG (dangerous goods) codes, and CAS numbers. Uses fine-tuned BERT/RoBERTa transformer models trained on regulatory classification datasets.
- **API Documentation:** https://classification.tools/
- **SDKs/Libraries:** REST API (language-agnostic)
- **Developer Guide:** https://classification.tools/
- **Standards:** REST/JSON; OpenAPI; claims 99%+ accuracy on standardised classification benchmarks
- **Authentication:** API key

---

### e2open Global Trade

- **Description:** Full-suite global trade management platform (Amber Road heritage), covering compliance screening, classification, FTA management, and supply chain collaboration.
- **API Documentation:** https://www.e2open.com/global-trade/ (detailed API docs require account)
- **SDKs/Libraries:** Carrier Marketplace API: https://marketplace.e2open.com/product/api-implementation/
- **Developer Guide:** https://apitracker.io/a/e2open (third-party tracker; official docs behind login)
- **Standards:** REST/JSON
- **Authentication:** Enterprise credentials

---

### ONESOURCE Global Trade (Thomson Reuters)

- **Description:** Established global trade compliance suite covering import, export, FTZ, and DPS with regulatory content across 220+ countries.
- **API Documentation:** Available to licensed customers; REST API noted as limited to US/Canada in current form
- **SDKs/Libraries:** No public SDKs
- **Developer Guide:** https://www.thomsonreuters.com/en/products/onesource.html (commercial enquiry required for API access)
- **Standards:** REST (partial coverage) and SOAP (legacy); SAP-certified connector
- **Authentication:** Enterprise credentials; API key

---

### WCO Trade Tools

- **Description:** Official WCO web platform for HS classification lookup, explanatory notes, and tariff guidance. Primary authoritative source for HS code data.
- **API Documentation:** https://www.wcotradetools.org/en (web interface; no public REST API as of 2026)
- **SDKs/Libraries:** No official SDKs
- **Developer Guide:** https://www.wcoomd.org/en/topics/facilitation/instrument-and-tools/tools/data-model.aspx (WCO Data Model documentation)
- **Standards:** WCO Data Model v4; UN/EDIFACT; XML
- **Authentication:** WCO member access for some features; web search is public

---

## Notes

**Emerging areas with limited standards coverage:**

- **EU CBAM Registry API**: The EU Commission's CBAM Authorisation Management Module (AMM) and Registry are in early implementation as of 2026. Programmatic API access to the CBAM Registry is not yet publicly documented, creating an opportunity for a compliance platform to build a standards-aligned integration ahead of vendor market entry.

- **Forced labour / supply chain traceability**: No unified standard exists for UFLPA compliance data exchange. UN/CEFACT is developing traceability data exchange standards, but adoption is nascent. This is an open standards gap for a platform building multi-tier supply chain diligence features.

- **AI model explainability for classification**: There is no established standard for how AI-assisted ECCN/HS classification decisions should be documented or audited for regulatory defensibility. ISO 42001 (AI Management Systems, 2023) provides a governance framework but does not address trade-specific explainability requirements.

- **MCP for trade compliance**: No published MCP server specifications exist for trade compliance data as of May 2026. An open-source platform publishing an MCP server for classification and screening queries would be a first-mover in enabling AI agent integrations.
