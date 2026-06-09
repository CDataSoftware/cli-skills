# driver-cli-skills

> **⚠️ Important:** This repository is under development — contents are subject to change.

CData Driver CLI Skills are agent skills for working with [CData Driver CLI](https://www.cdata.com/solutions/cli/) from AI coding tools. CData CLI enables AI coding agents like Claude Code, Cursor, GitHub Copilot, Google Gemini CLI or OpenAI Codex to access your business data via CData Drivers. Salesforce, NetSuite, Jira, SAP, QuickBooks and hundreds of SaaS or databases can be accessed from your AI terminal, allowing you to develop data-connected applications or simply run data-connected agents on your terminal.

The Skills follow the [vercel-labs/skills](https://github.com/vercel-labs/skills) standard — each skill is a directory under `skills/` containing a `SKILL.md` with YAML frontmatter (`name`, `description`, `license`) and instructions for the agent.

## Skills

| Skill | Description |
|---|---|
| [`cdata-driver-cli`](skills/cdata-driver-cli/SKILL.md) | Connect, query, explore schema, run SQL, search/download/activate CData JDBC drivers, and generate source-specific skills via the `cdatacli` tool. |

## Install

Install only the Driver CLI skills (this folder):

```bash
npx skills add CDataSoftware/skills/driver-cli-skills
```

## Prerequisites

The `cdata-driver-cli` skill drives the `cdatacli` executable — a Java CLI (requires Java 17+) for CData JDBC drivers. After installing the CLI, `cdatacli` is on `PATH` and discovers drivers from `./` or `./lib/` relative to the executable.

> Install commands are tentative and will be finalized for the CLI release. Separate installers are planned for Windows, macOS, and Linux.

- Windows (PowerShell): `iwr https://cdn.cdata.com/cli/install.ps1 | iex`
- macOS: `curl -fsSL https://cdn.cdata.com/cli/install-macos.sh | bash`
- Linux: `curl -fsSL https://cdn.cdata.com/cli/install-linux.sh | bash`

Driver jars can be fetched via `cdatacli drivers download` from the CData driver catalog.
