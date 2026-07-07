---
name: cdata-cli-python
description: "Use when writing Python application code that connects to a data source through a CData driver — after the connection and SQL have been validated with the cdata-cli discovery skill. Covers mapping the language to the CData Python Connector edition, downloading the connector wheel from CData's Python repository, activating its license, and connecting via DB-API 2.0 (cdata.<source>.connect). This is the Python build-phase companion to cdata-cli; invoke it whenever the target application language is Python (Salesforce, Snowflake, Jira, Gmail, etc.)."
license: MIT
metadata:
  author: CData Software
  version: "1.0"
---

# SKILL: CData CLI — Python Build

Build-phase companion to the **`cdata-cli`** skill. `cdata-cli` handles discovery
(connect, explore schema, validate SQL) using the JDBC driver; **this skill takes over
once you start writing Python application code** using the **CData Python Connector**.

Everything validated during discovery carries straight over: all CData drivers share the
same relational model and SQL dialect, so table names, column names, and SQL confirmed via
`cdatacli` work unchanged through the Python Connector.

---

## Language / Driver Edition

| Language | CData edition | Python import |
|---|---|---|
| **Python 3.x** | **CData Python Connector** | `import cdata.<source>` |

`<source>` is the lowercased driver name — e.g. `cdata.salesforce`, `cdata.gmail`,
`cdata.snowflake`. The connector is a compiled DB-API 2.0 module (`.pyd`/`.so`) plus
SQLAlchemy dialects, shipped as a platform-specific wheel.

---

## Prerequisite: Discovery Done First

This skill assumes `cdata-cli` has already been used to:

- confirm the source connects, and
- validate the exact tables/columns and the SQL the app will run.

Reuse those results here. In particular, if discovery created a `cdatacli` connection that
performed an **OAuth** browser sign-in, note its `OAuthSettingsLocation` — the Python app
can point at the same cached token file and skip re-authenticating (see **Connect**).

> The CLI's `drivers download` is JDBC-only and cannot fetch the Python wheel. The wheel
> comes from CData's Python repository over plain HTTP (below).

---

## Step 1: Get the Connector Wheel

CData Python Connectors are published at **`https://maven.cdata.com/python/`** as a
browseable index (plain HTTP — no API, no auth).

1. Find the artifact: `cdata-<source>-connector` (e.g. `cdata-salesforce-connector`).
2. Open the newest version folder (e.g. `26.0.9655`).
3. Pick the wheel matching the target OS/arch:

| Platform | Wheel suffix |
|---|---|
| Windows x64 | `...-win_amd64.whl` |
| Linux x64 | `...-linux_x86_64.whl` |
| macOS Apple Silicon | `...-macosx_<ver>_arm64.whl` |

The `cp310-abi3` tag means **Python 3.10+** with the stable ABI — one wheel covers all
Python ≥ 3.10.

Download it (validate the file — the server occasionally drops a connection mid-transfer):

```bash
# bash
curl -fSL --retry 3 --retry-all-errors -o cdata_<source>_connector-<ver>-cp310-abi3-win_amd64.whl \
  "https://maven.cdata.com/p/python/cdata-<source>-connector/<ver>/cdata_<source>_connector-<ver>-cp310-abi3-win_amd64.whl"
```

```powershell
# PowerShell — curl.exe is more robust here than Invoke-WebRequest
curl.exe -fSL --retry 3 --retry-all-errors -o "cdata_<source>_connector-<ver>-cp310-abi3-win_amd64.whl" `
  "https://maven.cdata.com/p/python/cdata-<source>-connector/<ver>/cdata_<source>_connector-<ver>-cp310-abi3-win_amd64.whl"
```

A `.whl` is a zip; if it won't open as an archive it truncated — re-download.

**Already installed locally?** If the connector was installed via a CData product bundle
you can use that wheel instead of downloading — look under the extraction path:

```
<install-path>\CData.Python.<Source>\win\Python<ver>\64\cdata_<source>_connector-...whl   # Windows
<install-path>/CData.Python.<Source>/linux/cdata_<source>_connector-...tar.gz             # Linux/macOS
```

---

## Step 2: Install the Connector

Install the wheel into the interpreter the app will run:

```bash
python -m pip install cdata_<source>_connector-<ver>-cp310-abi3-win_amd64.whl
```

A virtualenv is **recommended** (keeps the install project-local and the license-tool path
predictable) but **not required** — a global install works the same:

```bash
python -m venv .venv    # optional; activate, then pip install into it
```

Verify the module imports:

```bash
python -c "import cdata.<source> as m; print('paramstyle:', m.paramstyle, '| apilevel:', m.apilevel)"
# expect: paramstyle: qmark | apilevel: 2.0
```

---

## Step 3: Activate the License

> **A working `cdatacli` connection doesn't mean this is activated.** `cdatacli` runs on the
> JDBC driver, so it only proves *JDBC* is licensed — the Python Connector is a separate license.

The Python Connector is **licensed separately** from the JDBC/ADO.NET/ODBC editions, and
**per machine**. Without activation the first query raises *"Could not find a valid
license…"*.

The activator ships inside the installed package, in the active interpreter's
`site-packages/cdata/installlic_<source>/` (locate it with
`python -c "import cdata, os; print(os.path.dirname(cdata.__file__))"`):

```
installlic_<source>/install-license.exe   # Windows
installlic_<source>/install-license.sh    # macOS/Linux
```

It is **interactive** and must run **from its own folder** (it reads `prod.inf` from the
current directory). Because it prompts for input, **have the user run it in a normal
terminal — outside the AI coding session** (an AI session can't reliably answer the
prompts, and it hangs if run with no arguments).

**Trial:** run it and answer the `Name:` and `Email:` prompts:

```bash
cd <installlic_<source> folder>       # path located above
.\install-license.exe                 # Windows;  macOS/Linux: ./install-license.sh
# then answer the Name: and Email: prompts
```

**Purchased key (do this outside an AI session — the key is a secret):**

```bash
cd <installlic_<source> folder>
.\install-license.exe YOUR-PRODUCT-KEY   # Windows;  macOS/Linux: ./install-license.sh YOUR-PRODUCT-KEY
```

Re-run the import/query check after activation to confirm it took.

---

## Step 4: Connect (DB-API 2.0)

```python
import cdata.<source> as mod

conn = mod.connect(CONNECTION_STRING)   # a full CData connection string
cur = conn.cursor()
cur.execute("SELECT Id, Name FROM Account WHERE Name LIKE ?", ["%Acme%"])
rows = cur.fetchall()
```

- **`paramstyle` is `qmark`** — use `?` placeholders and pass a list to `execute`. Always
  parameterize user input; never string-format SQL.
- Call **`conn.commit()`** after INSERT/UPDATE/DELETE.


### Connection string & auth (reuse discovery, keep secrets out of code)

Pass the same properties you validated with `cdatacli`. For **OAuth** sources, point
`OAuthSettingsLocation` at the token file the discovery connection already populated so the
app refreshes silently with no browser flow:

```python
# Embedded-OAuth source (e.g. Salesforce) — cached token is sufficient:
cs = ("AuthScheme=OAuth;InitiateOAuth=GETANDREFRESH;UseSandbox=true;"
      r"OAuthSettingsLocation=C:\...\CData\Salesforce Data Provider\OAuthSettings-xxx.txt")
```
---

## Step 5: End-to-End Example

Framework-agnostic (wrap in Flask/FastAPI/Tkinter/etc. as needed):

```python
import cdata.salesforce as sf

CS = ("AuthScheme=OAuth;InitiateOAuth=GETANDREFRESH;UseSandbox=true;"
      r"OAuthSettingsLocation=C:\...\OAuthSettings-sbx.txt")

def get_conn():
    return sf.connect(CS)

def list_accounts(search=None, limit=50):
    conn = get_conn()
    try:
        cur = conn.cursor()
        sql = "SELECT Id, Name, Industry FROM Account"
        params = []
        if search:
            sql += " WHERE Name LIKE ?"; params.append(f"%{search}%")
        sql += f" ORDER BY Name LIMIT {int(limit)}"
        cur.execute(sql, params)
        return cur.fetchall()
    finally:
        conn.close()

def create_account(name, industry=None):
    conn = get_conn()
    try:
        cur = conn.cursor()
        cur.execute("INSERT INTO Account (Name, Industry) VALUES (?, ?)", [name, industry])
        conn.commit()
    finally:
        conn.close()

if __name__ == "__main__":
    for row in list_accounts(limit=5):
        print(row)
```

---

## SQLAlchemy (optional)

The wheel also installs SQLAlchemy dialects, so ORM/engine code works too:

- `cdata.sqlalchemy_<source>` — SQLAlchemy 1.4-style dialect
- `cdata.sqlalchemy2_<source>` — SQLAlchemy 2.0-style dialect

```python
from sqlalchemy import create_engine, text
# URL form: <source>:///?<connection-string-properties>
engine = create_engine("salesforce:///?AuthScheme=OAuth;InitiateOAuth=GETANDREFRESH;UseSandbox=true")
with engine.connect() as c:
    print(c.execute(text("SELECT COUNT(*) FROM Account")).scalar())
```

---

## Troubleshooting

| Error | Cause / fix |
|---|---|
| `Could not find a valid license for using CData Python Driver for <source>` | Python Connector not activated on this machine — run `install-license` (Step 3). Separate from any JDBC/ADO.NET/ODBC license. |
| `'OAuthClientId' and 'OAuthClientSecret' are needed to initiate OAuth` | Source has no embedded OAuth app; supply client id/secret in the connection string (from env vars). Must match the app whose refresh token is cached. |
| `object does not support the context manager protocol` | Used `with mod.connect(...)`. Replace with explicit `try/finally`|
| `ModuleNotFoundError: No module named 'cdata.<source>'` | Wheel not installed in the active interpreter — confirm the venv is active / the wheel installed, then re-check `import cdata.<source>`. |
| `install-license.exe` hangs or errors on `prod.inf` | Run it **from its own folder** with a `Name`/`Email` (trial) or a product key argument; don't run it with no arguments. |
