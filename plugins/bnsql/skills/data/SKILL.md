---
name: data
description: "Query Binary Ninja strings, bytes, and binary patterns. Use search_bytes() for native fast pattern search."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Data (Strings, Bytes, Binary Pattern Search)

## Tables and views

| Surface | Description |
|---------|-------------|
| `strings` | All discovered strings (content, length, address, type) |
| `string_refs` | Strings joined with the functions that reference them |

## SQL functions

| Function | Description |
|----------|-------------|
| `search_bytes(pattern)` | Find all matches; returns JSON array |
| `search_bytes(pattern, start, end)` | Search within an address range |
| `search_first(pattern)` | First match address (or NULL) |
| `bytes(addr, count)` | Read raw bytes (as hex string) |
| `bytes_raw(addr, count)` | Read raw bytes (as BLOB) |
| `hex(val)` | Format integer as hex |

### Pattern syntax (Binary Ninja native)

- `"48 8B 05"` — Exact bytes (hex, space-separated)
- `"48 ?? 05"` — `??` = any byte wildcard
- `"4?"` — `?` = any nibble (matches `40`–`4F`)
- `"[regex]"` — Regex patterns supported

## Quick patterns

### String search

```sql
-- Interesting strings
SELECT content, hex(address) AS addr
FROM strings
WHERE length > 10
ORDER BY length DESC
LIMIT 20;

-- Password-related
SELECT content, hex(address) AS addr
FROM strings
WHERE content LIKE '%password%';
```

### String cross-references

```sql
SELECT s.content, hex(s.address) AS str_addr,
       sr.func_name, hex(sr.func_addr) AS func_addr
FROM string_refs sr
JOIN strings s ON s.address = sr.string_addr
WHERE s.content LIKE '%error%';
```

### Binary pattern search

```sql
-- Find all RDTSC instructions (opcode: 0F 31)
SELECT search_bytes('0F 31');

-- Iterate matches
SELECT json_extract(value, '$.address') AS addr
FROM json_each(search_bytes('48 8B ?? 00'))
LIMIT 10;

-- First match only
SELECT printf('0x%X', search_first('CC CC CC'));
```

### Functions containing a specific instruction

```sql
-- Count unique functions using RDTSC
SELECT COUNT(DISTINCT func_start(json_extract(value, '$.address'))) AS count
FROM json_each(search_bytes('0F 31'))
WHERE func_start(json_extract(value, '$.address')) IS NOT NULL;

-- List those functions
SELECT DISTINCT
    func_start(json_extract(value, '$.address')) AS func_ea,
    name_at(func_start(json_extract(value, '$.address'))) AS func_name
FROM json_each(search_bytes('0F 31'))
WHERE func_start(json_extract(value, '$.address')) IS NOT NULL;
```

This is much faster than scanning all disassembly lines:
- `search_bytes()` uses native BN binary search
- `func_start()` is O(1) lookup in the function index

### Raw byte read

```sql
SELECT bytes(0x401000, 16);     -- "48 89 5C 24 08 48 89 ..."
```

## Related skills

- `xrefs` for finding the call sites of functions that contain a found
  pattern
- `disassembly` for the surrounding instruction context (`disasm(addr)`)
