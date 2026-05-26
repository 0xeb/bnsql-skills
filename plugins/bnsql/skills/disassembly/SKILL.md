---
name: disassembly
description: "Query Binary Ninja disassembly: functions, segments, instructions, blocks. Use for code-level analysis and instruction inspection."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Disassembly

Query Binary Ninja's disassembly: functions, segments, basic blocks, and
per-instruction rows.

## Tables

| Table | Description | Required filter |
|-------|-------------|-----------------|
| `funcs` | All detected functions | none |
| `segments` | Memory segments | none |
| `blocks` | Basic blocks inside functions | `func_ea = X` for perf |
| `instructions` | One row per instruction | `func_addr = X` for perf |
| `entries` | Entry points | none |

## Quick patterns

### Top functions by size

```sql
SELECT hex(address) AS addr, name, size
FROM funcs
ORDER BY size DESC
LIMIT 10;
```

### Segment layout

```sql
SELECT name, hex(start) AS start, hex(end) AS end,
       perm, class
FROM segments
ORDER BY start;
```

### Basic-block CFG of one function

```sql
SELECT hex(start_ea) AS bb_start, hex(end_ea) AS bb_end
FROM blocks
WHERE func_ea = 0x401000
ORDER BY start_ea;
```

### Disassembly listing

```sql
-- Single-address view
SELECT disasm(0x401000);

-- Per-instruction rows inside a function (constrained!)
SELECT hex(address) AS addr, mnemonic, operand
FROM instructions
WHERE func_addr = 0x401000
ORDER BY address;
```

## Performance

`instructions` and `blocks` are high-cost when scanned unconstrained — they
walk every function in the database. Always pass `func_addr = X`
(`instructions`) or `func_ea = X` (`blocks`).

## Related skills

- `xrefs` for call-graph and reference analysis (use the `callers`/`callees`
  views, not `func_start()`)
- `decompiler` for HLIL pseudocode
- `data` for `search_bytes()` binary pattern search

## Canonical reference

For exhaustive table column shapes and SQL function signatures, see
`prompts/bnsql_agent.md` in the bnsql repo. Use
`PRAGMA table_xinfo(<table>)` to confirm any column you're uncertain about.
