---
name: cdata-cli-adonet
description: "Use when writing C# / .NET application code (also VB.NET, F#) that connects to a data source through a CData driver — after the connection and SQL have been validated with the cdata-cli discovery skill. Covers mapping the language to the CData ADO.NET Data Provider edition, adding the provider from NuGet (CData.<Source>), activating its license, and connecting via the ADO.NET classes (<Source>Connection / <Source>Command). This is the .NET build-phase companion to cdata-cli; invoke it whenever the target application runs on .NET."
license: MIT
metadata:
  author: CData Software
  version: "1.0"
---

# SKILL: CData CLI — .NET / ADO.NET Build

Build-phase companion to the **`cdata-cli`** skill. `cdata-cli` handles discovery
(connect, explore schema, validate SQL) using the JDBC driver; **this skill takes over
once you start writing .NET application code** using the **CData ADO.NET Data Provider**.

Everything validated during discovery carries straight over: all CData drivers share the
same relational model and SQL dialect, so table names, column names, and SQL confirmed via
`cdatacli` work unchanged through the ADO.NET provider.

---

## Language / Driver Edition

| Language | CData edition | .NET namespace |
|---|---|---|
| **C#** (also VB.NET, F#) | **CData ADO.NET Data Provider** | `System.Data.CData.<Source>` |

The provider follows standard ADO.NET naming — for `<Source>` (e.g. `Salesforce`):

- `<Source>Connection`, `<Source>Command`, `<Source>DataReader`, `<Source>DataAdapter`,
  `<Source>Parameter`, `<Source>CommandBuilder`, `<Source>ConnectionStringBuilder`
- Assembly/DLL: `System.Data.CData.<Source>.dll`

---

## Prerequisite: Discovery Done First

**If `cdata-cli` has not been used yet in this session, stop and invoke it first.** This skill only covers the build phase — connection setup, schema discovery, and SQL validation must be done through `cdata-cli` before continuing here.

This skill assumes `cdata-cli` has already been used to confirm the source connects and to
validate the exact tables/columns and SQL the app will run. Reuse those results here. If
discovery created a `cdatacli` connection that did an **OAuth** browser sign-in, note its
`OAuthSettingsLocation` — the .NET app can point at the same cached token file and skip
re-authenticating (see **Connect**).

> **For runtime, the CLI works directly with Java/JDBC only.** It cannot install, activate,
> or create connections for the ADO.NET Data Provider — that is what this skill covers. The
> ADO.NET provider is a separate runtime driver edition and must be obtained and licensed
> independently (steps below).
>
> The CLI's `drivers download` fetches JDBC jars only; the ADO.NET provider comes from
> NuGet (below).

---

## Step 1: Get the Provider (NuGet)

The ADO.NET provider is published on NuGet by **CDataSoftware** as **`CData.<Source>`**
(e.g. `CData.Salesforce`). The idiomatic install is a package reference in the project:

```bash
dotnet add package CData.<Source>
```

- Grab the plain **`CData.<Source>`** package — that's the ADO.NET provider. The
  `CData.<Source>.EntityFrameworkCore*` packages are **EF Core add-ons**; only add those if
  you're using Entity Framework.
- To pin a version: `dotnet add package CData.<Source> --version <x.y.z>`.

**Already installed locally?** The standalone ADO.NET provider *product* is a **separate
distribution** from the NuGet package and may expose a **different set of target folders**.
If it's installed on the machine, you can reference its DLL directly instead of pulling
from NuGet:

```
C:\Program Files\CData\CData ADO.NET Provider for <Source> <Year>\lib\<target>\System.Data.CData.<Source>.dll
```

where `<target>` is one of the folders present under `lib\` (e.g. `net8.0`, `net6.0`,
`netstandard2.0`).

---

## Step 2: Reference It & Pick a Target Framework

The NuGet package ships its build targets under `lib/`. The set can vary by connector and
version, so **check the package's `lib/` folder**; for the connector verified here it was
`netstandard2.0` and `net40`:

| Target in package | Use for |
|---|---|
| `netstandard2.0` | .NET Core / .NET 5+ (net6.0, net8.0, net10.0, …) — **needs license activation (Step 3)** |
| `net40` | .NET Framework (4.x) — **self-licenses on NuGet install** |

**Pick the `TargetFramework` from the installed SDK, not the runtime list.** `dotnet new
--framework` and the build only accept TFMs the installed **SDK** knows — a box with only
the .NET 10 SDK rejects `net6.0`/`net8.0` even though those *runtimes* exist. Check
`dotnet --version` (or `dotnet --list-sdks`) and default to the SDK's matching TFM (e.g.
.NET 10 SDK → `net10.0`, or `net10.0-windows` for a WinForms/WPF UI); since an SDK ships
its own runtime, that target both builds and runs.

Only if you deliberately target an **older** TFM do you also need that TFM's runtime
installed (`dotnet --list-runtimes`) — otherwise launch fails with *"You must install or
update .NET to run this application."* The `netstandard2.0` provider itself works across
net6.0 / net8.0 / net10.0, so the limiting factor is your toolchain, not the driver.

Two ways to reference the provider:

- **PackageReference** (from `dotnet add package`) — simplest; NuGet restores the DLL.
- **Direct `HintPath`** — reference the extracted `netstandard2.0\System.Data.CData.<Source>.dll`.
  Useful when you want the DLL (and its activated `.lic`) in a known local path:

```xml
<ItemGroup>
  <Reference Include="System.Data.CData.<Source>">
    <HintPath>..\packages\CData.<Source>\lib\netstandard2.0\System.Data.CData.<Source>.dll</HintPath>
  </Reference>
</ItemGroup>
```

---

## Step 3: Activate the License

> **A working `cdatacli` connection doesn't mean this is activated.** `cdatacli` runs on the
> JDBC driver, so it only proves *JDBC* is licensed — the ADO.NET provider is a separate license.

Licensing depends on the build target and is **per machine**, separate from the
JDBC/Python/ODBC editions:

- **.NET Framework (`net40`)** — the NuGet package **auto-installs a license on restore**;
  no action needed.
- **.NET Core / .NET (`netstandard2.0`)** — **must be activated** with the `install-license`
  tool bundled in the package's `tools/` folder, or the first query raises *"Could not find
  a valid license…"*.

The tool is a .NET app. Its folder holds `install-license.dll`, `install-license.runtimeconfig.json`,
and `prod.inf`; find it under the package:

```
<project>\packages\CData.<Source>\tools\install-license.dll          # extracted package
%USERPROFILE%\.nuget\packages\cdata.<source>\<ver>\tools\install-license.dll   # global NuGet cache
```

It reads `prod.inf` from the current directory, so **run it from its own folder**. The tool
always prompts for `Name:` and `Email:` — it crashes with an unhandled exception if either
is blank. There are no CLI flags for them; they are always supplied interactively.

**Trial:** ask the user for their name and email, then pipe them in and run the tool
directly — no secrets are involved:

```powershell
# PowerShell — pipe name and email, run from the tools folder
cd "<project>\packages\CData.<Source>\tools"
"First Last`nname@example.com" | dotnet .\install-license.dll
```

```bash
# bash
cd "<project>/packages/CData.<Source>/tools"
printf "First Last\nname@example.com\n" | dotnet ./install-license.dll
```

**Purchased key** (the key is a secret — have the user run this outside the AI session):

```powershell
cd "<project>\packages\CData.<Source>\tools"
dotnet .\install-license.dll YOUR-PRODUCT-KEY
# Prompts: Name: <enter name>
#          Email: <enter email>
```

Activation writes `System.Data.CData.<Source>.lic` next to the provider DLL
(`lib\netstandard2.0\`). If you reference the DLL by `HintPath`, keep the `.lic` beside
that copy.

---

## Step 4: Connect (ADO.NET)

Standard ADO.NET usage — and unlike some other editions, these objects **do** implement
`IDisposable`, so `using` blocks are the right pattern:

```csharp
using System.Data.CData.Salesforce;

using var conn = new SalesforceConnection(connectionString);
conn.Open();

using var cmd = new SalesforceCommand(
    "SELECT Id, Name FROM Account WHERE Name LIKE @name", conn);
cmd.Parameters.Add(new SalesforceParameter("@name", "%Acme%"));

using var reader = cmd.ExecuteReader();
while (reader.Read())
    Console.WriteLine(reader["Name"]);
```

- **Named parameters** — placeholders are `@name` and you add `<Source>Parameter` objects.
  Always parameterize user input.
- `ExecuteNonQuery()` for INSERT/UPDATE/DELETE; `ExecuteScalar()` for single values;
  `<Source>DataAdapter` + `DataTable` to fill a grid.

### Connection string & auth (reuse discovery, keep secrets out of code)

Pass the same properties validated with `cdatacli`, from config (e.g. `appsettings.json`),
not hard-coded. For **OAuth** sources, point `OAuthSettingsLocation` at the token file the
discovery connection already populated so the app refreshes silently, no browser flow:

```text
AuthScheme=OAuth;InitiateOAuth=GETANDREFRESH;UseSandbox=true;OAuthSettingsLocation=C:\...\OAuthSettings-xxx.txt
```

Sources with **no embedded OAuth app** require the user's own `OAuthClientId`/
`OAuthClientSecret` even to refresh — read those from environment variables / user-secrets,
never commit them.

---

## Step 5: End-to-End Example

A minimal console/repository slice (wrap in WinForms/WPF/ASP.NET as needed):

```csharp
using System.Data;
using System.Data.CData.Salesforce;

const string CS =
    @"AuthScheme=OAuth;InitiateOAuth=GETANDREFRESH;UseSandbox=true;OAuthSettingsLocation=C:\...\OAuthSettings-sbx.txt";

static IEnumerable<(string Id, string Name)> ListAccounts(string? search, int limit = 50)
{
    using var conn = new SalesforceConnection(CS);
    conn.Open();
    var sql = "SELECT Id, Name FROM Account"
              + (string.IsNullOrWhiteSpace(search) ? "" : " WHERE Name LIKE @s")
              + $" ORDER BY Name LIMIT {limit}";
    using var cmd = new SalesforceCommand(sql, conn);
    if (!string.IsNullOrWhiteSpace(search))
        cmd.Parameters.Add(new SalesforceParameter("@s", $"%{search}%"));
    using var r = cmd.ExecuteReader();
    while (r.Read())
        yield return (r["Id"].ToString()!, r["Name"].ToString()!);
}

static void CreateAccount(string name, string? industry)
{
    using var conn = new SalesforceConnection(CS);
    conn.Open();
    using var cmd = new SalesforceCommand(
        "INSERT INTO Account (Name, Industry) VALUES (@n, @i)", conn);
    cmd.Parameters.Add(new SalesforceParameter("@n", name));
    cmd.Parameters.Add(new SalesforceParameter("@i", (object?)industry ?? DBNull.Value));
    cmd.ExecuteNonQuery();
}
```

Build to confirm the reference resolves: `dotnet build`.

---

## Troubleshooting

| Error | Cause / fix |
|---|---|
| `Could not find a valid license for using CData ADO.NET Provider for <source>` | The `netstandard2.0` build isn't activated on this machine — run `install-license` from the package `tools/` folder (Step 3). `net40` self-licenses, so this only affects .NET Core / .NET targets. |
| `You must install or update .NET to run this application` | The target TFM's **runtime** isn't installed. Simplest: target the SDK's own version (Step 2), which ships a matching runtime; otherwise install the missing runtime. |
| `'OAuthClientId' and 'OAuthClientSecret' are needed to initiate OAuth` | Source has no embedded OAuth app; supply client id/secret in the connection string (from env vars / user-secrets), matching the app whose refresh token is cached. |
| Reference/`using System.Data.CData.<Source>` not found | Package not restored or wrong package. Ensure `dotnet add package CData.<Source>` (the plain provider, not an `EntityFrameworkCore` variant) and `dotnet restore`. |
| `install-license.dll` crashes (`NullReferenceException`) | Name or Email was blank. The tool has no CLI flags for them — pipe them in for trial (Step 3), or enter them at the prompt for a purchased key. Always run from the `tools/` folder so it can find `prod.inf`. |
