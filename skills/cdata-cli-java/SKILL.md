---
name: cdata-cli-java
description: "Use when writing Java (or other JVM language — Kotlin, Scala) application code that connects to a data source through a CData driver, after the connection and SQL have been validated with the cdata-cli discovery skill. Covers the CData JDBC Driver edition: reusing the same jar cdata-cli already downloaded and activated, putting it on the classpath (or adding it as a Maven/Gradle dependency), and connecting via JDBC (DriverManager, the jdbc:<source>: URL, PreparedStatement). This is the JVM build-phase companion to cdata-cli."
license: MIT
metadata:
  author: CData Software
  version: "1.0"
---

# SKILL: CData CLI — Java / JDBC Build

Build-phase companion to the **`cdata-cli`** skill. `cdata-cli` handles discovery
(connect, explore schema, validate SQL) using the JDBC driver; **this skill takes over
once you start writing JVM application code** using that same **CData JDBC Driver**.

This is the most continuous of the build skills: the JDBC jar an app needs is **exactly the
jar `cdata-cli` already downloaded and activated** during discovery. Everything validated
there — tables, columns, SQL — runs unchanged through JDBC.

---

## Language / Driver Edition

| Language | CData edition | Entry points |
|---|---|---|
| **Java** (also Kotlin, Scala, other JVM) | **CData JDBC Driver** | Driver class `cdata.jdbc.<source>.<Source>Driver`; URL `jdbc:<source>:<connection-string>` |

`<source>` is the lowercased driver name (e.g. `salesforce`) and `<Source>` its cased form
(e.g. `Salesforce`): jar `cdata.jdbc.salesforce.jar`, class
`cdata.jdbc.salesforce.SalesforceDriver`, URL prefix `jdbc:salesforce:`.

---

## Prerequisite: Discovery Done First

This skill assumes `cdata-cli` has already been used to confirm the source connects and to
validate the tables/columns and SQL the app will run. Because the CLI runs on JDBC, two
things are usually **already done** for you:

- **The jar is already downloaded** — `cdatacli drivers download` placed
  `cdata.jdbc.<source>.jar` in `./lib/` (or you know the installed-product path).
- **The jar is likely already licensed** — if discovery ran `cdatacli drivers activate`,
  `drivers list` shows `"activated": true` and the same jar is ready for the app.

If discovery created an **OAuth** connection, note its `OAuthSettingsLocation` — the app's
JDBC URL can point at the same cached token file and skip re-authenticating (see
**Connect**).

---

## Step 1: Get the Driver Jar

Prefer the jar you already have from discovery; otherwise pick one of the alternatives.

**A. Reuse the CLI jar (simplest).** It's the exact driver, already local:

```
./lib/cdata.jdbc.<source>.jar        # where cdatacli drivers download put it
```

**B. Installed-product path** (if the JDBC driver product is installed):

```
C:\Program Files\CData\CData JDBC Driver for <Source> <Year>\lib\cdata.jdbc.<datasource>.jar   # Windows
/Applications/CData/CData JDBC Driver for <Source> <Year>/lib/cdata.jdbc.<datasource>.jar       # macOS
```

**C. Maven/Gradle dependency** from CData's JDBC maven repo. Coordinates follow the CLI
catalog URL — repo base `https://maven.cdata.com/p/jdbc/`, groupId `cdata`, artifactId
`<source>-jdbc`. Confirm the exact artifactId/version with `cdatacli drivers search --driver <source>`:

```kotlin
// Gradle (Kotlin DSL) — verify coordinates against `cdatacli drivers search` output
repositories { maven { url = uri("https://maven.cdata.com/p/jdbc/") } }
dependencies { implementation("cdata:salesforce-jdbc:26.0.9666") }
```

```xml
<!-- Maven -->
<repository><id>cdata</id><url>https://maven.cdata.com/p/jdbc/</url></repository>
<dependency>
  <groupId>cdata</groupId><artifactId>salesforce-jdbc</artifactId><version>26.0.9666</version>
</dependency>
```

If a build can't resolve the remote repo, fall back to installing the local jar into your
build (Gradle `implementation(files("libs/cdata.jdbc.<source>.jar"))`, or Maven
`mvn install:install-file`).

---

## Step 2: License

The JDBC driver is **licensed per machine**, separate from the ADO.NET/ODBC/Python
editions. If discovery already activated the jar you're using, the app needs nothing more.

To activate (if needed):

- **Via the CLI** (trial is safe in-session; a purchased key is a secret — run it outside
  the AI session):
  ```bash
  cdatacli drivers activate <Driver> --name "First Last" --email you@example.com --trial
  cdatacli drivers activate <Driver> --name "First Last" --email you@example.com --key XXXXX
  ```
- **Via the jar's GUI wizard:** double-click / `java -jar cdata.jdbc.<source>.jar`.

The driver looks for its license next to the jar (`cdata.jdbc.<source>.lic`) or in the
user's CData folder (`~/.CData/cdata.jdbc.<source>.lic`, i.e.
`%USERPROFILE%\.CData\` on Windows). Keep the `.lic` with the jar you ship, or ensure the
runtime user has it in `~/.CData/`.

---

## Step 3: Put It on the Classpath

> **OS note:** classpath separator is `;` on Windows (PowerShell/cmd), `:` on macOS/Linux.
> Relative paths resolve against the current directory — `cd` into the source folder or use
> absolute paths.

```bash
javac -cp "cdata.jdbc.<source>.jar" App.java
java  -cp ".:cdata.jdbc.<source>.jar" App        # macOS/Linux — Windows: ".;cdata.jdbc.<source>.jar"
```

With Maven/Gradle, the dependency from Step 1 puts it on the classpath automatically. Modern
JDBC (4.0+) **auto-registers** the driver via `ServiceLoader`, so `Class.forName(...)` is not
required — the jar just needs to be on the classpath.

---

## Step 4: Connect (JDBC)

JDBC objects are `AutoCloseable`, so use try-with-resources:

```java
import java.sql.*;

String url = "jdbc:salesforce:AuthScheme=OAuth;InitiateOAuth=GETANDREFRESH;UseSandbox=true;"
           + "OAuthSettingsLocation=C:\\...\\OAuthSettings-sbx.txt";

try (Connection conn = DriverManager.getConnection(url);
     PreparedStatement ps = conn.prepareStatement(
         "SELECT Id, Name FROM Account WHERE Name LIKE ?")) {
    ps.setString(1, "%Acme%");                  // ? positional params
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) System.out.println(rs.getString("Name"));
    }
}
```

- The **JDBC URL** is `jdbc:<source>:` followed by the same `;`-delimited connection
  properties you validated with `cdatacli`.
- **`?` positional parameters** (`setString`/`setObject`); use `executeUpdate()` for
  INSERT/UPDATE/DELETE.
- Under `ServiceLoader` no explicit driver registration is needed; if a non-standard
  classloader misses it, register with `Class.forName("cdata.jdbc.<source>.<Source>Driver")`.

### Connection string & auth (reuse discovery, keep secrets out of code)

Build the URL from config, not hard-coded. For **OAuth** sources, point
`OAuthSettingsLocation` at the token file discovery already populated so the app refreshes
silently. Sources with **no embedded OAuth app** (e.g. Gmail) require the user's own
`OAuthClientId`/`OAuthClientSecret` even to refresh — read those from environment variables,
never commit them.

---

## Step 5: End-to-End Example (Java)

```java
import java.sql.*;
import java.util.*;

public class Accounts {
    private static final String URL =
        "jdbc:salesforce:AuthScheme=OAuth;InitiateOAuth=GETANDREFRESH;UseSandbox=true;"
      + "OAuthSettingsLocation=C:\\...\\OAuthSettings-sbx.txt";

    static List<String> list(String search, int limit) throws SQLException {
        String sql = "SELECT Id, Name FROM Account"
                   + (search == null ? "" : " WHERE Name LIKE ?")
                   + " ORDER BY Name LIMIT " + limit;
        try (Connection c = DriverManager.getConnection(URL);
             PreparedStatement ps = c.prepareStatement(sql)) {
            if (search != null) ps.setString(1, "%" + search + "%");
            try (ResultSet rs = ps.executeQuery()) {
                List<String> out = new ArrayList<>();
                while (rs.next()) out.add(rs.getString("Name"));
                return out;
            }
        }
    }

    static void create(String name, String industry) throws SQLException {
        try (Connection c = DriverManager.getConnection(URL);
             PreparedStatement ps = c.prepareStatement(
                 "INSERT INTO Account (Name, Industry) VALUES (?, ?)")) {
            ps.setString(1, name);
            ps.setString(2, industry);
            ps.executeUpdate();
        }
    }
}
```

---

## Troubleshooting

| Error | Cause / fix |
|---|---|
| `No suitable driver found for jdbc:<source>:...` | The jar isn't on the classpath (or the URL prefix is wrong). Add `cdata.jdbc.<source>.jar` to the classpath / dependencies; confirm the URL starts with `jdbc:<source>:`. |
| `Could not find a valid license` | Jar not activated on this machine — activate via `cdatacli drivers activate` or the jar wizard (Step 2). The `.lic` must sit next to the jar or in `~/.CData/`. |
| `ClassNotFoundException: cdata.jdbc.<source>.<Source>Driver` | Only relevant if registering manually — the jar isn't on the classpath, or the class name is misspelled (`<Source>` is the cased driver name). |
| `'OAuthClientId' and 'OAuthClientSecret' are needed to initiate OAuth` | Source has no embedded OAuth app; add client id/secret (from env vars) to the JDBC URL, matching the app whose refresh token is cached. |
