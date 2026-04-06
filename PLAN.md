# Database Query Agent Skill — Implementation Plan

## Phase 0: Project Scaffolding

**Goal:** Set up the repo, tooling, and project skeleton.

### Tasks

- [ ] Initialize Python project with `pyproject.toml` (Python 3.11+)
- [ ] Set up directory structure (see below)
- [ ] Add `.gitignore`, `README.md`
- [ ] Configure dev dependencies: `pytest`, `ruff`, `mypy`
- [ ] Add `Makefile` with `lint`, `test`, `format` targets

### Directory Structure

```
domain-db-skills/
├── pyproject.toml
├── Makefile
├── README.md
├── SPEC.md
├── PLAN.md
├── config.example.yaml
│
├── src/
│   └── db_skill/
│       ├── __init__.py
│       ├── skill.py                # Top-level skill entry point (tool definitions)
│       │
│       ├── knowledge/              # Knowledge layer
│       │   ├── __init__.py
│       │   ├── provider.py         # KnowledgeProvider ABC
│       │   ├── vector_provider.py  # Vector DB implementation
│       │   ├── static_provider.py  # Static YAML/JSON implementation
│       │   └── models.py           # KnowledgeEntry, TableSummary, etc.
│       │
│       ├── database/               # Database execution layer
│       │   ├── __init__.py
│       │   ├── executor.py         # DatabaseExecutor ABC + factory
│       │   ├── adapters/           # DB-specific adapters
│       │   │   ├── __init__.py
│       │   │   ├── postgresql.py
│       │   │   ├── mysql.py
│       │   │   ├── sqlserver.py
│       │   │   └── sqlite.py
│       │   └── result.py           # QueryResult model
│       │
│       ├── validation/             # SQL validation & guardrails
│       │   ├── __init__.py
│       │   ├── validator.py        # SQL validator
│       │   └── guardrails.py       # Guardrail rules (row limits, statement types, etc.)
│       │
│       └── config.py               # Configuration loading
│
├── knowledge/                      # Default knowledge asset directory
│   └── example_schema.yaml         # Example knowledge entries
│
└── tests/
    ├── conftest.py
    ├── test_knowledge/
    │   ├── test_static_provider.py
    │   └── test_vector_provider.py
    ├── test_database/
    │   ├── test_executor.py
    │   └── test_adapters.py
    ├── test_validation/
    │   ├── test_validator.py
    │   └── test_guardrails.py
    └── test_skill.py               # Integration tests for the full skill
```

---

## Phase 1: Knowledge Layer

**Goal:** Build the schema knowledge retrieval system.

### Step 1.1 — Data Models

- [ ] Define Pydantic models in `knowledge/models.py`:
  - `KnowledgeEntry` (base: object_type, database, schema, table, description, tags)
  - `TableEntry(KnowledgeEntry)` — adds storage_format, eav_config
  - `ColumnEntry(KnowledgeEntry)` — adds column, data_type, coded_values
  - `RelationshipEntry(KnowledgeEntry)` — adds from/to table/column, join_type
  - `QueryPatternEntry(KnowledgeEntry)` — adds sql_template
  - `EAVConfig` — entity_column, attribute_column, value_column
  - `TableSummary`, `ColumnDetail` (output models for `describe_table`)

### Step 1.2 — Provider Interface

- [ ] Define `KnowledgeProvider` ABC in `knowledge/provider.py`:
  - `search(query, filters, top_k) → list[KnowledgeEntry]`
  - `list_tables(database, schema, name_pattern) → list[TableSummary]`
  - `describe_table(qualified_name) → TableDescription`
  - `load()` — initialize/connect

### Step 1.3 — Static Asset Provider

- [ ] Implement `StaticKnowledgeProvider` in `knowledge/static_provider.py`:
  - Reads YAML files from a directory
  - Indexes entries in memory
  - `search()` uses simple keyword/tag matching (TF-IDF or fuzzy match)
  - `list_tables()` / `describe_table()` are direct lookups
- [ ] Write tests with fixture YAML files

### Step 1.4 — Vector DB Provider

- [ ] Implement `VectorKnowledgeProvider` in `knowledge/vector_provider.py`:
  - Connects to ChromaDB (default) via client library
  - `search()` → vector similarity search on description embeddings + metadata filters
  - `list_tables()` → metadata filter on object_type=table
  - `describe_table()` → metadata filter on specific table
  - Embedding generation via configurable model (OpenAI, local, etc.)
- [ ] Write tests with ChromaDB in-memory mode

### Deliverables
- Working knowledge retrieval from both YAML files and vector DB
- 100% test coverage on knowledge layer

---

## Phase 2: SQL Validation & Guardrails

**Goal:** Ensure generated SQL is safe before execution.

### Step 2.1 — SQL Parser Integration

- [ ] Integrate `sqlglot` for SQL parsing (supports multiple dialects)
- [ ] Implement `SQLValidator` in `validation/validator.py`:
  - Parse SQL string into AST
  - Check statement type is SELECT
  - Detect subqueries with non-SELECT statements
  - Check referenced tables against allow/deny lists
  - Check for dangerous patterns (e.g., `INTO OUTFILE`, `LOAD DATA`)

### Step 2.2 — Guardrails

- [ ] Implement `Guardrails` in `validation/guardrails.py`:
  - Inject `LIMIT` if missing (dialect-aware: `LIMIT` vs `TOP`)
  - Enforce max query length
  - Log all queries for audit
  - Return structured validation result: `{valid, errors, warnings}`

### Deliverables
- SQL validation catches all disallowed operations
- Automatic LIMIT injection works across PostgreSQL, MySQL, SQL Server, SQLite dialects
- Tests covering edge cases (CTEs, subqueries, UNION, etc.)

---

## Phase 3: Database Executor

**Goal:** Execute validated queries safely against real databases.

### Step 3.1 — Executor Interface

- [ ] Define `DatabaseExecutor` ABC in `database/executor.py`:
  - `connect()` / `disconnect()`
  - `execute(sql, params, row_limit, timeout) → QueryResult`
- [ ] Define `QueryResult` model: columns, rows, row_count, truncated, execution_time_ms

### Step 3.2 — Adapters

- [ ] **PostgreSQL adapter** (`asyncpg` or `psycopg`) — primary target
- [ ] **SQLite adapter** — for testing and lightweight use
- [ ] MySQL adapter (stretch)
- [ ] SQL Server adapter (stretch)

### Step 3.3 — Connection Management

- [ ] Connection pooling (async pool for PostgreSQL)
- [ ] Read-only enforcement at connection level (`SET TRANSACTION READ ONLY`)
- [ ] Query timeout enforcement at connection level
- [ ] Graceful error handling: connection failures, query errors, timeouts

### Deliverables
- Working query execution against PostgreSQL and SQLite
- Timeout and row-limit enforcement
- Connection pool with auto-reconnect

---

## Phase 4: Skill Entry Point & Tool Definitions

**Goal:** Wire everything together into a coherent agent skill.

### Step 4.1 — Configuration

- [ ] Implement `config.py` — load from YAML, environment variables, defaults
- [ ] Validate config on startup

### Step 4.2 — Skill Class

- [ ] Implement `DatabaseQuerySkill` in `skill.py`:
  - Constructor: accepts config, initializes knowledge provider + executor + validator
  - Exposes 5 tool methods matching the spec:
    1. `search_schema_knowledge(query, filters, top_k)`
    2. `list_tables(database, schema, name_pattern)`
    3. `describe_table(table)`
    4. `validate_sql(sql)`
    5. `execute_query(sql, params, row_limit)`
  - Each method returns structured JSON-serializable output

### Step 4.3 — Tool Registration

- [ ] Create tool descriptors (name, description, input schema, output schema) for each method
- [ ] Export a `get_tools()` function that returns the tool list for agent frameworks
- [ ] Ensure compatibility with common agent frameworks (Claude tool_use format, OpenAI function calling, LangChain tools)

### Deliverables
- Single `DatabaseQuerySkill` class that can be instantiated with config
- `get_tools()` returns agent-ready tool definitions
- End-to-end test: question → knowledge search → SQL generation → validation → execution → result

---

## Phase 5: Testing & Documentation

### Step 5.1 — Integration Tests

- [ ] Set up SQLite-based integration tests (no external DB required)
- [ ] Create test fixtures: sample EAV tables + knowledge entries
- [ ] Test full flow: knowledge lookup → SQL gen → validate → execute
- [ ] Test EAV pivot scenarios specifically

### Step 5.2 — Example Knowledge Base

- [ ] Write example knowledge YAML covering:
  - 2-3 regular (wide) tables
  - 1-2 EAV tables with coded values
  - Relationships between them
  - 3-5 query patterns
- [ ] This doubles as documentation and a template for users

### Step 5.3 — Documentation

- [ ] `README.md` — quick start, configuration, usage
- [ ] Inline docstrings on public interfaces
- [ ] `knowledge/README.md` — how to write knowledge entries

---

## Phase Summary & Priority Order

| Phase | Priority | Estimated Effort | Dependencies |
|-------|----------|-----------------|--------------|
| Phase 0: Scaffolding | P0 | Small | None |
| Phase 1: Knowledge Layer | P0 | Medium | Phase 0 |
| Phase 2: SQL Validation | P0 | Medium | Phase 0 |
| Phase 3: Database Executor | P0 | Medium | Phase 0 |
| Phase 4: Skill Entry Point | P0 | Medium | Phases 1-3 |
| Phase 5: Testing & Docs | P1 | Medium | Phase 4 |

Phases 1, 2, and 3 are independent and can be worked on in parallel.
Phase 4 integrates them. Phase 5 hardens the result.

---

## Key Dependencies (Python Packages)

| Package | Purpose |
|---------|---------|
| `pydantic` | Data models, config validation |
| `sqlglot` | SQL parsing, validation, dialect translation |
| `pyyaml` | YAML config and knowledge file loading |
| `chromadb` | Vector DB (optional, for vector provider) |
| `psycopg[binary]` | PostgreSQL adapter |
| `asyncpg` | Async PostgreSQL (alternative) |
| `pytest` | Testing |
| `ruff` | Linting and formatting |
| `mypy` | Type checking |

---

## Open Questions

1. **Agent framework target** — Should the skill be framework-agnostic, or target
   a specific framework (Claude tool_use, LangChain, etc.) first?
2. **Embedding model** — For vector DB provider, which embedding model? Options:
   OpenAI `text-embedding-3-small`, local model via `sentence-transformers`, or
   framework-provided embeddings.
3. **Multi-database** — Should a single skill instance support multiple databases,
   or one instance per database?
4. **Knowledge curation workflow** — Should we build a CLI tool for ingesting
   schema metadata into the knowledge layer, or is manual YAML sufficient for v0.1?
5. **Result formatting** — Should the skill return raw data and let the agent
   format it, or provide pre-formatted markdown tables?
