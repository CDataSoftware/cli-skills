---
name: cdata-cli-odbc
description: "Use when writing application code in Node.js / JavaScript / TypeScript — or any non-JVM language (Go, PHP, Ruby, Rust, C/C++) — that connects to a data source through a CData driver, after the connection and SQL have been validated with the cdata-cli discovery skill. Covers the CData ODBC Driver edition: confirming the system-installed driver, configuring a DSN or a DSN-less connection string, and connecting via the language's ODBC bindings (e.g. the Node `odbc` package). This is the cross-language build-phase companion to cdata-cli — the fallback for any environment that speaks ODBC."
license: MIT
metadata:
  author: CData Software
  version: "1.0"
---

# SKILL: CData CLI — ODBC Build

Build-phase companion to the **`cdata-cli`** skill. `cdata-cli` handles discovery
(connect, explore schema, validate SQL) using the JDBC driver; **this skill takes over
once you start writing application code** against the **CData ODBC Driver** — the
cross-language path for Node.js and any non-JVM runtime that speaks ODBC.

Everything validated during discovery carries straight over: all CData drivers share the
same relational model and SQL dialect, so table names, column names, and SQL confirmed via
`cdatacli` work unchanged through ODBC.

---

## Language / Driver Edition

| Language | CData edition | ODBC binding |
|---|---|---|
| **Node.js / JS / TS** | **CData ODBC Driver** | `odbc` (npm) |
| **PHP** | CData ODBC Driver | `PDO_ODBC` (bundled PHP extension) |
| **C / C++** | CData ODBC Driver | native ODBC API (`sql.h`, `sqlext.h`) |
| **Go** | CData ODBC Driver | `database/sql` + a community ODBC driver package |
| **Ruby** | CData ODBC Driver | a community ODBC gem |
| **Rust** | CData ODBC Driver | a community ODBC crate |

The ODBC **driver** is CData's; the **binding** that lets a language call ODBC is part of
that language's own ecosystem — **not published by CData**. Some are effectively standard
(Node's `odbc`, PHP's `PDO_ODBC`, the C/C++ ODBC API); others (Go, Ruby, Rust) rely on a
**community** package, since Go's `database/sql`, etc. provide only a generic database
interface with no built-in ODBC implementation. Confirm the current, maintained binding for
the target language rather than assuming a specific package. Only the **Node `odbc`** path
is shown as concrete example code below.

ODBC is the **cross-language fallback**. If a language has a first-class CData edition,
prefer it — **Python → `cdata-cli-python`**, **.NET → `cdata-cli-adonet`**; reach for ODBC
for languages that have no dedicated edition (Node.js, Go, PHP, Ruby, C/C++, Rust) or when
ODBC is specifically required.

---

## Prerequisite: Discovery Done First

**If `cdata-cli` has not been used yet in this session, stop and invoke it first.** This skill only covers the build phase — connection setup, schema discovery, and SQL validation must be done through `cdata-cli` before continuing here.

This skill assumes `cdata-cli` has already been used to confirm the source connects and to
validate the exact tables/columns and SQL the app will run. Reuse those results here. If
discovery created a `cdatacli` connection that did an **OAuth** browser sign-in, note its
`OAuthSettingsLocation` — the app's DSN / connection string can point at the same cached
token file and skip re-authenticating (see **Configure the Connection**).

> **The CLI is JDBC-only.** It cannot install, activate, or create connections for the
> ODBC Driver — that is what this skill covers. The ODBC Driver is a separate driver
> edition installed as a system driver; the CLI has no role in obtaining or managing it.

---

## Step 1: Confirm the ODBC Driver Is Installed

Unlike the JDBC jar (maven) or the ADO.NET provider (NuGet), the **ODBC driver is a
system-installed driver** registered with the OS ODBC Driver Manager. It is **not** fetched
from a package repo — it's installed from a CData ODBC setup package (installer/`.msi` on
Windows; distro package on Linux/macOS), and `cdatacli` has no role in installing it.

Check whether it's already present:

```powershell
# Windows
Get-OdbcDriver | Where-Object Name -like "*CData*"
# e.g. "CData ODBC Driver for Salesforce"  (note the 32-bit / 64-bit rows)
```

```bash
# Linux/macOS (unixODBC)
odbcinst -q -d        # list registered driver names
```

The installed driver DLL/so typically lives at (Windows example):

```
C:\Program Files\CData\CData ODBC Driver for <Source>\lib64\CData.ODBC.<Source>.dll   # 64-bit
```

> **Bitness must match.** The app process, the ODBC Driver Manager, and the driver must all
> be the same architecture. A 64-bit Node/Go/PHP process needs the **64-bit** CData ODBC
> driver and a DSN created in the 64-bit registry hive. Mixing 32/64-bit is the most common
> ODBC failure (see Troubleshooting).

If the driver isn't installed, the user must install the CData ODBC driver for the source
first (from their CData download), then re-check.

---

## Step 2: License

> **A working `cdatacli` connection doesn't mean this is activated.** `cdatacli` runs on the
> JDBC driver, so it only proves *JDBC* is licensed — the ODBC driver is a separate license.

The CData ODBC driver is **licensed as part of its installation** (the setup wizard / CData
license tool applies a trial or product key), and the license is **per machine**, separate
from the JDBC/ADO.NET/Python editions. On a machine where the driver is already installed
and licensed, no further action is needed.

If queries fail with a license error, the user should re-activate the ODBC driver through
its installer / CData license tool (a product key is a secret — do this **outside the AI
coding session**). Activation specifics are installer- and platform-driven, so defer to the
CData ODBC driver's own setup rather than a scripted step here.

---

## Step 3: Configure the Connection

First, check whether a DSN for this source already exists — if so, confirm with the user whether to reuse it:

```powershell
# Windows — list all User and System DSNs
Get-OdbcDsn | Where-Object Name -like "*<Source>*"
# or list everything: Get-OdbcDsn
```

```bash
# Linux/macOS
odbcinst -q -s    # list all registered DSN names
```

If a suitable DSN exists, skip to Step 4 and connect using `DSN=<name>`. Otherwise, set one up below.

Two ways to connect — a **DSN** (named, stored config) or **DSN-less** (full connection
string in code). Either way, reuse the properties validated in discovery, and for OAuth
sources point `OAuthSettingsLocation` at the discovery token cache so no secrets live in the
DSN/string and no browser flow is triggered.

### Option A — DSN (recommended for reuse)

A **User DSN** needs no admin and is scoped to the current user; a **System DSN** is
machine-wide and needs admin. Create a User DSN reusing the cached OAuth token:

```powershell
# Windows — creates a 64-bit User DSN (no secrets stored: token comes from the cache file)
Add-OdbcDsn -Name "MySource SBX" -DriverName "CData ODBC Driver for <Source>" `
  -DsnType User -Platform "64-bit" -SetPropertyValue @(
    "AuthScheme=OAuth",
    "InitiateOAuth=GETANDREFRESH",
    "OAuthSettingsLocation=C:\...\CData\<Source> Data Provider\OAuthSettings-xxx.txt"
  )
```

```ini
# Linux/macOS (unixODBC): register the driver in odbcinst.ini, then a DSN in odbc.ini
# ~/.odbc.ini
[MySource SBX]
Driver = CData ODBC Driver for <Source>
AuthScheme = OAuth
InitiateOAuth = GETANDREFRESH
OAuthSettingsLocation = /home/you/.cdata/OAuthSettings-xxx.txt
```

Connect with just the name: `DSN=MySource SBX`.

### Option B — DSN-less (self-contained)

Put the driver name and all properties in the connection string — nothing to pre-register:

```text
Driver={CData ODBC Driver for <Source>};AuthScheme=OAuth;InitiateOAuth=GETANDREFRESH;OAuthSettingsLocation=C:\...\OAuthSettings-xxx.txt
```

The `Driver={...}` value must be the exact driver name from Step 1.

> **Secrets:** embedded-OAuth sources (e.g. Salesforce) need only the cached token.
> Sources with no embedded OAuth app (e.g. Gmail) require the user's own
> `OAuthClientId`/`OAuthClientSecret` even to refresh — supply those from environment
> variables at runtime, never in a checked-in DSN or string.

---

## Step 4: Connect From Code

### Node.js (`odbc`)

`npm install odbc` pulls a **prebuilt binary** — no C/C++ compiler needed on common
platforms.

```javascript
import odbc from "odbc";

// DSN or DSN-less connection string both work:
const CS = process.env.SF_ODBC_DSN || "DSN=MySource SBX";

const pool = await odbc.pool(CS);           // pool for a server app
const rows = await pool.query(
  "SELECT Id, Name FROM Account WHERE Name LIKE ?", ["%Acme%"]);   // ? placeholders
console.log(rows.length, rows[0]?.Name);
```

- **`?` placeholders** (qmark), values passed as an array — always parameterize input.
- `pool.query` for reads; the same for INSERT/UPDATE/DELETE. Use `odbc.connect(CS)` for a
  single short-lived connection.

### Other languages

The pattern is identical everywhere ODBC is available: connect with `DSN=<name>` (or a
DSN-less `Driver={...}` string), execute SQL with `?` parameter markers, iterate results.
Use the target language's own ODBC binding (see the table above), and the same SQL you
validated with `cdatacli`.

---

## Step 5: End-to-End Example (Node.js)

A minimal repository slice (wrap in Express/Fastify/etc. as needed):

```javascript
import odbc from "odbc";

const CS = process.env.SF_ODBC_DSN || "DSN=MySource SBX";
let poolPromise = null;
const getPool = () => (poolPromise ??= odbc.pool(CS));

export async function listAccounts(search, limit = 50) {
  const pool = await getPool();
  let sql = "SELECT Id, Name FROM Account";
  const params = [];
  if (search) { sql += " WHERE Name LIKE ?"; params.push(`%${search}%`); }
  sql += ` ORDER BY Name LIMIT ${Number(limit)}`;
  return pool.query(sql, params);
}

export async function createAccount(name, industry = null) {
  const pool = await getPool();
  return pool.query("INSERT INTO Account (Name, Industry) VALUES (?, ?)", [name, industry]);
}
```

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Data source name not found and no default driver specified` / `architecture mismatch` | Bitness mismatch, or DSN registered in the wrong hive. Ensure app process, Driver Manager, and driver are all 64-bit (or all 32-bit) and the DSN exists in the matching hive. |
| Driver name not found (DSN-less) | The `Driver={...}` name doesn't match an installed driver. Use the exact name from `Get-OdbcDriver` / `odbcinst -q -d`. |
| License error at query time | ODBC driver not licensed on this machine — re-activate via the CData ODBC installer / license tool (Step 2), outside the AI session. |
| `'OAuthClientId' and 'OAuthClientSecret' are needed to initiate OAuth` | Source has no embedded OAuth app; add client id/secret (from env vars) to the DSN/connection string, matching the app whose refresh token is cached. |
| `npm install odbc` tries to compile / fails | Usually a prebuilt binary is used; if compilation is triggered, ensure a supported Node version and platform, or install build tools for the fallback native build. |
