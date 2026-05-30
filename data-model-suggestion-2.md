# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Trade Compliance Platform · Created: 2026-05-20

## Philosophy

This model treats every action in the compliance lifecycle as an immutable event stored in an append-only event store. The event store is the single source of truth. Read-optimised materialised views (projections) are rebuilt from events to serve queries. This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from projections.

Trade compliance is a domain where "what happened and when" matters as much as "what is true now." Regulators may ask: "Was this party screened before the shipment on March 15th? What classification was in effect when the licence was issued? When exactly did the analyst clear that match?" Event sourcing answers these questions by design, because the full history is the data — not a secondary audit log bolted on after the fact.

This pattern is used in financial services (banking transaction ledgers, securities trading), healthcare (clinical event records), and by platforms like EventStoreDB and Apache Kafka for regulated systems. The FinTech industry has demonstrated that CQRS with event sourcing scales well for compliance-heavy workloads where auditability is non-negotiable.

**Best for:** Organisations where regulatory audit trail integrity is the top priority, temporal queries are frequent, and the team is comfortable with eventual consistency and event-driven architecture.

**Trade-offs:**
- Pro: Complete, immutable, tamper-evident audit trail by construction — not an afterthought
- Pro: Temporal queries ("state as of date X") are trivial — just replay events up to that point
- Pro: New read models can be added retroactively by replaying the event log
- Pro: Natural fit for AI analytics on compliance decision patterns over time
- Con: Higher implementation complexity — requires event store, projection engine, and snapshot management
- Con: Eventual consistency between event store and projections requires careful handling
- Con: Queries against current state require well-maintained projections; ad-hoc SQL is harder
- Con: Event schema evolution (versioning) must be managed carefully as the domain evolves
- Con: Larger storage footprint since events are never deleted

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OFAC Advanced Sanctions List Standard | Watchlist update events capture the full OFAC entry diff on each list publication |
| 21 CFR Part 11 / Annex 11 (Audit Trail) | Event store inherently satisfies "secure, computer-generated, time-stamped audit trail" requirements |
| ISO 27001:2022 | Immutable event log supports information security audit and incident investigation |
| NIST Cybersecurity Framework 2.0 | Event store provides the "Detect" and "Respond" evidence trail for security events |
| WCO Data Model v4 | Trade transaction events include WCO-aligned data elements |
| EAR / ITAR | Classification events capture the full regulatory reasoning chain for defensibility |

---

## Event Store

```sql
-- The single source of truth: an append-only event log
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                    -- aggregate root ID (party, product, licence, etc.)
    stream_type     VARCHAR(100) NOT NULL,             -- 'Party', 'Product', 'ScreeningSession', 'Licence'
    event_type      VARCHAR(200) NOT NULL,             -- 'PartyCreated', 'ScreeningCompleted', 'ClassificationDecided'
    event_version   INTEGER NOT NULL,                  -- monotonically increasing per stream
    tenant_id       UUID NOT NULL,
    payload         JSONB NOT NULL,                    -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',       -- user_id, ip_address, correlation_id, causation_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Optimise for stream replay and temporal queries
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, created_at);
CREATE INDEX idx_event_tenant ON event_store(tenant_id, created_at);
CREATE INDEX idx_event_created ON event_store(created_at);

-- Snapshots for performance: avoid replaying thousands of events
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    snapshot_version INTEGER NOT NULL,                 -- event_version at time of snapshot
    state           JSONB NOT NULL,                    -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Event Type Taxonomy

```
-- Party lifecycle
PartyCreated              { party_type, primary_name, nationality, ... }
PartyUpdated              { changed_fields: { field: [old, new], ... } }
PartyAliasAdded           { alias_type, alias_name, script }
PartyAddressAdded         { address_type, country_code, ... }
PartyIdentifierAdded      { identifier_type, identifier_value, ... }
PartyDeactivated          { reason }

-- Screening lifecycle
ScreeningRequested        { party_id, query_name, mode, sources }
ScreeningHitFound         { entry_id, match_score, match_algorithm, matched_fields }
ScreeningHitDisposed      { hit_id, status, disposition_note, reviewed_by }
ScreeningCompleted        { overall_status, hit_count, clear_count }
ContinuousRescreenTriggered { party_id, trigger: 'list_update' | 'schedule' }

-- Classification lifecycle
ClassificationRequested   { product_id, classification_type }
ClassificationDecided     { product_id, assigned_code, rationale, confidence_score, ai_assisted }
ClassificationSuperseded  { old_decision_id, new_decision_id, reason }

-- Licence lifecycle
LicenceApplicationCreated { licence_type, authority, product_id, destination_country, end_user }
LicenceSubmitted          { licence_number, submitted_to }
LicenceApproved           { approved_value, approved_quantity, expiry_date, conditions }
LicenceDenied             { denial_reason }
LicenceConsumed           { transaction_ref, value_consumed, quantity_consumed }
LicenceExpired            { }
LicenceRevoked            { revocation_reason }

-- Licence determination
LicenceDeterminationMade  { product_id, destination, end_user, licence_required, exception, basis }

-- Watchlist updates
WatchlistSourceUpdated    { source_code, entries_added, entries_modified, entries_removed }
WatchlistEntryAdded       { source_uid, primary_name, programme, ... }
WatchlistEntryModified    { source_uid, changed_fields }
WatchlistEntryDelisted    { source_uid, delisting_date }

-- Regulatory updates
RegulatoryUpdatePublished { source, title, effective_date, affected_eccns, affected_countries }

-- Trade transactions
TransactionCreated        { transaction_ref, type, destination_country, parties, lines }
TransactionScreened       { screening_request_id, result }
TransactionLicenceLinked  { licence_id }
TransactionFiled          { aes_itn, filed_at }
TransactionCompleted      { }
```

## Read Model Projections

Projections are materialised views rebuilt from events. They serve all read queries.

```sql
-- Projection: Current party state (rebuilt from Party* events)
CREATE TABLE proj_party (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    party_type      VARCHAR(50) NOT NULL,
    primary_name    VARCHAR(500) NOT NULL,
    aliases         JSONB NOT NULL DEFAULT '[]',
    addresses       JSONB NOT NULL DEFAULT '[]',
    identifiers     JSONB NOT NULL DEFAULT '[]',
    nationality     VARCHAR(3),
    lei             VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_screened_at TIMESTAMPTZ,
    last_screening_status VARCHAR(50),
    event_version   INTEGER NOT NULL,                  -- tracks which event this projection is current to
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Current screening results
CREATE TABLE proj_screening (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    party_id        UUID,
    query_name      VARCHAR(500) NOT NULL,
    mode            VARCHAR(20) NOT NULL,
    overall_status  VARCHAR(50) NOT NULL,
    hit_count       INTEGER NOT NULL DEFAULT 0,
    hits            JSONB NOT NULL DEFAULT '[]',       -- denormalised hit details
    requested_by    UUID NOT NULL,
    requested_at    TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    event_version   INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Current product classification
CREATE TABLE proj_product (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    part_number     VARCHAR(100),
    current_hs_code VARCHAR(10),
    current_eccn    VARCHAR(5),
    current_usml    VARCHAR(50),
    ear99           BOOLEAN NOT NULL DEFAULT false,
    classification_status VARCHAR(50) NOT NULL DEFAULT 'unclassified',
    classification_history JSONB NOT NULL DEFAULT '[]',  -- array of past decisions
    event_version   INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Current licence state
CREATE TABLE proj_licence (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    licence_number  VARCHAR(100),
    licence_type    VARCHAR(50) NOT NULL,
    authority       VARCHAR(100) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    destination_country VARCHAR(3) NOT NULL,
    approved_value  NUMERIC(18,2),
    consumed_value  NUMERIC(18,2) NOT NULL DEFAULT 0,
    remaining_value NUMERIC(18,2),
    approved_quantity NUMERIC(18,4),
    consumed_quantity NUMERIC(18,4) NOT NULL DEFAULT 0,
    remaining_quantity NUMERIC(18,4),
    expiry_date     DATE,
    event_version   INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Watchlist entries (current active set)
CREATE TABLE proj_watchlist_entry (
    id              UUID PRIMARY KEY,
    source_code     VARCHAR(50) NOT NULL,
    source_uid      VARCHAR(255) NOT NULL,
    entry_type      VARCHAR(50) NOT NULL,
    primary_name    VARCHAR(500) NOT NULL,
    aliases         JSONB NOT NULL DEFAULT '[]',
    addresses       JSONB NOT NULL DEFAULT '[]',
    identifiers     JSONB NOT NULL DEFAULT '[]',
    programme       VARCHAR(255),
    listing_date    DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    event_version   INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Audit timeline (flattened for compliance reporting)
CREATE TABLE proj_audit_timeline (
    event_id        UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(200) NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    stream_id       UUID NOT NULL,
    actor_id        UUID,
    actor_email     VARCHAR(255),
    summary         TEXT NOT NULL,                     -- human-readable event description
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_party_tenant ON proj_party(tenant_id);
CREATE INDEX idx_proj_party_name ON proj_party(tenant_id, primary_name);
CREATE INDEX idx_proj_screening_tenant ON proj_screening(tenant_id);
CREATE INDEX idx_proj_screening_status ON proj_screening(overall_status);
CREATE INDEX idx_proj_product_tenant ON proj_product(tenant_id);
CREATE INDEX idx_proj_licence_tenant ON proj_licence(tenant_id);
CREATE INDEX idx_proj_licence_status ON proj_licence(status);
CREATE INDEX idx_proj_audit_tenant ON proj_audit_timeline(tenant_id, created_at DESC);
CREATE INDEX idx_proj_audit_stream ON proj_audit_timeline(stream_type, stream_id, created_at);
```

## Reference Data (Non-Event-Sourced)

Reference data that does not change per-tenant is stored in regular relational tables.

```sql
CREATE TABLE ref_hs_code (
    code            VARCHAR(10) PRIMARY KEY,
    chapter         VARCHAR(2) NOT NULL,
    heading         VARCHAR(4) NOT NULL,
    subheading      VARCHAR(6) NOT NULL,
    description     TEXT NOT NULL,
    edition         VARCHAR(10) NOT NULL DEFAULT 'HS2022'
);

CREATE TABLE ref_eccn (
    code            VARCHAR(5) PRIMARY KEY,
    category        CHAR(1) NOT NULL,
    product_group   CHAR(1) NOT NULL,
    control_reason  VARCHAR(100),
    description     TEXT NOT NULL
);

CREATE TABLE ref_country (
    code            VARCHAR(3) PRIMARY KEY,           -- ISO 3166-1 alpha-3
    name            VARCHAR(255) NOT NULL,
    alpha2          VARCHAR(2) NOT NULL,
    is_embargoed    BOOLEAN NOT NULL DEFAULT false,
    embargo_programmes VARCHAR(255)[]
);

CREATE TABLE ref_watchlist_source (
    code            VARCHAR(50) PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    administering_body VARCHAR(255) NOT NULL,
    url             TEXT,
    update_frequency VARCHAR(50)
);
```

## Example Queries

### Temporal Query: Party state as of a specific date

```sql
-- Reconstruct party state as it existed on 2026-03-15
SELECT payload
FROM event_store
WHERE stream_id = '...' 
  AND stream_type = 'Party'
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;

-- Application code replays these events to reconstruct the aggregate state
```

### Compliance Report: All screening activity for a party

```sql
SELECT 
    event_type,
    payload,
    metadata->>'user_id' AS actor,
    created_at
FROM event_store
WHERE stream_type = 'ScreeningSession'
  AND payload->>'party_id' = '...'
ORDER BY created_at ASC;
```

### Audit Query: Who changed a classification and why?

```sql
SELECT 
    event_type,
    payload->>'assigned_code' AS code,
    payload->>'rationale' AS rationale,
    payload->>'ai_assisted' AS ai_assisted,
    metadata->>'user_id' AS decided_by,
    created_at
FROM event_store
WHERE stream_id = '...'  -- product stream
  AND event_type IN ('ClassificationDecided', 'ClassificationSuperseded')
ORDER BY created_at ASC;
```

### Fast Current-State Query (via projection)

```sql
-- All pending screening hits for a tenant (reads from projection, not event store)
SELECT s.query_name, s.overall_status, s.hits, s.requested_at
FROM proj_screening s
WHERE s.tenant_id = '...'
  AND s.overall_status = 'potential_match'
ORDER BY s.requested_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store, event_snapshot |
| Projections | 6 | proj_party, proj_screening, proj_product, proj_licence, proj_watchlist_entry, proj_audit_timeline |
| Reference Data | 4 | ref_hs_code, ref_eccn, ref_country, ref_watchlist_source |
| **Total** | **12** | Far fewer tables but higher application-layer complexity |

---

## Key Design Decisions

1. **Single event_store table for all aggregates** — Rather than one event table per aggregate type, a single event_store table with stream_type discrimination keeps the storage layer simple and allows cross-aggregate temporal queries. The stream_id + event_version unique constraint ensures ordering within each aggregate.

2. **Events carry full payload, not diffs** — Each event contains all the data needed to understand what happened without requiring the previous state. This makes events self-contained and simplifies projection rebuilding. The trade-off is larger storage, which is acceptable for compliance data volumes.

3. **Projections are disposable and rebuildable** — Every projection table can be dropped and rebuilt by replaying the event store. This means new read models (e.g., a new compliance dashboard) can be added at any time by writing a new projection handler and replaying history.

4. **Snapshots for performance** — The event_snapshot table stores serialised aggregate state at periodic intervals. When reconstructing an aggregate, the system loads the latest snapshot and replays only events after that version, avoiding full replay of long-lived aggregates.

5. **Reference data is not event-sourced** — HS codes, ECCNs, country lists, and watchlist source metadata are reference data that changes infrequently and globally. These are stored in regular relational tables to avoid unnecessary event complexity.

6. **Metadata captures actor and correlation** — Every event's metadata JSONB includes user_id, ip_address, correlation_id (linking related events across aggregates), and causation_id (the event that triggered this one). This supports full causal chain analysis for regulatory investigations.

7. **Eventual consistency is explicit** — The system acknowledges that projections may lag behind the event store by milliseconds to seconds. For screening decisions, the command handler validates against the event store directly (strong consistency), while dashboards read from projections (eventual consistency).

8. **Event versioning strategy** — Events use a type-based versioning approach. When an event schema changes (e.g., ClassificationDecided gains a new field), the projection handler supports both old and new versions via upcasting. Old events are never modified.
