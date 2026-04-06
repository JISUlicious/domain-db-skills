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
| **Naming chaos** | Column names are non-descriptive, abbreviated, and called different things by different teams |

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
| G7 | Agent-assisted knowledge enrichment | Interactive mode where agent helps build descriptions, aliases, and coded-value mappings |
| G8 | Resolve naming ambiguity | Match user terms (aliases, abbreviations, domain jargon) to actual column/table names |
| G9 | Framework-agnostic | Works with Claude tool_use, but adaptable to any agent framework |

---

## 3. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Agent (LLM)                                 │
│                                                                      │
│  QUERY MODE (runtime)              ENRICH MODE (setup/maintenance)   │
│  1. Receive user question          1. Review table/column from crawl │
│  2. search_schema_knowledge        2. Sample data, suggest desc      │
│  3. describe_table (+ aliases)     3. Ask user for aliases           │
│  4. Generate SQL                   4. Detect abbreviation patterns   │
│  5. validate_sql                   5. Collect coded value meanings   │
│  6. explain_query                  6. Write enriched YAML            │
│  7. execute_query                                                    │
│  8. Format & return results                                          │
└───────┬──────────────┬──────────────┬────────────────┬───────────────┘
        │              │              │                │
        ▼              ▼              ▼                ▼
┌───────────────┐ ┌──────────┐ ┌──────────────┐ ┌────────────────────┐
│Knowledge Layer│ │   SQL    │ │  Database    │ │  Schema Crawler    │
│(Static YAML,  │ │Validator │ │  Executor    │ │  + Enrichment CLI  │
│ aliases,      │ │& Guard-  │ │  (read-only, │ │                    │
│ abbreviations,│ │ rails    │ │   pooled)    │ │  db-skill crawl    │
│ conventions)  │ │          │ │              │ │  db-skill enrich   │
└───────────────┘ └──────────┘ └──────────────┘ └────────────────────┘
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
# Example: a regular wide table with aliases
- object_type: table
  database: warehouse
  schema: dbo
  table: dim_customer
  display_name: Customer Dimension
  description: >
    Master customer table. One row per customer.
  aliases:                              # ← NEW: what people call this table
    - customer table
    - customers
    - customer master
    - customer dim
    - cust table
  naming_convention: >                  # ← NEW: explain the naming pattern
    "dim_" prefix = dimension table. "customer" is the entity.
  storage_format: wide
  row_estimate: 5_000_000
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

# Example: a column description with aliases and coded values
- object_type: column
  database: warehouse
  schema: dbo
  table: tbl_cx_attr
  column: attr_cd
  display_name: Attribute Code
  description: >
    Coded attribute name.
  data_type: varchar(50)
  aliases:                              # ← NEW: all the ways people refer to this
    - attribute code
    - attribute type
    - attr type
    - attribute key
  coded_values:
    REGION: "Geographic sales region (East, West, Central, South)"
    TIER: "Customer tier (Gold, Silver, Bronze)"
    ACQ_DT: "Acquisition date (YYYY-MM-DD stored as string)"
    LTV: "Lifetime value in USD (numeric stored as string)"

# Example: a column with non-descriptive name and many aliases
- object_type: column
  database: warehouse
  schema: dbo
  table: fact_orders
  column: rev_amt
  display_name: Revenue Amount
  description: >
    Revenue in USD for this order line. Net of discounts.
  data_type: decimal(12,2)
  aliases:                              # different teams call this different things
    - revenue
    - sales amount
    - order amount
    - order revenue
    - rev
    - amount
    - sales
  abbreviation_note: >                  # ← NEW: explain the naming convention
    "rev" = revenue, "amt" = amount. This DB uses the pattern
    {metric_abbreviation}_{unit_abbreviation} for numeric columns.

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

### Interactive Enrichment Mode

Beyond basic extraction, the crawler supports an **agent-assisted enrichment
mode** where a user has a conversation with the agent to build richer metadata.

```bash
# Launch interactive enrichment session
db-skill enrich \
  --knowledge ./knowledge/auto_extracted.yaml \
  --output ./knowledge/enriched.yaml
```

In this mode, the agent:

1. **Reviews each table/column** and asks the user clarifying questions
2. **Suggests descriptions** based on column name patterns, data types, and
   sample values
3. **Collects aliases** — prompts the user for all the names people use
4. **Identifies coded values** — samples distinct values from columns and asks
   the user what they mean
5. **Detects naming conventions** — finds patterns like `rev_amt`, `ord_dt` and
   asks the user to confirm the abbreviation scheme

#### Example Enrichment Conversation

```
Agent: I found column `rev_amt` in `fact_orders` (decimal(12,2)).
       Sample values: 149.99, 2500.00, 89.50
       This looks like a monetary amount. What does this column represent?

User:  That's the revenue amount for each order line, in USD.

Agent: Got it. What do people usually call this column? I'll record all the
       names so I can match them later.

User:  Revenue, sales amount, order amount. Some people just say "rev" or
       "amount". The finance team calls it "order revenue".

Agent: I notice a naming pattern: "rev" = revenue, "amt" = amount.
       Do other columns follow this {abbreviation}_{unit} pattern?

User:  Yes, we also have qty for quantity, dt for date, cd for code.

Agent: I'll apply that pattern across all columns. Let me show you what
       I generated for the other columns...
```

#### Enrichment Output

The enrichment session produces updated YAML with:
- `aliases[]` filled in from user responses
- `abbreviation_note` explaining naming conventions
- `description` written by the agent, confirmed by the user
- `coded_values` populated from sample data + user explanations
- `enrichment_status: complete | partial | pending` for tracking progress

#### Enrichment Tracking

```yaml
# Auto-generated enrichment status per entry
- object_type: column
  table: fact_orders
  column: rev_amt
  enrichment_status: complete       # agent + user reviewed this
  enrichment_date: "2026-04-06"
  enrichment_notes: >
    Confirmed by data team. Revenue in USD, net of discounts.
    Abbreviation pattern: rev=revenue, amt=amount.
```

---

## 8. Naming Resolution & Alias Matching

A core challenge: users refer to the same column/table by many names. The
knowledge layer must resolve these ambiguities at query time.

### 8.1 Alias Fields in Knowledge Schema

Every table and column entry supports:

```yaml
aliases: list[str]           # all known names for this object
abbreviation_note: str       # explanation of abbreviation conventions
naming_convention: str       # pattern used in this database
```

The agent's `search_schema_knowledge` matches against:
- `table` / `column` (exact DB name)
- `display_name`
- All entries in `aliases[]`
- `description` text (keyword match)
- `tags`

### 8.2 Abbreviation Pattern Registry

Common abbreviation patterns are stored at the database or schema level:

```yaml
- object_type: naming_convention
  database: warehouse
  scope: schema              # applies to all tables in this schema
  patterns:
    - abbreviation: rev
      full_name: revenue
    - abbreviation: amt
      full_name: amount
    - abbreviation: qty
      full_name: quantity
    - abbreviation: dt
      full_name: date
    - abbreviation: cd
      full_name: code
    - abbreviation: cx
      full_name: customer
    - abbreviation: ord
      full_name: order
    - abbreviation: prod
      full_name: product
    - abbreviation: attr
      full_name: attribute
    - abbreviation: val
      full_name: value
    - abbreviation: num
      full_name: number / numeric
    - abbreviation: str
      full_name: string
    - abbreviation: tbl
      full_name: table (prefix)
    - abbreviation: dim
      full_name: dimension (prefix)
    - abbreviation: fact
      full_name: fact (prefix)
  description: >
    This database uses abbreviated column names in the format
    {concept_abbr}_{type_abbr}. Table prefixes indicate the table
    type: dim_ for dimensions, fact_ for fact tables, tbl_ for
    general/EAV tables.
```

### 8.3 Resolution Algorithm

When the user says "revenue" and the column is `rev_amt`:

1. **Direct match** — check column name, display_name, aliases
2. **Abbreviation expansion** — expand "revenue" → check against known
   abbreviations → match `rev` in `rev_amt`
3. **Fuzzy match** — token overlap between user term and all indexed terms
4. **Semantic match** — (future, with vector DB) embedding similarity

The resolver returns **ranked candidates** with confidence scores. If ambiguous,
the agent asks the user to clarify.

### 8.4 Context-Dependent Aliases

Different teams may use the same word for different things. The knowledge layer
supports **context tags** on aliases:

```yaml
- object_type: column
  table: fact_orders
  column: rev_amt
  aliases:
    - name: revenue
      contexts: [all]
    - name: sales
      contexts: [finance, executive]    # finance team says "sales"
    - name: order amount
      contexts: [operations]            # ops team says "order amount"
    - name: GMV
      contexts: [marketplace]           # marketplace team says "GMV"
```

When the user's context is known (e.g. "I'm from the finance team"), the
resolver prioritizes context-specific aliases.

---

## 9. Large Result Set Strategy (unchanged from prior revision)

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

## 10. Configuration

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

## 11. Security Considerations

| Concern | Mitigation |
|---------|------------|
| SQL injection | Parse and validate SQL AST via `sqlglot` before execution; use parameterized queries for bind params |
| Data exfiltration | Row limits, query timeout, optional table allowlist |
| Privilege escalation | DB user has read-only grants only; no `GRANT`, `CREATE`, `EXECUTE` |
| Prompt injection via data | Sanitize query results before returning to LLM context; truncate long cell values |
| Credential leakage | Passwords via env vars only; never logged or returned to user |
| Large result DoS | Export row cap; server-side cursors prevent OOM |

---

## 12. Tool Registration Format

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

## 13. Non-Goals (v0.1)

- **Write operations** — this skill is read-only
- **Cross-database joins** — single database per skill instance
- **Visualization / charting** — returns data; presentation is the caller's job
- **Query caching** — not in initial scope
- **Real-time streaming to UI** — export to file instead
