# APEX Copilot Project Template

A project template for AI-assisted Oracle APEX development using GitHub Copilot, SQLcl, and Playwright MCP. It provides a structured workflow for exporting, versioning, and iterating on APEX applications with an AI coding agent.

## Prerequisites

- [VS Code](https://code.visualstudio.com/)
- [GitHub Copilot](https://github.com/features/copilot) extension
- [SQL Developer Extension for VS Code](https://marketplace.visualstudio.com/items?itemName=Oracle.sql-developer)
- [Playwright MCP](https://github.com/microsoft/playwright-mcp) for browser-based E2E testing
- [Git Bash](https://git-scm.com/) (or compatible Bash shell)
- [Python 3.12+](https://www.python.org/)
- [SQLcl](https://www.oracle.com/database/sqldeveloper/technologies/sqlcl/) (Oracle SQL command-line tool)

## Project Structure

```
├── .github/
│   ├── .copilot-context.md        # Environment details & project standards
│   ├── copilot-instructions.md    # Global Copilot instructions
│   ├── instructions/              # File-specific Copilot instructions
│   │   ├── apex-coding.instructions.md
│   │   └── apex-testing.instructions.md
│   └── skills/                    # Custom Copilot skills
│       ├── apex-component-modifier/
│       └── apex-plugin-patcher/
├── apex/                          # Exported APEX application (generated)
├── db-scripts/                    # Exported DDL scripts (generated)
│   └── {object-type}/
├── export-app.sql                 # SQLcl script to export the APEX app
├── refresh-export.sh              # Shell script to refresh the APEX export
├── set-ddl-format.sql             # DDL formatting rules for SQLcl
├── WORKFLOW.md                    # Detailed development workflow
└── .gitignore
```

## Getting Started

### 1. Clone or copy this template

```bash
git clone <repository-url>
cd apex-copilot-project-template
```

### 2. Create a working copy of your APEX app

In APEX Builder, copy the application you want to work on. This keeps the source/production app untouched while you develop.

### 3. Configure the project

Update the environment details in [.github/.copilot-context.md](.github/.copilot-context.md):

- `connection-name` — your SQLcl connection name
- `appid` — the working copy's application ID
- `app-alias` — the app alias
- `app-schema` — the parsing schema
- `apex-workspace-runtime-url` — the runtime URL for the APEX workspace

Replace `{appid}` in `export-app.sql` and `{connection-name}` in `refresh-export.sh` with your actual values.

### 4. Export the APEX application

Run the export script to download the app definition into the `apex/` folder:

```bash
sql -name {connection-name} @export-app.sql
```

Or use the helper script:

```bash
./refresh-export.sh
```

### 5. Export relevant DDL

Export database objects to `db-scripts/{object-type}/` using SQLcl:

```sql
-- Connect to the database
sql -name {connection-name}

-- Apply formatting rules (once per session)
@set-ddl-format.sql

-- Export an object
ddl {object-name} {object-type} save db-scripts/{object-type}/{object-name}.sql
```

### 6. Initialize Git and commit the baseline

```bash
git init
git add .
git commit -m "Initial baseline export"
```

## Development Workflow

1. **Prompt the AI agent** — Describe your fix, feature, or enhancement. Start in plan mode to review the approach before implementation.
2. **Refresh the APEX export** — After changes are applied to the database, run `./refresh-export.sh` to sync the local export with the latest working copy.
3. **Commit changes** — Review the diff and commit.
4. **Repeat** — Continue iterating through steps 1–3.

See [WORKFLOW.md](WORKFLOW.md) for the full detailed workflow.

## Naming Conventions

**Examples:**

| Object Type        | Convention         |
|--------------------|--------------------|
| Views              | Suffixed with `_VW` |
| Materialized Views | Suffixed with `_MV` |

Additional standards can be configured in [.github/.copilot-context.md](.github/.copilot-context.md).

## License

Internal use — see your organization's policies.
