# Trade Compliance Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source trade compliance platform for denied party screening, export classification, and licence management.

The Trade Compliance Platform helps exporters, importers, and compliance teams screen counterparties, classify goods, and manage export licences across multiple jurisdictions. It targets the gap left by legacy enterprise suites that price out small and mid-size exporters while remaining UI-dated and slow to adopt AI. The project aims to bring modern, explainable AI workflows to compliance work that today depends on scarce specialist expertise.

---

## Why Trade Compliance Platform?

- Incumbents (Descartes Visual Compliance, ONESOURCE Global Trade, e2open, SAP GTS) all use custom enterprise pricing — entry tiers start around USD 5,000–15,000/year and full deployments reach seven figures, pricing out SMB exporters.
- Legacy platforms have dated UIs and complex configuration; ONESOURCE's REST API is criticised as immature (limited to US/Canada) and SOAP is being deprecated without feature parity.
- AI-native capabilities are still proprietary and partial: Descartes AI Assist, KYG Trade's ChatECN/ChatHTS, and e2open's transliteration each address one slice but no open, auditable solution exists.
- EU CBAM (definitive from January 2026), UFLPA forced-labour diligence, and multi-tier sanctions exposure are underserved across the incumbent landscape.
- Regulatory data (OFAC, BIS, EU Consolidated List) is government-published and freely available; an open-source aggregator combined with explainable AI classification is feasible without IP conflicts.

---

## Key Features

### Screening and Sanctions

- Denied/restricted party screening against OFAC SDN, BIS Entity List, BIS Denied Persons List, EU Consolidated List, and UN sanctions lists
- Real-time and batch screening modes
- OFAC 50% rule / sanctioned ownership screening for indirect beneficiaries
- Continuous re-screening with alerts when existing counterparties appear on new lists
- AI-assisted false-positive triage to reduce manual review queues

### Classification

- AI-assisted ECCN classification with explainable rationale and cited regulatory sources
- HS/HTS code classification from product descriptions, BOMs, specs, and images
- USML/ITAR classification for defence articles
- Configurable decision trees and progressive disclosure for routine vs. complex goods
- Inline surfacing of WCO Explanatory Notes and CROSS rulings

### Licence Management

- Export licence determination given product, destination, and end-user
- Licence application tracking, consumption monitoring, and expiry alerts
- End-use assessment and end-user certificate management
- Embargo and sanctions checks on transactions

### Regulatory Intelligence

- Continuous monitoring of CCL, USML, Wassenaar, BIS, OFAC, and EU dual-use updates
- Natural-language regulatory change digests flagging affected products and counterparties
- Audit trail and compliance reporting for every classification and screening decision

### Integration and APIs

- REST API with OpenAPI 3.x specification
- SAP ECC and S/4HANA connector
- Oracle ERP and Salesforce integrations
- Bulk data downloads and self-hostable screening API

---

## AI-Native Advantage

AI assists where compliance work is most expensive: ECCN/HS classification (which today depends on scarce specialists), false-positive triage in denied party screening, and reading regulatory updates. The platform pairs natural-language licence determination with multi-tier counterparty graph analysis to surface indirect sanctions exposure. Every AI decision is explainable and audit-logged, with cited regulatory sources, so compliance officers retain accountability.

---

## Tech Stack & Deployment

The platform is designed for self-hosted and cloud deployment, with a JSON/REST API-first architecture and OpenAPI 3.x specifications. It aligns with established standards: EAR/ECCN, ITAR/USML, OFAC sanctions lists, EU Dual-Use Regulation (2021/821), Harmonized System / Schedule B, AES electronic filing, and the WCO SAFE Framework. Open-source sanctions data (OpenSanctions, FollowtheMoney model) is integrated where compatible, and certified ERP connectors are provided for SAP and Oracle.

---

## Market Context

The trade compliance software market is valued at approximately USD 3.09 billion in 2025 and is growing through 2029, driven by expanding sanctions regimes, new tariff structures, and supply chain scrutiny (Globe Newswire, 2026). Consolidation is accelerating — WiseTech Global acquired e2open for USD 2.1 billion in August 2025. Primary buyers are export and trade compliance officers at manufacturers of controlled goods, customs managers, legal/regulatory affairs teams at defence and aerospace firms, and supply chain directors managing multi-jurisdiction flows.

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
