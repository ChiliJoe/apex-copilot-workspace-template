Before generating any code or CLI commands, refer to .copilot-context.md to resolve placeholders like {connection-name} or {appid}. Always check the instructions in .github/instructions/ for any specific guidelines related to the file you are working on.

## Database Connectivity

Always prefer the **SQLcl MCP server** for database operations when available:

1. Connect using `mcp__sqlcl__connect` with the `{connection-name}` defined in `.github/.copilot-context.md`.
2. Execute SQL and scripts using `mcp__sqlcl__run-sql` or `mcp__sqlcl__run-sql-async`.
3. Fall back to the terminal command `sql -name {connection-name}` **only** when the SQLcl MCP server is unavailable.

The `{connection-name}` in `.github/.copilot-context.md` is the single source of truth for which database connection to use. Resolve it before every database interaction.