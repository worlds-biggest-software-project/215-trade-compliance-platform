# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Trade Compliance Platform · Created: 2026-05-20

## Philosophy

This model combines a relational PostgreSQL core for operational CRUD with a property graph layer for relationship-heavy queries. The key insight for trade compliance is that the hardest compliance problems are network problems: sanctioned ownership chains (OFAC 50% rule requires tracing indirect ownership), multi-tier supply chain exposure (a tier-3 supplier in a sanctioned country), and conflict-of-interest detection across counterparty relationships. Traditional relational queries struggle with variable-depth traversal ("find all entities within 3 hops of a sanctioned party") — graph queries excel at exactly this.

OpenSanctions and OpenOwnership already model sanctions data as property graphs using the FollowtheMoney ontology. Neo4j publishes case studies showing how financial institutions use graph databases for sanctions screening and AML. The graph-relational hybrid lets the platform use PostgreSQL for structured compliance workflows (screening queues, licence management, audit logs) while pushing relationship-intensive queries to a graph engine (either Neo4j or PostgreSQL's Apache AGE extension for those who want a single database).

This is the architecture pattern used by compliance teams at major banks for OFAC 50% rule calculations, by supply chain risk platforms for multi-tier supplier mapping, and by investigative journalism tools (ICIJ's Offshore Leaks uses Neo4j) for uncovering hidden relationships.

**Best for:** Organisations where counterparty network analysis, ownership chain traversal, and multi-tier supply chain risk assessment are primary use cases — particularly those already dealing with OFAC 50% rule compliance or multi-tier supplier due diligence.

**Trade-offs:**
- Pro: Variable-depth relationship traversal in milliseconds (ownership chains, supply networks)
- Pro: OFAC 50% rule calculation becomes a graph algorithm, not a recursive CTE
- Pro: Natural fit for OpenSanctions/FollowtheMoney data integration
- Pro: Visual network exploration for compliance investigators
- Pro: Supply chain graph enables proactive risk identification before screening
- Con: Dual-store complexity — must keep PostgreSQL and graph database synchronised
- Con: Additional infrastructure (Neo4j or Apache AGE) to deploy and maintain
- Con: Graph query languages (Cypher, openCypher) require different developer skills
- Con: Graph databases are less mature for transactional ACID workloads than PostgreSQL
- Con: Higher operational cost than a single-database solution

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FollowtheMoney (FtM) Ontology | Graph node types map directly to FtM entity schemas (Person, LegalEntity, Company, Vessel) |
| OpenSanctions Entity Model | Watchlist entities are imported as graph nodes with FtM-compatible properties |
| OFAC 50% Rule | Ownership edges with percentage weights enable graph traversal to calculate aggregate sanctioned ownership |
| OpenOwnership / BODS | Beneficial Ownership Data Standard informs ownership edge properties |
| ISO 3166-1 | Country nodes in the graph enable jurisdiction-based traversal and filtering |
| WCO SAFE Framework | Trusted trader relationships modelled as graph edges between trading entities |
| Neo4j/openCypher | Graph queries use the openCypher standard for portability between Neo4j and Apache AGE |

---

## PostgreSQL Relational Layer (Operational CRUD)

The relational layer handles all transactional operations: screening workflows, classification decisions, licence management, and audit logging.

```sql
-- Tenant and user tables (same as normalised model)
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    roles           TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

### Party (Relational master record, synced to graph)

```sql
CREATE TABLE party (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    party_type      VARCHAR(50) NOT NULL,
    primary_name    VARCHAR(500) NOT NULL,
    country_code    VARCHAR(3),
    properties      JSONB NOT NULL DEFAULT '{}',
    graph_node_id   BIGINT,                            -- references the graph node for this party
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_tenant ON party(tenant_id);
CREATE INDEX idx_party_name ON party(tenant_id, primary_name);
CREATE INDEX idx_party_graph ON party(graph_node_id) WHERE graph_node_id IS NOT NULL;
```

### Screening (Relational workflow with graph-enhanced matching)

```sql
CREATE TABLE screening (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    party_id        UUID REFERENCES party(id),
    mode            VARCHAR(20) NOT NULL,
    query_name      VARCHAR(500) NOT NULL,
    sources_checked VARCHAR(50)[] NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    hit_count       INTEGER NOT NULL DEFAULT 0,
    
    -- Graph-enhanced fields
    network_risk_score  NUMERIC(5,4),                  -- calculated from graph proximity to sanctioned nodes
    network_hops_to_sanctioned INTEGER,                -- shortest path to any sanctioned entity
    network_analysis    JSONB,                         -- graph traversal results
    -- network_analysis example: {
    --   "paths_to_sanctioned": [
    --     { "path": ["PartyA", "owns 60%", "SubCo", "owns 30%", "SanctionedEntity"],
    --       "aggregate_ownership": 0.18, "sanctioned_entity": "SDN-12345" }
    --   ],
    --   "risk_factors": ["indirect_ownership_above_10pct", "shared_address"]
    -- }
    
    requested_by    UUID NOT NULL REFERENCES "user"(id),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE screening_hit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    screening_id    UUID NOT NULL REFERENCES screening(id) ON DELETE CASCADE,
    watchlist_entry_id UUID NOT NULL,
    match_score     NUMERIC(5,4) NOT NULL,
    match_type      VARCHAR(50) NOT NULL,              -- 'direct_name', 'direct_id', 'network_ownership', 'network_address'
    matched_via     JSONB,                             -- for network matches: the graph path
    status          VARCHAR(50) NOT NULL DEFAULT 'potential_match',
    reviewed_by     UUID REFERENCES "user"(id),
    reviewed_at     TIMESTAMPTZ,
    disposition_note TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_screening_tenant ON screening(tenant_id);
CREATE INDEX idx_screening_status ON screening(status);
CREATE INDEX idx_screening_hit_screening ON screening_hit(screening_id);
```

### Product, Classification, Licence, Transaction (Relational)

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    part_number     VARCHAR(100),
    hs_code         VARCHAR(10),
    eccn            VARCHAR(5),
    classification_status VARCHAR(50) NOT NULL DEFAULT 'unclassified',
    classifications JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE classification_decision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    regime          VARCHAR(50) NOT NULL,
    assigned_code   VARCHAR(50) NOT NULL,
    rationale       TEXT NOT NULL,
    ai_assisted     BOOLEAN NOT NULL DEFAULT false,
    ai_confidence   NUMERIC(5,4),
    decided_by      UUID NOT NULL REFERENCES "user"(id),
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE licence (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    licence_number      VARCHAR(100),
    licence_type        VARCHAR(50) NOT NULL,
    authority           VARCHAR(100) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    product_id          UUID REFERENCES product(id),
    destination_country VARCHAR(3) NOT NULL,
    end_user_party_id   UUID REFERENCES party(id),
    approved_value      NUMERIC(18,2),
    consumed_value      NUMERIC(18,2) NOT NULL DEFAULT 0,
    approved_quantity   NUMERIC(18,4),
    consumed_quantity   NUMERIC(18,4) NOT NULL DEFAULT 0,
    currency_code       VARCHAR(3) DEFAULT 'USD',
    expiry_date         DATE,
    authority_data      JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE trade_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    transaction_ref     VARCHAR(255) NOT NULL,
    transaction_type    VARCHAR(50) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    exporter_party_id   UUID REFERENCES party(id),
    consignee_party_id  UUID REFERENCES party(id),
    destination_country VARCHAR(3) NOT NULL,
    total_value         NUMERIC(18,2),
    currency_code       VARCHAR(3) DEFAULT 'USD',
    screening_id        UUID REFERENCES screening(id),
    licence_id          UUID REFERENCES licence(id),
    line_items          JSONB NOT NULL DEFAULT '[]',
    filing_data         JSONB NOT NULL DEFAULT '{}',
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
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Watchlist entries (relational for ingestion, synced to graph)
CREATE TABLE watchlist_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_code     VARCHAR(50) NOT NULL,
    source_uid      VARCHAR(255) NOT NULL,
    entry_type      VARCHAR(50) NOT NULL,
    primary_name    VARCHAR(500) NOT NULL,
    programme       VARCHAR(255),
    listing_date    DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    aliases         JSONB NOT NULL DEFAULT '[]',
    addresses       JSONB NOT NULL DEFAULT '[]',
    identifiers     JSONB NOT NULL DEFAULT '[]',
    source_data     JSONB NOT NULL DEFAULT '{}',
    graph_node_id   BIGINT,                            -- references the graph node
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_code, source_uid)
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_classification_product ON classification_decision(product_id);
CREATE INDEX idx_licence_tenant ON licence(tenant_id);
CREATE INDEX idx_transaction_tenant ON trade_transaction(tenant_id);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_watchlist_name ON watchlist_entry(primary_name);
CREATE INDEX idx_watchlist_active ON watchlist_entry(is_active) WHERE is_active = true;
```

---

## Graph Layer (Neo4j / Apache AGE)

The graph layer stores entities as nodes and relationships as edges. It is synchronised from the relational layer via change-data-capture or application-level dual writes.

### Node Types

```cypher
// Party node (synced from relational party table)
CREATE (p:Party {
    id: "uuid-...",
    tenant_id: "uuid-...",
    party_type: "organisation",
    primary_name: "ACME Industries GmbH",
    country_code: "DEU",
    is_active: true,
    risk_score: 0.0
})

// WatchlistEntry node (synced from relational watchlist_entry table)
CREATE (w:WatchlistEntry {
    id: "uuid-...",
    source_code: "OFAC_SDN",
    source_uid: "12345",
    entry_type: "organisation",
    primary_name: "Sanctioned Corp LLC",
    programme: "IRAN",
    is_active: true,
    listing_date: date("2024-06-15")
})

// Country node (reference data)
CREATE (c:Country {
    code: "IRN",
    name: "Iran",
    is_embargoed: true,
    embargo_programmes: ["IRAN", "IRAN-HR"]
})

// Product node (optional — only if product-to-product or product-to-entity relationships matter)
CREATE (pr:Product {
    id: "uuid-...",
    name: "Widget A",
    eccn: "3A001",
    hs_code: "8471300000"
})
```

### Edge Types (Relationships)

```cypher
// Ownership — critical for OFAC 50% rule
CREATE (parent)-[:OWNS {
    percentage: 60.0,
    effective_date: date("2020-01-01"),
    source: "company_registry",
    verified: true
}]->(subsidiary)

// Control (non-ownership influence)
CREATE (controller)-[:CONTROLS {
    control_type: "board_majority",      // 'board_majority', 'voting_rights', 'contractual'
    effective_date: date("2021-03-15"),
    source: "annual_report"
}]->(controlled)

// Supply chain relationships
CREATE (buyer)-[:BUYS_FROM {
    product_category: "raw_materials",
    since: date("2019-06-01"),
    tier: 1,                              // direct=1, indirect=2+
    volume_annual_usd: 500000
}]->(supplier)

// Trading relationships
CREATE (exporter)-[:EXPORTS_TO {
    transaction_count: 15,
    last_transaction: date("2026-02-28"),
    total_value_usd: 1200000
}]->(consignee)

// Shared address (indicator of potential front/shell company)
CREATE (entity1)-[:SHARES_ADDRESS {
    address: "123 Commerce St, Dubai, UAE",
    since: date("2022-01-01")
}]->(entity2)

// Shared director/officer
CREATE (entity1)-[:SHARES_OFFICER {
    officer_name: "John Smith",
    role_at_entity1: "Director",
    role_at_entity2: "CEO"
}]->(entity2)

// Party located in country
CREATE (party)-[:LOCATED_IN]->(country)

// Watchlist entry associated with country
CREATE (entry)-[:ASSOCIATED_WITH]->(country)

// Screening match (links operational screening to graph)
CREATE (party)-[:MATCHED_TO {
    match_score: 0.92,
    match_type: "direct_name",
    screening_id: "uuid-...",
    screened_at: datetime("2026-03-15T10:30:00Z")
}]->(watchlist_entry)
```

### Graph Indexes

```cypher
CREATE INDEX party_id_idx FOR (p:Party) ON (p.id);
CREATE INDEX party_name_idx FOR (p:Party) ON (p.primary_name);
CREATE INDEX party_tenant_idx FOR (p:Party) ON (p.tenant_id);
CREATE INDEX watchlist_source_uid_idx FOR (w:WatchlistEntry) ON (w.source_uid);
CREATE INDEX watchlist_name_idx FOR (w:WatchlistEntry) ON (w.primary_name);
CREATE INDEX country_code_idx FOR (c:Country) ON (c.code);
CREATE FULLTEXT INDEX party_name_fulltext FOR (p:Party) ON EACH [p.primary_name];
CREATE FULLTEXT INDEX watchlist_name_fulltext FOR (w:WatchlistEntry) ON EACH [w.primary_name];
```

---

## Key Graph Queries

### OFAC 50% Rule: Aggregate Sanctioned Ownership

```cypher
// Find all ownership paths from a party to any sanctioned entity,
// calculating aggregate ownership percentage at each step.
// OFAC 50% rule: if aggregate ownership by sanctioned parties >= 50%, entity is blocked.

MATCH path = (target:Party {id: $party_id})<-[:OWNS*1..5]-(ancestor)
WHERE ancestor:WatchlistEntry AND ancestor.is_active = true
WITH target, path,
     reduce(pct = 1.0, r IN relationships(path) | pct * (r.percentage / 100.0)) AS aggregate_ownership
WHERE aggregate_ownership >= 0.01  // filter noise
RETURN 
    target.primary_name AS entity,
    [n IN nodes(path) | n.primary_name] AS ownership_chain,
    round(aggregate_ownership * 100, 2) AS aggregate_pct,
    aggregate_ownership >= 0.50 AS is_blocked
ORDER BY aggregate_ownership DESC
```

### Multi-Tier Supply Chain Risk

```cypher
// Find all supply chain paths from a party to any entity in an embargoed country
MATCH path = (company:Party {id: $party_id})-[:BUYS_FROM*1..4]->(supplier:Party)-[:LOCATED_IN]->(country:Country)
WHERE country.is_embargoed = true
RETURN 
    [n IN nodes(path) WHERE n:Party | n.primary_name] AS supply_chain,
    country.name AS risk_country,
    length(path) - 1 AS supply_chain_depth
ORDER BY supply_chain_depth ASC
```

### Network Proximity to Sanctioned Entities

```cypher
// Find shortest paths between a party and any sanctioned entity (any relationship type)
MATCH (party:Party {id: $party_id}),
      (sanctioned:WatchlistEntry {is_active: true}),
      path = shortestPath((party)-[*..6]-(sanctioned))
RETURN 
    sanctioned.primary_name AS sanctioned_entity,
    sanctioned.programme AS programme,
    length(path) AS hops,
    [r IN relationships(path) | type(r)] AS relationship_types,
    [n IN nodes(path) | n.primary_name] AS path_entities
ORDER BY hops ASC
LIMIT 20
```

### Shared Infrastructure Detection (Shell Company Indicators)

```cypher
// Find entities that share addresses or officers with sanctioned entities
MATCH (sanctioned:WatchlistEntry {is_active: true})-[:SHARES_ADDRESS|SHARES_OFFICER]-(connected:Party)
WHERE connected.tenant_id = $tenant_id
RETURN 
    connected.primary_name AS entity,
    sanctioned.primary_name AS sanctioned_entity,
    type(collect(relationships((sanctioned)--(connected)))[0]) AS connection_type,
    sanctioned.programme AS programme
```

### Visualise Party Network (for investigator UI)

```cypher
// Get the full network around a party for visual exploration
MATCH (center:Party {id: $party_id})-[r*1..2]-(connected)
RETURN center, r, connected
```

---

## Synchronisation Strategy

```
┌─────────────────┐     CDC / Dual Write     ┌─────────────────┐
│   PostgreSQL    │ ─────────────────────────>│   Neo4j / AGE   │
│  (source of     │                           │  (graph layer)  │
│   truth for     │                           │                 │
│   CRUD ops)     │  ┌───────────────────┐    │  Nodes: Party,  │
│                 │  │  Sync Service     │    │  WatchlistEntry │
│  party ─────────┼──┤  - Debezium CDC   │───>│  Country        │
│  watchlist_entry┼──┤  - or app-level   │    │                 │
│  screening_hit ─┼──┤    dual write     │    │  Edges: OWNS,   │
│                 │  └───────────────────┘    │  BUYS_FROM,     │
│                 │                           │  EXPORTS_TO,    │
│                 │  <── Graph query results  │  SHARES_ADDRESS │
│                 │      enrichment back      │  MATCHED_TO     │
└─────────────────┘                           └─────────────────┘
```

The synchronisation service:
1. Listens for INSERT/UPDATE/DELETE on party and watchlist_entry tables (via Debezium CDC or application events)
2. Creates/updates corresponding graph nodes
3. Relationship edges (OWNS, BUYS_FROM, etc.) are created through a dedicated relationship management API that writes to both stores
4. Graph query results (network risk scores, ownership calculations) are written back to the relational screening and party tables for fast operational access

---

## Table Count Summary

| Category | Tables (PostgreSQL) | Nodes/Edges (Graph) | Notes |
|----------|-------------------|-------------------|-------|
| Identity & Multi-Tenancy | 2 | — | tenant, user |
| Party Management | 1 | Party nodes | Synced to graph |
| Watchlist & Sanctions | 1 | WatchlistEntry nodes | Synced to graph |
| Screening | 2 | MATCHED_TO edges | screening, screening_hit |
| Product Classification | 2 | Product nodes (optional) | product, classification_decision |
| Licence Management | 1 | — | licence |
| Trade Transactions | 1 | EXPORTS_TO edges | trade_transaction |
| Audit | 1 | — | audit_log |
| Reference Data | — | Country nodes | In graph only |
| Relationships | — | 7 edge types | OWNS, CONTROLS, BUYS_FROM, EXPORTS_TO, SHARES_ADDRESS, SHARES_OFFICER, LOCATED_IN |
| **Total** | **11 tables** | **4 node types, 7+ edge types** | Dual-store architecture |

---

## Key Design Decisions

1. **PostgreSQL as source of truth, graph as derived store** — All writes go through PostgreSQL. The graph database is a synchronised read model optimised for relationship traversal. This avoids the transactional consistency challenges of graph databases and keeps the operational data in a battle-tested RDBMS.

2. **Graph solves the OFAC 50% rule** — The OFAC 50% rule requires aggregating ownership percentages across multi-level corporate hierarchies. In SQL, this requires recursive CTEs with multiplication at each level, which is complex and slow for deep hierarchies. In Cypher, it is a natural pattern: traverse OWNS edges, multiply percentages along the path.

3. **Screening enhanced with network context** — Traditional screening matches a party name against watchlists. Graph-enhanced screening adds: "Is this party within N hops of a sanctioned entity? Does it share addresses or officers with sanctioned entities? Is its supply chain exposed to embargoed countries?" This network_analysis field on the screening table stores these graph-derived insights alongside traditional match results.

4. **Relationship edges are first-class data** — Ownership percentages, supply chain tiers, and trading volumes are properties on graph edges, not just boolean connections. This enables weighted traversal algorithms and quantitative risk scoring.

5. **Apache AGE as single-database alternative** — For teams that want graph capabilities without deploying Neo4j, the Apache AGE extension for PostgreSQL provides openCypher query support within PostgreSQL. The same Cypher queries work, with the trade-off of lower graph query performance at scale compared to a dedicated graph database.

6. **Dual-write with eventual consistency** — The synchronisation between PostgreSQL and the graph database is eventually consistent. For screening workflows, the graph is queried during the screening process (not in the critical path of individual CRUD operations). If the graph is briefly behind, the next screening run will pick up the latest data.

7. **OpenSanctions/FollowtheMoney native import** — OpenSanctions publishes data in the FollowtheMoney graph format. The graph layer can ingest FtM entities and relationships directly as nodes and edges, without the lossy translation required to flatten them into relational tables. This preserves the rich relationship data that OpenSanctions provides.

8. **Visual network explorer** — The graph layer naturally supports a visual investigation UI where compliance officers can explore the network around a party, following ownership chains, supply relationships, and shared infrastructure. This is a significant UX differentiator over traditional list-based screening tools.
