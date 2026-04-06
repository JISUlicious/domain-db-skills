# Database Query Agent Skill — Implementation Plan

## Design Decisions (from clarification)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Knowledge backend | Static YAML first | ~3 tables now; simple; version-controlled; vector DB is a future upgrade |
| Database engine | PostgreSQL primary, SQLite for tests | Postgres is the production DB; SQLite enables zero-dependency testing |
| EAV handling | Support multiple EAV variants | Various EAV patterns in use (single_value, typed_values, multi_value) |
| Agent reasoning | Multi-step (explore → draft → validate → execute) | Messy schemas need iterative discovery, not single-shot SQL generation |
| Result handling | Pagination + CSV export | Users expect large outputs (10k+ rows); can't fit in LLM context |
| Framework target | Framework-agnostic with format adapters | Primary: Claude tool_use. Also supports OpenAI, LangChain formats |
| Packaging | Simple Python library (`pip install`) | Download and use; no server deployment |
| Schema bootstrap | CLI crawler from INFORMATION_SCHEMA | Data team enriches auto-extracted YAML with descriptions |
| Knowledge enrichment | Agent-assisted interactive mode | User converses with agent to add descriptions, aliases, abbreviation patterns |
| Naming resolution | Aliases + abbreviation registry | Column names are non-descriptive; multiple teams use different names |
| SQL execution | Yes, read-only | Skill both generates and executes SQL |

---

## Phase 0: Project Scaffolding ✅

Already completed. Project structure, pyproject.toml, Makefile in place.

---

## Phase 1: Data Models & Knowledge Layer

**Goal:** Define all data models and build the static YAML knowledge provider.

### Directory Structure

```
src/db_skill/knowledge/
├── __init__.py
├── models.py           # All Pydantic models
├── provider.py         # KnowledgeProvider ABC
└── static_provider.py  # Static YAML implementation
```

### Step 1.1 — Pydantic Models (`knowledge/models.py`)

- [ ] **EAV configuration models:**
  - `ValueColumn` — name, type, description
  - `EAVConfig` — variant (single_value | typed_values | multi_value), entity_column, attribute_column, value_columns[], type_discriminator?
- [ ] **Alias and naming models:**
  - `ContextualAlias` — name, contexts[] (e.g. ["finance", "operations", "all"])
  - `AbbreviationPattern` — abbreviation, full_name
  - `NamingConventionEntry` — database, scope, patterns: list[AbbreviationPattern], description
- [ ] **Knowledge entry models:**
  - `KnowledgeEntry` (base) — object_type, database, schema, table, description, tags, aliases[]?, abbreviation_note?
  - `TableEntry` — storage_format, eav_config?, row_estimate?, partition_key?, naming_convention?
  - `ColumnEntry` — column, data_type, display_name, coded_values?, aliases[]? (str or ContextualAlias)
  - `RelationshipEntry` — from_table, from_column, to_table, to_column, join_type
  - `QueryPatternEntry` — sql_template, tags
  - `PerformanceHintEntry` — table, indexes[], description
  - `NamingConventionEntry` — schema-level abbreviation registry
- [ ] **Enrichment tracking models:**
  - `EnrichmentStatus` — enum: complete | partial | pending
  - Fields on entries: enrichment_status?, enrichment_date?, enrichment_notes?
- [ ] **Output models (returned by tools):**
  - `TableSummary` — name, display_name, description, storage_format, row_estimate, aliases
  - `TableDescription` — full metadata for describe_table output (including aliases, naming_convention)
  - `ColumnDetail` — name, display_name, data_type, description, coded_values, aliases
- [ ] **Query result models:**
  - `QueryResult` — columns, rows, row_count, has_more, execution_time_ms
  - `ExportResult` — file_path, row_count, execution_time_ms
  - `ValidationResult` — valid, errors, warnings, tables_referenced
  - `ExplainResult` — plan, warnings

### Step 1.2 — Provider Interface (`knowledge/provider.py`)

- [ ] Define `KnowledgeProvider` ABC:
  ```python
  class KnowledgeProvider(ABC):
      def load(self) -> None: ...
      def search(self, query: str, filters: dict | None, top_k: int = 10) -> list[KnowledgeEntry]: ...
      def list_tables(self, database: str | None, schema: str | None, name_pattern: str | None) -> list[TableSummary]: ...
      def describe_table(self, qualified_name: str) -> TableDescription: ...
  ```

### Step 1.3 — Static YAML Provider (`knowledge/static_provider.py`)

- [ ] Load all YAML files from configured directory
- [ ] Parse into typed KnowledgeEntry subclasses via discriminated union
- [ ] Build in-memory indexes:
  - By object_type (for fast list_tables)
  - By qualified table name (for fast describe_table)
  - By tags (for search filtering)
  - **By alias** — inverted index from alias → entry (for name resolution)
  - **Abbreviation registry** — loaded from NamingConventionEntry entries
- [ ] Implement `search()` with **alias-aware keyword matching**:
  - Tokenize query
  - Expand user terms via abbreviation registry (e.g. "revenue" → also matches "rev")
  - Score entries by keyword overlap in: aliases > display_name > description > tags
    (aliases weighted highest since they represent confirmed user terminology)
  - Match against contextual aliases if user context is provided in filters
  - Filter by metadata if filters provided
  - Return top_k ranked results
- [ ] Implement `list_tables()` — direct index lookup with optional filters
- [ ] Implement `describe_table()` — aggregate table + columns + relationships + hints + patterns

### Step 1.4 — Tests

- [ ] Test model parsing from YAML (all entry types, all EAV variants, aliases)
- [ ] Test static provider search (keyword matching, filtering, ranking)
- [ ] **Test alias resolution** — search("revenue") matches column `rev_amt` via alias
- [ ] **Test abbreviation expansion** — search("order date") matches `ord_dt` via abbreviation registry
- [ ] **Test contextual aliases** — "sales" matches `rev_amt` when context=finance
- [ ] Test describe_table assembles full metadata correctly (including aliases)
- [ ] Edge cases: missing optional fields, unknown object_type, empty knowledge base, no aliases

---

## Phase 2: SQL Validation & Guardrails

**Goal:** Parse, validate, and enforce safety rules on generated SQL.

### Directory Structure

```
src/db_skill/validation/
├── __init__.py
├── validator.py        # SQL parsing + statement type checking
└── guardrails.py       # Limit injection, cost warnings, audit
```

### Step 2.1 — SQL Validator (`validation/validator.py`)

- [ ] Integrate `sqlglot` for parsing (set dialect from config: postgres default)
- [ ] `validate(sql, dialect) -> ValidationResult`:
  - Parse SQL into AST
  - Reject if not a SELECT (including inside CTEs, subqueries)
  - Extract referenced table names → check against allow/deny lists
  - Detect dangerous patterns: `INTO OUTFILE`, `LOAD DATA`, `pg_sleep`, etc.
  - Return `{valid, errors, warnings, tables_referenced}`

### Step 2.2 — Guardrails (`validation/guardrails.py`)

- [ ] `apply_guardrails(sql, config) -> tuple[str, list[str]]`:
  - If no LIMIT present, inject `LIMIT {default_row_limit}` (dialect-aware)
  - If LIMIT exceeds max_row_limit, cap it and add warning
  - Check query length against max_query_length
  - Return (modified_sql, warnings)
- [ ] Dialect awareness: PostgreSQL uses `LIMIT`, SQL Server uses `TOP`

### Step 2.3 — Tests

- [ ] Valid SELECT queries pass
- [ ] INSERT/UPDATE/DELETE/DROP rejected
- [ ] CTEs with SELECT pass, CTEs with INSERT rejected
- [ ] UNION queries handled
- [ ] LIMIT injection works when missing
- [ ] LIMIT capping works when too high
- [ ] Table allowlist/denylist enforcement
- [ ] Dangerous patterns caught (pg_sleep, INTO OUTFILE, etc.)

---

## Phase 3: Database Executor

**Goal:** Execute validated SQL safely with pagination and export.

### Directory Structure

```
src/db_skill/database/
├── __init__.py
├── executor.py         # DatabaseExecutor ABC + factory
├── result.py           # QueryResult, ExportResult (or reuse from models.py)
└── adapters/
    ├── __init__.py
    ├── postgresql.py   # Primary: psycopg-based
    └── sqlite.py       # For testing
```

### Step 3.1 — Executor Interface (`database/executor.py`)

- [ ] `DatabaseExecutor` ABC:
  ```python
  class DatabaseExecutor(ABC):
      async def connect(self) -> None: ...
      async def disconnect(self) -> None: ...
      async def execute(self, sql: str, params: dict | None,
                        row_limit: int, offset: int) -> QueryResult: ...
      async def explain(self, sql: str) -> ExplainResult: ...
      async def export_csv(self, sql: str, params: dict | None,
                           file_path: str, max_rows: int) -> ExportResult: ...
      async def sample_rows(self, table: str, where: str | None,
                            limit: int) -> QueryResult: ...
  ```
- [ ] Factory function: `create_executor(config) -> DatabaseExecutor`

### Step 3.2 — PostgreSQL Adapter (`database/adapters/postgresql.py`)

- [ ] Connection using `psycopg` (sync) or `psycopg` async
- [ ] Connection pooling via `psycopg_pool`
- [ ] Read-only enforcement: `SET default_transaction_read_only = on`
- [ ] Query timeout: `SET statement_timeout = {ms}`
- [ ] `execute()`: run query, fetch rows up to limit, report has_more
- [ ] `explain()`: run `EXPLAIN (FORMAT TEXT)` — NOT `EXPLAIN ANALYZE`
- [ ] `export_csv()`: use server-side cursor, stream rows to CSV writer
- [ ] `sample_rows()`: `SELECT * FROM {table} WHERE {where} LIMIT {limit}`
  - Table name validated against knowledge layer to prevent injection

### Step 3.3 — SQLite Adapter (`database/adapters/sqlite.py`)

- [ ] Lightweight adapter using Python's built-in `sqlite3`
- [ ] Same interface as PostgreSQL adapter
- [ ] Primary use: integration testing without a DB server

### Step 3.4 — Tests

- [ ] Test execute with SQLite (inline results, pagination via offset)
- [ ] Test explain output parsing
- [ ] Test CSV export (write to temp file, verify contents)
- [ ] Test timeout enforcement
- [ ] Test read-only enforcement (reject INSERT etc. at DB level)
- [ ] Test sample_rows

---

## Phase 4: Configuration & Skill Entry Point

**Goal:** Wire everything together into a single `DatabaseQuerySkill` class.

### Step 4.1 — Configuration (`config.py`)

- [ ] Pydantic settings model:
  ```python
  class SkillConfig(BaseModel):
      knowledge: KnowledgeConfig
      database: DatabaseConfig
      guardrails: GuardrailsConfig
      export: ExportConfig
  ```
- [ ] Load from YAML file path
- [ ] Override individual fields from environment variables
- [ ] Validate on construction

### Step 4.2 — Skill Class (`skill.py`)

- [ ] `DatabaseQuerySkill` — main entry point:
  ```python
  class DatabaseQuerySkill:
      @classmethod
      def from_config(cls, config_path: str) -> "DatabaseQuerySkill": ...

      # 7 tools matching the spec
      def search_schema_knowledge(self, query, filters, top_k) -> dict: ...
      def list_tables(self, database, schema, name_pattern) -> dict: ...
      def describe_table(self, table) -> dict: ...
      def validate_sql(self, sql, dialect) -> dict: ...
      def explain_query(self, sql) -> dict: ...
      def execute_query(self, sql, params, row_limit, offset, export_format) -> dict: ...
      def get_sample_rows(self, table, where, limit) -> dict: ...

      # Tool registration
      def get_tools(self, format="generic") -> list[dict]: ...
      def call(self, tool_name: str, arguments: dict) -> dict: ...
  ```
- [ ] Each method returns JSON-serializable dict
- [ ] `call()` dispatches by tool name (for agent framework integration)

### Step 4.3 — Tool Registration & Format Adapters

- [ ] Generic format: JSON Schema for input, description string
- [ ] Claude format adapter: `{"name", "description", "input_schema"}`
- [ ] OpenAI format adapter: `{"type": "function", "function": {"name", "description", "parameters"}}`
- [ ] Each tool includes a detailed description telling the agent **when and why** to use it

### Step 4.4 — Tests

- [ ] Test from_config loads correctly
- [ ] Test each tool method independently
- [ ] End-to-end: search → describe → validate → execute against SQLite
- [ ] Test get_tools returns valid schemas for each format

---

## Phase 5: Schema Crawler & Interactive Enrichment

**Goal:** Auto-extract schema metadata and provide agent-assisted enrichment
for building rich descriptions, aliases, and abbreviation registries.

### Directory Structure

```
src/db_skill/
├── ...
└── crawler/
    ├── __init__.py
    ├── cli.py              # CLI entry points (crawl + enrich)
    ├── extractor.py        # INFORMATION_SCHEMA queries
    ├── eav_detector.py     # Heuristic EAV detection
    ├── enrichment.py       # Interactive enrichment engine
    └── abbreviations.py    # Abbreviation pattern detection
```

### Step 5.1 — Schema Extractor (`crawler/extractor.py`)

- [ ] Query `information_schema.tables` → table names, row estimates
- [ ] Query `information_schema.columns` → column names, types, nullability
- [ ] Query `information_schema.table_constraints` + `key_column_usage` → PKs, FKs
- [ ] Query `pg_indexes` or `information_schema` equivalent → index info
- [ ] For PostgreSQL: `pg_stat_user_tables` for row count estimates
- [ ] Output: list of TableEntry + ColumnEntry + RelationshipEntry with empty descriptions
- [ ] Set `enrichment_status: pending` on all generated entries

### Step 5.2 — EAV Detector (`crawler/eav_detector.py`)

- [ ] Heuristic rules:
  - Table has 3-6 columns
  - One column is varchar with name matching `*val*`, `*value*`
  - One column is varchar with name matching `*key*`, `*attr*`, `*code*`, `*type*`, `*prop*`
  - One column looks like an FK (integer/bigint, name ending in `_id`)
- [ ] Flag matching tables as `storage_format: eav_suspected`
- [ ] Best-guess `eav_config` based on column name heuristics

### Step 5.3 — Abbreviation Detector (`crawler/abbreviations.py`)

- [ ] Analyze column names across all tables to detect naming patterns:
  - Split column names by `_` delimiter
  - Count frequency of each token (e.g. `amt` appears 12 times, `dt` appears 8 times)
  - Flag high-frequency short tokens (≤4 chars) as likely abbreviations
  - Group columns sharing the same tokens to suggest patterns
- [ ] Generate a draft `naming_convention` entry with:
  - Detected abbreviations and guessed full names (based on common conventions)
  - Confidence score per abbreviation
- [ ] Known abbreviation dictionary (seed):
  - Common DB abbreviations: amt→amount, qty→quantity, dt→date, cd→code, etc.
  - Common prefix conventions: dim_→dimension, fact_→fact, tbl_→table, stg_→staging

### Step 5.4 — Interactive Enrichment Engine (`crawler/enrichment.py`)

The enrichment engine drives a structured conversation between the agent and
the user to build complete knowledge entries. It produces **enrichment prompts**
that the agent presents to the user, and processes user responses into YAML
updates.

- [ ] `EnrichmentSession` class:
  ```python
  class EnrichmentSession:
      def __init__(self, knowledge_path: str, db_executor: DatabaseExecutor | None): ...
      def get_next_item(self) -> EnrichmentPrompt | None: ...
      def apply_response(self, response: EnrichmentResponse) -> None: ...
      def get_progress(self) -> EnrichmentProgress: ...
      def save(self) -> None: ...
  ```

- [ ] `EnrichmentPrompt` — what the agent shows the user:
  ```python
  class EnrichmentPrompt:
      entry: KnowledgeEntry           # the item being enriched
      prompt_type: str                # "describe_table" | "describe_column" | "collect_aliases" |
                                      # "confirm_abbreviations" | "identify_coded_values" |
                                      # "confirm_eav"
      suggested_description: str | None   # agent's best guess
      suggested_aliases: list[str] | None # based on naming patterns
      sample_data: list[list] | None      # actual rows from DB
      context: str                        # what to ask the user
  ```

- [ ] **Enrichment workflow** (ordered by priority):
  1. **Abbreviation confirmation** — show detected abbreviation patterns,
     ask user to confirm/correct/add. This unlocks alias suggestions for all
     subsequent items.
  2. **Table descriptions** — for each table with `enrichment_status: pending`:
     - Show table name, column list, row estimate, sample rows
     - Suggest description based on column names and detected patterns
     - Ask user for description + what people call this table (aliases)
  3. **Column descriptions** — for each column with non-descriptive name:
     - Show column name, data type, sample values
     - Suggest description based on abbreviation patterns
     - Ask: "What does this column mean? What do people call it?"
     - Collect aliases (including team-specific aliases with context)
  4. **Coded value identification** — for varchar columns with low cardinality:
     - Sample distinct values
     - Ask user what each coded value means
  5. **EAV confirmation** — for tables flagged as `eav_suspected`:
     - Show the table structure and sample rows
     - Ask user to confirm/correct the EAV config
  6. **Relationship validation** — for auto-detected FK relationships:
     - Ask user to confirm join logic

- [ ] **Enrichment progress tracking:**
  ```python
  class EnrichmentProgress:
      total_items: int
      completed: int
      partial: int
      pending: int
      by_type: dict[str, int]   # e.g. {"table": 3, "column": 12, "coded_values": 5}
  ```

- [ ] Write updated YAML after each response (incremental save)
- [ ] Support resuming interrupted sessions (read enrichment_status from YAML)

### Step 5.5 — CLI (`crawler/cli.py`)

- [ ] `db-skill crawl` command (auto-extract):
  - `--engine`, `--host`, `--port`, `--database`, `--schema`, `--user`
  - `--output` path for YAML
  - `--detect-eav` flag (default on)
  - `--detect-abbreviations` flag (default on)
  - Output well-formatted YAML with `enrichment_status: pending` markers
- [ ] `db-skill enrich` command (interactive enrichment):
  - `--knowledge` path to existing YAML (from crawl or manual)
  - `--output` path for enriched YAML (default: overwrite in-place)
  - `--db-*` optional DB connection args (for sampling data during enrichment)
  - `--resume` flag to continue from where the user left off
  - `--skip-confirmed` flag to skip items already marked `complete`
  - Outputs enrichment prompts as structured JSON (for agent consumption)
- [ ] `db-skill enrich-status` command:
  - Shows enrichment progress summary per table
- [ ] Register all commands as console_script entry points

### Step 5.6 — Tests

- [ ] Test extractor against SQLite with test schema
- [ ] Test EAV detector identifies known EAV patterns
- [ ] Test abbreviation detector finds common patterns (amt, dt, cd, qty)
- [ ] Test enrichment session lifecycle (get_next → apply → save → resume)
- [ ] Test enrichment produces valid YAML with aliases and descriptions
- [ ] Test YAML output is valid and re-parseable by StaticKnowledgeProvider

---

## Phase 6: Integration Testing & Documentation

### Step 6.1 — Integration Test Suite

- [ ] SQLite-based fixture DB with:
  - 1 wide dimension table (dim_customer)
  - 1 EAV single_value table (tbl_cx_attr)
  - 1 EAV typed_values table (tbl_entity_props)
  - 1 wide fact table (fact_orders)
  - Realistic sample data (1000+ rows)
- [ ] Knowledge YAML matching the fixture DB
- [ ] Test scenarios:
  - Simple SELECT on wide table
  - EAV single_value pivot query
  - EAV typed_values pivot query
  - Join across wide + EAV tables
  - Pagination (offset-based)
  - CSV export
  - Guardrail enforcement (no INSERT, LIMIT injection)
  - **Alias resolution** — user says "revenue", finds `rev_amt`
  - **Abbreviation expansion** — user says "order date", finds `ord_dt`
  - Schema crawler against the fixture DB
  - Enrichment session round-trip (crawl → enrich → use)

### Step 6.2 — Documentation

- [ ] `README.md`:
  - Quick start (install, configure, run)
  - Tool descriptions and example usage
  - How to write knowledge YAML
  - How to use the schema crawler + enrichment workflow
  - How to manage aliases and abbreviation patterns
  - Framework integration examples (Claude, OpenAI)
- [ ] Inline docstrings on all public classes and methods
- [ ] `knowledge/README.md` — guide for data teams writing knowledge entries, aliases, and conventions

---

## Phase Summary

| Phase | What | Depends On | Status |
|-------|------|------------|--------|
| **0: Scaffolding** | Project structure | — | ✅ Done |
| **1: Models & Knowledge** | Pydantic models, static YAML provider | Phase 0 | ⬜ Next |
| **2: SQL Validation** | sqlglot parsing, guardrails | Phase 0 | ⬜ Parallel with 1 |
| **3: Database Executor** | PostgreSQL + SQLite adapters, pagination, export | Phase 0 | ⬜ Parallel with 1,2 |
| **4: Skill Entry Point** | Wire together, tool registration, format adapters | Phases 1-3 | ⬜ |
| **5: Crawler & Enrichment** | CLI crawl + interactive agent-assisted enrichment | Phase 1 (models) | ⬜ |
| **6: Integration & Docs** | End-to-end tests, README | Phases 4-5 | ⬜ |

Phases 1, 2, and 3 are independent and should be worked in parallel.
Phase 4 integrates them. Phase 5 can start once models are defined.
Phase 6 finalizes.

---

## Key Dependencies (Python Packages)

| Package | Purpose | Required? |
|---------|---------|-----------|
| `pydantic>=2.0` | Data models, config validation | Yes |
| `pyyaml>=6.0` | YAML config and knowledge files | Yes |
| `sqlglot>=20.0` | SQL parsing, validation, dialect support | Yes |
| `psycopg[binary]>=3.1` | PostgreSQL adapter | Optional (postgresql extra) |
| `psycopg_pool>=3.1` | PostgreSQL connection pooling | Optional (postgresql extra) |
| `chromadb>=0.4` | Vector DB knowledge provider | Optional (vector extra) |
| `pytest>=8.0` | Testing | Dev |
| `ruff>=0.4` | Linting and formatting | Dev |
| `mypy>=1.10` | Type checking | Dev |

---

## Resolved Questions

| Question | Resolution |
|----------|------------|
| Agent framework target | Framework-agnostic with format adapters (Claude primary) |
| Multi-database | One instance per database |
| Knowledge curation | Auto-extract CLI + agent-assisted interactive enrichment |
| Naming ambiguity | Aliases on every entry + schema-level abbreviation registry + contextual aliases |
| Result formatting | Raw data to agent; agent formats. Skill provides CSV export for large results. |
| Embedding model | Deferred — not needed until vector DB upgrade |
