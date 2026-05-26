# bnsql-skills

Claude Code and Codex plugin packaging for [bnsql](https://github.com/0xeb/bnsql) — a [live SQL](https://github.com/0xeb/libxsql) interface to Binary Ninja databases.

## Prerequisites

- **Binary Ninja** installed with `BN_INSTALL_DIR` set (or Binary Ninja's install dir on `PATH`)
- **bnsql** from [Releases](https://github.com/0xeb/bnsql/releases) placed on `PATH` (or next to `binaryninja.exe`)
- Verify: `bnsql -h`

## Installation

### Claude Code

```bash
/plugin marketplace add 0xeb/bnsql-skills
```

### Codex

Use the plugin packaging, not a flat copy into `~/.codex/skills`. The plugin path preserves the `bnsql` namespace so generic skill names like `xrefs`, `data`, and `types` do not collide with other skills.

#### Home-local install

1. Clone this repo anywhere temporary:

```bash
git clone https://github.com/0xeb/bnsql-skills
```

2. Copy the plugin directory into your home plugins folder:

```bash
mkdir -p ~/plugins
cp -R bnsql-skills/plugins/bnsql ~/plugins/
```

3. Create or update `~/.agents/plugins/marketplace.json` so it contains:

```json
{
  "name": "bnsql-tools",
  "interface": {
    "displayName": "BNSQL"
  },
  "plugins": [
    {
      "name": "bnsql",
      "source": {
        "source": "local",
        "path": "./plugins/bnsql"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Reverse Engineering"
    }
  ]
}
```

4. Restart Codex.

The Codex plugin manifest lives at `plugins/bnsql/.codex-plugin/plugin.json`, and the skills live under `plugins/bnsql/skills/`.

#### Repo-local install

If you want to test directly from a checkout, keep the repo where it is and use the bundled marketplace file at `.agents/plugins/marketplace.json`. Codex should resolve `./plugins/bnsql` relative to the repo root.

## Compatibility

- `SKILL.md` is the canonical skill contract and remains the source of truth.
- Per-skill Codex UI metadata lives in each skill's optional `agents/openai.yaml` and does not replace `SKILL.md`.
- Codex plugin packaging lives alongside the Claude packaging so this repo can support both without renaming the underlying skills.

## Skills

| Skill | Description | When to Use |
|-------|-------------|-------------|
| `connect` | Connection, CLI, HTTP, MCP, routing index | Starting a session, CLI options, HTTP/MCP server, pragmas |
| `disassembly` | Functions, segments, instructions, blocks | Querying disassembly, instruction analysis |
| `data` | Strings, bytes, binary pattern search (`search_bytes`) | String search, byte access, binary pattern search |
| `xrefs` | Cross-references and imports | Caller/callee analysis (`callers`/`callees` views), import queries |
| `decompiler` | HLIL decompiler reference | Decompilation, HLIL variables, function calls |
| `annotations` | Edit and annotate decompilation | Comments, renames, type application, explicit save |
| `types` | Type system mechanics | Structs, enums, typedefs, function signatures |
| `functions` | SQL functions reference | Look up any bnsql SQL function signature |
| `analysis` | Analysis workflows and scenarios | Security audits, library detection, advanced SQL patterns |

## Notes

- The supported Codex path is plugin-based installation. Flat installs into `~/.codex/skills` are possible, but only if you manually rename every skill with a `bnsql-` prefix to avoid collisions.
- The plugin-based layout keeps the repo's natural `bnsql` ownership model intact.

## Links

- [bnsql](https://github.com/0xeb/bnsql) — CLI and plugin
- [libxsql](https://github.com/0xeb/libxsql) — Core SQLite virtual table framework

## Author

**Elias Bachaalany** ([@0xeb](https://github.com/0xeb))

## License

This project and all its contents — including skill definitions, reference documentation, and configuration files — are licensed under the [MIT License](LICENSE).
