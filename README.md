<p align="center">
  <img src="logo.svg" alt="csvql" width="420"/>
</p>

[![CI](https://github.com/melihbirim/csvql/actions/workflows/ci.yml/badge.svg)](https://github.com/melihbirim/csvql/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE.md)
[![Release](https://img.shields.io/github/v/release/melihbirim/csvql)](https://github.com/melihbirim/csvql/releases)

**Give an AI agent read-only SQL over your CSV files, where they already live.**

> A database is something you load your data *into*. csvql is a query you run on the data where it already lives.

csvql runs `SELECT` on CSV files in place: no import, no ingest, no server to stand up. It ships as an [MCP](https://modelcontextprotocol.io/) server, so an agent can ask a question of a gigabyte file and get back a few rows instead of the file. It is one static binary written in Zig, it makes no network calls, and it has no way to write to your data.

```bash
$ csvql "SELECT cab_type, COUNT(*) FROM 'trips.csv' GROUP BY cab_type"
cab_type,COUNT(*)
green,32447
yellow,967553
```

[Website](https://melihbirim.github.io/csvql/) · [Quick Start](#quick-start) · [For agents](#for-agents-mcp) · [Install](#installation) · [Performance](#performance) · [SQL](#sql) · [Docs](#documentation)

---

## Why this exists

Agents are getting access to corporate data. The common path is to hand them a database connection or to upload files to the model. Both move data. csvql is the third option: run it *next to the data*, on your own box, and let the agent query it there.

What that buys you, and what you can verify in the source:

- **Read-only by construction.** The parser has no `INSERT`, `UPDATE`, `DELETE` or `DROP`. The engine has no code path that writes to an input file.
- **No network.** No `std.net`, `std.http` or socket call anywhere in the codebase. No telemetry, no update check. Runs air-gapped. `strace -f -e trace=network` will show you an empty trace.
- **Sandboxed by a flag.** `--root /data` confines every file the process can open. `/etc/passwd` and `../` traversal are rejected with `PathOutsideAllowedRoot`.
- **Bounded output.** MCP results are capped at about 100 rows / 12 KB per call, so a careless `SELECT *` cannot flood the context or exfiltrate a table one call at a time.
- **Flat memory.** An aggregate query holds a running accumulator, not a dataset: 74 MB on a 0.69 GB file and 74 MB on an 8.14 GB one, measured the same way. It fits on the small VM sitting beside the data.
- **Auditable.** `--audit <file>` appends one JSONL record per query. Over SSH you also inherit your existing keys, bastion and logs.

Full posture, including the two known gaps on macOS and Windows: [SECURITY.md](SECURITY.md).

## What it does not do

Say this up front so nobody picks csvql for a workload it will lose:

- **It is not a database.** No `UNION`, no window functions, no `LEFT`/`RIGHT`/`FULL`/`CROSS JOIN` (inner hash join only), and the only subquery shape is `col IN (SELECT ...)`. Each of these errors clearly rather than returning wrong data. If you need full SQL, use DuckDB.
- **Only `DATEDIFF`, `DATEADD` and `EXTRACT` work in `WHERE`.** Other scalar functions (`UPPER`, `LENGTH`, `ROUND`, ...) are `SELECT`-only for now.
- **Quoted CSV is roughly 3x slower than unquoted.** The 16-byte skip that makes unquoted scans fast is disabled by quotes. Still faster than DuckDB on every shape measured, but by 1.6 to 2x rather than 6x.
- **`MEDIAN` buffers every value in a group**, so on 20M rows it is 5x slower than DuckDB ([#205](https://github.com/melihbirim/csvql/issues/205)).
- **CSV only.** JSON and JSONL are output formats, not input formats.

The complete list with tracking issues: [SQL_REFERENCE.md#known-limitations](SQL_REFERENCE.md#known-limitations).

## Quick Start

```bash
# SQL mode
csvql "SELECT name, salary FROM 'data.csv' WHERE age > 30 ORDER BY salary DESC LIMIT 10"

# Simple mode: same query, positional
csvql data.csv "name,salary" "age>30" 10 "salary:desc"

# Pipes
cat data.csv | csvql "SELECT name, age FROM '-' WHERE age > 25"
csvql "SELECT * FROM 'data.csv' WHERE status = 'active'" > output.csv

# Other formats
csvql "SELECT name, age FROM 'data.tsv'" -d $'\t'
csvql "SELECT name, age FROM 'data.csv' LIMIT 3" --json      # or --jsonl, --markdown

# What is in this file?
csvql --schema data.csv
```

Flags you will reach for: `-d/--delimiter`, `--json`, `--jsonl`, `--markdown`, `--no-header`, `--no-input-header` (headerless files get `c1..cN`), `-o/--output`, `--threads`, `--strict`, `--schema`, `--root`, `--audit`, `--mcp`. The full list with semantics is in `csvql --help`. Simple mode is documented in [SIMPLE_QUERY_LANGUAGE.md](SIMPLE_QUERY_LANGUAGE.md).

## For agents (MCP)

### Setup

```bash
csvql install          # registers csvql in Claude Code and Claude Desktop; --print to dry-run
```

It runs `claude mcp add` if the Claude Code CLI is present and merges an `mcpServers.csvql` entry into the Claude Desktop config, preserving your other servers. Restart Claude afterward.

Alternatives: a one-click `csvql-<platform>.mcpb` bundle for Claude Desktop on the [Releases](https://github.com/melihbirim/csvql/releases) page (Settings → Extensions), or a manual entry:

```json
{ "mcpServers": { "csvql": { "command": "/usr/local/bin/csvql", "args": ["--mcp", "--root", "/data"] } } }
```

VS Code (Copilot) takes the same `command`/`args` under `servers` in `.vscode/mcp.json` with `"type": "stdio"`.

### Tools

| Tool | What it returns |
|------|-----------------|
| `csv_query(sql)` | Query results as JSON, capped at about 100 rows / 12 KB |
| `csv_schema(file)` | Column names and a few sample rows |
| `csv_list(directory?)` | CSV files in a directory |

The agent writes ordinary SQL against single-quoted paths:

```sql
SELECT customer, SUM(revenue) AS total FROM 'sales.csv' GROUP BY customer ORDER BY total DESC LIMIT 10
SELECT STRFTIME('%Y-%m', order_date) AS month, COUNT(*) FROM 'orders.csv' GROUP BY month ORDER BY 1
SELECT o.id, c.name FROM 'orders.csv' o JOIN 'customers.csv' c ON o.customer_id = c.id WHERE c.region = 'West'
```

### Remote and on-prem

Run csvql on the server next to the data and connect over SSH. Only the SQL and the small result cross the wire. No open port, no reverse proxy, no new credentials:

```jsonc
// client MCP config; csvql runs on the remote server
{ "command": "ssh", "args": ["analyst@dataserver", "csvql", "--mcp", "--root", "/data"] }
```

Always set `--root` when an agent or another user can reach csvql. Pair it with an unprivileged OS user and a read-only mount for defense in depth. Hardening checklist: [SECURITY.md#hardening-for-on-prem-and-remote-access](SECURITY.md#hardening-for-on-prem-and-remote-access).

### Token cost

Pasting a 417 MB CSV into a model would cost about 230 million tokens. It fits no context window, so the comparison is about feasibility, not savings. Over MCP the agent sends SQL and gets rows back:

| Question an agent asks | Tokens |
| ---------------------- | ------ |
| *"How many trips per cab type?"* | 43 |
| *"Which year was busiest?"* | 49 |
| *"Average fare by passenger count?"* | 123 |
| Five questions, plus the tool schema paid once per conversation | ~560 |

That cost is flat with file size: the same five questions cost the same on a 1 MB file and a 400 MB replica of it, to the token. Two caveats, both measured. The figures above count the CLI's CSV output; the MCP server returns JSON, which on these questions costs about 25% more because keys repeat per row. And the fairer comparison is not pasting but an agent that already has a shell and runs `duckdb -csv -c "..."`: that uses slightly *fewer* tokens than csvql over MCP. DuckDB's default box-drawing output costs about 1.7x more, and a model has to remember to turn it off. What csvql changes for that agent is not token count but `--root`, read-only execution, the output cap, and 54 MB of RAM instead of a few hundred. Measure it yourself: [`bench/bench_tokens.py`](bench/bench_tokens.py).

## Installation

**Homebrew** (macOS / Linux):

```bash
brew install melihbirim/csvql/csvql
```

**Prebuilt binaries** from [GitHub Releases](https://github.com/melihbirim/csvql/releases):

```bash
# macOS (Apple Silicon)
curl -L https://github.com/melihbirim/csvql/releases/latest/download/csvql-macos-aarch64.tar.gz | tar xz
sudo mv csvql-macos-aarch64 /usr/local/bin/csvql

# macOS (Intel)
curl -L https://github.com/melihbirim/csvql/releases/latest/download/csvql-macos-x86_64.tar.gz | tar xz
sudo mv csvql-macos-x86_64 /usr/local/bin/csvql

# Linux (x86_64)
curl -L https://github.com/melihbirim/csvql/releases/latest/download/csvql-linux-x86_64.tar.gz | tar xz
sudo mv csvql-linux-x86_64 /usr/local/bin/csvql
```

Windows (x86_64): download `csvql-windows-x86_64.zip` from the same page and put `csvql-windows-x86_64.exe` on your `PATH`.

**Libraries**, same engine, no subprocess:

```bash
pip install csvql-query      # Python: csvql.query(), .query_df(), .query_csv()   → docs/PYTHON.md
npm install csvql-query      # Node.js: require('csvql-query'), plus a `csvql-query` CLI → docs/NODE.md
```

**From source**, Zig 0.15.2:

```bash
git clone https://github.com/melihbirim/csvql.git && cd csvql
zig build -Doptimize=ReleaseFast
sudo cp zig-out/bin/csvql /usr/local/bin/
```

## Performance

Speed is the reason the memory stays flat and the box stays small, so it is worth stating. It is not the reason to pick csvql over DuckDB: on an agent's timescale both are instant. Numbers below are Apple M2 Pro unless stated; every one has a reproduce script next to it.

**Memory is the headline.** Same `GROUP BY department`, four engines, one process each, 16 GB machine. Peak RSS and wall time:

| file | csvql | DuckDB | polars 2.0 | pandas |
| --- | --- | --- | --- | --- |
| 0.69 GB | **74 MB / 0.2s** | 254 MB / 0.8s | 1151 MB / 0.6s | 1445 MB / 6.5s |
| 1.89 GB | **74 MB / 0.4s** | 253 MB / 1.2s | 2107 MB / 1.1s | 2093 MB / 17.5s |
| 3.80 GB | **75 MB / 0.7s** | 275 MB / 2.1s | 2666 MB / 9.8s | 2355 MB / 36.4s |
| 8.14 GB | **74 MB / 2.4s** | 367 MB / 5.5s | 1754 MB / 26.7s | 4024 MB / 90.8s |

csvql's memory does not move with file size because an aggregate holds an accumulator, not a dataset. Three honest caveats: DuckDB is also nearly flat, so against it the memory gap is about 3.7x, not the 15x polars suggests; every engine here finished the 8 GB file on 16 GB of RAM, so there is no cliff the others fall off; and flat memory is a property of aggregates, not of every query. `ORDER BY` has to hold rows, and a top-N on the 711 MB file peaks at 715 MB. All four columns are measured in-process from Python, so each carries the interpreter's ~20 MB; the CLI measured directly reads 54 MB.

**Speed on raw CSV, 20M rows / 711 MB**, median of 5, one query per process, engines interleaved:

| Query | csvql | polars 2.0 | DuckDB |
| --- | --- | --- | --- |
| `SUM(salary)` | **0.081s** | 0.227s | 0.372s |
| `COUNT(*) GROUP BY department` | **0.124s** | 0.317s | 0.498s |
| `COUNT(*) WHERE city = '...'` | **0.080s** | 0.282s | 0.435s |
| `COUNT(*)` | 0.026s | **0.022s** | 0.401s |
| `ORDER BY salary LIMIT 10` | 0.344s | **0.305s** | 0.420s |
| `MEDIAN(salary) GROUP BY department` | 2.832s | | **0.580s** |

Wins by 2.6 to 5.4x on the scans and aggregates agents actually run. Draws with polars on `COUNT(*)`, which is bandwidth-bound for everyone at this size, and on top-N. Loses on `MEDIAN` ([#205](https://github.com/melihbirim/csvql/issues/205)). On the 8 GB NYC taxi file the gap over DuckDB settles at about 3.2x with roughly 6x less memory and no 2.1 GB native store to build first.

Everything else, including quoted versus unquoted CSV, column-count scaling, wide files, joins, `LIKE`, subqueries, DataFusion, and the methodology: **[BENCHMARKS.md](BENCHMARKS.md)**. Reproduce any number: [`bench/bench_all.sh`](bench/bench_all.sh), [`bench/bench_vs_polars.sh`](bench/bench_vs_polars.sh), [`bench/bench_taxi.sh`](bench/bench_taxi.sh).

## SQL

`SELECT` / `FROM` / `WHERE` / `GROUP BY` / `HAVING` / `ORDER BY` / `LIMIT` / `OFFSET`, inner `JOIN` (chained), `col IN (SELECT ...)`, `LIKE` / `ILIKE` / `BETWEEN` / `IS NULL` / `AND` / `OR` / `NOT`, `DISTINCT`, `CASE WHEN`. Aggregates: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `VARIANCE`, `STDDEV`, `MEDIAN`, `GROUP_CONCAT`, `COUNT(DISTINCT)`. Scalars in `SELECT`: string (`UPPER`, `LOWER`, `TRIM`, `LENGTH`, `REVERSE`, `CONCAT`, `SUBSTR`, `REPLACE`, `SPLIT_PART`, `LPAD`, `RPAD`), math (`ABS`, `SIGN`, `CEIL`, `FLOOR`, `MOD`, `ROUND`, `TRUNC`, `SQRT`, `POWER`, `LN`, `LOG`, `EXP`, `GREATEST`, `LEAST`), `CAST`, `COALESCE`, and dates (`STRFTIME`, `DATE_PART`, `EXTRACT`, `DATEDIFF`, `DATEADD`).

```bash
csvql "SELECT department, COUNT(*), AVG(salary) FROM 'data.csv' WHERE salary > 50000 GROUP BY department HAVING COUNT(*) > 10 ORDER BY department"
csvql "SELECT e.name, d.dept_name FROM 'employees.csv' e JOIN 'departments.csv' d ON e.dept_id = d.id WHERE d.dept_name = 'Engineering'"
csvql "SELECT id FROM 'orders.csv' WHERE customer_id IN (SELECT id FROM 'customers.csv' WHERE region = 'EU')"
```

Full syntax, runnable examples, DuckDB differences and limitations: **[SQL_REFERENCE.md](SQL_REFERENCE.md)**. What is differentially tested against DuckDB and how: **[CORRECTNESS.md](CORRECTNESS.md)**.

## Documentation

| Document | Description |
| --- | --- |
| [SQL_REFERENCE.md](SQL_REFERENCE.md) | Full SQL syntax, runnable examples, DuckDB differences, limitations |
| [SECURITY.md](SECURITY.md) | Security posture, network and disk-write guarantees, hardening |
| [CORRECTNESS.md](CORRECTNESS.md) | What is tested against DuckDB, how, known gaps, error behaviour |
| [BENCHMARKS.md](BENCHMARKS.md) | Full performance analysis vs DuckDB, polars, pandas, DataFusion |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Engine design, optimization techniques |
| [SIMPLE_QUERY_LANGUAGE.md](SIMPLE_QUERY_LANGUAGE.md) | Simple mode syntax reference |
| [docs/PYTHON.md](docs/PYTHON.md) · [docs/NODE.md](docs/NODE.md) · [docs/LIBRARY.md](docs/LIBRARY.md) | Python, Node.js and Zig library APIs |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Contribution guidelines |

## Planned

- HTTP/SSE MCP transport, so one csvql can serve a team as a shared service ([#60](https://github.com/melihbirim/csvql/issues/60)).
- `MEDIAN` in O(groups) memory ([#205](https://github.com/melihbirim/csvql/issues/205)).
- Shell completions, help wanted ([#73](https://github.com/melihbirim/csvql/issues/73)).

Shipped features are in the [release notes](https://github.com/melihbirim/csvql/releases). The [good first issues](https://github.com/melihbirim/csvql/labels/good%20first%20issue) are scoped with file pointers and done-when criteria.

## License

MIT, see [LICENSE.md](LICENSE.md).
