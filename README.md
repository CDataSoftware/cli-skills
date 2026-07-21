# cli-skills

CData CLI Skills are agent skills for working with [CData CLI](https://www.cdata.com/solutions/cli/) from AI coding tools. CData CLI enables AI coding agents like Claude Code, Cursor, GitHub Copilot, Google Gemini CLI or OpenAI Codex to access your business data via CData Drivers. Salesforce, NetSuite, Jira, SAP, QuickBooks and hundreds of SaaS or databases can be accessed from your AI terminal, allowing you to develop data-connected applications or simply run data-connected agents on your terminal.

The Skills follow the [vercel-labs/skills](https://github.com/vercel-labs/skills) standard — each skill is a directory under `skills/` containing a `SKILL.md` with YAML frontmatter (`name`, `description`, `license`) and instructions for the agent.

## Skills

| Skill | Description |
|---|---|
| [`cdata-cli`](skills/cdata-cli/SKILL.md) | **Discovery** — connect, query, explore schema, run SQL, download/activate JDBC drivers, and generate source-specific skills via `cdatacli`. Foundation for the build skills below. |
| [`cdata-cli-python`](skills/cdata-cli-python/SKILL.md) | **Build** Python apps with the CData Python Connector (DB-API 2.0). |
| [`cdata-cli-adonet`](skills/cdata-cli-adonet/SKILL.md) | **Build** C# / .NET apps with the CData ADO.NET Data Provider (NuGet). |
| [`cdata-cli-odbc`](skills/cdata-cli-odbc/SKILL.md) | **Build** Node.js / non-JVM apps with the CData ODBC Driver. |
| [`cdata-cli-java`](skills/cdata-cli-java/SKILL.md) | **Build** Java apps with the CData JDBC Driver. |

## Install

Install all skills:

```bash
npx skills add CDataSoftware/cli-skills
```

Or install individual skills by name:

```bash
npx skills add CDataSoftware/cli-skills --skill cdata-cli
npx skills add CDataSoftware/cli-skills --skill cdata-cli-python
```

The build skills assume `cdata-cli` for discovery — install it alongside whichever build skill(s) you need.

## Prerequisites

The `cdata-cli` skill drives the `cdatacli` executable — a Java CLI (requires Java 17+) for CData JDBC drivers. After installing the CLI, `cdatacli` is on `PATH` and discovers drivers from `./` or `./lib/` relative to the executable.

- Windows (PowerShell): `irm https://downloads.cdata.com/cdatabuilds/builds/free/cdatacli/install-cdatacli-windows.ps1 | iex`
- macOS: `curl -fsSL https://downloads.cdata.com/cdatabuilds/builds/free/cdatacli/install-cdatacli-macos.sh | bash`
- Linux: `curl -fsSL https://downloads.cdata.com/cdatabuilds/builds/free/cdatacli/install-cdatacli-linux.sh | bash`

Driver jars can be fetched via `cdatacli drivers download` from the CData driver catalog.
