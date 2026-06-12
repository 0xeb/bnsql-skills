# CLI Reference

Command-line interface, REPL commands, server modes, runtime controls, database
modification, hex formatting.

---

## Command-Line Interface

BNSQL provides SQL access to Binary Ninja databases via command line or as a
server.

### Binary Provenance

When validating behavior that must match the live Binary Ninja plugin session,
use SDK-path binaries:

- CLI: `bnsql.exe` on `PATH` (or copied alongside `binaryninja.exe`)
- Plugin: `bnsql_plugin.dll` inside Binary Ninja's plugin directory

Set `BN_INSTALL_DIR` so the CLI's delay-loaded BN DLLs resolve.

### Invocation Modes

**0. Fresh Analysis from Raw Binary**

`bnsql <file>` (and the equivalent `-s <file>`) accepts either an existing
Binary Ninja database **or** a raw binary. When the path does not end in
`.bndb`, bnsql calls Binary Ninja to analyze it from scratch. The resulting
`.bndb` is auto-saved next to the source on clean exit. If a sibling `.bndb`
already exists, that's loaded instead.

```bash
bnsql sample.exe --http 8080
bnsql sample.dll -c "SELECT * FROM db_info"
bnsql firmware.bin
```

No manual `binaryninja --headless` pre-step is required.

**1. Query or Script (Local)**

```bash
bnsql sample.bndb -c "SELECT * FROM funcs LIMIT 10"
bnsql sample.bndb -c "SELECT * FROM db_info; SELECT COUNT(*) FROM funcs;"
```

**2. SQL File Execution**

```bash
bnsql sample.bndb -f analysis.sql
```

**3. Interactive REPL**

```bash
bnsql sample.bndb
```

**4. HTTP Server Mode**

```bash
bnsql sample.bndb --http 8080
# Then query via: curl -X POST http://localhost:8080/query -d "SELECT * FROM funcs"
```

**5. MCP Server Mode (for Claude Desktop and other MCP clients)**

```bash
bnsql sample.bndb --mcp 9998
```

### CLI Options

| Option | Description |
|--------|-------------|
| `-s, --source <path>` | Binary Ninja database (`.bndb`) **or** raw binary (`.exe`/`.dll`/firmware/etc.) — raw binaries trigger fresh BN analysis (auto-saved next to source) |
| `-c, --command <sql>` | Execute SQL query or semicolon-separated script |
| `-f, --file <path>` | Execute SQL from file |
| `-i, --interactive` | Interactive REPL mode (default) |
| `--http [port]` | Start HTTP REST server (default port: 8080) |
| `--mcp [port]` | Start MCP server (default port: 9998) |
| `--bind <addr>` | Bind address for HTTP/MCP server (default: 127.0.0.1) |
| `--token <token>` | Auth token for HTTP/MCP mode |
| `-q, --quiet` | Suppress banner |
| `-h, --help` | Show help |
| `-v, --version` | Show version |

### REPL Commands

| Command | Description |
|---------|-------------|
| `.tables` | List all virtual tables |
| `.schema [table]` | Show table schema |
| `.info` | Show database metadata |
| `.clear` / `.reset` | Clear/reset session |
| `.quit` / `.exit` | Exit REPL |
| `.help` | Show available commands |
| `.http start` | Start HTTP server on random port |
| `.http stop` | Stop HTTP server |
| `.http` | Show HTTP server status (start if not running) |

### Performance Strategy

Opening a database has startup overhead (Binary Ninja initialization, analysis
wait for raw binaries). For one small query, use `-c`. For iterative work,
keep one long-lived session (interactive REPL, `--http`, or `--mcp`) and run
many queries against it.

**One-shot:**

```bash
bnsql sample.bndb -c "SELECT COUNT(*) FROM funcs"
```

**Iterative exploration:** Start a server once, then query repeatedly over
HTTP. Each HTTP `/query` body may also be a semicolon-separated script.

```bash
# Terminal 1: open once
bnsql sample.bndb --http 8080

# Terminal 2: query many times, no startup cost
curl -X POST http://localhost:8080/query -d "SELECT * FROM funcs LIMIT 5"
curl -X POST http://localhost:8080/query -d "SELECT * FROM strings WHERE content LIKE '%error%'"
```

---

## Runtime Controls (SQL)

`bnsql` exposes runtime settings through pragmas:

```sql
PRAGMA bnsql.query_timeout_ms;            -- get
PRAGMA bnsql.query_timeout_ms = 60000;    -- set (0 disables)
PRAGMA bnsql.hints_enabled = 1;           -- 1/0
```

The full pragma surface is enumerated by the `runtime_settings` table:

```sql
SELECT * FROM runtime_settings;
```

The `capabilities` table exposes the runtime feature matrix — which mutations
and features are available in the current binary view:

```sql
SELECT * FROM capabilities;
```

---

## Database Modification (Explicit-Save Model, v0.0.9+)

BNSQL opens the database read-mostly. Mutations are applied in-memory and
must be explicitly saved with `SELECT save();`. If you exit without saving,
the edits are lost. The HTTP and MCP servers follow the same model.

Quick capability matrix (verify against the live `capabilities` table for
your version):

| Surface | INSERT | UPDATE | DELETE |
|---------|--------|--------|--------|
| `funcs` | Yes (creates function via BN API) | `name` | Yes |
| `names` | Yes | `name` | Yes |
| `comments` | Yes | `comment` | Yes |
| `hlil_vars` | — | `name`, `type`, `comment` | — |
| `types`, `type_members`, `type_enum_values` | Yes | Yes | Yes |
| `patches` | Yes | Yes | Yes |

After any of these mutations, run:

```sql
SELECT save();
```

---

## Hex Address Formatting

BN uses integer addresses. For display, use `printf()` or `hex()`:

```sql
-- 32-bit format
SELECT printf('0x%08X', address) AS addr FROM funcs;

-- 64-bit format
SELECT printf('0x%016llX', address) AS addr FROM funcs;

-- Auto-width via hex() UDF
SELECT hex(address) AS addr FROM funcs;
```

---

## Server Modes

BNSQL supports HTTP-based server modes: **HTTP REST** and **MCP** (Model
Context Protocol).

### HTTP REST Server (Recommended)

Standard REST API for curl, Python, or any HTTP client.

```bash
bnsql sample.bndb --http              # default port 8080
bnsql sample.bndb --http 9000         # custom port
bnsql sample.bndb --http --token X    # with auth
```

```bash
curl -X POST http://localhost:8080/query -d "SELECT name, size FROM funcs LIMIT 5"
```

For endpoints, Python automation patterns, response format, and compatibility
notes, see `server-guide.md`.

### MCP Server

For Claude Desktop and other MCP-aware clients:

```bash
bnsql sample.bndb --mcp 9998
```
