---
name: xrefs
description: "Analyze Binary Ninja cross-references: callers, callees, imports, data refs. Use the pre-computed callers/callees views for performance."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Cross-References

## Tables and views

| Surface | Description | Performance |
|---------|-------------|-------------|
| `xrefs` | `from_ea`, `to_ea`, `type`, `is_code`, `from_func` | Filter by `to_ea = X` or `from_ea = X` |
| `callers` | Pre-computed view: who calls each function | Pure SQL, fast |
| `callees` | Pre-computed view: what each function calls | Pure SQL, fast |
| `imports` | Imported symbols by module | Filter by `module` for narrow scans |
| `string_refs` | Strings and the functions that reference them | Pure SQL |

## CRITICAL: Use views over UDFs

```sql
-- SLOW: 8000+ BN API calls on a medium binary
SELECT func_start(from_ea), to_ea FROM xrefs WHERE is_code = 1;

-- FAST: Pure SQL, milliseconds
SELECT func_addr, callee_addr FROM callees;
```

The `xrefs` table already provides a pre-computed `from_func` column. Use
that when you need the caller function without an extra UDF call:

```sql
SELECT from_func, to_ea FROM xrefs WHERE is_code = 1;
```

## Quick patterns

### Functions ranked by caller count

```sql
WITH caller_counts AS (
    SELECT to_ea, COUNT(*) AS callers
    FROM xrefs
    WHERE is_code = 1
    GROUP BY to_ea
)
SELECT f.name, hex(f.address) AS addr, c.callers
FROM funcs f
JOIN caller_counts c ON c.to_ea = f.address
ORDER BY c.callers DESC
LIMIT 10;
```

### Who calls `malloc`?

```sql
SELECT DISTINCT caller_name
FROM callers
WHERE callee_name = 'malloc';
```

### Functions called at most 10 times

```sql
WITH cnt AS (
    SELECT to_ea, COUNT(*) AS n
    FROM xrefs WHERE is_code = 1
    GROUP BY to_ea
)
SELECT f.name, COALESCE(c.n, 0) AS calls
FROM funcs f
LEFT JOIN cnt c ON c.to_ea = f.address
WHERE COALESCE(c.n, 0) <= 10
ORDER BY calls DESC;
```

### Orphan functions (no callers)

```sql
WITH has_callers AS (
    SELECT DISTINCT to_ea FROM xrefs WHERE is_code = 1
)
SELECT f.name, hex(f.address) AS addr
FROM funcs f
WHERE f.address NOT IN (SELECT to_ea FROM has_callers);
```

### Import dependencies

```sql
SELECT module, COUNT(*) AS api_count
FROM imports
GROUP BY module
ORDER BY api_count DESC;
```

## Related skills

- `disassembly` for the underlying instruction stream
- `decompiler` for HLIL call sites via `hlil_calls`
- `data` for string cross-references via `string_refs`
