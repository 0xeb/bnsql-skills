---
name: decompiler
description: "Decompile Binary Ninja functions via HLIL: pseudocode text, local variables, call sites. Always filter by func_addr."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Decompiler (HLIL)

Binary Ninja's High Level IL is the closest thing to C-like pseudocode in BN.
BNSQL exposes it through one SQL function and three tables.

## Surfaces

| Surface | Description | Required filter |
|---------|-------------|-----------------|
| `decompile(addr)` UDF | Full-function decompiled text | — (single address) |
| `decompile(addr, limit)` UDF | Limit-aware variant | — |
| `pseudocode` table | Line-level filtering by `func_addr` / `ea` | **`func_addr = X`** |
| `hlil_vars` table | Local variables; writable (name, type, comment) | **`func_addr = X`** |
| `hlil_calls` table | Call sites discovered in HLIL | **`func_addr = X`** |

**Always filter decompiler tables by `func_addr`** — without it, BN
decompiles every function in the binary.

## Quick patterns

### Decompile one function

```sql
-- Full function as one text block
SELECT decompile(0x401000);

-- Limit to 40 lines (good for surveys)
SELECT decompile(0x401000, 40);
```

### Local variables

```sql
SELECT name, type, storage
FROM hlil_vars
WHERE func_addr = 0x401000;
```

### Call sites inside a function

```sql
SELECT DISTINCT callee_name
FROM hlil_calls
WHERE func_addr = 0x401000;
```

### Decompile + line-level filter

```sql
SELECT line_text
FROM pseudocode
WHERE func_addr = 0x401000
  AND line_text LIKE '%CreateFile%';
```

### Find functions calling a target

```sql
SELECT DISTINCT func_at(func_addr) AS caller
FROM hlil_calls
WHERE callee_name = 'malloc';
```

## Mutation surface (annotations)

`hlil_vars` is writable on `name`, `type`, `comment`. Apply edits inside
the mutation loop:

```sql
-- 1. Read current state
SELECT name, type FROM hlil_vars
WHERE func_addr = 0x401000 AND storage = 'var_8';

-- 2. Apply mutation
UPDATE hlil_vars
SET name = 'user_buffer', type = 'char *'
WHERE func_addr = 0x401000 AND storage = 'var_8';

-- 3. Re-read to verify
SELECT name, type FROM hlil_vars
WHERE func_addr = 0x401000 AND storage = 'var_8';

-- 4. Persist
SELECT save();
```

Writes to `hlil_vars`, `pseudocode`, or names that affect a function
invalidate the decompiler cache for that function. The next decompile re-runs
HLIL on the updated state — no manual refresh needed.

## Related skills

- `annotations` for the broader mutation workflow (renames, comments, types)
- `xrefs` for the surrounding call graph context
- `types` for type creation and application to variables

## Performance

Decompiler tables are **on-demand virtual tables** — each row requires
decompiling its containing function. Unconstrained queries iterate every
function and decompile each one. On a medium binary that's minutes-to-hours.
Always pin to a specific `func_addr`.
