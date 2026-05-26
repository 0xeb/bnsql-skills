# Schema Catalog (bnsql)

Canonical column shapes and owner-skill mapping for bnsql virtual tables.

**Always confirm the live shape with `PRAGMA table_xinfo(<table>);`** before
authoring complex queries — the catalog reflects the source-of-truth at
authoring time but bnsql's surfaces evolve.

## How to refresh this catalog

```sql
SELECT schema, name, type, ncol
FROM pragma_table_list
WHERE schema = 'main'
ORDER BY type, name;

PRAGMA table_xinfo(<surface>);
```

## Owner-skill mapping

| Table / View | Primary skill | Notes |
|--------------|---------------|-------|
| `db_info` | `connect` | Orientation surface; use first |
| `capabilities` | `connect` | Feature matrix — gate writes by checking this |
| `runtime_settings` | `connect` | PRAGMA bnsql.* enumeration |
| `funcs` | `disassembly` | Function table; writable for create/rename |
| `segments` | `disassembly` | Memory layout |
| `blocks` | `disassembly` | Basic blocks; filter by `func_ea` |
| `instructions` | `disassembly` | Per-instruction rows; filter by `func_addr` |
| `entries` | `analysis` | Entry points |
| `imports` | `xrefs` | Import table |
| `names` | `annotations` | Global names; writable |
| `comments` | `annotations` | Comments; writable |
| `xrefs` | `xrefs` | `from_ea`, `to_ea`, `is_code`, `from_func` |
| `callers` | `xrefs` | Pre-computed view; prefer over `func_start()` UDF |
| `callees` | `xrefs` | Pre-computed view; same |
| `string_refs` | `data` | Strings + the functions that reference them |
| `strings` | `data` | All discovered strings |
| `pseudocode` | `decompiler` | HLIL decompiled text; filter by `func_addr` |
| `hlil_vars` | `decompiler` | HLIL local variables; writable |
| `hlil_calls` | `decompiler` | HLIL call sites |
| `types` | `types` | All defined types |
| `type_members` | `types` | Struct/union member fields |
| `type_enum_values` | `types` | Enum members |
| `func_signatures` | `types` | Per-function parameter and return type info |
| `types_v_structs` | `types` | Convenience: structs only |
| `types_v_enums` | `types` | Convenience: enums only |
| `patches` | `annotations` | Byte patches; writable |
| `entities` | `analysis` | Unified discovery view |

## High-cost surfaces (require constraints)

Filter the following by `func_addr` (or equivalent) to avoid full scans /
decompiling every function:

- `instructions` — `func_addr = X`
- `pseudocode` — `func_addr = X`
- `hlil_vars` — `func_addr = X`
- `hlil_calls` — `func_addr = X`
- `xrefs` — `to_ea = X` or `from_ea = X`
- `blocks` — `func_ea = X`
