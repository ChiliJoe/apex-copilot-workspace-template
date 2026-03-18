---
description: "Use when writing, modifying, or deploying Oracle APEX application code or database objects (tables, views, packages, procedures, functions, triggers). Covers script layout, file naming, and execution conventions."
---
# APEX Coding

## Executing Scripts

Use SQLcl with a named connection to run scripts on the database:

```bash
sql -name {connection-name}
```

Replace `{connection-name}` with the active connection name for the target schema. Once connected, execute scripts with `@path/to/script.sql`.

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
- Write the script file first, then execute it via `sql -name {connection-name}` with `@db-scripts/{object-type}/{object-name}.{ext}`. For example:

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
