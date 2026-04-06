# Database Query Agent Skill — Specification

## 1. Problem Statement

Enterprise teams hold valuable data in relational databases, but the schemas are
messy, evolved organically, and use non-descriptive column names. Some tables
follow the **Entity-Attribute-Value (EAV)** pattern, making them especially hard
to query without tribal knowledge. Today, only a handful of people who "know the
data" can write correct queries. Everyone else files tickets and waits.

This skill gives an AI agent the ability to **translate natural-language
questions into correct SQL** against these databases by first consulting a
**knowledge layer** (vector DB or static background-info asset) that explains
what the tables, columns, and coded values actually mean.

---

## 2. Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | Translate natural-language questions to correct SQL | ≥ 80 % of test questions produce valid, accurate SQL on first attempt |
| G2 | Handle EAV tables transparently | Agent pivots EAV rows correctly without user needing to know the storage format |
| G3 | Leverage schema knowledge layer | Agent retrieves relevant table/column descriptions before generating SQL |
| G4 | Support iterative refinement | User can say "add a filter for region = East" and the agent amends the query |
| G5 | Safe execution | Read-only access; no DDL/DML; query size/timeout guardrails |

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Agent (LLM)                            │
│                                                             │
│  1. Receive user question                                   │
│  2. Search knowledge layer for relevant schema context      │
│  3. Generate SQL                                            │
│  4. Validate & execute SQL                                  │
│  5. Format & return results                                 │
└────────┬──────────────────┬──────────────────┬──────────────┘
         │                  │                  │
         ▼                  ▼                  ▼
┌────────────────┐ ┌────────────────┐ ┌────────────────────┐
│ Knowledge Layer│ │  SQL Validator  │ │  Database Executor │
│ (Vector DB or  │ │  & Guardrails   │ │  (read-only conn)  │
│  Static Asset) │ │                 │ │                    │
└────────────────┘ └────────────────┘ └────────────────────┘
```

### 3.1 Knowledge Layer

The knowledge layer is the bridge between messy schemas and the LLM. It stores
**human-written descriptions** of tables, columns, relationships, coded values,
and common query patterns.

**Two interchangeable backends:**

| Backend | When to use |
|---------|-------------|
| **Vector DB** (e.g. ChromaDB, Qdrant, Pinecone) | Large number of tables/columns (100+); need semantic search across descriptions |
| **Static background-info asset** (YAML/JSON files) | Smaller schemas; simpler deployment; version-controlled alongside code |

Both backends expose the same interface (`KnowledgeProvider`), so the agent
skill doesn't need to know which one is active.

### 3.2 Knowledge Document Schema

Each knowledge entry describes one database object:

```yaml
# Example: a table description
- object_type: table
  database: warehouse
  schema: dbo
  table: tbl_cx_attr
  display_name: Customer Attributes
  description: >
    EAV table storing customer-level attributes. Each row is one
    attribute for one customer. Join to dim_customer on cust_id.
  storage_format: eav          # "wide" | "eav"
  eav_config:                  # only when storage_format = eav
    entity_column: cust_id
    attribute_column: attr_cd
    value_column: attr_val
  tags: [customer, attributes, eav]

# Example: a column description
- object_type: column
  database: warehouse
  schema: dbo
  table: tbl_cx_attr
  column: attr_cd
  display_name: Attribute Code
  description: >
    Coded attribute name. Known values include 'REGION', 'TIER',
    'ACQ_DT' (acquisition date), 'LTV' (lifetime value).
  data_type: varchar(50)
  coded_values:
    REGION: "Geographic sales region (East, West, Central, South)"
    TIER: "Customer tier (Gold, Silver, Bronze)"
    ACQ_DT: "Customer acquisition date (YYYY-MM-DD format stored as string)"
    LTV: "Lifetime value in USD (numeric stored as string)"

# Example: a relationship / join hint
- object_type: relationship
  description: "Join tbl_cx_attr.cust_id = dim_customer.id"
  from_table: tbl_cx_attr
  from_column: cust_id
  to_table: dim_customer
  to_column: id
  join_type: many-to-one

# Example: a query pattern (common question → known-good SQL)
- object_type: query_pattern
  description: "Get all Gold-tier customers acquired in 2024"
  sql_template: |
    SELECT e.cust_id, c.cust_name, acq.attr_val AS acquisition_date
    FROM tbl_cx_attr e
    JOIN dim_customer c ON c.id = e.cust_id
    JOIN tbl_cx_attr acq ON acq.cust_id = e.cust_id AND acq.attr_cd = 'ACQ_DT'
    WHERE e.attr_cd = 'TIER' AND e.attr_val = 'Gold'
      AND acq.attr_val LIKE '2024%'
  tags: [customer, tier, acquisition]
```

### 3.3 SQL Validator & Guardrails

Before execution, every generated query passes through validation:

| Check | Action |
|-------|--------|
| **Statement type** | Only `SELECT` allowed. Reject `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `TRUNCATE`, etc. |
| **Row limit** | Inject `LIMIT` / `TOP` if not present (default: 1000) |
| **Timeout** | Enforce query timeout (default: 30 s) |
| **Table allowlist** | Optional — restrict which tables the agent can query |
| **Syntax check** | Parse SQL to catch syntax errors before hitting the DB |

### 3.4 Database Executor

- Opens a **read-only** connection (enforced at DB user/role level)
- Supports multiple DB engines via pluggable adapters: PostgreSQL, MySQL, SQL Server, SQLite
- Returns results as structured data (list of dicts) + row count
- Enforces timeout at the connection level

---

## 4. Agent Skill Interface

The skill exposes the following **tools** to the agent:

### 4.1 `search_schema_knowledge`

Search the knowledge layer for relevant schema information.

```
Input:
  query: str          # natural-language description of what the user is looking for
  filters: dict       # optional metadata filters (database, schema, table, object_type, tags)
  top_k: int          # number of results (default 10)

Output:
  results: list[KnowledgeEntry]   # ranked list of matching entries
```

### 4.2 `list_tables`

List available tables with their descriptions.

```
Input:
  database: str       # optional filter
  schema: str         # optional filter
  name_pattern: str   # optional glob/regex on table name

Output:
  tables: list[TableSummary]   # table name, display_name, description, storage_format
```

### 4.3 `describe_table`

Get full metadata for a specific table.

```
Input:
  table: str          # fully qualified table name

Output:
  description: str
  storage_format: "wide" | "eav"
  eav_config: EAVConfig | null
  columns: list[ColumnDetail]
  relationships: list[Relationship]
  sample_queries: list[QueryPattern]
```

### 4.4 `validate_sql`

Validate SQL without executing.

```
Input:
  sql: str

Output:
  valid: bool
  errors: list[str]
  warnings: list[str]       # e.g. "no LIMIT clause — will be capped at 1000"
```

### 4.5 `execute_query`

Execute a validated SQL query.

```
Input:
  sql: str
  params: dict              # bind parameters
  row_limit: int            # override default limit

Output:
  columns: list[str]
  rows: list[list[any]]
  row_count: int
  truncated: bool           # true if limit was hit
  execution_time_ms: int
```

---

## 5. EAV Handling Strategy

EAV tables are the primary source of complexity. The agent handles them via:

### 5.1 Knowledge-Driven Pivoting

When the knowledge layer identifies a table as `storage_format: eav`, the agent
receives `eav_config` telling it which columns hold entity, attribute, and value.
The agent then generates **pivot-style SQL** (self-joins or conditional
aggregation) to present the data in a user-friendly wide format.

### 5.2 Example Transformation

User asks: *"Show me customer name, region, and tier for customers acquired in 2024"*

The agent recognizes `tbl_cx_attr` is EAV and generates:

```sql
SELECT
  c.cust_name,
  MAX(CASE WHEN a.attr_cd = 'REGION' THEN a.attr_val END) AS region,
  MAX(CASE WHEN a.attr_cd = 'TIER'   THEN a.attr_val END) AS tier
FROM dim_customer c
JOIN tbl_cx_attr a ON a.cust_id = c.id
JOIN tbl_cx_attr acq ON acq.cust_id = c.id
  AND acq.attr_cd = 'ACQ_DT'
  AND acq.attr_val LIKE '2024%'
WHERE a.attr_cd IN ('REGION', 'TIER')
GROUP BY c.id, c.cust_name
```

### 5.3 Fallback

If pivot SQL is too complex or the agent is unsure, it can fall back to
returning raw EAV rows with a note explaining the format.

---

## 6. Agent Workflow (Step-by-Step)

```
User: "How many Gold-tier customers do we have per region?"

Agent:
  1. search_schema_knowledge("customer tier region count")
     → gets: tbl_cx_attr (EAV), dim_customer, coded values for TIER and REGION

  2. describe_table("dbo.tbl_cx_attr")
     → confirms EAV config, sees attr_cd values

  3. Generates SQL:
     SELECT
       MAX(CASE WHEN a.attr_cd = 'REGION' THEN a.attr_val END) AS region,
       COUNT(DISTINCT a.cust_id) AS customer_count
     FROM tbl_cx_attr a
     WHERE a.cust_id IN (
       SELECT cust_id FROM tbl_cx_attr
       WHERE attr_cd = 'TIER' AND attr_val = 'Gold'
     )
     AND a.attr_cd = 'REGION'
     GROUP BY a.cust_id
     -- (agent refines: actually group by region, not cust_id)

  4. validate_sql(sql)
     → valid: true, warnings: ["LIMIT 1000 will be applied"]

  5. execute_query(sql)
     → returns region counts

  6. Formats result as a table and explains the answer
```

---

## 7. Configuration

```yaml
# config.yaml
skill:
  name: database-query
  version: "0.1.0"

knowledge:
  provider: vector_db          # "vector_db" | "static_asset"

  # Vector DB settings (when provider = vector_db)
  vector_db:
    engine: chromadb            # chromadb | qdrant | pinecone
    collection: schema_knowledge
    embedding_model: text-embedding-3-small
    host: localhost
    port: 8000

  # Static asset settings (when provider = static_asset)
  static_asset:
    path: ./knowledge/          # directory of YAML files

database:
  engine: postgresql            # postgresql | mysql | sqlserver | sqlite
  host: localhost
  port: 5432
  database: warehouse
  user: readonly_agent
  password_env: DB_PASSWORD     # read from environment variable
  connect_timeout: 10
  query_timeout: 30

guardrails:
  max_row_limit: 10000
  default_row_limit: 1000
  allowed_statements: [SELECT]
  blocked_tables: []            # optional denylist
  allowed_tables: []            # optional allowlist (empty = all)
  max_query_length: 10000       # characters
```

---

## 8. Security Considerations

| Concern | Mitigation |
|---------|------------|
| SQL injection | Use parameterized queries; parse and validate SQL AST before execution |
| Data exfiltration | Row limits, query timeout, optional table allowlist |
| Privilege escalation | DB user has read-only grants only; no `GRANT`, `CREATE`, `EXECUTE` |
| Prompt injection via data | Sanitize query results before returning to LLM context |
| Credential leakage | Passwords via env vars only; never logged or returned to user |

---

## 9. Non-Goals (v0.1)

- **Write operations** — this skill is read-only
- **Cross-database joins** — single database per skill instance
- **Automated schema discovery** — knowledge layer is manually curated (auto-discovery is a future enhancement)
- **Visualization / charting** — the skill returns data; presentation is the caller's responsibility
- **Query caching** — not in initial scope
