---
name: functions
description: "Complete bnsql SQL function reference catalog."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# SQL Function Catalog

Use `SELECT name FROM pragma_function_list` for the live function list. The
catalog below is canonical for v0.0.9 but verify against the runtime when in
doubt.

## Performance class

| Class | Cost | Examples | Use case |
|-------|------|----------|----------|
| Fast (pure SQL) | µs | `hex`, `printf`, `json_*` | Anywhere |
| Moderate | ms | `disasm(addr)` | Small result sets |
| Slow (one BN API call per row) | ms × N | `func_start`, `func_at`, `func_end`, `name_at`, `xrefs_to`, `xrefs_from` | Single-address lookups only |
| Very slow (decompiles function) | seconds | `decompile(addr)` | Single function only |

**Rule:** Don't put slow UDFs inside bulk SELECTs over many rows. Use
pre-computed views (`callers`, `callees`) or pre-computed columns
(`xrefs.from_func`) instead.

## Disassembly

| Function | Description |
|----------|-------------|
| `disasm(addr)` | Single-instruction disassembly text |
| `bytes(addr, count)` | Raw bytes as hex string |
| `bytes_raw(addr, count)` | Raw bytes as BLOB |
| `mnemonic(addr)` | Instruction mnemonic only |

## Binary search

| Function | Description |
|----------|-------------|
| `search_bytes(pattern)` | All matches; returns JSON array |
| `search_bytes(pattern, start, end)` | Search within range |
| `search_first(pattern)` | First match address (or NULL) |

See the `data` skill for pattern syntax (`??` wildcards, regex).

## Names and functions

| Function | Description |
|----------|-------------|
| `name_at(addr)` | Name at address (or NULL) |
| `func_at(addr)` | Function name containing address |
| `func_start(addr)` | Start of containing function |
| `func_end(addr)` | End of containing function |
| `func_qty()` | Total function count |
| `func_at_index(n)` | Function address at index `n` |

## Cross-references

| Function | Description |
|----------|-------------|
| `xrefs_to(addr)` | JSON array of xrefs TO address |
| `xrefs_from(addr)` | JSON array of xrefs FROM address |

For bulk xref work, query the `xrefs` table or `callers` / `callees` views
directly — these UDFs are for single-address lookups.

## Navigation

| Function | Description |
|----------|-------------|
| `next_head(addr)` | Next defined item |
| `prev_head(addr)` | Previous defined item |
| `segment_at(addr)` | Segment name at address |

## Comments

| Function | Description |
|----------|-------------|
| `comment_at(addr)` | Get comment at address |
| `set_comment(addr, text)` | Set comment (requires `save()` to persist) |

## Mutation

| Function | Description |
|----------|-------------|
| `set_name(addr, name)` | Set name at address |
| `save()` | Persist in-memory edits to disk (explicit-save model) |

## Decompilation

| Function | Description |
|----------|-------------|
| `decompile(addr)` | Full-function HLIL text |
| `decompile(addr, limit)` | Limit-aware variant |

## Display helpers

| Function | Description |
|----------|-------------|
| `hex(val)` | Format integer as hex string |
| `printf(fmt, ...)` | Standard SQLite printf |
