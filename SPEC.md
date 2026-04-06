# Database Query Agent Skill — Specification

## 1. Problem Statement

Enterprise teams hold valuable data in relational databases, but the schemas are
messy, evolved organically, and use non-descriptive column names. Some tables
follow the **Entity-Attribute-Value (EAV)** pattern, making them especially hard
to query without tribal knowledge. Today, only a handful of people who "know the
data" can write correct queries. Everyone else files tickets and waits.

This skill gives an AI agent the ability to **translate natural-language
questions into correct SQL** against these databases by first consulting a
**knowledge layer** (static background-info asset, upgradeable to vector DB)
that explains what the tables, columns, and coded values actually mean.

### Key Constraints

| Constraint | Detail |
|------------|--------|
| **Data volume** | Tables range from millions to billions of rows — query performance is critical |
| **EAV variety** | Multiple EAV patterns exist (single value col, typed value cols, etc.) |
| **Schema size** | ~3 tables initially; knowledge layer must scale but starts small |
| **Output size** | Users expect large result sets — pagination and export are required |
| **Portability** | Must be a simple downloadable Python skill, not a hosted service |

---

## 2. Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | Translate natural-language questions to correct SQL | ≥ 80 % of test questions produce valid, accurate SQL on first attempt |
| G2 | Handle diverse EAV patterns transparently | Agent pivots EAV rows correctly regardless of EAV variant |
| G3 | Leverage schema knowledge layer | Agent retrieves relevant table/column descriptions before generating SQL |
| G4 | Support iterative multi-step reasoning | Agent explores schema → drafts SQL → validates → refines → executes |
| G5 | Safe execution with large-data awareness | Read-only; timeout guardrails; pagination for large results |
| G6 | Auto-extract schema metadata | CLI tool to bootstrap knowledge YAML from `INFORMATION_SCHEMA` |
| G7 | Framework-agnostic | Works with Claude tool_use, but adaptable to any agent framework |

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Agent (LLM)                              │
│                                                                 │
│  1. Receive user question                                       │
│  2. search_schema_knowledge → understand relevant tables        │
│  3. describe_table → get EAV config, columns, join hints        │
│  4. Generate SQL (with EAV pivoting, performance hints)         │
│  5. validate_sql → check safety                                 │
│  6. explain_query → verify plan on large tables                 │
│  7. execute_query → get results (paginated)                     │
│  8. Format & return results (or export to file)                 │
└────────┬──────────────────┬──────────────────┬──────────────────┘
         │                  │                  │
         ▼                  ▼                  ▼
┌────────────────┐ ┌────────────────┐ ┌───────────────────────┐
│ Knowledge Layer│ │  SQL Validator  │ │  Database Executor    │
│ (Static YAML,  │ │  & Guardrails   │ │  (read-only, pooled,  │
│  upgradeable   │ │                 │ │   paginated results)  │
│  to Vector DB) │ │                 │ │                       │
└────────────────┘ └────────────────┘ └───────────────────────┘
         ▲
         │
┌────────────────┐
│ Schema Crawler  │  ← CLI: auto-extract from INFORMATION_SCHEMA
│ (bootstrap)     │    then data team enriches descriptions
└────────────────┘
```

### 3.1 Knowledge Layer

The knowledge layer bridges messy schemas and the LLM. It stores **human-written
descriptions** of tables, columns, relationships, coded values, and common query
patterns.

**Primary backend: Static YAML files** — version-controlled, simple to deploy,
editable by the data team. For the current scale (~3 tables), this is sufficient.

**Future upgrade path: Vector DB** — when the schema grows to 100+ tables, swap
in a vector provider behind the same `KnowledgeProvider` interface. Both backends
are interchangeable.

### 3.2 Knowledge Document Schema

Each knowledge entry describes one database object:

```yaml
# Example: a regular wide table
- object_type: table
  database: warehouse
  schema: dbo
  table: dim_customer
  display_name: Customer Dimension
  description: >
    Master customer table. One row per customer.
  storage_format: wide
  row_estimate: 5_000_000           # helps agent reason about performance
  partition_key: null
  tags: [customer, dimension]

# Example: EAV with single value column
- object_type: table
  database: warehouse
  schema: dbo
  table: tbl_cx_attr
  display_name: Customer Attributes
  description: >
    EAV table storing customer-level attributes.
  storage_format: eav
  row_estimate: 50_000_000
  eav_config:
    variant: single_value           # see §3.3 for variants
    entity_column: cust_id
    attribute_column: attr_cd
    value_columns:
      - name: attr_val
        type: varchar(500)
  tags: [customer, attributes, eav]

# Example: EAV with typed value columns
- object_type: table
  database: warehouse
  schema: dbo
  table: tbl_entity_props
  display_name: Entity Properties
  description: >
    EAV table with separate columns for different data types.
  storage_format: eav
  row_estimate: 200_000_000
  eav_config:
    variant: typed_values           # multiple value columns by type
    entity_column: entity_id
    attribute_column: prop_key
    value_columns:
      - name: val_str
        type: varchar(1000)
        description: "String values"
      - name: val_num
        type: numeric(18,4)
        description: "Numeric values"
      - name: val_dt
        type: timestamp
        description: "Date/time values"
    type_discriminator: val_type    # optional column that indicates which value_column to read
  tags: [entity, properties, eav]

# Example: a column description with coded values
- object_type: column
  database: warehouse
  schema: dbo
  table: tbl_cx_attr
  column: attr_cd
  display_name: Attribute Code
  description: >
    Coded attribute name.
  data_type: varchar(50)
  coded_values:
    REGION: "Geographic sales region (East, West, Central, South)"
    TIER: "Customer tier (Gold, Silver, Bronze)"
    ACQ_DT: "Acquisition date (YYYY-MM-DD stored as string)"
    LTV: "Lifetime value in USD (numeric stored as string)"

# Example: a relationship / join hint
- object_type: relationship
  description: "Customer attributes belong to a customer"
  from_table: tbl_cx_attr
  from_column: cust_id
  to_table: dim_customer
  to_column: id
  join_type: many-to-one

# Example: a query pattern (common question → known-good SQL)
- object_type: query_pattern
  description: "Count of Gold-tier customers by region"
  sql_template: |
    SELECT
      r.attr_val AS region,
      COUNT(DISTINCT t.cust_id) AS gold_customer_count
    FROM tbl_cx_attr t
    JOIN tbl_cx_attr r ON r.cust_id = t.cust_id AND r.attr_cd = 'REGION'
    WHERE t.attr_cd = 'TIER' AND t.attr_val = 'Gold'
    GROUP BY r.attr_val
    ORDER BY gold_customer_count DESC
  tags: [customer, tier, region, eav]

# Example: performance hint
- object_type: performance_hint
  table: tbl_cx_attr
  description: >
    This table has a composite index on (attr_cd, cust_id). Always filter
    by attr_cd first for best performance. Avoid full table scans — the
    table has 50M+ rows.
  indexes:
    - columns: [attr_cd, cust_id]
      type: btree
    - columns: [cust_id]
      type: btree
  tags: [performance, index]
```

### 3.3 EAV Variant Taxonomy

Different EAV tables use different patterns. The knowledge layer must describe
which variant each table uses so the agent generates correct pivoting SQL.

| Variant | Description | Example |
|---------|-------------|---------|
| `single_value` | One value column (varchar). All types stored as strings. | `(entity_id, attr_code, attr_value)` |
| `typed_values` | Separate columns per data type. A discriminator column may indicate which to read. | `(entity_id, prop_key, val_str, val_num, val_dt, val_type)` |
| `multi_value` | Multiple value columns for different aspects of the same attribute. | `(entity_id, attr_code, val_1, val_2, unit)` |

The `eav_config` block in the knowledge YAML captures:
- `variant`: which pattern
- `entity_column`: the FK / entity identifier
- `attribute_column`: the column that names the attribute
- `value_columns[]`: one or more value columns with name, type, description
- `type_discriminator` (optional): column indicating which value column to read

### 3.4 SQL Validator & Guardrails

Before execution, every generated query passes through validation:

| Check | Action |
|-------|--------|
| **Statement type** | Only `SELECT` allowed. Reject `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `TRUNCATE`, etc. |
| **Row limit** | Inject `LIMIT` if not present (default configurable, default: 1000) |
| **Timeout** | Enforce query timeout (default: 60 s — higher than v0 draft, given large tables) |
| **Table allowlist** | Optional — restrict which tables the agent can query |
| **Syntax check** | Parse SQL via `sqlglot` to catch syntax errors before hitting the DB |
| **Cost estimation** | Warn if query might trigger full table scan on a billion-row table |

### 3.5 Database Executor

- Opens a **read-only** connection (enforced at DB user/role level)
- **Primary adapter: PostgreSQL** (via `psycopg`)
- **SQLite adapter** for local testing without a DB server
- Additional RDBMS adapters can be added behind the `DatabaseExecutor` interface
- Returns results as structured data + metadata
- Enforces timeout at the connection level
- **Server-side cursors** for large result sets — stream rows instead of loading all into memory

---

## 4. Agent Skill Interface

The skill exposes the following **tools** to the agent. The agent is expected
to use them in a multi-step reasoning loop (search → explore → generate →
validate → execute), not as a single-shot call.

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
  tables: list[TableSummary]   # name, display_name, description, storage_format, row_estimate
```

### 4.3 `describe_table`

Get full metadata for a specific table including EAV config.

```
Input:
  table: str          # fully qualified table name

Output:
  description: str
  storage_format: "wide" | "eav"
  eav_config: EAVConfig | null
  row_estimate: int | null
  columns: list[ColumnDetail]
  relationships: list[Relationship]
  performance_hints: list[PerformanceHint]
  sample_queries: list[QueryPattern]
```

### 4.4 `validate_sql`

Validate SQL without executing.

```
Input:
  sql: str
  dialect: str        # optional, default from config (e.g. "postgres")

Output:
  valid: bool
  errors: list[str]
  warnings: list[str]       # e.g. "no LIMIT clause — will be capped at 1000"
  tables_referenced: list[str]
```

### 4.5 `explain_query`

Run EXPLAIN (not EXPLAIN ANALYZE) to show the query plan without executing.
Useful for catching full table scans on billion-row tables before committing.

```
Input:
  sql: str

Output:
  plan: str                 # raw EXPLAIN output
  warnings: list[str]       # e.g. "Seq Scan on tbl_cx_attr (50M rows)"
```

### 4.6 `execute_query`

Execute a validated SQL query with pagination support.

```
Input:
  sql: str
  params: dict              # bind parameters
  row_limit: int            # rows per page (default: 1000)
  offset: int               # pagination offset (default: 0)
  export_format: str | null # null = return inline, "csv" = write to file

Output (inline):
  columns: list[str]
  rows: list[list[any]]
  row_count: int
  total_count: int | null   # if determinable without extra query
  has_more: bool
  execution_time_ms: int

Output (export):
  file_path: str            # path to exported CSV
  row_count: int
  execution_time_ms: int
```

### 4.7 `get_sample_rows`

Preview actual data from a table (useful for the agent to understand coded
values and data format before writing the real query).

```
Input:
  table: str
  where: str | null         # optional filter (e.g. "attr_cd = 'REGION'")
  limit: int                # default 5

Output:
  columns: list[str]
  rows: list[list[any]]
```

---

## 5. EAV Handling Strategy

EAV tables are few in number but hold critical data. The agent handles them via
a **multi-step approach**:

### 5.1 Discovery

The agent calls `search_schema_knowledge` or `describe_table` and learns:
- The table is EAV
- Which variant (single_value, typed_values, multi_value)
- The entity/attribute/value column names
- Known attribute codes and their meanings

### 5.2 Exploration

The agent can call `get_sample_rows` to see actual data:
```
get_sample_rows("tbl_cx_attr", where="attr_cd = 'TIER'", limit=5)
```
This helps ground the agent in reality before generating SQL.

### 5.3 SQL Generation — Variant-Aware Pivoting

**single_value variant:**
```sql
SELECT
  c.cust_name,
  MAX(CASE WHEN a.attr_cd = 'REGION' THEN a.attr_val END) AS region,
  MAX(CASE WHEN a.attr_cd = 'TIER'   THEN a.attr_val END) AS tier
FROM dim_customer c
JOIN tbl_cx_attr a ON a.cust_id = c.id
WHERE a.attr_cd IN ('REGION', 'TIER')
GROUP BY c.id, c.cust_name
```

**typed_values variant:**
```sql
SELECT
  e.entity_id,
  MAX(CASE WHEN p.prop_key = 'score'  THEN p.val_num END) AS score,
  MAX(CASE WHEN p.prop_key = 'status' THEN p.val_str END) AS status,
  MAX(CASE WHEN p.prop_key = 'created' THEN p.val_dt END) AS created
FROM entity e
JOIN tbl_entity_props p ON p.entity_id = e.id
WHERE p.prop_key IN ('score', 'status', 'created')
GROUP BY e.entity_id
```

**multi_value variant:**
```sql
SELECT
  entity_id,
  attr_code,
  val_1 AS primary_value,
  val_2 AS secondary_value,
  unit
FROM tbl_measurements
WHERE attr_code = 'BLOOD_PRESSURE'
```

### 5.4 Performance Awareness

For large EAV tables (50M+ rows), the agent should:
1. Check `performance_hints` for index information
2. Always filter on the indexed `attribute_column` first
3. Use `explain_query` before executing if the table has `row_estimate > 10M`
4. Prefer conditional aggregation over self-joins when pivoting (fewer table scans)

### 5.5 Fallback

If pivot SQL is too complex, the agent falls back to raw EAV rows with an
explanation of the format.

---

## 6. Agent Workflow (Multi-Step Reasoning)

The agent uses an **iterative exploration loop**, not a single-shot approach:

```
User: "How many Gold-tier customers do we have per region?"

Agent Step 1: SEARCH
  → search_schema_knowledge("customer tier region count")
  → gets: tbl_cx_attr (EAV, single_value), dim_customer, coded values

Agent Step 2: EXPLORE
  → describe_table("dbo.tbl_cx_attr")
  → learns: EAV config, attr_cd values, 50M rows, index on (attr_cd, cust_id)

Agent Step 3: SAMPLE (optional, if unsure about data)
  → get_sample_rows("tbl_cx_attr", where="attr_cd = 'TIER'", limit=5)
  → sees actual values: 'Gold', 'Silver', 'Bronze'

Agent Step 4: GENERATE SQL
  → writes conditional aggregation query with proper filters

Agent Step 5: VALIDATE
  → validate_sql(sql)
  → valid: true

Agent Step 6: EXPLAIN (because row_estimate > 10M)
  → explain_query(sql)
  → plan looks good: Index Scan on (attr_cd, cust_id)

Agent Step 7: EXECUTE
  → execute_query(sql)
  → returns region counts

Agent Step 8: PRESENT
  → formats result as table, explains the answer
```

---

## 7. Schema Crawler (Auto-Extract CLI)

A CLI tool to bootstrap knowledge YAML from a live database. The data team
then enriches the auto-generated YAML with human descriptions, coded value
meanings, and query patterns.

### Usage

```bash
# Extract schema metadata into YAML
db-skill crawl \
  --engine postgresql \
  --host localhost \
  --database warehouse \
  --schema dbo \
  --output ./knowledge/auto_extracted.yaml

# Output: YAML file with table/column entries pre-filled with:
# - table names, column names, data types
# - row counts (from pg_stat_user_tables or COUNT estimate)
# - primary keys, foreign keys, indexes
# - empty description fields for humans to fill in
# - EAV detection heuristics (tables with few columns, one varchar value col)
```

### EAV Detection Heuristics

The crawler can flag likely-EAV tables based on:
- Low column count (3-6 columns)
- One column with high cardinality in a lookup-like pattern
- Column names matching common EAV patterns (key/value, attr/val, property/value)

These are flagged as `storage_format: eav_suspected` for human review.

---

## 8. Large Result Set Strategy

Given that users expect large data output (10k+ rows), the skill needs to
handle results that don't fit in a single LLM context window.

| Strategy | When |
|----------|------|
| **Inline (default)** | ≤ 1000 rows — return directly in tool response |
| **Paginated** | > 1000 rows — agent can request next page via offset |
| **Export to CSV** | Agent or user requests file export — write to local file, return path |
| **Summary + export** | Agent returns summary stats inline + exports full data to file |

The agent should default to **summary + export** for large results:
1. Execute with `row_limit=100` to show a preview
2. If `has_more=true`, offer to export full results to CSV
3. Provide summary statistics (row count, key aggregates) inline

---

## 9. Configuration

```yaml
# config.yaml
skill:
  name: database-query
  version: "0.1.0"

knowledge:
  provider: static_asset          # "static_asset" | "vector_db"

  static_asset:
    path: ./knowledge/            # directory of YAML files

  # Vector DB settings (future upgrade path)
  vector_db:
    engine: chromadb
    collection: schema_knowledge
    embedding_model: text-embedding-3-small
    host: localhost
    port: 8000

database:
  engine: postgresql              # postgresql | sqlite (more adapters later)
  host: localhost
  port: 5432
  database: warehouse
  user: readonly_agent
  password_env: DB_PASSWORD       # read from environment variable
  connect_timeout: 10
  query_timeout: 60               # higher default for large tables
  pool_size: 5
  use_server_cursor: true         # stream large results

guardrails:
  max_row_limit: 100000
  default_row_limit: 1000
  allowed_statements: [SELECT]
  blocked_tables: []
  allowed_tables: []
  max_query_length: 10000         # characters
  warn_seq_scan_above: 10000000   # warn if EXPLAIN shows seq scan on table > 10M rows

export:
  directory: ./exports/           # where CSV exports are written
  max_export_rows: 1000000        # cap on CSV export size
```

---

## 10. Security Considerations

| Concern | Mitigation |
|---------|------------|
| SQL injection | Parse and validate SQL AST via `sqlglot` before execution; use parameterized queries for bind params |
| Data exfiltration | Row limits, query timeout, optional table allowlist |
| Privilege escalation | DB user has read-only grants only; no `GRANT`, `CREATE`, `EXECUTE` |
| Prompt injection via data | Sanitize query results before returning to LLM context; truncate long cell values |
| Credential leakage | Passwords via env vars only; never logged or returned to user |
| Large result DoS | Export row cap; server-side cursors prevent OOM |

---

## 11. Tool Registration Format

The skill exports tools in a **framework-agnostic JSON schema** format.
Thin adapters translate this to framework-specific formats.

```python
# Usage — framework agnostic
from db_skill import DatabaseQuerySkill

skill = DatabaseQuerySkill.from_config("config.yaml")
tools = skill.get_tools()          # list of tool definitions (JSON schema)
result = skill.call("execute_query", {"sql": "SELECT 1"})

# Usage — Claude tool_use
claude_tools = skill.get_tools(format="claude")   # Claude tool_use format

# Usage — OpenAI function calling
openai_tools = skill.get_tools(format="openai")   # OpenAI function format
```

---

## 12. Non-Goals (v0.1)

- **Write operations** — this skill is read-only
- **Cross-database joins** — single database per skill instance
- **Visualization / charting** — returns data; presentation is the caller's job
- **Query caching** — not in initial scope
- **Real-time streaming to UI** — export to file instead
