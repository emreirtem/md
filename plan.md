# COBOL Parser

## Project goal

Build a standalone, local-first tool that analyzes COBOL source projects and presents program-to-program calls and embedded DB2 table/column references in an interactive graph. The tool must preserve source locations, make analysis limitations visible, and keep its parser and analysis model independent from the UI and storage choices.

This project is a general, standalone COBOL source analysis utility. Its scope is limited to source parsing, dependency extraction, browsing, and export of analysis results.

## Product scope

### Initial capabilities

- Scan a directory of COBOL source files and optional COPY libraries.
- Support fixed and free source formats, with explicit configuration and best-effort detection.
- Identify programs and source locations.
- Extract static and dynamic `CALL` statements.
- Track whether a call is unconditional, conditional, unreachable in the supported control-flow subset, or unknown.
- Parse a defined subset of embedded DB2 SQL and extract table and column references.
- Provide a local web UI with search, filters, an interactive graph, and source-location details.
- Export and import a versioned, storage-independent JSON analysis snapshot.
- Persist/query results with Neo4j through a replaceable storage adapter. The parser and JSON workflows must still work without a running Neo4j instance.

### Explicit limitations

- This is not a complete COBOL compiler and does not promise complete language or dialect coverage.
- Dynamic calls and dynamic SQL may not be resolvable statically; preserve them as unresolved findings.
- Report analysis status and explanations. Never label a call definitely unreachable or guaranteed solely because the parser cannot understand its context.
- SQL column resolution must distinguish resolved, ambiguous, wildcard, and unsupported cases.

## Recommended technology

- Python 3.12 or newer for the analysis package and local API.
- `Lark` for a maintainable grammar-based parser for the supported COBOL/SQL subsets. Keep the grammar narrow and explicit; do not build the implementation around large regular expressions.
- Python standard library for source scanning, path handling, serialization, and core data structures.
- `Pydantic` for API and interchange-schema validation, with explicit schema versions.
- `FastAPI` for an optional local HTTP API.
- React + TypeScript for the local web UI.
- Cytoscape.js for graph rendering and interaction.
- Neo4j Python driver (`neo4j`) only in the Neo4j storage adapter; the core parser must not import or require it.
- `pytest` for unit and integration tests; `mypy` or `pyright` for type checking; `ruff` for linting and formatting.
- Package/dependency management with `uv` and `pyproject.toml`.

Choose compatible versions when implementation begins and record them in lockfiles. Keep optional components in extras (for example `web`, `neo4j`) so parser-only use does not install frontend/server/database dependencies.

## Architecture principles

1. **Core first:** Parsing, source models, findings, and analysis must run as a Python library without a web server or database.
2. **Explicit stages:** Keep file discovery, source-format normalization, lexing/parsing, semantic analysis, graph construction, persistence, and presentation separate.
3. **Provenance everywhere:** Each finding includes source file, original line/column span, and analyzer status. Preserve original locations through normalization and COPY expansion.
4. **Uncertainty is data:** Represent unresolved, ambiguous, conditional, unsupported, and unreachable findings explicitly with explanations.
5. **Replaceable boundaries:** Expose interfaces for source loading, graph storage, and export. Do not leak Neo4j or Cytoscape types into core models.
6. **Deterministic output:** Identical source and configuration should yield stable IDs and equivalent serialized output.
7. **Incremental extensibility:** Add dialect and statement support as isolated grammar/semantic modules, not scattered special cases.
8. **Safe local operation:** Analyze user-selected local source directories; do not execute source code or upload source automatically.

## Proposed repository structure

```text
cobol-parser/
├── plan.md
├── README.md
├── pyproject.toml
├── uv.lock
├── .gitignore
├── config/
│   └── example.toml
├── docs/
│   ├── architecture.md
│   ├── supported-cobol.md
│   ├── supported-db2-sql.md
│   └── analysis-statuses.md
├── examples/
│   ├── cobol/
│   └── expected-graph.json
├── src/
│   └── cobol_parser/
│       ├── __init__.py
│       ├── api.py                 # Stable public Python API
│       ├── config.py
│       ├── errors.py
│       ├── models/                # Version-independent internal data models
│       │   ├── source.py
│       │   ├── program.py
│       │   ├── call.py
│       │   ├── sql.py
│       │   └── graph.py
│       ├── input/                 # File discovery and source loading
│       │   ├── discovery.py
│       │   ├── encoding.py
│       │   └── copybooks.py
│       ├── source_format/         # Fixed/free normalization and source maps
│       │   ├── detect.py
│       │   ├── fixed.py
│       │   ├── free.py
│       │   └── source_map.py
│       ├── lexer/                 # Tokens with original source spans
│       ├── grammar/               # COBOL and embedded SQL grammar modules
│       │   ├── cobol.lark
│       │   └── db2_sql.lark
│       ├── parser/                # Parse-tree construction and recovery
│       │   ├── cobol.py
│       │   └── embedded_sql.py
│       ├── analysis/              # Semantic extraction and control flow
│       │   ├── programs.py
│       │   ├── calls.py
│       │   ├── control_flow.py
│       │   ├── sql_tables.py
│       │   ├── sql_columns.py
│       │   └── diagnostics.py
│       ├── graph/                 # Storage-independent graph and queries
│       │   ├── builder.py
│       │   ├── query.py
│       │   └── ids.py
│       ├── serialization/         # Versioned JSON import/export
│       │   ├── schema.py
│       │   └── json_io.py
│       ├── storage/               # Optional persistence adapters
│       │   ├── protocol.py
│       │   ├── memory.py
│       │   └── neo4j/
│       │       ├── store.py
│       │       └── schema.cypher
│       ├── cli/                   # Command-line interface
│       │   └── main.py
│       └── web/                   # API composition layer; not imported by core
│           ├── app.py
│           └── routes/
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── app/
│   │   ├── components/
│   │   ├── graph/
│   │   ├── search/
│   │   └── api/
│   └── ...
└── tests/
    ├── fixtures/
    ├── unit/
    ├── integration/
    └── golden/
```

The implementation may start with fewer files. Keep these boundaries even if some modules initially share a file; split modules as responsibilities grow. Avoid creating empty packages solely to match the tree. Every module included in the initial deliverable must have a real responsibility and working implementation.

## Analysis pipeline

```text
COBOL files + COPY libraries
    → file discovery and encoding detection
    → fixed/free normalization with original-source mapping
    → tokenization and parse with recoverable diagnostics
    → semantic extraction (programs, CALLs, control flow, embedded SQL)
    → storage-independent graph + diagnostics
    → JSON export and/or Neo4j storage adapter
    → local API and interactive UI
```

Each stage should consume and produce typed models. A parse error in one compilation unit should produce a diagnostic and allow other files to be analyzed where safe.

## Core data model

### Program

- Stable program identifier, program name, source path, source format, and optional dialect.
- Source span for the declaration and list of analysis diagnostics.

### Call site

- Caller program ID and resolved callee ID/name, if known.
- Literal or variable target classification.
- Original file, line, and column span.
- Reachability status: `reachable`, `unreachable`, or `unknown`.
- Invocation status: `unconditional`, `conditional`, or `unknown`.
- Condition summary and explanation, when available.
- Resolution status: `resolved`, `unresolved`, or `ambiguous`.

### SQL reference

- Program ID, DB2 schema/table, optional column, SQL operation, and original source span.
- Access kind: `read`, `write`, or `unknown`.
- Resolution status: `resolved`, `ambiguous`, `wildcard`, `dynamic`, or `unsupported`.
- SQL snippet or normalized statement reference, subject to configurable source-retention policy.

### Diagnostics

- Stable diagnostic code, severity, message, source span, and optional remediation/support note.
- Diagnostics are first-class output, not only log messages.

## Graph conventions

- Program nodes represent COBOL programs.
- Directed `CALLS` edges run from caller to callee and retain individual call-site records. Multiple call sites between the same programs must not lose their locations or statuses.
- `ACCESSES_TABLE` and `ACCESSES_COLUMN` edges connect programs to DB2 objects and retain SQL operation and source location.
- Preserve node and edge identifiers independently of Neo4j internal IDs.
- Neo4j labels, relationship types, indexes, and constraints belong only in the Neo4j adapter/migrations.
- The in-memory graph and JSON snapshot are the canonical interchange representation.

## User experience

- Local project selection and analysis configuration (source roots, COPY paths, source format/dialect overrides, include/exclude globs).
- Search program, table, schema, and column names.
- Interactive directed graph with pan, zoom, selection, neighborhood expansion, and status filters.
- Suggested call-edge colors: green for proven unconditional reachable calls, yellow for conditional calls, red for statically unreachable call sites, gray for unresolved/unknown. Always show a legend and text status.
- A selected node/edge detail panel shows source location, finding status, explanation, and diagnostics.
- Search results remain useful without rendering the full project graph; large graphs should allow scoped neighborhood views.

## CLI and public API

Provide a small, stable Python API and CLI before coupling the parser to the web app.

Example library shape:

```python
from cobol_parser import AnalysisConfig, analyze_project

result = analyze_project(AnalysisConfig(source_roots=["./src"], copybook_paths=["./copybooks"]))
```

Example CLI shape:

```text
cobol-parser analyze ./src --copybooks ./copybooks --format auto --output analysis.json
cobol-parser serve ./src --host 127.0.0.1 --port 8000
```

The public API should return a typed analysis result containing graph data, diagnostics, and run metadata. The CLI is a thin wrapper over the library API.

## Delivery phases

### Phase 1: Parser library and interchange model

- Define typed source, finding, diagnostic, and graph models.
- Implement directory discovery, source format handling, source maps, program identification, and literal/dynamic `CALL` extraction.
- Add versioned JSON output and a usable CLI.
- Document the initially supported COBOL syntax and known limitations.

### Phase 2: Control-flow statuses and embedded DB2 SQL

- Parse a documented subset of `IF`/`ELSE` and statement boundaries.
- Classify call sites as unconditional, conditional, unreachable, or unknown only where supported rules justify it.
- Parse supported `EXEC SQL ... END-EXEC` statements and extract tables/columns with explicit unresolved cases.
- Add COPY library handling while preserving original source locations.

### Phase 3: Local interactive web UI

- Add FastAPI endpoints that compose the library and return validated graph snapshots.
- Add React/TypeScript UI with search, filters, graph exploration, and source detail panels.
- Keep API response schemas versioned and UI-independent from internal Python class names.

### Phase 4: Neo4j adapter and query support

- Implement a Neo4j storage adapter behind a protocol and make it the normal persistent backend when configured.
- Add uniqueness constraints/indexes for program names/IDs, table schema+name, and column identity.
- Support bounded caller/callee traversal and export equivalent results independent of storage backend.
- Keep in-memory/JSON workflows available when Neo4j is not configured.

## Single-prompt implementation brief

The following section is intended to be given to a coding agent as one implementation prompt. It defines a bounded first release; do not broaden the project into a complete COBOL compiler.

> Implement this standalone COBOL source analysis project in the current repository, following this plan. First inspect the existing repository and preserve any user files. Deliver a runnable vertical slice, not only scaffolding or pseudocode.
>
> Build a Python 3.12+ package under `src/cobol_parser`, a local FastAPI backend, and a React + TypeScript + Vite frontend. Use Lark for a documented, intentionally limited COBOL/embedded-DB2-SQL grammar; use Pydantic for public schemas; Cytoscape.js for the interactive directed graph; and the official Neo4j Python driver behind a storage protocol. Provide a `docker-compose.yml` for local Neo4j and clear setup/run instructions. Keep parser-only analysis and JSON import/export usable without Neo4j.
>
> Implement an end-to-end workflow: the user enters/selects a local source directory and optional COPY paths, runs analysis, searches program/table/column names, inspects source-located findings, explores a scoped graph, and can export/import a versioned JSON snapshot. Bind the local server to `127.0.0.1` by default. Do not upload source or execute COBOL.
>
> Source handling must support fixed and free format, explicit format override, and a documented best-effort auto-detection. Preserve original file/line/column positions after normalization. Identify program IDs and extract literal `CALL "name"`/`CALL 'name'` and variable-target CALLs. Parse nested `IF`/`ELSE`/`END-IF` sufficiently to mark a call conditional. Mark a call unreachable only when a supported, unconditional control-flow terminator proves the call cannot execute; otherwise use `unknown`. Keep unresolved variable calls in results rather than dropping them. If full COPY expansion is too large for the first vertical slice, implement COPY discovery/resolution for configured directories and report unsupported replacement syntax clearly.
>
> Support a clearly documented DB2 SQL subset inside `EXEC SQL ... END-EXEC`: `SELECT` with one or more tables and joins, `INSERT`, `UPDATE`, and `DELETE`; extract table references and column references from projections, predicates, join conditions, and write clauses. Resolve qualified columns and aliases. For unqualified or wildcard references that cannot be resolved safely, emit explicit ambiguous/wildcard status rather than guessing. Treat dynamic SQL and unsupported statements as diagnostics/findings.
>
> Use storage-independent typed models and stable IDs. Model programs, DB2 tables, and columns as nodes; model individual call sites and access findings as directed relationships with their own source locations and statuses. Multiple call sites between the same programs must remain individually inspectable. Do not represent COBOL variables as graph nodes or edges in this release. Store provenance and analyzer status in the canonical result. Neo4j labels/internal IDs must not leak into the public JSON schema.
>
> Provide Neo4j constraints/indexes for stable program identity, schema+table identity, and table+column identity. Implement idempotent snapshot persistence and bounded-depth incoming/outgoing graph queries. If Neo4j is unavailable, explain how to start it and allow analysis/export to JSON to continue. Do not silently pretend that Neo4j-backed queries succeeded when disconnected.
>
> The UI must include source-directory and format configuration, an analyze action with progress/errors, search for program/table/column, filters for call/access status, an interactive graph with pan/zoom/select and neighborhood expansion, and a detail panel with file, line, finding explanation, SQL/call information, and diagnostic status. Use green for proven unconditional reachable calls, yellow for conditional calls, red for proven unreachable calls, and gray for unresolved/unknown calls; include a legend and text labels so color is not the only signal. Keep large-project views scoped to search results and selected neighborhoods.
>
> Include a small example COBOL project and expected JSON output. Document supported syntax, known limitations, architecture, environment variables, Neo4j startup, frontend/backend startup, and sample usage. Add automated coverage for core source normalization, call extraction/statuses, SQL extraction/resolution statuses, serialization, and Neo4j adapter behavior where practical. Run the available checks and fix failures before finishing. Do not leave core features as TODOs or stubs; if a case is unsupported, return a useful diagnostic.
>
> Keep boundaries modular: source discovery/normalization, parsing, semantic analysis, graph construction/query, serialization, storage, API, and UI must not be collapsed into one module. The CLI/API should call the same public library entry points. At completion, report what works, exact commands to run it, checks performed, and any remaining limitations.

## First-release acceptance checklist

- A clean checkout can install dependencies and start the app using the documented commands.
- The included sample project produces stable, readable JSON and a useful graph in the browser.
- Fixed/free source handling and original source locations are visible in findings.
- Search works for program, DB2 table, and column names.
- Call statuses distinguish conditional, proven unreachable, unresolved, and unknown without treating parser failure as proof.
- Supported SQL forms return table/column results and explicit ambiguity/wildcard/unsupported diagnostics.
- Graph queries can traverse callers and callees with a bounded depth.
- Neo4j persistence is isolated behind an adapter; JSON analysis remains usable when Neo4j is stopped.
- Setup, supported syntax, limitations, and architecture are documented.

## Quality strategy

- Maintain small, representative COBOL fixtures for each supported syntax feature and source format.
- Use golden JSON snapshots to detect accidental changes in graph output and source locations.
- Test malformed/partially supported input and verify diagnostics and graceful recovery.
- Validate SQL extraction against fixtures that include aliases, joins, qualified/unqualified columns, wildcard selections, host variables, and unsupported/dynamic forms.
- Keep performance benchmarks separate from correctness tests and use representative project sizes.

## Definition of done for the first usable release

- A user can point the CLI or local UI at a COBOL source directory and receive a graph snapshot with source locations and diagnostics.
- Program names and statically identifiable calls are searchable and navigable.
- Conditional, unreachable, dynamic, and unknown states are represented without overstating certainty.
- Supported embedded DB2 statements produce table/column references with operation and source location.
- JSON import/export works without Neo4j; Neo4j persistence can be enabled independently.
- The documented package boundaries allow adding another source format rule, SQL construct, storage adapter, or UI without rewriting the parser core.
