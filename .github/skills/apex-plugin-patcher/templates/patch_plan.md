# Plugin Patch Plan

## Change Request
<!-- paste user's original request here -->

## Plugin
- **Name:** `COM.<VENDOR>.<NAME>`
- **SQL file:** `apex/f123/application/shared_components/plugins/item_type/<name>.sql`
- **Readable JS:** `apex/f123/readable/application/shared_components/plugin_static_files/item_type_<name>/<name>.js`
- **Readable CSS:** `apex/f123/readable/application/shared_components/plugin_static_files/item_type_<name>/<name>.css`

## Patch Type
- [ ] PL/SQL only (render/AJAX functions embedded in SQL string literals)
- [ ] Static files only (JS and/or CSS)
- [ ] Both PL/SQL and static files

---

## PL/SQL Changes
*(leave empty if not applicable)*

| # | Procedure | Grep anchor to locate line | Action | Detail |
|---|-----------|---------------------------|--------|--------|
|   |           |                           |        |        |

Quoting reminder: internal `'` → `''`, `"` unchanged. See `tools/plsql_string_literals.md`.

---

## Static File Changes
*(leave empty if not applicable)*

| # | File | Function / section | Action | Detail |
|---|----|-------------------|--------|--------|
|   |    |                   |        |        |

---

## Hex Blobs to Re-encode
*(list every blob that must be regenerated after edits)*

| Source file changed | Blobs to update in SQL |
|---------------------|------------------------|
| `<name>.js`         | `<name>.js`, `<name>.min.js` |
| `<name>.css`        | `<name>.css`, `<name>.min.css` |

Use the canonical script in `tools/hex_blob_management.md`.

---

## Execution Order
1. Edit readable static file(s) (if applicable)
2. Edit PL/SQL string literals in the SQL file (if applicable)
3. Run re-encode Python script for each changed static file
4. Run SQL structure verification (`grep` for begin/end/create_plugin_file)
5. *(Optional)* Install via SQLcl and validate in browser

---

## Risks / Notes
-
