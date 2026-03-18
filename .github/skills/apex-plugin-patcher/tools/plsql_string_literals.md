# Editing PL/SQL String Literals

The plugin's render and AJAX procedures are stored inside the `create_plugin`
call as a single `wwv_flow_string.join(wwv_flow_t_varchar2(...))` value for
`p_plsql_code`. Each line of PL/SQL is a separate VARCHAR2 literal.

---

## Structure

```sql
,p_plsql_code=>wwv_flow_string.join(wwv_flow_t_varchar2(
'PROCEDURE render_<plugin_name> (',
'    p_item   IN apex_plugin.t_item,',
'    p_plugin IN apex_plugin.t_plugin,',
'    ...',
'IS',
'    l_item_name VARCHAR2(255) := p_item.name;',
'    ...',
'BEGIN',
'    htp.p(''<div class="apex-item-group">'');',
'    ...',
'END render_<plugin_name>;',
'',
'PROCEDURE ajax_<plugin_name> (',
'    ...',
'END ajax_<plugin_name>;',
''))
```

The closing `''` (empty string, no comma) terminates the array. The outer
`)` closes the `join(...)` call.

---

## Quoting Rules

| Inside PL/SQL | In the SQL file |
|---------------|----------------|
| `'string'`    | `''string''`   |
| `''`          | `''''`         |
| `"identifier"` | `"identifier"` (no change — double quotes are safe) |

Example — a line with a string literal inside PL/SQL:
```plsql
-- original PL/SQL:
htp.p('<input type="hidden" value="' || l_value || '">');
```
```sql
-- in the SQL file (outer single quotes escaped):
'    htp.p(''<input type="hidden" value="'' || l_value || ''">'');',
```

---

## Edit Operations

### Modify an existing line

Use `Grep` to find a distinctive anchor near the target line, then use `Edit`
with a short but unique `old_string` / `new_string` context.

**Example — change a default separator value:**
```
old: 'l_separator := NVL(p_item.attribute_02, ''-'');',
new: 'l_separator := NVL(p_item.attribute_02, '':'');',
```

### Add a new `htp.p(...)` call

1. Find the surrounding `htp.p` calls that bracket the insertion point.
2. Insert a new `'    htp.p(...);',` line between them.

**Example — add an input element after the `<ul>` line:**
```
old:
'    htp.p(''  <ul class="apex-item-multi" id="'' || l_item_name || ''_DISPLAY"></ul>'');',
'    htp.p(''  <button type="button"'');',

new:
'    htp.p(''  <ul class="apex-item-multi" id="'' || l_item_name || ''_DISPLAY"></ul>'');',
'    htp.p(''  <input type="text" id="'' || l_item_name || ''_SEARCH" class="acn-inline-search">'');',
'    htp.p(''  <button type="button"'');',
```

### Add a new local variable

Find the `IS` / variable declaration block. Insert before `BEGIN`:
```
old:
'    l_separator VARCHAR2(10);',
'BEGIN',

new:
'    l_separator VARCHAR2(10);',
'    l_new_var   VARCHAR2(100);',
'BEGIN',
```

### Add a new PL/SQL procedure

Append before the final closing `''`:
```
old:
'END ajax_<plugin_name>;',
''))

new:
'END ajax_<plugin_name>;',
'',
'PROCEDURE my_new_procedure (...)',
'IS',
'BEGIN',
'    null;',
'END my_new_procedure;',
''))
```

---

## Safety Checks

After any edit to the PL/SQL section:

1. **Balanced `begin`/`end`** — Count occurrences of `BEGIN` and `END` inside
   the string literals (excluding SQL-level `begin`/`end;` block wrappers).
   They must match.

2. **No unmatched quotes** — Every line entry must start with `'` and end with
   `',` (or `''` for the last line). A missing closing `'` corrupts all
   subsequent lines.

3. **Line length** — Keep individual varchar2 lines under ~3900 characters.
   If a generated string (e.g., a long JavaScript config) would exceed that,
   split across multiple lines using string concatenation:
   ```sql
   '    htp.p(''<div class="very-long-class-name-'');',
   '    htp.p(''continued-here">'');',
   ```

4. **Grep to verify the anchor still exists** after editing, to confirm the
   `Edit` tool applied the change correctly.

---

## Finding the Right Line

Never rely on line numbers — they shift when blobs are re-encoded. Use `Grep`:

```bash
# Find a specific htp.p call by a distinctive CSS class or ID fragment
grep -n "<distinctive_css_class_or_id>" <plugin_sql_file>

# Find where a plugin attribute is read
grep -n "attribute_01\|<attribute_static_id>" <plugin_sql_file>

# Find the AJAX operation handler
grep -n "FETCH_DATA\|GET_DISPLAY\|l_operation" <plugin_sql_file>
```

PL/SQL keywords inside the string literals appear with doubled quotes, so
search for the unescaped form (e.g., `FETCH_DATA`, not `''FETCH_DATA''`) —
the outer SQL quoting is transparent to `grep`.
