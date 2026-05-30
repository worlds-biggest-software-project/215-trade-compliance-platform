# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Trade Compliance Platform · Created: 2026-05-20

## Philosophy

This model follows classical third-normal-form relational design, giving every domain concept its own table with explicit foreign keys enforcing referential integrity. Every watchlist entry, screening result, classification decision, licence, and regulatory rule is a distinct row in a purpose-built table. The schema mirrors how compliance officers think: parties are screened against lists, products are classified under codes, and licences are consumed by transactions.

This is the approach taken by OFAC's own data products, which assign UIDs to every primary entry and link them to aliases, addresses, and identifiers via relational joins. It is also how SAP GTS and ONESOURCE store compliance data inside ERP systems. The normalized design maximises query flexibility and data integrity at the cost of more tables and more joins.

**Best for:** Teams with strong relational database expertise deploying a system where data integrity, complex cross-entity reporting, and regulatory auditability are paramount.

**Trade-offs:**
- Pro: Strong referential integrity prevents orphaned records and data inconsistencies
- Pro: Standard SQL tooling, wide talent pool, mature ORM support
- Pro: Easy to add new entity types without disrupting existing tables
- Pro: Natural fit for regulatory reporting queries that join across many dimensions
- Con: High table count (~55-65 tables) increases migration complexity
- Con: Multi-jurisdiction variations require many nullable columns or additional junction tables
- Con: Schema changes require migrations; less flexible for rapidly evolving regulatory fields
- Con: Complex joins for common queries (e.g., "all screenings for party X with matches and dispositions")

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OFAC Advanced Sanctions List Standard (XML) | Party, alias, address, and identifier tables mirror the OFAC UID-linked relational structure |
| FollowtheMoney (FtM) Ontology | Entity schema types (Person, LegalEntity, Company) inform the party type hierarchy |
| Harmonized System (WCO) | HS code table uses the 6-digit international structure with national subheading extensions |
| EAR/CCL ECCN System | ECCN table encodes the 5-character alphanumeric classification with category, group, and reason |
| ISO 3166-1/2 | Country and subdivision codes used for jurisdictions, destinations, and embargo rules |
| OpenAPI 3.x | REST resource structure maps 1:1 to table entities |
| WCO Data Model v4 | Trade transaction and declaration tables align with WCO cross-border data elements |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- e.g., 'compliance_officer', 'auditor', 'admin'
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES "user"(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES "user"(id),
    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
CREATE INDEX idx_user_role_user ON user_role(user_id);
```

## Party Management

```sql
CREATE TYPE party_type AS ENUM ('individual', 'organisation', 'vessel', 'aircraft');

CREATE TABLE party (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    party_type      party_type NOT NULL,
    primary_name    VARCHAR(500) NOT NULL,
    date_of_birth   DATE,
    place_of_birth  VARCHAR(255),
    nationality     VARCHAR(3),           -- ISO 3166-1 alpha-3
    tax_id          VARCHAR(100),
    lei             VARCHAR(20),          -- ISO 17442 Legal Entity Identifier
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE party_alias (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id        UUID NOT NULL REFERENCES party(id) ON DELETE CASCADE,
    alias_type      VARCHAR(50) NOT NULL,  -- 'aka', 'fka', 'dba', 'original_script'
    alias_name      VARCHAR(500) NOT NULL,
    script          VARCHAR(10),           -- ISO 15924 script code
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE party_address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id        UUID NOT NULL REFERENCES party(id) ON DELETE CASCADE,
    address_type    VARCHAR(50) NOT NULL,  -- 'registered', 'operating', 'shipping'
    line1           VARCHAR(255),
    line2           VARCHAR(255),
    city            VARCHAR(255),
    state_province  VARCHAR(255),
    postal_code     VARCHAR(50),
    country_code    VARCHAR(3) NOT NULL,   -- ISO 3166-1 alpha-3
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE party_identifier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id        UUID NOT NULL REFERENCES party(id) ON DELETE CASCADE,
    identifier_type VARCHAR(100) NOT NULL,  -- 'passport', 'imo_number', 'duns', 'vat', 'mmsi'
    identifier_value VARCHAR(255) NOT NULL,
    issuing_country VARCHAR(3),             -- ISO 3166-1 alpha-3
    issue_date      DATE,
    expiry_date     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_tenant ON party(tenant_id);
CREATE INDEX idx_party_name ON party(tenant_id, primary_name);
CREATE INDEX idx_party_alias_name ON party_alias(alias_name);
CREATE INDEX idx_party_identifier ON party_identifier(identifier_type, identifier_value);
```

## Watchlist & Sanctions Data

```sql
CREATE TABLE watchlist_source (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,   -- 'OFAC_SDN', 'BIS_EL', 'EU_CONSOLIDATED', 'UN_SC'
    name            VARCHAR(255) NOT NULL,
    administering_body VARCHAR(255) NOT NULL,
    country_code    VARCHAR(3),                     -- ISO 3166-1 alpha-3
    url             TEXT,
    update_frequency VARCHAR(50),                   -- 'daily', 'weekly', 'as_needed'
    last_fetched_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE watchlist_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES watchlist_source(id),
    source_uid      VARCHAR(255) NOT NULL,          -- OFAC UID or equivalent
    entry_type      party_type NOT NULL,
    primary_name    VARCHAR(500) NOT NULL,
    programme       VARCHAR(255),                   -- e.g., 'SDGT', 'IRAN', 'UKRAINE-EO13662'
    listing_date    DATE,
    delisting_date  DATE,
    remarks         TEXT,
    raw_data        JSONB,                          -- original source record for reference
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, source_uid)
);

CREATE TABLE watchlist_entry_alias (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id        UUID NOT NULL REFERENCES watchlist_entry(id) ON DELETE CASCADE,
    alias_type      VARCHAR(50) NOT NULL,
    alias_name      VARCHAR(500) NOT NULL,
    script          VARCHAR(10),
    quality         VARCHAR(50)                     -- 'strong', 'weak', 'low' per OFAC schema
);

CREATE TABLE watchlist_entry_address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id        UUID NOT NULL REFERENCES watchlist_entry(id) ON DELETE CASCADE,
    line1           VARCHAR(255),
    city            VARCHAR(255),
    country_code    VARCHAR(3) NOT NULL,
    full_address    TEXT
);

CREATE TABLE watchlist_entry_identifier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id        UUID NOT NULL REFERENCES watchlist_entry(id) ON DELETE CASCADE,
    identifier_type VARCHAR(100) NOT NULL,
    identifier_value VARCHAR(255) NOT NULL,
    issuing_country VARCHAR(3)
);

CREATE INDEX idx_watchlist_entry_source ON watchlist_entry(source_id);
CREATE INDEX idx_watchlist_entry_name ON watchlist_entry(primary_name);
CREATE INDEX idx_watchlist_alias_name ON watchlist_entry_alias(alias_name);
CREATE INDEX idx_watchlist_entry_active ON watchlist_entry(is_active) WHERE is_active = true;
```

## Screening

```sql
CREATE TYPE screening_mode AS ENUM ('realtime', 'batch', 'continuous');
CREATE TYPE screening_status AS ENUM ('pending', 'clear', 'potential_match', 'confirmed_match', 'escalated', 'false_positive');

CREATE TABLE screening_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    party_id        UUID REFERENCES party(id),
    mode            screening_mode NOT NULL,
    query_name      VARCHAR(500) NOT NULL,
    query_country   VARCHAR(3),
    query_type      party_type,
    sources_checked UUID[] NOT NULL,              -- array of watchlist_source IDs
    overall_status  screening_status NOT NULL DEFAULT 'pending',
    requested_by    UUID NOT NULL REFERENCES "user"(id),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE screening_hit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES screening_request(id) ON DELETE CASCADE,
    entry_id        UUID NOT NULL REFERENCES watchlist_entry(id),
    match_score     NUMERIC(5,4) NOT NULL,        -- 0.0000 to 1.0000
    match_algorithm VARCHAR(50) NOT NULL,          -- 'fuzzy_name', 'exact_id', 'phonetic'
    matched_fields  JSONB NOT NULL,                -- which fields matched and how
    status          screening_status NOT NULL DEFAULT 'potential_match',
    reviewed_by     UUID REFERENCES "user"(id),
    reviewed_at     TIMESTAMPTZ,
    disposition_note TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_screening_tenant ON screening_request(tenant_id);
CREATE INDEX idx_screening_party ON screening_request(party_id);
CREATE INDEX idx_screening_status ON screening_request(overall_status);
CREATE INDEX idx_screening_hit_request ON screening_hit(request_id);
CREATE INDEX idx_screening_hit_status ON screening_hit(status) WHERE status = 'potential_match';
```

## Product Classification

```sql
CREATE TABLE hs_code (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(10) NOT NULL UNIQUE,   -- up to 10 digits (6 international + national)
    chapter         VARCHAR(2) NOT NULL,            -- first 2 digits
    heading         VARCHAR(4) NOT NULL,            -- first 4 digits
    subheading      VARCHAR(6) NOT NULL,            -- first 6 digits (WCO international)
    description     TEXT NOT NULL,
    unit_of_measure VARCHAR(50),
    edition         VARCHAR(10) NOT NULL DEFAULT 'HS2022',
    parent_code     VARCHAR(10),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE eccn (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(5) NOT NULL UNIQUE,     -- e.g., '3A001'
    category        CHAR(1) NOT NULL,               -- 0-9
    product_group   CHAR(1) NOT NULL,               -- A-E
    control_reason  VARCHAR(100),                    -- 'NS', 'MT', 'AT', 'CB', etc.
    description     TEXT NOT NULL,
    ccl_heading     TEXT,
    licence_requirements TEXT,
    related_usml    VARCHAR(50),                    -- cross-reference to USML category if applicable
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    part_number     VARCHAR(100),
    manufacturer    VARCHAR(255),
    hs_code_id      UUID REFERENCES hs_code(id),
    eccn_id         UUID REFERENCES eccn(id),
    usml_category   VARCHAR(50),
    ear99           BOOLEAN NOT NULL DEFAULT false,  -- true if determined to be EAR99
    classification_status VARCHAR(50) NOT NULL DEFAULT 'unclassified',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE classification_decision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    classification_type VARCHAR(20) NOT NULL,       -- 'HS', 'ECCN', 'USML'
    assigned_code   VARCHAR(50) NOT NULL,
    confidence_score NUMERIC(5,4),
    rationale       TEXT NOT NULL,                   -- human or AI-generated reasoning
    cited_sources   JSONB,                          -- regulatory references, CROSS rulings, WCO notes
    ai_assisted     BOOLEAN NOT NULL DEFAULT false,
    decided_by      UUID NOT NULL REFERENCES "user"(id),
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_by   UUID REFERENCES classification_decision(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_hs ON product(hs_code_id);
CREATE INDEX idx_product_eccn ON product(eccn_id);
CREATE INDEX idx_classification_product ON classification_decision(product_id);
```

## Licence Management

```sql
CREATE TYPE licence_status AS ENUM ('draft', 'submitted', 'approved', 'denied', 'expired', 'revoked', 'consumed');

CREATE TABLE licence (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    licence_number      VARCHAR(100),
    licence_type        VARCHAR(50) NOT NULL,       -- 'individual', 'bulk', 'general', 'ITAR_TAA'
    authority           VARCHAR(100) NOT NULL,       -- 'BIS', 'DDTC', 'EU_member_state'
    status              licence_status NOT NULL DEFAULT 'draft',
    product_id          UUID REFERENCES product(id),
    destination_country VARCHAR(3) NOT NULL,          -- ISO 3166-1 alpha-3
    end_user_party_id   UUID REFERENCES party(id),
    end_use_description TEXT,
    approved_value      NUMERIC(18,2),
    approved_quantity   NUMERIC(18,4),
    consumed_value      NUMERIC(18,2) NOT NULL DEFAULT 0,
    consumed_quantity   NUMERIC(18,4) NOT NULL DEFAULT 0,
    currency_code       VARCHAR(3) DEFAULT 'USD',    -- ISO 4217
    issued_date         DATE,
    expiry_date         DATE,
    conditions          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE licence_consumption (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    licence_id      UUID NOT NULL REFERENCES licence(id),
    transaction_ref VARCHAR(255),
    value_consumed  NUMERIC(18,2),
    quantity_consumed NUMERIC(18,4),
    consumed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_by     UUID NOT NULL REFERENCES "user"(id)
);

CREATE TABLE licence_determination (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    product_id          UUID NOT NULL REFERENCES product(id),
    destination_country VARCHAR(3) NOT NULL,
    end_user_party_id   UUID REFERENCES party(id),
    end_use_type        VARCHAR(100),
    licence_required    BOOLEAN NOT NULL,
    licence_exception   VARCHAR(50),                 -- e.g., 'LVS', 'TMP', 'TSR'
    determination_basis TEXT NOT NULL,                -- regulatory reasoning
    ai_assisted         BOOLEAN NOT NULL DEFAULT false,
    determined_by       UUID NOT NULL REFERENCES "user"(id),
    determined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_licence_tenant ON licence(tenant_id);
CREATE INDEX idx_licence_status ON licence(status);
CREATE INDEX idx_licence_expiry ON licence(expiry_date) WHERE status = 'approved';
CREATE INDEX idx_licence_consumption_licence ON licence_consumption(licence_id);
```

## Embargo & Regulatory Rules

```sql
CREATE TABLE embargo_rule (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_code           VARCHAR(100) NOT NULL UNIQUE,
    authority           VARCHAR(100) NOT NULL,        -- 'OFAC', 'EU', 'UN_SC'
    target_type         VARCHAR(50) NOT NULL,          -- 'country', 'region', 'programme'
    target_value        VARCHAR(255) NOT NULL,          -- country code or programme name
    restriction_type    VARCHAR(100) NOT NULL,          -- 'comprehensive', 'sectoral', 'arms', 'financial'
    description         TEXT NOT NULL,
    effective_date      DATE NOT NULL,
    expiry_date         DATE,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    source_url          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE regulatory_update (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source          VARCHAR(100) NOT NULL,            -- 'BIS', 'OFAC', 'EU_OJ'
    title           VARCHAR(500) NOT NULL,
    summary         TEXT,
    ai_digest       TEXT,                             -- AI-generated plain-language summary
    publication_date DATE NOT NULL,
    effective_date  DATE,
    source_url      TEXT,
    affected_eccns  VARCHAR(5)[],
    affected_countries VARCHAR(3)[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embargo_active ON embargo_rule(is_active) WHERE is_active = true;
CREATE INDEX idx_embargo_target ON embargo_rule(target_type, target_value);
CREATE INDEX idx_reg_update_source ON regulatory_update(source, publication_date DESC);
```

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES "user"(id),
    action          VARCHAR(100) NOT NULL,           -- 'screening.created', 'classification.decided', 'licence.approved'
    entity_type     VARCHAR(100) NOT NULL,           -- 'screening_request', 'classification_decision', 'licence'
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at DESC);
```

## Trade Transactions

```sql
CREATE TABLE trade_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    transaction_ref     VARCHAR(255) NOT NULL,
    transaction_type    VARCHAR(50) NOT NULL,          -- 'export', 'import', 'reexport', 'deemed_export'
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    exporter_party_id   UUID REFERENCES party(id),
    consignee_party_id  UUID REFERENCES party(id),
    end_user_party_id   UUID REFERENCES party(id),
    origin_country      VARCHAR(3),                    -- ISO 3166-1 alpha-3
    destination_country VARCHAR(3) NOT NULL,
    total_value         NUMERIC(18,2),
    currency_code       VARCHAR(3) DEFAULT 'USD',
    incoterm            VARCHAR(3),                    -- e.g., 'FOB', 'CIF', 'DDP'
    screening_request_id UUID REFERENCES screening_request(id),
    licence_id          UUID REFERENCES licence(id),
    aes_itn             VARCHAR(50),                   -- AES Internal Transaction Number
    filed_at            TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE trade_transaction_line (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id      UUID NOT NULL REFERENCES trade_transaction(id) ON DELETE CASCADE,
    line_number         INTEGER NOT NULL,
    product_id          UUID NOT NULL REFERENCES product(id),
    quantity            NUMERIC(18,4) NOT NULL,
    unit_value          NUMERIC(18,2) NOT NULL,
    line_value          NUMERIC(18,2) NOT NULL,
    hs_code             VARCHAR(10),
    eccn_code           VARCHAR(5),
    schedule_b          VARCHAR(10),
    licence_exception   VARCHAR(50),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transaction_tenant ON trade_transaction(tenant_id);
CREATE INDEX idx_transaction_status ON trade_transaction(status);
CREATE INDEX idx_transaction_line_tx ON trade_transaction_line(transaction_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | tenant, user, role, user_role |
| Party Management | 4 | party, party_alias, party_address, party_identifier |
| Watchlist & Sanctions | 5 | watchlist_source, watchlist_entry, aliases, addresses, identifiers |
| Screening | 2 | screening_request, screening_hit |
| Product Classification | 4 | hs_code, eccn, product, classification_decision |
| Licence Management | 3 | licence, licence_consumption, licence_determination |
| Embargo & Regulatory | 2 | embargo_rule, regulatory_update |
| Trade Transactions | 2 | trade_transaction, trade_transaction_line |
| Audit | 1 | audit_log |
| **Total** | **27** | Core schema; additional tables for CBAM, UFLPA, FTA would add ~10-15 |

---

## Key Design Decisions

1. **Separate party and watchlist_entry tables** — Internal counterparties (party) are distinct from government watchlist entries (watchlist_entry). Screening creates links between them via screening_hit. This prevents conflating customer data with sanctions data and allows watchlist data to be refreshed independently.

2. **OFAC-aligned watchlist structure** — The watchlist_entry, alias, address, and identifier tables mirror the OFAC Advanced Sanctions List Standard's UID-linked structure, making it straightforward to ingest OFAC XML and other government list formats.

3. **Classification decisions as immutable records** — Each classification_decision is a standalone row linked to a product. When a classification changes, a new row is created and the old one gets a superseded_by pointer. This provides a complete audit trail of classification history without temporal table complexity.

4. **Licence consumption tracking** — The licence table tracks approved and consumed values/quantities, with individual consumption events recorded in licence_consumption. This enables real-time licence balance queries and consumption reporting.

5. **Row-level tenant isolation** — Every business table has a tenant_id foreign key. Combined with PostgreSQL Row-Level Security policies, this provides multi-tenant data isolation without schema-per-tenant complexity.

6. **Audit log as supplementary record** — The audit_log table captures all state changes with old/new values in JSONB. This provides regulatory audit trail capability without the complexity of event sourcing, though it does not support replaying state.

7. **ISO standard identifiers throughout** — Country codes use ISO 3166-1 alpha-3, currency uses ISO 4217, legal entities can carry LEI (ISO 17442), and scripts use ISO 15924. This aligns with WCO Data Model v4 recommendations and ensures interoperability with government systems.
