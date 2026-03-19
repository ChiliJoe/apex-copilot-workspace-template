
# AI-Assisted APEX Development Workflow

## Software Requirements

* VS Code
* Github Copilot
* SQL Developer Extension
* Playwright MCP | [Reference](https://github.com/microsoft/playwright-mcp) | [Install](https://insiders.vscode.dev/redirect?url=vscode%3Amcp%2Finstall%3F%257B%2522name%2522%253A%2522playwright%2522%252C%2522command%2522%253A%2522npx%2522%252C%2522args%2522%253A%255B%2522%2540playwright%252Fmcp%2540latest%2522%255D%257D)
* Git Bash
* Python 3.12+


## Development Workflow

1. **Create a working copy** — Copy the APEX app you are working on in APEX Builder. This keeps the source/production app untouched while you develop.

2. **Update export scripts** — Replace `{appid}` in `export-app.sql` with the working copy's app ID. Replace `{connection-name}` in `refresh-export.sh`.

3. **Export the working copy** — Run the following from your workspace to export the app to the local `apex/` folder:

   ```bash
   sql -name {connection-name} @export-app.sql
   ```

   Or execute the `refresh-export.sh` script:

   ```bash
   ./refresh-export.sh
   ```

   The connection must use the app's parsing schema.

4. **Export relevant DDL** — Export DDL for any database objects involved in the work to `db-scripts/{object-type}/` using the DDL commands below. To login to SQLcl, run `sql -name {connection-name}`.

5. Update the environment details and project standards in `.github/.copilot-context.md` file.

6. **Initialize git and baseline** — Run `git init` and commit all exported files as the baseline before making any changes.

7. **Prompt the AI coding agent** — Describe your fix, feature, or enhancement. Always start in **plan mode** to review the approach before implementation begins.

8. Refresh APEX export by executing `refresh-export.sql`. This will synchronize the copy of the application export to ensure you are working with the latest working copy definition and retrieve any additional transformations that have been applied by APEX when saving the changes to the database.

9. Check in changes and commit.

10. Repeat steps 7-9.

---

## Exporting application

```sql
apex export -applicationid {appid} -split -expType APPLICATION_SOURCE,READABLE_YAML -dir apex/
```

## Exporting DDL

```sql
-- Apply formatting rules (execute once per session)
@set-ddl-format.sql

ddl {object-name} {object-type} save db-scripts/{object-type}/{object-name}.sql
```
