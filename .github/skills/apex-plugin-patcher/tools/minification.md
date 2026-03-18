# JS / CSS Minification

This document explains the minification approach used in this skill and provides
the canonical Python 3 script that minifies source files and re-encodes all four
hex blobs in a single pass.

---

## Approach

Minification is performed with pure Python 3 (no external libraries required).
The minifiers are intentionally simple — sufficient for plugin JS/CSS files while
avoiding build-tool dependencies.

| Technique | JS | CSS |
|-----------|:--:|:---:|
| Remove `/* … */` block comments | ✓ | ✓ |
| Remove `// …` line comments | ✓ | — |
| Collapse whitespace / newlines | ✓ | ✓ |
| Remove spaces around punctuation (`{}:;,`) | — | ✓ |

> **Limitation:** The JS minifier does not handle comment-like sequences inside
> string literals (e.g. `"// not a comment"`). For typical plugin JS this is
> not a problem, but keep it in mind if your source has unusual string content.

---

## Canonical Minify + Re-encode Script

Adjust the four path constants at the top, then run once after editing readable
source files. It writes all four blobs (`.js`, `.min.js`, `.css`, `.min.css`)
in a single pass.

```python
# Usage: python3 - << 'EOF'  (paste this script, substituting actual paths)

import re

READABLE_JS  = "<readable_js_path>"   # e.g. apex/f123/readable/.../plugin_static_files/item_type_<name>/<plugin>.js
READABLE_CSS = "<readable_css_path>"  # e.g. apex/f123/readable/.../plugin_static_files/item_type_<name>/<plugin>.css
SQL_FILE     = "<plugin_sql_file>"    # e.g. apex/f123/application/.../plugins/item_type/<plugin>.sql
PLUGIN_NAME  = "<plugin_name>"        # e.g. "com_vendor_my_plugin"

# ── minifiers ────────────────────────────────────────────────────────────────

def minify_js(source: str) -> str:
    """Strip comments and collapse whitespace from JS source."""
    # Remove block comments  /* … */
    source = re.sub(r'/\*.*?\*/', '', source, flags=re.DOTALL)
    # Remove line comments  // … (only when not inside a string — simple heuristic)
    lines = []
    for line in source.splitlines():
        # Walk char by char to skip string contents
        in_single = False
        in_double = False
        i = 0
        out = []
        while i < len(line):
            ch = line[i]
            if ch == "'" and not in_double:
                in_single = not in_single
            elif ch == '"' and not in_single:
                in_double = not in_double
            elif ch == '/' and not in_single and not in_double and i + 1 < len(line) and line[i+1] == '/':
                break  # rest of line is a comment
            out.append(ch)
            i += 1
        stripped = ''.join(out).strip()
        if stripped:
            lines.append(stripped)
    # Collapse all remaining whitespace runs to a single space
    return re.sub(r'\s+', ' ', ' '.join(lines)).strip()


def minify_css(source: str) -> str:
    """Strip comments and collapse whitespace from CSS source."""
    # Remove block comments  /* … */
    source = re.sub(r'/\*.*?\*/', '', source, flags=re.DOTALL)
    # Collapse whitespace
    source = re.sub(r'\s+', ' ', source)
    # Remove spaces around structural punctuation
    source = re.sub(r'\s*([{};:,])\s*', r'\1', source)
    return source.strip()

# ── encode ───────────────────────────────────────────────────────────────────

def to_chunks(data: bytes) -> str:
    """Convert bytes to g_varchar2_table assignment lines (100-byte / 200-hex-char chunks)."""
    chunks = [data[i:i+100].hex().upper() for i in range(0, len(data), 100)]
    return "\n".join(
        f"wwv_flow_imp.g_varchar2_table({i+1}) := '{chunk}';"
        for i, chunk in enumerate(chunks)
    ), len(chunks)

# ── patch helper ──────────────────────────────────────────────────────────────

def replace_blob(sql_text: str, file_name: str, new_entries: str) -> str:
    """Replace the g_varchar2_table entries for a named plugin file."""
    marker     = f"p_file_name=>'{file_name}'"
    marker_pos = sql_text.find(marker)
    if marker_pos == -1:
        raise ValueError(f"Marker not found: {marker}")

    search  = "end;\n/\nbegin\n"
    end_pos = sql_text.rfind(search, 0, marker_pos)

    header     = "wwv_flow_imp.g_varchar2_table := wwv_flow_imp.empty_varchar2_table;\n"
    header_pos = sql_text.rfind(header, 0, end_pos)
    if header_pos == -1:
        raise ValueError(f"Header not found before: {file_name}")

    entries_start = header_pos + len(header)
    return sql_text[:entries_start] + new_entries + "\n" + sql_text[end_pos:]

# ── apply ─────────────────────────────────────────────────────────────────────

with open(READABLE_JS, 'r', encoding='utf-8') as f:
    js_source = f.read()

with open(READABLE_CSS, 'r', encoding='utf-8') as f:
    css_source = f.read()

js_min  = minify_js(js_source)
css_min = minify_css(css_source)

js_entries,     js_chunks     = to_chunks(js_source.encode('utf-8'))
js_min_entries, js_min_chunks = to_chunks(js_min.encode('utf-8'))
css_entries,    css_chunks    = to_chunks(css_source.encode('utf-8'))
css_min_entries,css_min_chunks= to_chunks(css_min.encode('utf-8'))

with open(SQL_FILE, 'r', encoding='utf-8') as f:
    sql = f.read()

sql = replace_blob(sql, f"{PLUGIN_NAME}.js",     js_entries)
sql = replace_blob(sql, f"{PLUGIN_NAME}.min.js", js_min_entries)
sql = replace_blob(sql, f"{PLUGIN_NAME}.css",    css_entries)
sql = replace_blob(sql, f"{PLUGIN_NAME}.min.css",css_min_entries)

with open(SQL_FILE, 'w', encoding='utf-8') as f:
    f.write(sql)

print(f"JS  : {js_chunks} chunks (source)  |  {js_min_chunks} chunks (minified)")
print(f"CSS : {css_chunks} chunks (source)  |  {css_min_chunks} chunks (minified)")
print("Done.")
```

---

## JS-only or CSS-only Patches

If only JS changed, comment out the CSS `replace_blob` calls (and vice versa).
The unused minifier functions can stay — they are harmless.

---

## Verifying Minified Output

Use the decoding helper from `hex_blob_management.md`, substituting
`<plugin_name>.min.js` or `<plugin_name>.min.css` as the `FILE_NAME`, to
confirm the blob contains the minified (comment-free, whitespace-collapsed) version.
