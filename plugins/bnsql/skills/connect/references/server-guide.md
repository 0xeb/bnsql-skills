# Server Guide (bnsql)

HTTP REST and MCP server modes.

---

## HTTP REST Server

### Starting

```bash
bnsql sample.bndb --http              # default port 8080
bnsql sample.bndb --http 9000         # custom port
bnsql sample.bndb --http --bind 0.0.0.0
bnsql sample.bndb --http --token mysecret
```

### Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/` | GET | No | Welcome message |
| `/help` | GET | No | API documentation (LLM-discoverable) |
| `/query` | POST | Yes* | Execute SQL (body = raw SQL or semicolon-separated script) |
| `/status` | GET | Yes* | Health check |
| `/shutdown` | POST | Yes* | Stop server |

*Auth required only if `--token` was specified.

### Example queries

```bash
# Single query
curl -X POST http://localhost:8080/query \
  -d "SELECT name, size FROM funcs ORDER BY size DESC LIMIT 10"

# Semicolon-separated script
curl -X POST http://localhost:8080/query \
  -d "SELECT * FROM db_info; SELECT COUNT(*) FROM funcs;"

# With auth token
curl -X POST http://localhost:8080/query \
  -H "Authorization: Bearer mysecret" \
  -d "SELECT * FROM funcs LIMIT 5"
```

### Response format

The `/query` endpoint returns JSON. Single statements return the legacy
column/row shape; semicolon-separated scripts return a `statements[]` array
with one entry per statement.

### Python automation

```python
import requests

resp = requests.post(
    "http://localhost:8080/query",
    data="SELECT name, size FROM funcs ORDER BY size DESC LIMIT 10",
    headers={"Authorization": "Bearer mysecret"} if TOKEN else {},
)
data = resp.json()
for row in data["rows"]:
    print(row)
```

---

## MCP Server

For Claude Desktop and other Model Context Protocol clients.

```bash
bnsql sample.bndb --mcp 9998
bnsql sample.bndb --mcp --token mysecret
```

The MCP server exposes a `bnsql_query` tool that takes raw SQL or
semicolon-separated scripts. The tool schema is auto-published over the MCP
protocol; consult your MCP client's tool inventory after connection.

---

## Iterative analysis pattern

Open the database once, query many times:

```bash
# Terminal 1
bnsql sample.bndb --http 8080

# Terminal 2
curl -X POST http://localhost:8080/query -d "SELECT * FROM funcs LIMIT 5"
curl -X POST http://localhost:8080/query -d "SELECT * FROM strings WHERE content LIKE '%password%'"
# ... as many as needed; no per-query BN startup cost
```

This is significantly faster than re-invoking the CLI for each query.

---

## Raw binary serving

The same fresh-analysis behavior applies in server mode — point `--http` at
a raw binary and the first query waits for BN's analysis to complete:

```bash
bnsql sample.exe --http 8080
# First query may take longer (analysis pending);
# subsequent queries are fast.
```

The resulting `.bndb` is auto-saved next to the source on clean exit.
