# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Trade Compliance Platform · Created: 2026-05-20

## Philosophy

This model uses relational tables for core structural entities and relationships but stores jurisdiction-specific, regulation-specific, and rapidly evolving fields in JSONB columns. The insight is that trade compliance spans dozens of regulatory regimes (EAR, ITAR, EU Dual-Use, UK Export Control, Wassenaar, OFAC, EU sanctions, UN sanctions) and each regime adds its own fields, categories, and rules. A fully normalized schema would require hundreds of columns — most of them nullable and specific to one jurisdiction — or dozens of junction tables. JSONB columns absorb this variability while keeping the relational backbone for entities that are structurally stable.

This is the approach used by modern SaaS platforms that serve multiple markets (e.g., Stripe's payment method metadata, Shopify's product metafields). It is also how OpenSanctions' FollowtheMoney model works conceptually: a core entity type (Person, LegalEntity, Company) with a flexible property bag where all properties are multi-valued strings. The hybrid approach translates this pattern to PostgreSQL, leveraging GIN indexes on JSONB for fast containment queries while retaining relational foreign keys for structural integrity.

**Best for:** Rapid MVP development, multi-jurisdiction deployments where regulatory fields vary widely by country, and teams that want relational joins for core queries but flexible schema extension without migrations.

**Trade-offs:**
- Pro: Fewer tables than fully normalized (~20-25 vs. 55+) — faster to build and migrate
- Pro: Jurisdiction-specific fields can be added without schema changes
- Pro: JSONB GIN indexes enable fast containment queries on flexible fields
- Pro: Natural fit for ingesting diverse government list formats (each source has unique fields)
- Pro: Easier to prototype and iterate during early product development
- Con: JSONB fields lack database-enforced constraints — validation must happen in application code
- Con: Reporting across JSONB fields requires more complex queries (->>, @>, jsonb_path_query)
- Con: Schema documentation must be maintained manually; JSONB structure is not self-documenting in the DDL
- Con: ORM support for JSONB varies; some frameworks handle it poorly
- Con: Risk of "JSONB creep" — over time, too many fields migrate to JSONB, eroding data integrity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FollowtheMoney (FtM) Ontology | The party entity model mirrors FtM's schema+properties pattern, with party_type as the schema and properties as JSONB |
| OFAC Advanced Sanctions List Standard | Watchlist entries store source-specific fields (programme, vessel details, etc.) in JSONB, accommodating the OFAC XML schema's variable structure |
| ISO 3166-1/2 | Country codes are relational columns (indexed, foreign-key-able); subdivision details go in JSONB |
| EAR/CCL + EU Dual-Use + Wassenaar | Classification details vary by regime; the core code is relational, regime-specific parameters are JSONB |
| WCO Data Model v4 | Core transaction fields (parties, goods, values) are relational; customs-authority-specific fields are JSONB |
| OpenAPI 3.x | API responses merge relational and JSONB fields into flat JSON objects |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    jurisdictions   VARCHAR(3)[] NOT NULL DEFAULT '{}',  -- ISO 3166-1 alpha-3 codes this tenant operates in
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {
    --   "default_screening_sources": ["OFAC_SDN", "BIS_EL", "EU_CONSOLIDATED"],
    --   "risk_thresholds": { "auto_clear_below": 0.3, "auto_escalate_above": 0.9 },
    --   "enabled_modules": ["screening", "classification", "licensing"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    roles           TEXT[] NOT NULL DEFAULT '{}',       -- simple role array: 'admin', 'compliance_officer', 'auditor'
    preferences     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
CREATE INDEX idx_user_roles ON "user" USING GIN(roles);
```

## Party Management

```sql
CREATE TABLE party (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    party_type      VARCHAR(50) NOT NULL,              -- 'individual', 'organisation', 'vessel', 'aircraft'
    primary_name    VARCHAR(500) NOT NULL,
    country_code    VARCHAR(3),                        -- ISO 3166-1 alpha-3, primary jurisdiction
    
    -- Structured core fields (always present, always queried)
    aliases         JSONB NOT NULL DEFAULT '[]',
    -- aliases example: [
    --   { "type": "aka", "name": "ACME Industries Ltd", "script": "Latn" },
    --   { "type": "original_script", "name": "エーシーエム工業株式会社", "script": "Jpan" }
    -- ]
    
    addresses       JSONB NOT NULL DEFAULT '[]',
    -- addresses example: [
    --   { "type": "registered", "line1": "123 Trade St", "city": "Houston", 
    --     "state": "TX", "postal": "77001", "country": "USA", "primary": true }
    -- ]
    
    identifiers     JSONB NOT NULL DEFAULT '[]',
    -- identifiers example: [
    --   { "type": "lei", "value": "529900T8BM49AURSDO55", "issuer": "GLEIF" },
    --   { "type": "duns", "value": "123456789" },
    --   { "type": "vat", "value": "DE123456789", "country": "DEU" }
    -- ]
    
    -- Flexible properties that vary by party type and jurisdiction
    properties      JSONB NOT NULL DEFAULT '{}',
    -- For vessel: { "imo_number": "9074729", "flag": "PAN", "tonnage": 28000, "vessel_type": "Bulk Carrier" }
    -- For individual: { "date_of_birth": "1965-03-15", "place_of_birth": "Tehran", "nationality": "IRN" }
    -- For organisation: { "incorporation_date": "2010-06-01", "registration_number": "HRB12345" }
    
    risk_score      NUMERIC(5,2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_tenant ON party(tenant_id);
CREATE INDEX idx_party_name ON party(tenant_id, primary_name);
CREATE INDEX idx_party_country ON party(country_code);
CREATE INDEX idx_party_aliases ON party USING GIN(aliases jsonb_path_ops);
CREATE INDEX idx_party_identifiers ON party USING GIN(identifiers jsonb_path_ops);
CREATE INDEX idx_party_properties ON party USING GIN(properties jsonb_path_ops);
```

## Watchlist Data

```sql
CREATE TABLE watchlist_source (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    administering_body VARCHAR(255) NOT NULL,
    country_code    VARCHAR(3),
    url             TEXT,
    schema_mapping  JSONB NOT NULL DEFAULT '{}',       -- maps source fields to internal fields
    last_fetched_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE watchlist_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES watchlist_source(id),
    source_uid      VARCHAR(255) NOT NULL,
    entry_type      VARCHAR(50) NOT NULL,
    primary_name    VARCHAR(500) NOT NULL,
    
    -- Core searchable fields (relational)
    programme       VARCHAR(255),
    listing_date    DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    
    -- All aliases, addresses, identifiers in JSONB (variable per source)
    aliases         JSONB NOT NULL DEFAULT '[]',
    addresses       JSONB NOT NULL DEFAULT '[]',
    identifiers     JSONB NOT NULL DEFAULT '[]',
    
    -- Source-specific fields that vary wildly between OFAC, EU, UN, etc.
    source_data     JSONB NOT NULL DEFAULT '{}',
    -- OFAC example: { "programmes": ["SDGT", "IRAN"], "remarks": "...", "vessel_info": {...} }
    -- EU example: { "regulation": "2024/XXX", "legal_basis": "...", "designation_details": "..." }
    -- UN example: { "un_list_type": "Al-Qaida", "reference_number": "QDi.001" }
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, source_uid)
);

CREATE INDEX idx_watchlist_source ON watchlist_entry(source_id);
CREATE INDEX idx_watchlist_name ON watchlist_entry(primary_name);
CREATE INDEX idx_watchlist_active ON watchlist_entry(is_active) WHERE is_active = true;
CREATE INDEX idx_watchlist_aliases ON watchlist_entry USING GIN(aliases jsonb_path_ops);
CREATE INDEX idx_watchlist_identifiers ON watchlist_entry USING GIN(identifiers jsonb_path_ops);
CREATE INDEX idx_watchlist_source_data ON watchlist_entry USING GIN(source_data jsonb_path_ops);
```

## Screening

```sql
CREATE TABLE screening (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    party_id        UUID REFERENCES party(id),
    mode            VARCHAR(20) NOT NULL,              -- 'realtime', 'batch', 'continuous'
    
    -- Query parameters
    query           JSONB NOT NULL,
    -- query example: { "name": "ACME Corp", "country": "IRN", "type": "organisation",
    --                   "identifiers": [{"type": "vat", "value": "..."}] }
    
    sources_checked VARCHAR(50)[] NOT NULL,             -- watchlist_source codes
    
    -- Results
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    hit_count       INTEGER NOT NULL DEFAULT 0,
    hits            JSONB NOT NULL DEFAULT '[]',
    -- hits example: [
    --   { "entry_id": "...", "source": "OFAC_SDN", "match_score": 0.92,
    --     "algorithm": "fuzzy_name", "matched_fields": {"name": 0.92, "country": 1.0},
    --     "entry_name": "ACME Corporation", "programme": "IRAN",
    --     "disposition": null, "reviewed_by": null, "reviewed_at": null }
    -- ]
    
    overall_disposition VARCHAR(50),                    -- 'cleared', 'escalated', 'blocked'
    disposition_note    TEXT,
    
    requested_by    UUID NOT NULL REFERENCES "user"(id),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_screening_tenant ON screening(tenant_id);
CREATE INDEX idx_screening_party ON screening(party_id);
CREATE INDEX idx_screening_status ON screening(status);
CREATE INDEX idx_screening_hits ON screening USING GIN(hits jsonb_path_ops);
```

## Product Classification

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    part_number     VARCHAR(100),
    manufacturer    VARCHAR(255),
    
    -- Current classification (relational for fast joins and filtering)
    hs_code         VARCHAR(10),
    eccn            VARCHAR(5),
    usml_category   VARCHAR(50),
    ear99           BOOLEAN NOT NULL DEFAULT false,
    classification_status VARCHAR(50) NOT NULL DEFAULT 'unclassified',
    
    -- Technical specifications (vary wildly by product type)
    specs           JSONB NOT NULL DEFAULT '{}',
    -- specs example: { "material": "aluminum alloy 7075", "tensile_strength_mpa": 570,
    --                   "operating_temp_c": [-40, 150], "encryption": "AES-256",
    --                   "images": ["s3://bucket/img1.jpg"] }
    
    -- Multi-jurisdiction classification details
    classifications JSONB NOT NULL DEFAULT '{}',
    -- classifications example: {
    --   "EAR": { "eccn": "3A001", "reason": "NS,AT", "determined_by": "...", "date": "2026-01-15",
    --            "rationale": "Contains encryption above 56-bit...", "ai_confidence": 0.87 },
    --   "EU_DUAL_USE": { "code": "3A001", "annex": "I", "determined_by": "...", "date": "2026-01-16" },
    --   "ITAR": { "category": null, "determination": "Not ITAR-controlled", "date": "2026-01-15" },
    --   "UK": { "rating": "3A001", "determined_by": "...", "date": "2026-02-01" }
    -- }
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE classification_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id),
    regime          VARCHAR(50) NOT NULL,              -- 'EAR', 'EU_DUAL_USE', 'ITAR', 'UK', 'HS'
    old_code        VARCHAR(50),
    new_code        VARCHAR(50) NOT NULL,
    rationale       TEXT NOT NULL,
    cited_sources   JSONB,
    ai_assisted     BOOLEAN NOT NULL DEFAULT false,
    ai_confidence   NUMERIC(5,4),
    decided_by      UUID NOT NULL REFERENCES "user"(id),
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_hs ON product(hs_code);
CREATE INDEX idx_product_eccn ON product(eccn);
CREATE INDEX idx_product_classifications ON product USING GIN(classifications jsonb_path_ops);
CREATE INDEX idx_product_specs ON product USING GIN(specs jsonb_path_ops);
CREATE INDEX idx_classification_history_product ON classification_history(product_id, decided_at DESC);
```

## Licence Management

```sql
CREATE TABLE licence (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    licence_number      VARCHAR(100),
    licence_type        VARCHAR(50) NOT NULL,
    authority           VARCHAR(100) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    
    -- Core relational fields
    product_id          UUID REFERENCES product(id),
    destination_country VARCHAR(3) NOT NULL,
    end_user_party_id   UUID REFERENCES party(id),
    
    -- Financial tracking
    approved_value      NUMERIC(18,2),
    consumed_value      NUMERIC(18,2) NOT NULL DEFAULT 0,
    remaining_value     NUMERIC(18,2) GENERATED ALWAYS AS (approved_value - consumed_value) STORED,
    approved_quantity   NUMERIC(18,4),
    consumed_quantity   NUMERIC(18,4) NOT NULL DEFAULT 0,
    currency_code       VARCHAR(3) DEFAULT 'USD',
    
    issued_date         DATE,
    expiry_date         DATE,
    
    -- Authority-specific fields (vary by BIS, DDTC, EU member state, etc.)
    authority_data      JSONB NOT NULL DEFAULT '{}',
    -- BIS example: { "licence_exception": "LVS", "conditions": ["Must notify BIS annually"],
    --               "provisos": "Limited to 50 units per calendar year" }
    -- DDTC example: { "agreement_type": "TAA", "signatories": [...], "articles_covered": [...] }
    -- EU example: { "member_state": "DEU", "bafa_reference": "...", "dual_use_annex": "IV" }
    
    -- Consumption log (denormalised for fast balance queries)
    consumption_log     JSONB NOT NULL DEFAULT '[]',
    -- example: [
    --   { "date": "2026-03-01", "ref": "EXP-2026-0042", "value": 15000.00, "qty": 5, "by": "..." }
    -- ]
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_licence_tenant ON licence(tenant_id);
CREATE INDEX idx_licence_status ON licence(status);
CREATE INDEX idx_licence_expiry ON licence(expiry_date) WHERE status IN ('approved');
CREATE INDEX idx_licence_authority_data ON licence USING GIN(authority_data jsonb_path_ops);
```

## Embargo, Regulatory Updates & Audit

```sql
CREATE TABLE embargo_rule (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    authority           VARCHAR(100) NOT NULL,
    target_country      VARCHAR(3),
    restriction_type    VARCHAR(100) NOT NULL,
    description         TEXT NOT NULL,
    effective_date      DATE NOT NULL,
    expiry_date         DATE,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    details             JSONB NOT NULL DEFAULT '{}',
    -- details example: { "programmes": ["UKRAINE-EO13660", "UKRAINE-EO13661"],
    --                     "sectors": ["financial", "energy"], "exceptions": [...] }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE regulatory_update (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source          VARCHAR(100) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    publication_date DATE NOT NULL,
    effective_date  DATE,
    source_url      TEXT,
    ai_digest       TEXT,
    impact_analysis JSONB NOT NULL DEFAULT '{}',
    -- impact_analysis example: {
    --   "affected_eccns": ["3A001", "5A002"],
    --   "affected_countries": ["CHN", "RUS"],
    --   "severity": "high",
    --   "action_required": "Review all open licences for 3A001 to China"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE trade_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    transaction_ref     VARCHAR(255) NOT NULL,
    transaction_type    VARCHAR(50) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    
    -- Core relational fields
    exporter_party_id   UUID REFERENCES party(id),
    consignee_party_id  UUID REFERENCES party(id),
    destination_country VARCHAR(3) NOT NULL,
    
    -- Transaction details
    total_value         NUMERIC(18,2),
    currency_code       VARCHAR(3) DEFAULT 'USD',
    
    -- Line items (denormalised in JSONB for simpler transactions)
    line_items          JSONB NOT NULL DEFAULT '[]',
    -- line_items example: [
    --   { "product_id": "...", "product_name": "Widget A", "qty": 100, "unit_value": 50.00,
    --     "hs_code": "8471300000", "eccn": "EAR99", "schedule_b": "8471300000" }
    -- ]
    
    -- Compliance results
    screening_id        UUID REFERENCES screening(id),
    licence_id          UUID REFERENCES licence(id),
    compliance_checks   JSONB NOT NULL DEFAULT '{}',
    -- compliance_checks example: {
    --   "screening": { "status": "cleared", "date": "2026-03-01" },
    --   "licence_check": { "required": false, "exception": "LVS" },
    --   "embargo_check": { "status": "passed", "date": "2026-03-01" }
    -- }
    
    -- Filing data (varies by customs authority)
    filing_data         JSONB NOT NULL DEFAULT '{}',
    -- US example: { "aes_itn": "X20260301234567", "filed_at": "2026-03-02T14:30:00Z" }
    -- EU example: { "mrn": "26DE1234567890", "customs_office": "DE004000" }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES "user"(id),
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB NOT NULL,
    -- changes example: { "status": ["pending", "cleared"], "disposition_note": [null, "False positive - different country"] }
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embargo_active ON embargo_rule(is_active) WHERE is_active = true;
CREATE INDEX idx_reg_update_source ON regulatory_update(source, publication_date DESC);
CREATE INDEX idx_transaction_tenant ON trade_transaction(tenant_id);
CREATE INDEX idx_transaction_status ON trade_transaction(status);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

## Example Queries

### Find parties with a specific identifier across JSONB

```sql
-- Find party by LEI stored in JSONB identifiers array
SELECT id, primary_name, identifiers
FROM party
WHERE tenant_id = '...'
  AND identifiers @> '[{"type": "lei", "value": "529900T8BM49AURSDO55"}]';
```

### Multi-jurisdiction classification lookup

```sql
-- Find all products classified under EAR but not yet classified under EU Dual-Use
SELECT id, name, eccn, classifications
FROM product
WHERE tenant_id = '...'
  AND classifications ? 'EAR'
  AND NOT classifications ? 'EU_DUAL_USE';
```

### Find all OFAC-specific watchlist data

```sql
-- Query source-specific fields in watchlist entries
SELECT primary_name, source_data->>'programmes' AS programmes, aliases
FROM watchlist_entry
WHERE source_id = (SELECT id FROM watchlist_source WHERE code = 'OFAC_SDN')
  AND source_data @> '{"programmes": ["SDGT"]}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenant, user (roles as array, no junction table) |
| Party Management | 1 | party (aliases, addresses, identifiers in JSONB) |
| Watchlist & Sanctions | 2 | watchlist_source, watchlist_entry |
| Screening | 1 | screening (hits denormalised in JSONB) |
| Product Classification | 2 | product, classification_history |
| Licence Management | 1 | licence (consumption log in JSONB) |
| Embargo & Regulatory | 2 | embargo_rule, regulatory_update |
| Trade Transactions | 1 | trade_transaction (line items in JSONB) |
| Audit | 1 | audit_log |
| **Total** | **13** | Compact schema; JSONB absorbs most variability |

---

## Key Design Decisions

1. **Core query fields are relational; everything else is JSONB** — Fields that appear in WHERE clauses, JOINs, or GROUP BYs (tenant_id, party_type, country_code, hs_code, eccn, status, dates) are relational columns with proper types and indexes. Fields that vary by jurisdiction, source, or entity subtype go in JSONB.

2. **Party follows the FollowtheMoney pattern** — A single party table with party_type discrimination and a properties JSONB bag mirrors the FtM ontology's schema+properties design. This makes it natural to ingest OpenSanctions data and to represent the wide variety of party types (individuals, companies, vessels, aircraft) without separate tables.

3. **Multi-jurisdiction classifications in one JSONB column** — The product.classifications column stores per-regime classification details keyed by regime code. This avoids a separate classification table per jurisdiction and makes it trivial to add new regimes (e.g., adding "AUSTRALIA" as a key) without schema changes.

4. **Screening hits denormalised into the screening row** — Rather than a separate screening_hit table, hits are stored as a JSONB array within the screening record. This simplifies the common query pattern ("show me screening X with all its hits") to a single-row fetch. The trade-off is that updating a single hit's disposition requires a JSONB array element update.

5. **JSONB validation in application layer** — PostgreSQL CHECK constraints with jsonb_typeof and jsonb_path_exists can enforce basic JSONB structure, but complex validation (e.g., "if party_type is 'vessel' then properties must contain imo_number") is enforced in application code with JSON Schema validation.

6. **GIN indexes on all JSONB columns** — Every JSONB column used for querying has a GIN index with jsonb_path_ops, which supports the @> containment operator for fast lookups. This is critical for identifier searches and source-data queries.

7. **Consumption log as JSONB array** — For licences with modest consumption counts (typically <100 per licence), storing the consumption log as a JSONB array avoids a separate table and provides the full consumption history in a single read. For high-volume licences, this could be extracted to a separate table.
