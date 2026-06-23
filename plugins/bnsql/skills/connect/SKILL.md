---
name: connect
description: "Connect to Binary Ninja databases and bootstrap sessions. Use when starting analysis, routing to other skills, or setting up CLI/HTTP/MCP connections."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

## CLI Help (verbatim `bnsql --help`)

```
bnsql vX.Y.Z - SQL interface for Binary Ninja

Usage:
  bnsql <file>                                 Interactive mode
  bnsql <file> -c <query>                      Execute query and exit
  bnsql <file> -f <file.sql>                   Execute SQL file
  bnsql <file> --http [port]                   Start HTTP REST server (default: 8080)
  bnsql <file> --mcp [port]                    Start MCP server (default: 9998)

  <file> is a Binary Ninja database (.bndb) OR a raw binary
  (.exe/.dll/firmware/etc.). Raw binaries trigger fresh BN analysis;
  the resulting .bndb is auto-saved next to the source on clean exit.

Options:
  -s, --source <path>    Binary Ninja database (.bndb) OR raw binary
                         (.exe/.dll/firmware/etc.) — raw binaries trigger
                         fresh BN analysis (auto-saved next to source)
  -c, --command <sql>    SQL query to execute
  -f, --file <path>      SQL file to execute
  -i, --interactive      Interactive SQL mode (default)
  --http [port]          Start HTTP REST server (default: 8080)
  --mcp [port]           Start MCP server for Claude Desktop, etc. (default: 9998)
  --bind <addr>          Bind address for server (default: 127.0.0.1)
  --token <token>        Auth token for HTTP/MCP mode
  -q, --quiet            Suppress banner
  -h, --help             Show this help
  -v, --version          Show version

Examples:
  bnsql sample.bndb -c "SELECT name, size FROM funcs LIMIT 10"
  bnsql sample.bndb --http 8080
  bnsql sample.exe --http 8080       # raw PE: BN auto-analyzes, then serves SQL
  bnsql firmware.bin -c "SELECT COUNT(*) FROM funcs"
```

**Key takeaway:** `bnsql <file>` (and the equivalent `-s <file>`) accepts either
an existing Binary Ninja database **or** a raw binary. No manual
`binaryninja --headless` pre-step is needed — point `bnsql` straight at the
`.exe`/`.dll`/firmware/etc. and Binary Ninja creates the database on first
open. The resulting `.bndb` is auto-saved next to the source on clean exit.

---

## Additional Resources

- For CLI reference, REPL commands, server modes, and runtime controls: [references/cli-reference.md](references/cli-reference.md)
- For canonical schema catalog: [references/schema-catalog.md](references/schema-catalog.md)
- For HTTP server guide: [references/server-guide.md](references/server-guide.md)

---

## Quick Start CLI (Do This First)

Use these commands first to avoid guessing behavior or schema:

```bash
# Query a Binary Ninja database
bnsql sample.bndb -c "SELECT * FROM db_info"
bnsql sample.bndb -c "SELECT * FROM db_info; SELECT COUNT(*) FROM funcs;"

# Interactive REPL
bnsql sample.bndb

# Long-lived HTTP server for iterative analysis
bnsql sample.bndb --http 8080

# Raw binary (.exe/.dll/firmware/etc.) — BN auto-analyzes on first open
bnsql sample.exe --http 8080
bnsql sample.exe -c "SELECT * FROM db_info"

# Query over HTTP
curl -X POST http://127.0.0.1:8080/query -d "SELECT name, size FROM funcs LIMIT 5"
```

Critical guardrails:

- Always provide a file argument. The file can be either a Binary Ninja
  database (`.bndb`) **or** a raw binary (`.exe`, `.dll`, firmware blob, etc.)
  — raw binaries trigger fresh BN analysis and the resulting `.bndb` is
  auto-saved next to the source on clean exit. Do **not** pre-build a `.bndb`
  with `binaryninja --headless`; `bnsql raw.exe` does it in one shot.
- Mutations (UPDATE/INSERT/DELETE, `set_name`, `set_comment`, type edits,
  patches) are applied in-memory only. Call `SELECT save();` to persist to
  disk before exiting. **This is the explicit-save model introduced in v0.0.9.**
- Discover schema before writing queries against unfamiliar surfaces:
  - REPL: `.schema <table>`
  - SQL: `PRAGMA table_xinfo(<table>);` (or `PRAGMA table_info(<table>);`)
- Start orientation with `SELECT * FROM db_info;`.

---

## Schema Catalog (Canonical)

Canonical table/view formats live in `references/schema-catalog.md`.

- Source of truth for column shapes and owner skill mapping.
- Sourced from SQL metadata (`pragma_table_list` + `pragma_table_xinfo`).
- Use this before assuming column names for less-common surfaces.

Manual refresh:

1. `SELECT schema, name, type, ncol FROM pragma_table_list WHERE schema='main' ORDER BY type, name;`
2. `PRAGMA table_xinfo(<surface>);`

---

## Session Bootstrap Contract

Use this exact startup flow before deep analysis:

1. Connect to database (`bnsql <file>`, `--http`, or `--mcp`). The file can
   be an existing `.bndb` or a raw binary — never pre-build a `.bndb` with
   `binaryninja --headless`; let `bnsql raw.exe` do it.
2. Run orientation query:
   ```sql
   SELECT * FROM db_info;
   ```
3. Validate key entities exist:
   ```sql
   SELECT COUNT(*) AS funcs FROM funcs;
   SELECT COUNT(*) AS xrefs FROM xrefs;
   SELECT COUNT(*) AS strings FROM strings;
   ```
4. Check the capabilities matrix to know what mutations are allowed:
   ```sql
   SELECT * FROM capabilities;
   ```
5. Introspect schema for target surfaces before authoring complex SQL:
   ```sql
   PRAGMA table_xinfo(funcs);
   PRAGMA table_xinfo(xrefs);
   ```
6. Route to a domain skill using the routing matrix below.

Never skip steps 2-5 when the user prompt is broad or ambiguous.

---

## Global Agent Contracts

These contracts apply across all bnsql skills and should be treated as one
shared agent behavior model.

### Read-First Contract

- Read current state first (`SELECT`) before writes (`INSERT`/`UPDATE`/`DELETE`).
- Confirm target precision using stable identifiers (`address`, `func_addr`, `from_ea`, `to_ea`).

### Anti-Guessing Contract

- Do not assume columns/types for long-tail surfaces (`types`, `type_members`,
  `func_signatures`, `patches`, `capabilities`, `runtime_settings`).
- Introspect via `.schema` or `PRAGMA table_xinfo(...)` before issuing
  uncertain queries.

### Explicit-Save Contract (v0.0.9+)

- Mutations to `funcs`, `names`, `comments`, `hlil_vars`, `types*`, `patches`,
  and the `set_name`/`set_comment` UDFs are **applied in-memory but not
  written to disk** until `SELECT save();` is called.
- Always pair a mutation with a `save()` when the user expects the change to
  survive after exit.
- Decompiler tables (`pseudocode`, `hlil_vars`, `hlil_calls`) auto-invalidate
  their cache when a function's mutations land — no manual refresh needed.

### Performance Contract

- Always constrain high-cost surfaces (`xrefs`, `instructions`, `pseudocode`,
  `hlil_vars`, `hlil_calls`) by key columns (`func_addr` / `func_ea`).
- For decompiler surfaces, enforce `func_addr = X` unless explicitly asked for
  broad scans.
- Prefer the `callers` / `callees` pre-computed views over `func_start(addr)`
  / `func_at(addr)` UDFs in bulk xref queries — UDFs call the BN API once per
  row.

### Failure Recovery Contract

- On `no such table/column`: introspect schema and retry.
- On empty results: validate address range, then check `capabilities` for
  whether the surface is available in this binary view.
- On timeout: narrow scope, add constraints, paginate, or split the query.

### Output Contract

- **Selection** — decide *whether and how much* to surface from user intent.
  Answer questions directly ("biggest is `_main`, 2851 bytes"); show supporting
  rows only when they help the user verify; don't dump full tables unprompted;
  never surface data fetched only as an intermediate reasoning step.
- **Fidelity** — when you *do* present code/data, show the real artifact
  (decompilation, actual rows), never a paraphrase.
- **Mechanics** — the HTTP `/query` response is a JSON envelope
  (`{success, results:[{columns,rows,…}]}`). Consume it directly and render in
  your reply. Do **not** pipe responses through `python`/`jq` to pre-render a
  table — that discards the `success`/`elapsed_ms`/`error` fields and makes you
  reason over a lossy view. The CLI (`-c`/`-f`) already prints a table. Reserve
  `jq`/`python` for extracting a value to feed a later query.

---

## Skill Routing Matrix (Intent → Skill)

Use this deterministic mapping for initial routing:

| User intent | Primary skill | Typical first query |
|-------------|---------------|---------------------|
| "what does this binary do?" / triage | `analysis` | `SELECT * FROM entries;` |
| disassembly, segments, instructions | `disassembly` | `SELECT * FROM funcs LIMIT 20;` |
| xrefs/callers/callees/import dependencies | `xrefs` | `SELECT * FROM callers WHERE callee_addr = ...;` |
| strings/bytes/pattern search (`search_bytes`) | `data` | `SELECT * FROM strings LIMIT 20;` |
| decompile/pseudocode/hlil variables/calls | `decompiler` | `SELECT decompile(0x...);` |
| comments/renames/retyping/save | `annotations` | `SELECT ... FROM funcs WHERE address = ...;` |
| type creation/struct/enum/member work | `types` | `SELECT * FROM types LIMIT 20;` |
| SQL function lookup/signature recall | `functions` | `SELECT name FROM pragma_function_list;` |

When prompts span domains, execute in this order:

1. Orientation in `connect`
2. Primary domain skill
3. Adjacent skills for enrichment (e.g. `xrefs` + `decompiler` + `annotations`)

---

## welcome / db_info

Database orientation surface for quick session metadata. Use
`PRAGMA table_xinfo(db_info)` to confirm the column shape against your bnsql
version — common columns include processor/architecture, bitness, entry point,
and counts of funcs/segments/names/strings.

```sql
SELECT * FROM db_info;
```

For canonical schema and owner mapping, see `references/schema-catalog.md`.

---

## What is Binary Ninja and Why SQL?

**Binary Ninja** is a modern reverse engineering platform. It analyzes
compiled binaries (executables, DLLs, firmware) and produces:

- **Disassembly** — Human-readable assembly code
- **Functions** — Detected code boundaries with names
- **Cross-references** — Who calls what, who references what data
- **Types** — Structures, enums, function prototypes
- **HLIL/MLIL** — High/Medium Level IL representations

**BNSQL** exposes all this analysis data through SQL virtual tables, enabling:

- Complex queries across multiple data types (JOINs)
- Aggregations and statistics (COUNT, GROUP BY)
- Pattern detection across the entire binary
- Scriptable analysis without writing plugins or Python scripts

---

## Core Concepts for Binary Analysis

### Addresses

Everything in a binary has an **address** — a memory location where code or
data lives. BNSQL exposes addresses as unsigned integers. Use `hex(addr)`
or `printf('0x%X', addr)` for display.

```sql
SELECT hex(address) AS addr, name FROM funcs WHERE size > 1000;
```

### Functions

Binary Ninja groups code into **functions** with:

- `address` — Where the function begins
- `name` — Assigned or auto-generated name (e.g. `main`, `sub_401000`)
- `size` — Total bytes in the function

For single-address disassembly (code or data), prefer `disasm(addr)` over
function-scoped queries.

### Cross-References (xrefs)

Binary analysis is about understanding **relationships**:

- **Code xrefs** — Function calls, jumps between code
- **Data xrefs** — Code reading/writing data locations, or data referring to other data (pointers)
- `from_ea` → `to_ea` represents "address X references address Y"

Use table: `xrefs(from_ea, to_ea, type, is_code, from_func)`.

The `from_func` column is pre-computed (no UDF call per row) — use it for
bulk call-graph queries instead of `func_start(from_ea)`.

### Segments

Memory is divided into **segments** with different purposes. For a typical PE:

- `.text` — Executable code
- `.data` — Initialized global data
- `.rdata` — Read-only data (strings, constants)
- `.bss` — Uninitialized data

Use table: `segments`.

### Basic Blocks

Within a function, **basic blocks** are straight-line code sequences:

- No branches in the middle
- Single entry, single exit
- Useful for control flow analysis

Use table: `blocks(start_ea, end_ea, func_ea)`.

### Decompilation (HLIL)

Binary Ninja's **HLIL** (High Level IL) is the closest thing to C-like
pseudocode in BN:

- `pseudocode` table — full-function decompiled text
- `hlil_vars` — local variables detected by HLIL
- `hlil_calls` — call sites discovered in HLIL

Core decompiler surfaces:

- `decompile(addr)` — Returns the entire function as one text block.
  Use this first when the user asks to "decompile", "show pseudocode", or
  "explain function logic".
- `decompile(addr, limit)` — Limit-aware variant; useful when survey-decompiling
  many functions.
- `pseudocode` table — for line-level filtering by `func_addr` / `ea`.
- `hlil_vars` — local variable rename / type / comment updates.
- `hlil_calls` — call-site analysis.

---

## Performance Rules

### CRITICAL: Constraint Pushdown

Some tables have **optimized filters** that use efficient BN API:

| Table | Optimized Filter | Without Filter |
|-------|------------------|----------------|
| `instructions` | `func_addr = X` | O(all instructions) — SLOW |
| `blocks` | `func_ea = X` | O(all blocks) |
| `xrefs` | `to_ea = X` or `from_ea = X` | O(all xrefs) |
| `pseudocode` | `func_addr = X` | **Decompiles ALL functions** |
| `hlil_vars` | `func_addr = X` | **Decompiles ALL functions** |
| `hlil_calls` | `func_addr = X` | **Decompiles ALL functions** |

**Always filter decompiler tables by `func_addr`!**

### Use Pre-computed Views, Not UDFs

```sql
-- SLOW: One BN API call per row (8000+ for medium binaries)
SELECT func_start(from_ea), to_ea FROM xrefs WHERE is_code = 1;

-- FAST: Pure SQL using pre-computed view
SELECT func_addr, callee_addr FROM callees;
```

The `callers` and `callees` views compute function boundaries once at startup,
not per row. Use them for any bulk call-graph analysis.

---

## Error Handling

- **No decompiler license:** Some HLIL surfaces (`pseudocode`, `hlil_vars`,
  `hlil_calls`) may be empty or unavailable depending on the BN tier. Check
  `capabilities`.
- **No constraint on decompiler tables:** Query will be extremely slow
  (decompiles all functions).
- **Invalid address:** Containing-function UDFs return NULL; use a scalar
  subquery when you need a nullable scalar result.
- **Forgot `save()`:** Mutations succeed but disappear on exit. Always pair a
  mutation with `SELECT save();` if persistence is required.
