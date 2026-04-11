---
description: "Use when writing, modifying, or deploying Oracle APEX application code or database objects (tables, views, packages, procedures, functions, triggers). Covers script layout, file naming, and execution conventions."
---
# APEX Coding

## Executing Scripts

Always resolve `{connection-name}` from `.github/.copilot-context.md` before connecting to the database.

### Preferred: SQLcl MCP Server

When the SQLcl MCP server is available, use it for all database operations:

1. **Connect** — call `mcp__sqlcl__connect` with the `{connection-name}` from `.github/.copilot-context.md`.
2. **Run SQL / scripts** — call `mcp__sqlcl__run-sql` (or `mcp__sqlcl__run-sql-async` for long-running operations) passing the SQL statement.
3. **File execution (`@`) is restricted** — `mcp__sqlcl__run-sql` blocks `@path/to/script.sql` with `SP2-0738: Restricted command`. To execute `.sql` files, use the **Terminal** fallback below.

### Fallback: Terminal

If the SQLcl MCP server is unavailable, use SQLcl in the terminal.

> **PowerShell `@` splatting conflict:** PowerShell interprets `@` as a splatting operator, so `sql -name {connection-name} @script.sql` will fail with `SplattingNotPermitted`. Always wrap the command in `cmd /c` when running from PowerShell:
>
> ```powershell
> cmd /c "cd /d ""{workspace-path}"" && sql -name {connection-name} @script.sql"
> ```

In **bash / Git Bash**, `@` works natively:

```bash
sql -name {connection-name} @path/to/script.sql
```

Or connect interactively first, then execute scripts:

```bash
sql -name {connection-name}
SQL> @path/to/script.sql
```

### Importing APEX Export Files

APEX export files (pages, shared components, etc.) contain JavaScript and CSS with `&` characters (e.g., `&fc`, `&n`). SQLcl interprets `&` as a substitution variable prefix by default, causing the session to hang with `Enter value for ...` prompts.

**Always use `SET DEFINE OFF` before importing APEX export files.**

The reliable pattern is to create a wrapper SQL script:

```sql
-- _import_wrapper.sql
SET DEFINE OFF
@apex/f{appid}/application/pages/page_NNNNN.sql
EXIT
```

Then execute:

```bash
# Bash / Git Bash
sql -name {connection-name} @_import_wrapper.sql
```

```powershell
# PowerShell (must wrap in cmd /c to avoid @ splatting error)
cmd /c "cd /d ""{workspace-path}"" && sql -name {connection-name} @_import_wrapper.sql"
```

> **Do not** use heredoc (`<<'EOF'`) piped into `sql` — this fails in MINGW64/Git Bash environments. Always write a wrapper `.sql` file instead.

Always specify the connection name to ensure scripts run against the correct database and schema.

## Writing Scripts to the Workspace

Always write scripts to the project workspace **before** executing them. Never run ad-hoc SQL directly in the terminal without a corresponding file.

Maintain a log of all scripts executed, including the file name, purpose, and date executed. This ensures traceability and allows for version control of database changes. Save these logs in `sql-logs/script-execution-log.md` with the following format:

```
| Date and time     | Script Name                      | Purpose                          |
|-------------------|----------------------------------|----------------------------------|
| 2024-06-01 10:00  | table/create_example_table.sql   | Create example table             |
```

### APEX Application Files

Export and maintain APEX application source as split files under the `apex/` directory:

```
apex/
```

Use the standard APEX split export structure produced by:

```sql
apex export -applicationid {appid} -split -expType APPLICATION_SOURCE,READABLE_YAML -dir apex/
```

### Database Object Scripts

Store database object scripts under `db-scripts/` organised by object type:

```
db-scripts/
  table/
    {object-name}.sql
  view/
    {object-name}.sql
  package/
    {object-name}.pkb    -- package body
    {object-name}.pks    -- package specification
  procedure/
    {object-name}.pls
  function/
    {object-name}.pls
  trigger/
    {object-name}.sql
  type/
    {object-name}.sql
  adhoc-queries/
    {query-name}.sql     -- for one-off queries or scripts not tied to a specific object
```

#### File Extension Rules

| Object Type           | Extension |
|-----------------------|-----------|
| Package Specification | `.pks`   |
| Package Body          | `.pkb`   |
| Procedure             | `.pls`   |
| Function              | `.pls`   |
| All other objects     | `.sql`   |

#### Conventions

- Use **lowercase** for file and folder names.
- One object per file.
- Include a trailing `/` (forward slash) to execute the PL/SQL block where appropriate.
- Write the script file first, then execute it. Prefer the SQLcl MCP server: call `mcp__sqlcl__run-sql` with `@db-scripts/{object-type}/{object-name}.{ext}`. If MCP is unavailable, fall back to the terminal:

  ```bash
  sql -name {connection-name} @db-scripts/package/my_package.pks
  ```

- Use **lowercase** for object names in scripts to maintain consistency with APEX exports and avoid case sensitivity issues.
- Use CHAR semantics for VARCHAR2 columns in tables to prevent issues with multi-byte characters. For example:

  ```sql
  CREATE TABLE example_table (
      example_column VARCHAR2(100 CHAR)
  );
  ```

- When creating or altering tables, include descriptive comments on the table and columns to improve maintainability. For example:

  ```sql
  COMMENT ON TABLE example_table IS 'This table stores example data';
  COMMENT ON COLUMN example_table.example_column IS 'This column stores example values';
  ```

- Include useful docstrings to describe the purpose and functionality of packages, procedures, and functions. For example:

  ```sql
  CREATE OR REPLACE PACKAGE my_package AS
  /* 
     This package provides utility functions for processing data 

     Created by: Github Copilot ([model-name if available])
     Created on: [Creation Date
  */
      FUNCTION process_data(p_input VARCHAR2) RETURN VARCHAR2;
  END my_package;
  ```

  ```sql
  CREATE OR REPLACE FUNCTION my_function(p_input VARCHAR2) RETURN VARCHAR2 IS
    /*
      This function processes the input data and returns a result

      Input parameters:
          p_input - The input data to be processed
      Returns:
          A string indicating the result of processing
      Exceptions:
          Raises an exception if the input data is invalid

      Created by: Github Copilot ([model-name if available])
      Created on: [Creation Date]
    */
  BEGIN
      RETURN 'Processed: ' || p_input;
  END my_function;
  ```