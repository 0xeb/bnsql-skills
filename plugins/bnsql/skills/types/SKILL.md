---
name: types
description: "Binary Ninja type system via SQL: structs, unions, enums, typedefs, function signatures. All writable; remember SELECT save()."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Type System

## Tables

| Table | Description |
|-------|-------------|
| `types` | All defined types (structs, unions, enums, typedefs, function prototypes) |
| `type_members` | Struct/union member fields |
| `type_enum_values` | Enum member values |
| `func_signatures` | Per-function parameter and return type info |
| `types_v_structs` | Convenience view: structs only |
| `types_v_enums` | Convenience view: enums only |

All are writable (INSERT/UPDATE/DELETE) and follow the explicit-save model —
call `SELECT save();` to persist.

**Use `PRAGMA table_xinfo(<table>)`** to confirm the live column shape before
authoring complex queries.

## Quick patterns

### Type inventory

```sql
SELECT
    SUM(is_struct)  AS structs,
    SUM(is_enum)    AS enums,
    SUM(is_typedef) AS typedefs,
    SUM(is_func)    AS func_protos
FROM types;
```

### Struct layout

```sql
PRAGMA table_xinfo(type_members);

SELECT offset, name, type, size
FROM type_members
WHERE type_name = 'PACKET_HEADER'
ORDER BY offset;
```

### Enum values

```sql
SELECT name, value
FROM type_enum_values
WHERE enum_name = 'IRP_MJ'
ORDER BY value;
```

### Function signatures

```sql
SELECT arg_index, name, type
FROM func_signatures
WHERE func_addr = 0x401000
ORDER BY arg_index;
```

## Type creation

Create a struct:

```sql
INSERT INTO types(name, is_struct) VALUES ('MY_HEADER', 1);
INSERT INTO type_members(type_name, offset, name, type)
VALUES
    ('MY_HEADER', 0,  'magic',  'uint32_t'),
    ('MY_HEADER', 4,  'length', 'uint32_t'),
    ('MY_HEADER', 8,  'data',   'char[1]');

SELECT save();
```

Create an enum:

```sql
INSERT INTO types(name, is_enum) VALUES ('ERROR_CODE', 1);
INSERT INTO type_enum_values(enum_name, name, value) VALUES
    ('ERROR_CODE', 'SUCCESS',    0),
    ('ERROR_CODE', 'INVALID',    1),
    ('ERROR_CODE', 'NOT_FOUND',  2);

SELECT save();
```

## Apply a type to a variable

```sql
UPDATE hlil_vars
SET type = 'MY_HEADER *'
WHERE func_addr = 0x401000 AND name = 'packet';
SELECT save();
```

## Pattern: type-based feature detection

```sql
-- Functions whose first parameter is a specific struct
SELECT func_addr, hex(func_addr) AS addr
FROM func_signatures
WHERE arg_index = 0
  AND type LIKE '%PIRP%';
```

## Related skills

- `annotations` for applying types to variables and renaming
- `decompiler` for inspecting variables before retyping
