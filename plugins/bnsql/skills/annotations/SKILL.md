---
name: annotations
description: "Edit Binary Ninja databases: comments, renames, types, patches. Mutations require SELECT save() to persist (explicit-save model, v0.0.9+)."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Annotations (Renames, Comments, Types, Patches)

## Explicit-Save Model (v0.0.9+)

BNSQL opens the database read-mostly. Mutations are applied in-memory but are
**not** written to disk until `SELECT save();` is called.

```sql
UPDATE funcs SET name = 'main' WHERE address = 0x401000;
SELECT save();      -- without this, the edit is lost on exit
```

This applies to every mutation: UPDATE/INSERT/DELETE on writable tables,
plus `set_name()` / `set_comment()` UDFs.

## Mutation Loop

1. Read current state with `SELECT` (precision before edit).
2. Apply mutation (UPDATE/INSERT/DELETE or `set_*` UDF).
3. Re-read to verify expected change.
4. `SELECT save();` to persist.

## Writable surfaces

| Surface | Operations | Columns |
|---------|------------|---------|
| `funcs` | INSERT/UPDATE/DELETE | `name` (rename); INSERT creates a function via BN API |
| `names` | INSERT/UPDATE/DELETE | `name` (global rename) |
| `comments` | INSERT/UPDATE/DELETE | `comment` |
| `hlil_vars` | UPDATE | `name`, `type`, `comment` |
| `types`, `type_members`, `type_enum_values` | INSERT/UPDATE/DELETE | Type system edits |
| `patches` | INSERT/UPDATE/DELETE | Byte-level patches |

Plus UDFs: `set_name(addr, name)`, `set_comment(addr, text)`.

## Quick patterns

### Rename a function

```sql
SELECT name FROM funcs WHERE address = 0x401000;
UPDATE funcs SET name = 'parse_packet' WHERE address = 0x401000;
SELECT name FROM funcs WHERE address = 0x401000;
SELECT save();
```

### Add a comment

```sql
SELECT comment FROM comments WHERE address = 0x401000;
INSERT INTO comments(address, comment) VALUES (0x401000, 'Entry point dispatcher');
SELECT save();
```

Or via UDF:

```sql
SELECT set_comment(0x401000, 'Entry point dispatcher');
SELECT save();
```

### Rename an HLIL variable

```sql
UPDATE hlil_vars
SET name = 'packet_len'
WHERE func_addr = 0x401000 AND storage = 'rdi';
SELECT save();
```

### Apply a type to a variable

```sql
UPDATE hlil_vars
SET type = 'PACKET_HEADER *'
WHERE func_addr = 0x401000 AND name = 'packet_len';
SELECT save();
```

### Bulk rename `sub_*` functions matching a pattern

```sql
UPDATE funcs
SET name = 'handler_' || printf('%X', address)
WHERE name LIKE 'sub_%'
  AND address IN (
      SELECT to_ea FROM xrefs
      WHERE from_ea = (SELECT address FROM funcs WHERE name = 'dispatcher_table')
  );
SELECT save();
```

### Patch bytes

```sql
INSERT INTO patches(address, patched) VALUES (0x401050, X'90909090');
SELECT save();
```

Use `SELECT * FROM patches` to inventory current patches; the `original`
column preserves the pre-patch bytes for safe revert via DELETE.

## Decompiler cache invalidation

Writes to `hlil_vars`, `pseudocode`, or names that affect a function
**invalidate the decompiler cache** for that function. The next read against
`pseudocode` / `hlil_vars` / `hlil_calls` for that function re-decompiles.
No manual refresh step is required.

## Related skills

- `decompiler` for HLIL variable inspection before rename
- `types` for type creation
- `xrefs` for finding the right targets before mutating

## Failure modes

- **Forgot `save()`:** Edits succeed but vanish on exit. Always pair mutations
  with `SELECT save();`.
- **Capability denied:** Some views may have restricted writes. Check
  `SELECT * FROM capabilities;` for the live feature matrix.
- **Address out of range:** Containing-function lookups return NULL; precede
  mutations with a `SELECT` to confirm the target row exists.
