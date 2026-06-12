---
name: cdata-driver-cli
description: "Use when the user wants to connect to, query, or explore a data source through a CData driver via the command line. This includes: creating or testing a connection to an enterprise system (Salesforce, NetSuite, Snowflake, SQL Server, Jira, etc.); running SQL queries against connected data; exploring schema (listing tables, columns, or stored procedures); executing source-specific stored procedures; and downloading, installing, or activating CData drivers. Acts as the prerequisite foundation for all driver-level cdata- skills (cdata-salesforce, cdata-airtable, etc.) that add source-specific guidance."
license: MIT
metadata:
  author: CData Software
  version: "1.0"
---

# SKILL: CData Driver CLI

## Data Model

The driver exposes the data source as a relational model:

- Tables that represent queriable entities
- Columns that make up the attributes of tables
- Procedures that support performing actions

---

## CLI Invocation

The CLI installs as `cdatacli` and is on `PATH` after install:

```bash
cdatacli <group> <subcommand> [options]
```

Requires Java 17+. Drivers are discovered from `./` or `./lib/` relative to the CLI executable.

> **Shell syntax:** Examples below use bash/zsh syntax. On Windows PowerShell, adapt them:
> - `mkdir -p <dir>` → `New-Item -ItemType Directory -Force <dir>`
> - `\` line-continuation → backtick `` ` ``
> - `> file` writes UTF-16 on **Windows PowerShell 5.1**, which can corrupt a generated `SKILL.md`; use `| Out-File -Encoding utf8 file` instead (PowerShell 7+ already writes UTF-8).
> - Wrap a connection string in single quotes if it contains `$` or other characters the shell would expand.

If `cdatacli --version` is missing, install:

> Install commands are tentative and will be finalized for the CLI release. Separate installers are planned for Windows, macOS, and Linux.

- Windows (PowerShell): `iwr https://cdn.cdata.com/cli/install.ps1 | iex`
- macOS: `curl -fsSL https://cdn.cdata.com/cli/install-macos.sh | bash`
- Linux: `curl -fsSL https://cdn.cdata.com/cli/install-linux.sh | bash`

---

## Command Reference

| Goal | Command |
|---|---|
| List installed drivers | `drivers list` |
| Search remote driver catalog | `drivers search [--driver <name-or-artifact-id>]` |
| Download driver jar | `drivers download --artifact-id <id> [--output <dir>]` |
| Download driver jar (by URL) | `drivers download --url <jar-url> [--output <dir>]` |
| Activate (trial) | `drivers activate <Driver> --name "First Last" --email E --trial` |
| Activate (key) | `drivers activate <Driver> --name "First Last" --email E --key KEY` |
| Inspect connection properties | `drivers connectionprops <Driver> [--full]` |
| Generate driver-specific skill | `drivers skill <Driver>` |
| Create connection | `connection create --driver <Driver> --name N --connectionstring CS` |
| List connections | `connection list` |
| Delete connection | `connection delete --name N` |
| List catalogs | `metadata catalogs --connection N` |
| List schemas | `metadata schemas --connection N` |
| List tables | `metadata tables --connection N` |
| Get columns | `metadata columns --connection N --table T` |
| List procedures | `metadata procedures --connection N` |
| Procedure params | `metadata parameters --connection N --procedure P` |
| Run SQL (SELECT/DML) | `query sql --connection N --sql "SQL"` |
| Execute procedure | `query sql --connection N --sql "EXEC Proc Param='val'"` |

Every subcommand supports `--help` — returns purpose, required args, optional args, and flags. Use when Command Reference doesn't cover your case.

---

## Source-Specific Skills

For popular sources, CData drivers ship source-specific instructions (connection patterns, schema notes, query examples). Generate a ready-to-use skill with:

```bash
cdatacli drivers skill <Driver>
```

This prints a ready-to-use YAML skill frontmatter prefix (`name: cdata-<source>` plus a description) followed by the driver's bundled instructions (`resources/instructions.md` inside the jar).

Save the output as a `SKILL.md` in **whichever directory the user's AI coding tool loads skills (or rules/instructions) from** — the right location depends on the tool, so don't assume a fixed path. For example:

- **Claude Code:** `~/.claude/skills/cdata-<source>/SKILL.md` (user-level) or `.claude/skills/cdata-<source>/SKILL.md` (project-level)
- **Other agents** (Cursor, GitHub Copilot, Gemini CLI, Codex, etc.): use that tool's own skills/rules/instructions directory and file convention

If you're not sure where the current tool loads skills from, ask the user before writing the file. Then load the new skill for future work on that source. Example (replace `<skills-dir>` with the correct location for your tool):

```bash
mkdir -p <skills-dir>/cdata-<source>
cdatacli drivers skill <Driver> > <skills-dir>/cdata-<source>/SKILL.md
```

If the driver has no checked-in instructions, the command returns `No instructions available for <driver>` — in that case, proceed with the generic workflow below.

---

## Best Practices

1. **Get columns before querying** — use exact names from schema discovery.
2. **LIMIT first** — sample before full queries.
3. **[Bracket] table and column names** — handles reserved words and spaces.
4. **Filter before joining** — apply WHERE first, add JOINs one at a time.
5. **Date filters prevent timeouts** — large sources time out on unfiltered queries.
6. **Check procedures before concluding impossible** — file operations, bulk exports, auth flows are often stored procedures.
7. **Enum/picklist fields** — look for a dedicated table (e.g. `PickListValues`) before filtering. Don't guess values.

---

## Recommended Workflow

### Step 1: Understand the Goal

Confirm the **source** and **goal** (what to query or accomplish) before proceeding.

If a source-specific SKILL is already installed in the AI tool's skills directory (`cdata-<source>`), invoke it. Otherwise try `cdatacli drivers skill <Driver>` — if it returns content, save it as a new skill in the location appropriate for the user's AI tool (see Source-Specific Skills above) and invoke it. If it returns `No instructions available for <driver>`, continue with the generic workflow.

---

### Step 2: Verify Driver is Available and Activated

```bash
cdatacli drivers list
```

If the driver appears with `"activated": true`, skip to Step 3. If the driver is missing, download it from the CData driver catalog:

```bash
cdatacli drivers search --driver <source>          # find the artifactId
cdatacli drivers download --artifact-id <id>       # downloads to ./lib by default
cdatacli drivers download --url <jar-url>          # alternative: direct URL
```

`drivers download` resolves the URL from CData's published `artifacts.json` catalog and writes the jar to `--output <dir>` (default `./lib/`) — the same location the CLI auto-discovers. Re-run `drivers list` to confirm discovery. As a manual fallback, place a CData JDBC JAR in `./` or `./lib/` next to the CLI executable.

#### Activate

> **⚠️ License-key notice — show this to the user before running `drivers activate --key`:**
>
> It is not recommended to paste a license key into this AI session. A `--trial` activation is safe to run here, but a purchased `--key` is a secret credential — anything typed into the session is visible to the AI and may be persisted in transcripts.
>
> If you need to activate with a purchased key, do it outside the AI session:
> - Run `cdatacli drivers activate <Driver> --name ... --email ... --key ...` in a non-AI terminal, or
> - Double-click `cdata.jdbc.<source>.jar` to activate via the GUI wizard.
>
> Once the driver is activated, the license is saved alongside the jar — `drivers list` will show `"activated": true` and no further activation is needed in the AI session.

The driver is a positional argument (e.g. `Salesforce`). `--name` and `--email` are the registrant's name and email — the individual registering the license. Use `--trial` for a 30-day trial. For a purchased key, follow the notice above.

```bash
cdatacli drivers activate <Driver> --name "John Doe" --email "you@example.com" --trial
cdatacli drivers activate <Driver> --name "John Doe" --email "you@example.com" --key "XXXXX-XXXXX"
```

---

### Step 3: Check Connection Properties

If a saved connection already exists (`connection list`), confirm with the user they would like to use it. Otherwise, inspect the driver's connection properties to determine what credentials to ask for:

```bash
cdatacli drivers connectionprops <Driver>          # basic properties (default)
cdatacli drivers connectionprops <Driver> --full   # advanced properties instead
```

By default this returns the **basic** properties (`jdbcJson.basic`) — the set most connections need. Use `--full` only when the user needs an advanced property not in the basic set (`--full` returns `jdbcJson.advanced`, a flat list of every property). Each property reports:

- `name` — the property to put in the connection string (strip spaces, e.g. `Auth Scheme` → `AuthScheme`)
- `display` — required-ness, e.g. `RequiredBasic` (must provide) vs `UnrequiredBasic` (optional)
- `type` — expected value type
- `description` — what it does
- `default` — value used if you omit the property (shown only when it has one)
- `enum` — allowed values, when the property accepts a fixed set
- `hierarchyRules` — conditional dependencies (see below)

**Hierarchy rules** describe how properties depend on each other. A property's `hierarchyRules` is keyed by that property's possible values, and each key lists the properties that become relevant for that choice. For example, Salesforce's `AuthScheme` has rules for `Basic`, `OAuth`, `OAuthClient`, etc.: choosing `AuthScheme=Basic` surfaces `User` and `Password` (both `RequiredBasic`) plus `SecurityToken` (`UnrequiredBasic`). Use these rules to ask the user only for the properties that the chosen auth scheme actually requires.

Key properties across all sources:
- `AuthScheme` — OAuth, Basic, etc.
- `InitiateOAuth` — OFF, GETANDREFRESH, REFRESH
- `OAuthClientId` / `OAuthClientSecret` — for custom OAuth apps
- `OAuthSettingsLocation` — where OAuth tokens are cached

**Use only the properties the user provides or that `connectionprops` marks as required.** Do not add, infer, or carry over any property the user didn't ask for — including values from a previously created connection.

---

### Step 4: Create a Saved Connection

> **⚠️ Credential notice — show this to the user before running `connection create`:**
>
> It is not recommended to paste passwords, API tokens, or other secret credentials into this AI session. Even though `cdatacli connection create` will accept them in a connection string, anything typed here is visible to the AI and may be persisted in transcripts.
>
> **Recommended for AI-assisted use:**
> - `AuthScheme=OAuth` (embedded OAuth) — credentials never enter the connection string; the browser flow handles auth.
>
> **If your data source requires Basic / PAT / API-token auth, set up the connection outside the AI session:**
> - Run `cdatacli connection create ...` in a non-AI terminal, or
> - Double-click `cdata.jdbc.<source>.jar` to use the GUI setup wizard, or
> - Reuse a connection from a prior non-AI setup.
>
> Afterward, `cdatacli` uses the saved connection by name only — credentials never re-enter the terminal.

Confirm the (non-secret) connection string values with the user, then save:

```bash
cdatacli connection create --driver "<Driver>" --name "<connection-name>" \
  --connectionstring "<properties>"
```

Common patterns:

| Auth Type | Connection String |
|---|---|
| OAuth (browser flow) | `AuthScheme=OAuth;InitiateOAuth=GETANDREFRESH` |
| Basic (user/pass) | `AuthScheme=Basic;User=you@example.com;Password=pass` |
| API Token | `User=you@example.com;APIToken=yourtoken` |
| Read-only | Append `ReadOnly=true` to any connection string |

The first query after creating an OAuth connection opens a browser for authentication. Subsequent queries auto-refresh tokens.

```bash
cdatacli connection list
cdatacli connection delete --name "<connection-name>"
```

Connections are saved as encrypted `.conn` files (AES-256):

| OS | Location |
|---|---|
| Windows | `%APPDATA%\CData\<Driver Name> Data Provider\` |
| macOS | `~/Library/Application Support/CData/<Driver Name> Data Provider/` |
| Linux | `~/.config/CData/<Driver Name> Data Provider/` |

---

### Step 5: Discover Schema

```bash
cdatacli metadata tables --connection <name>
cdatacli query sql --connection <name> --sql "SELECT TableName, TableType, Description FROM sys_tables WHERE TableName LIKE '%<keyword>%'"
cdatacli metadata columns --connection <name> --table <TableName>
cdatacli metadata procedures --connection <name>
cdatacli metadata parameters --connection <name> --procedure <ProcedureName>
cdatacli metadata catalogs --connection <name>
cdatacli metadata schemas --connection <name>
```

Most `metadata` subcommands accept optional `--catalog` / `--schema` filters. `metadata tables` also takes a `--table` name pattern; `metadata columns` requires `--table` and accepts `--column`; `metadata procedures` accepts `--procedure`; `metadata parameters` requires `--procedure` and accepts `--parameter`.

Name patterns use `%` as a wildcard and are **case-insensitive**. A bare value is an **exact match** — `--table Order` returns only `Order`, not `OrderItem`; use `--table "%Order%"` to match substrings. `_` is treated literally (not a wildcard), so `--column "%__c"` matches Salesforce custom fields. Quote any pattern containing `%`.

---

### Step 6: Execute Queries

Use only column names confirmed by previous steps. Build incrementally.

`query sql` accepts two optional limits beyond `--connection` and `--sql`:

- `--timeout <seconds>` — query timeout (default `30`). Raise it for slow or large sources.
- `--max-rows <n>` — maximum rows returned **per result set** (default `1000`). Results are silently capped at this number, so a `SELECT *` on a larger table returns only the first 1000 rows unless you raise `--max-rows`. Use a SQL `LIMIT` to sample deliberately, and raise `--max-rows` when you need a full extract.

#### SELECT

```bash
cdatacli query sql --connection <name> --sql "SELECT * FROM [TableName] LIMIT 5"
cdatacli query sql --connection <name> --sql "SELECT [Id], [Name], [Status] FROM [TableName] WHERE [Status] = 'Active' LIMIT 10"
cdatacli query sql --connection <name> --sql "SELECT a.[Id], a.[Name], b.[Name] AS Related FROM [TableA] a LEFT JOIN [TableB] b ON a.[ForeignKey] = b.[Id] LIMIT 10"
```

**Reading the output.** `query sql` returns the same JSON shape for every data source (the CLI builds it from the result set; `--compact` only removes whitespace). The top-level shape depends on the query kind:

```json
// SELECT (one result set):
{ "metadata": [ { "name": "cnt", "typeName": "INT", ... } ], "resultset": [ { "cnt": 368 } ] }

// INSERT / UPDATE / DELETE, or a statement that returns nothing:
{ "affectedRows": 3 }

// Multiple result sets (batch): each entry is one of the two shapes above
{ "results": [ { "metadata": [...], "resultset": [...] }, { "affectedRows": 2 } ] }
```

- **Row data for a SELECT is always under `resultset`** — an array of row objects keyed by column name. To count rows, count the `resultset` array (above, 1 row whose `cnt` is 368). For batches, read `results[].resultset`.
- `metadata` describes the columns (names, types), not the data — ignore it when reading values.
- There is **no `rows` key**, for any source. A SELECT returned no data only when `resultset` is `[]`. Do not infer "0 rows" from a missing `rows` field, and do not assume the table name is wrong because you can't find row data — look under `resultset`.

Pipe to `jq` with `--compact`, reading rows from `.resultset`:

```bash
cdatacli query sql --connection <name> --sql "SELECT Id, Name FROM Account" --compact | jq '.resultset[].Name'
```

#### INSERT / UPDATE / DELETE

```bash
cdatacli query sql --connection <name> --sql "INSERT INTO [TableName] (Col1, Col2) VALUES ('val1', 'val2')"
cdatacli query sql --connection <name> --sql "UPDATE [TableName] SET [Col1] = 'new' WHERE [Id] = '123'"
cdatacli query sql --connection <name> --sql "DELETE FROM [TableName] WHERE [Id] = '123'"
```

If writes fail, the connection may have `ReadOnly=true`. Do not change this on your own — point it out to the user and ask whether they want to recreate the connection without `ReadOnly=true` before taking any action.

#### Stored Procedures

```bash
cdatacli query sql --connection <name> --sql "EXEC ProcedureName Param1='value1', Param2='value2'"
```

---

### Step 7: Generate Application Code

If the user's goal includes generating application code, use the validated SQL, driver path (from `drivers list`), and connection string to generate standalone code.

#### Driver Locations

**JDBC (Java):**
- Windows: `C:\Program Files\CData\CData JDBC Driver for <DataSource> <Year>\lib\cdata.jdbc.<datasource>.jar`
- macOS: `/Applications/CData/CData JDBC Driver for <DataSource> <Year>/lib/cdata.jdbc.<datasource>.jar`
- CLI-bundled: `./` or `./lib/cdata.jdbc.<datasource>.jar` next to the CLI executable
- Driver class: `cdata.jdbc.<source>.<Source>Driver`
- JDBC URL: `jdbc:<source>:<connection-string>`
- License file: same directory, `cdata.jdbc.<datasource>.lic`
