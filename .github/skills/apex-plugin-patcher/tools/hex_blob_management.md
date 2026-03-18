# Hex Blob Management

APEX plugin SQL files store static file content (JS, CSS) as hex-encoded
`wwv_flow_imp.g_varchar2_table` entries. This document explains the storage
format and provides the canonical Python script for re-encoding after edits.

---

## Storage Format

Each static file is stored as two consecutive SQL blocks:

**Block 1 — the data blob:**
```sql
begin
wwv_flow_imp.g_varchar2_table := wwv_flow_imp.empty_varchar2_table;
wwv_flow_imp.g_varchar2_table(1) := '2F2A2A0A202A20416363656E74...';  -- 200 hex chars = 100 bytes
wwv_flow_imp.g_varchar2_table(2) := '6E7465726E616C20557365204F...';
...
wwv_flow_imp.g_varchar2_table(N) := 'LAST_CHUNK...';
end;
/
```

**Block 2 — the file registration:**
```sql
begin
wwv_flow_imp_shared.create_plugin_file(
 p_id=>wwv_flow_imp.id(...)
,p_plugin_id=>wwv_flow_imp.id(...)
,p_file_name=>'<plugin_name>.js'
,p_mime_type=>'text/javascript'
,p_file_charset=>'utf-8'
,p_file_content=>wwv_flow_imp.varchar2_to_blob(wwv_flow_imp.g_varchar2_table)
);
end;
/
```

**Key rules:**
- Chunk size is always **100 bytes = 200 hex chars** per entry — do not change
- Hex is uppercase (A–F)
- Block 2 immediately follows Block 1 for each file
- The `g_varchar2_table` global is shared; the re-encoding script uses Block 2's `p_file_name` as a landmark to locate Block 1

---

## File Pairs in This Project

This project stores both readable and minified versions of each static file.

| Pair | Readable source                  | Blob(s) to update                                                 |
|------|----------------------------------|-------------------------------------------------------------------|
| JS   | `readable/.../<plugin_name>.js`  | `<plugin_name>.js` (source) + `<plugin_name>.min.js` (minified)   |
| CSS  | `readable/.../<plugin_name>.css` | `<plugin_name>.css` (source) + `<plugin_name>.min.css` (minified) |

The `.min.*` blob is encoded from the minified version of the source.
Use the combined script in `tools/minification.md` to produce both blobs in one pass.

**Always update both blobs in a pair** whenever the readable source changes.

---

## Canonical Re-encode Script

```python
# Usage: python3 re_encode_plugin.py
# Adjust the path constants at the top for each use.

READABLE_JS  = "<readable_js_path>"   # from Step 1 — e.g. apex/f123/readable/.../plugin_static_files/item_type_<plugin_sql_name>/<plugin_name>.js
SQL_FILE     = "<plugin_sql_file>"    # from Step 1 — e.g. apex/f123/application/.../plugins/item_type/<plugin_sql_name>.sql

# ── encode ──────────────────────────────────────────────────────────────────
with open(READABLE_JS, 'rb') as f:
    content = f.read()

chunks = [content[i:i+100].hex().upper() for i in range(0, len(content), 100)]
new_entries = "\n".join(
    f"wwv_flow_imp.g_varchar2_table({i+1}) := '{chunk}';"
    for i, chunk in enumerate(chunks)
)

# ── patch helper ─────────────────────────────────────────────────────────────
def replace_blob(sql_text, file_name, new_entries):
    """Replace the g_varchar2_table entries for a named plugin file."""
    marker     = f"p_file_name=>'{file_name}'"
    marker_pos = sql_text.find(marker)
    if marker_pos == -1:
        raise ValueError(f"Marker not found: {marker}")

    # The entries block ends at the "end;\n/\nbegin\n" just before the marker
    search  = "end;\n/\nbegin\n"
    end_pos = sql_text.rfind(search, 0, marker_pos)

    # The entries block starts right after "g_varchar2_table := empty_varchar2_table;\n"
    header     = "wwv_flow_imp.g_varchar2_table := wwv_flow_imp.empty_varchar2_table;\n"
    header_pos = sql_text.rfind(header, 0, end_pos)
    if header_pos == -1:
        raise ValueError(f"Header not found before: {file_name}")

    entries_start = header_pos + len(header)
    return sql_text[:entries_start] + new_entries + "\n" + sql_text[end_pos:]

# ── apply ────────────────────────────────────────────────────────────────────
with open(SQL_FILE, 'r', encoding='utf-8') as f:
    sql = f.read()

sql = replace_blob(sql, "<plugin_name>.js",     new_entries)  # readable
sql = replace_blob(sql, "<plugin_name>.min.js", new_entries)  # minified (same)

with open(SQL_FILE, 'w', encoding='utf-8') as f:
    f.write(sql)

print(f"Done. {len(chunks)} chunks written.")
```

To re-encode the CSS instead, change `READABLE_JS` to the `.css` path and
update the two `replace_blob` calls to use `'<plugin_name>.css'`
and `'<plugin_name>.min.css'`.

---

## Decoding a Blob for Inspection

To read the current content of a blob without modifying anything:

```bash
python3 - << 'EOF'
import re

SQL_FILE  = "<plugin_sql_file>"   # from Step 1
FILE_NAME = "<plugin_name>.js"    # change as needed

with open(SQL_FILE) as f:
    sql = f.read()

# Locate the blob boundaries using the file_name marker
marker     = f"p_file_name=>'{FILE_NAME}'"
marker_pos = sql.find(marker)
end_pos    = sql.rfind("end;\n/\nbegin\n", 0, marker_pos)
header     = "wwv_flow_imp.g_varchar2_table := wwv_flow_imp.empty_varchar2_table;\n"
header_pos = sql.rfind(header, 0, end_pos)
blob_section = sql[header_pos:end_pos]

hex_chunks = re.findall(r"g_varchar2_table\(\d+\) := '([0-9A-Fa-f]+)';", blob_section)
decoded = bytes.fromhex("".join(hex_chunks)).decode("utf-8")
print(decoded[:2000])   # print first 2000 chars
EOF
```

---

## Verification After Re-encode

```bash
grep -n "create_plugin_file\|g_varchar2_table :=\|empty_varchar2_table\|^end;$\|^begin$" \
     <plugin_sql_file>
```

Confirm:
1. `begin` / `end;` lines alternate cleanly — no orphan blocks
2. Each `g_varchar2_table := wwv_flow_imp.empty_varchar2_table;` is followed by entries, then `end;`, then `begin`, then `create_plugin_file(...)`
3. The file ends with `wwv_flow_imp.component_end`
