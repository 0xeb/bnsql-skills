---
name: analysis
description: "BNSQL analysis workflows: triage, security audit, crypto/network detection, multi-table queries."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Analysis Workflows

High-level recipes for cross-domain analysis. Each recipe composes multiple
bnsql skills (`disassembly`, `xrefs`, `data`, `decompiler`).

## Recipe: Unknown binary triage

```sql
-- 1. Orientation
SELECT * FROM db_info;
SELECT * FROM entries;
SELECT * FROM capabilities;

-- 2. Imports hint at capability surface
SELECT module, COUNT(*) AS apis
FROM imports
GROUP BY module
ORDER BY apis DESC;

-- 3. Long strings often reveal purpose
SELECT content, hex(address) AS addr
FROM strings
WHERE length > 16
ORDER BY length DESC
LIMIT 20;

-- 4. Most-called functions are usually utilities/dispatchers
WITH caller_counts AS (
    SELECT to_ea, COUNT(*) AS n
    FROM xrefs WHERE is_code = 1
    GROUP BY to_ea
)
SELECT f.name, hex(f.address) AS addr, c.n AS callers
FROM funcs f
JOIN caller_counts c ON c.to_ea = f.address
ORDER BY c.n DESC
LIMIT 10;
```

## Recipe: Security audit

```sql
-- Dangerous string functions
SELECT module, name FROM imports
WHERE name IN ('strcpy', 'strcat', 'sprintf', 'gets', 'scanf');

-- Find call sites in the binary
SELECT DISTINCT c.caller_name, hex(c.caller_addr) AS addr
FROM callers c
WHERE c.callee_name IN ('strcpy', 'strcat', 'sprintf', 'gets');
```

## Recipe: Crypto detection

```sql
-- Crypto-related imports
SELECT module, name FROM imports
WHERE name LIKE '%Crypt%' OR name LIKE '%Hash%'
   OR name LIKE '%AES%'   OR name LIKE '%RSA%'
   OR name LIKE '%MD5%'   OR name LIKE '%SHA%';

-- Constants in code (common AES S-box bytes via search_bytes)
SELECT search_first('63 7C 77 7B F2 6B 6F C5');  -- AES S-box prefix

-- Crypto-named functions
SELECT name, hex(address) AS addr
FROM funcs
WHERE name LIKE '%crypt%' OR name LIKE '%aes%' OR name LIKE '%sha%';
```

## Recipe: Network detection

```sql
-- Network-related imports
SELECT module, name FROM imports
WHERE name LIKE '%socket%' OR name LIKE '%connect%'
   OR name LIKE '%recv%'   OR name LIKE '%send%'
   OR name LIKE '%bind%'   OR name LIKE '%listen%';

-- URL / domain strings
SELECT content, hex(address) AS addr
FROM strings
WHERE content LIKE 'http://%' OR content LIKE 'https://%'
   OR content LIKE '%.com%'   OR content LIKE '%.net%';

-- Functions containing network call sites
SELECT DISTINCT c.caller_name, hex(c.caller_addr) AS addr
FROM callers c
WHERE c.callee_name IN ('socket', 'connect', 'send', 'recv');
```

## Recipe: Function-of-interest deep dive

```sql
-- 1. Survey
SELECT * FROM funcs WHERE address = 0x401000;

-- 2. Who calls it?
SELECT * FROM callers WHERE callee_addr = 0x401000;

-- 3. What does it call?
SELECT DISTINCT callee_name
FROM hlil_calls
WHERE func_addr = 0x401000;

-- 4. Local variables
SELECT name, type, storage
FROM hlil_vars
WHERE func_addr = 0x401000;

-- 5. Decompiled body
SELECT decompile(0x401000);
```

## Recipe: Bridge functions (high connectivity)

```sql
WITH inbound AS (
    SELECT to_ea AS addr, COUNT(*) AS in_count
    FROM xrefs WHERE is_code = 1
    GROUP BY to_ea
),
outbound AS (
    SELECT from_func AS addr, COUNT(*) AS out_count
    FROM xrefs WHERE is_code = 1 AND from_func IS NOT NULL
    GROUP BY from_func
)
SELECT f.name, hex(f.address) AS addr,
       i.in_count, o.out_count,
       (i.in_count * o.out_count) AS connectivity
FROM funcs f
JOIN inbound  i ON i.addr = f.address
JOIN outbound o ON o.addr = f.address
ORDER BY connectivity DESC
LIMIT 10;
```

## Recipe: Patch inventory

```sql
SELECT hex(address) AS addr, original, patched, status
FROM patches
ORDER BY address;
```

## Related skills

- `connect` for session bootstrap
- `xrefs` for the underlying call graph
- `data` for `search_bytes` patterns
- `decompiler` for HLIL-level investigation
- `annotations` for marking findings with comments / renames
