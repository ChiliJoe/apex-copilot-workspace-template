# APEX Plugin SQL File Anatomy

Generic reference for APEX plugin SQL file structure.
Use as a guide for locating sections when patching any plugin.

> **Note:** Line numbers and file names below are illustrative. Always verify
> the actual structure by reading the target plugin's SQL file — section sizes
> vary by plugin. Replace `<plugin_sql_name>` with the SQL filename stem (e.g.
> the part before `.sql`) and `<plugin_name>` with the base name used for
> JS/CSS files.

---

## Overall Structure

```
Line 1        begin
Lines 2-N     wwv_flow_imp_shared.create_plugin(
               p_name=>'COM.<VENDOR>.<NAME>',
               p_plsql_code=>wwv_flow_string.join(wwv_flow_t_varchar2(
                   ... render and ajax PL/SQL ...
               )),
               ...attribute definitions...
              );
Line N        end;
              /

              begin                              ← blob for .min.js
              g_varchar2_table := empty...
              g_varchar2_table(n) := 'HEX...';
              end;
              /
              begin
              create_plugin_file(p_file_name=>'<plugin_name>.min.js', ...)
              end;
              /

              begin                              ← blob for .min.css
              ...
              create_plugin_file(p_file_name=>'<plugin_name>.min.css', ...)
              end;
              /

              begin                              ← blob for .css
              ...
              create_plugin_file(p_file_name=>'<plugin_name>.css', ...)
              end;
              /

              begin                              ← blob for .js
              ...
              create_plugin_file(p_file_name=>'<plugin_name>.js', ...)
              end;
              /

              begin
              wwv_flow_imp.component_end;        ← must remain last
              end;
              /
```

---

## Static File Order

The four static files appear in this fixed order:

| Order | File name                  | Type       | APEX serves when        |
|-------|----------------------------|------------|-------------------------|
| 1     | `<plugin_name>.min.js`     | JavaScript | Production / debug off  |
| 2     | `<plugin_name>.min.css`    | CSS        | Production / debug off  |
| 3     | `<plugin_name>.css`        | CSS        | Development / debug on  |
| 4     | `<plugin_name>.js`         | JavaScript | Development / debug on  |

In this project there is **no separate minification step** — `.min.js` and
`.min.css` are re-encoded from the same readable sources as `.js` and `.css`.

---

## Locating a Section

**Verify blob boundaries (run this grep first):**
```bash
grep -n "create_plugin_file\|g_varchar2_table :=\|empty_varchar2_table\|^end;$\|^begin$" \
  <plugin_sql_file>
```

Expected output pattern:
```
2:begin
<N>:end;
<N+2>:begin
<N+3>:wwv_flow_imp.g_varchar2_table := wwv_flow_imp.empty_varchar2_table;
<M>:end;
<M+2>:begin
<M+3>:wwv_flow_imp_shared.create_plugin_file(   ← p_file_name=>'*.min.js'
...
```

**Find PL/SQL section by function name:**
```bash
grep -n "PROCEDURE render_\|PROCEDURE ajax_\|BEGIN\|END render_\|END ajax_" \
  <plugin_sql_file> | head -20
```

**Find a specific `htp.p` call:**
```bash
grep -n "<distinctive_css_class_or_id>" <plugin_sql_file>
```

---

## create_plugin Call Parameters (key fields)

```sql
wwv_flow_imp_shared.create_plugin(
 p_id                    => wwv_flow_imp.id(<generated_id>)
,p_plugin_type           => 'ITEM TYPE'
,p_name                  => 'COM.<VENDOR>.<NAME>'
,p_display_name          => '<Display Name>'
,p_image_prefix          => nvl(wwv_flow_application_install.get_static_plugin_file_prefix(...), '')
,p_plsql_code            => wwv_flow_string.join(wwv_flow_t_varchar2( ... ))
,p_api_version           => 3
,p_render_function       => 'render_<plugin_name>'
,p_ajax_function         => 'ajax_<plugin_name>'
,p_standard_attributes   => 'VISIBLE:SESSION_STATE:READONLY:SOURCE:ELEMENT:WIDTH:LOV:...'
,p_substitute_attributes => true
,p_version_scn           => <scn>
,p_subscribe_plugin_settings => true
,p_version_identifier    => '1.0'
,p_files_version         => <n>
);
```

When bumping the version after a patch, increment `p_files_version` by 1
so APEX knows to invalidate cached static files.

---

## Plugin Attribute Definitions

After the main `create_plugin` call, attributes are registered separately
(the number and names are plugin-specific — read the SQL file to discover them):

```sql
-- Attribute 01: <attribute name> (<type>, default <value>)
wwv_flow_imp_shared.create_plugin_attribute(
 p_id=>wwv_flow_imp.id(<generated_id>)
,p_plugin_id=>wwv_flow_imp.id(<plugin_id>)
,p_attribute_sequence=>1
,p_static_id=>'<static_id>'
,p_prompt=>'<Prompt>'
,p_attribute_type=>'<TYPE>'
,p_default_value=>'<default>'
...
);
```

Attributes are referenced in PL/SQL as `p_item.attribute_01`,
`p_item.attribute_02`, etc. (always positional, not by static_id).

---

## Installation and Validation

**Install:**
```bash
sql <connection_name> @<plugin_sql_file>
```

**Validate:**
1. Hard-refresh the browser (`Ctrl+Shift+R`) to bust JS/CSS cache
2. Exercise the changed behaviour in the running APEX application
3. Check browser DevTools → Network tab → filter by `.js` or `.css` → confirm
   the file size changed from the previous version

**Readable source paths** (discovered via Glob in Step 1):
```
apex/f<APP_ID>/readable/application/shared_components/plugin_static_files/
  item_type_<plugin_sql_name>/
    <plugin_name>.js
    <plugin_name>.css
```

**SQL installation file** (discovered via Glob in Step 1):
```
apex/f<APP_ID>/application/shared_components/plugins/item_type/
  <plugin_sql_name>.sql
```
