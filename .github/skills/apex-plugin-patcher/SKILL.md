---
name: apex-plugin-patcher
description: "Patch an Oracle APEX plugin's PL/SQL render/AJAX logic or its static JS/CSS files. Edits the human-readable source files, re-encodes hex blobs, and syncs the plugin installation SQL file — then optionally installs and validates via SQLcl MCP. TRIGGER when: user asks to fix, modify, or add behaviour to an APEX plugin (COM.* plugin names, plugin SQL files, or plugin JS/CSS static files)."
argument-hint: "[conn] [plugin-name] -- <change description>"
disable-model-invocation: false
allowed-tools: Read, Write, Edit, Grep, Glob, Bash(python3 *), mcp__sqlcl__connect, mcp__sqlcl__run-sql, mcp__sqlcl__run-sql-async
---

# Oracle APEX Plugin Patcher

## Purpose
Use this skill to **patch an Oracle APEX plugin** — its embedded PL/SQL (render/AJAX functions), its static JS/CSS files, or both — using a controlled workflow that keeps the human-readable sources and the binary-encoded installation SQL in sync.

The workflow produces:
1. Edited readable source file(s) (`.js`, `.css`)
2. A patched installation SQL file with re-encoded hex blobs
3. An optional SQLcl install and validation step

> This workflow modifies versioned files and (optionally) the live APEX database. The skill can be invoked by both the user and the model (`disable-model-invocation: false`).

## Inputs
Recommended argument structure (free-form is fine, but this is preferred):
- `$0`: SQLcl connection name (e.g., `DEV`, `STG`) — passed to `sql -name <conn>` for optional install step
- `$1`: plugin name or path fragment (e.g., `COM_VENDOR_MY_PLUGIN` or just `my_plugin`)
- Remaining args: what to change and why

If the connection name is omitted, skip the install step and just produce patched files.
If the plugin name or file paths are unclear, use `Glob` to discover them under `apex/f<APP_ID>/application/shared_components/plugins/`.

## Preconditions / Assumptions
- Human-readable source files exist under `apex/f<APP_ID>/readable/application/shared_components/plugin_static_files/`
- The installation SQL file exists under `apex/f<APP_ID>/application/shared_components/plugins/item_type/`
- Python 3 is available on PATH for hex re-encoding
- For live install: connect via `mcp__sqlcl__connect` using the `{connection-name}` from `.github/.copilot-context.md`. Fall back to terminal `sql -name {connection-name}` only when the SQLcl MCP server is unavailable.

## Reference documentation
This skill directory contains:
- `tools/hex_blob_management.md` — canonical re-encode script and blob boundary rules
- `tools/minification.md` — canonical JS/CSS minify + re-encode script
- `tools/plsql_string_literals.md` — how to edit PL/SQL embedded as string literals
- `templates/patch_plan.md` — planning template
- `references/plugin_structure.md` — full plugin SQL file anatomy

## Workflow (follow every time)

### 1) Locate plugin files
1. Find the installation SQL: `Glob apex/f*/application/shared_components/plugins/item_type/*<plugin>*.sql`
2. Find readable static files: `Glob apex/f*/readable/application/shared_components/plugin_static_files/*<plugin>*/*`
3. Note all four blobs that may need updating: `.js`, `.min.js`, `.css`, `.min.css`

### 2) Read and understand current code
1. Read the relevant section(s) of the readable `.js` or `.css` before making any change.
2. For PL/SQL changes: read `tools/plsql_string_literals.md`, then search the SQL file for the target function or `htp.p` block using `Grep` with a distinctive anchor.
3. Understand the current logic before proposing a fix — do not guess.

### 3) Plan the patch
Fill in `templates/patch_plan.md`:
- Mark whether the change touches PL/SQL, static files, or both
- List exact edit anchors (function names, surrounding lines) for each change
- List which hex blobs need re-encoding after edits

### 4) Edit readable source files
- **JS/CSS**: use `Edit` on the readable `.js` / `.css` file with exact `old_string` / `new_string`
- **PL/SQL string literals**: follow `tools/plsql_string_literals.md` — edit the `wwv_flow_string.join(...)` section in the SQL file using stable grep-anchored strings, doubling any internal single quotes
- Keep changes minimal: only touch what the request requires

### 5) Minify and re-encode hex blobs
For every static file changed, run the canonical Python script from `tools/minification.md`:
```bash
python3 - << 'EOF'
# ... (paste the minify + re-encode script, substituting actual file paths)
EOF
```
Rules:
- `.js` / `.css` blobs are encoded from the readable source (unchanged)
- `.min.js` / `.min.css` blobs are encoded from the minified version (comments stripped, whitespace collapsed)
- Always sync both blobs in a pair whenever the readable source changes
- Never manually edit hex entries

### 6) Verify SQL structure
```bash
grep -n "create_plugin_file\|g_varchar2_table :=\|empty_varchar2_table\|^end;\|^begin$" path/to/plugin.sql
```
Confirm:
- `begin` / `end;` count is balanced
- Each blob section is followed by its correct `create_plugin_file(p_file_name=>...)` block
- `wwv_flow_imp.component_end` still present at the end

### 7) Install and validate (optional)
Always resolve `{connection-name}` from `.github/.copilot-context.md` before connecting.

**Preferred — SQLcl MCP Server:**
1. Connect via `mcp__sqlcl__connect` with `{connection-name}`.
2. Run install via `mcp__sqlcl__run-sql` with `@apex/<appid>/application/shared_components/plugins/item_type/<plugin_file>.sql`.

**Fallback — Terminal** (if MCP is unavailable):
```sh
sql -name {connection-name} @apex/<appid>/application/shared_components/plugins/item_type/<plugin_file>.sql
```
Then:
- Hard-refresh the browser (Ctrl+Shift+R) to bust cached JS/CSS
- Exercise the changed behaviour in the running APEX application
- Check browser Network tab: confirm file size changed

### 8) Report
Deliver to the user:
- Summary of what was changed (readable file edits + hex blobs re-encoded)
- List of modified files
- Install command to deploy
- Any warnings or caveats

## Error-handling playbook
- **Hex blob boundary not found**: verify the `p_file_name=>` marker exists and the `end;\n/\nbegin\n` pattern separates blobs correctly (see `references/plugin_structure.md`)
- **SQL structure broken after re-encode**: run the verify step in §6; if `begin`/`end;` counts are off, inspect around the patched blob section manually
- **PL/SQL string literal corrupted**: count opening vs closing single quotes on the edited lines; ensure all internal `'` are doubled; re-read `tools/plsql_string_literals.md`
- **APEX install error "component not found"**: the workspace/schema context may differ — check the `component_begin` parameters match the target environment
- **Browser still shows old behaviour after install**: hard-refresh; if still stale, check APEX is serving the updated file (Network tab → file size)

## Examples
- `/apex-plugin-patcher my_plugin -- Fix X button removal failing for numeric return values`
- `/apex-plugin-patcher my_plugin -- Add inline search input that auto-opens modal on typing`
- `/apex-plugin-patcher my_plugin -- Add a new plugin attribute for minimum search term length`
