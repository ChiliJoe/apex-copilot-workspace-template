---
description: "Use when testing the APEX application via Playwright, browser automation, or E2E testing. Covers the APEX app URL, login flow, and session handling for the APEX app."
---
# APEX Application Testing

## Application URL

The Oracle APEX application is hosted at: {apex-workspace-runtime-url}/{app-alias}/. Use this URL for all testing activities, including Playwright tests, browser automation, and manual testing.

When navigating / rewriting the URL for testing, ensure the `session` query parameter is included to maintain session state and application context. The URL should follow this format:

```
{apex-workspace-runtime-url}/{app-alias}/{page-alias}/?session={session-id}
```

Where `{session-id}` is the active session identifier for the APEX application. This ensures that any automated testing or browser navigation maintains the correct session context and allows for accurate testing of authenticated features and application behavior. Always verify that the session parameter is included when accessing the app to avoid unexpected redirects or authentication issues during testing.

## Login Flow

The APEX app requires authentication. When testing via Playwright or browser automation:

1. Navigate to the application URL
2. The app will redirect to the APEX login page if no active session exists
3. **Do NOT attempt to automate login**. Instead, pause and ask the user to log in manually
4. After the user confirms login, take a snapshot to verify the authenticated page loaded
5. Proceed with automated testing

## Troubleshooting Internal Errors

If you encounter an internal error, check the contents of APEX_DEBUG_MESSAGES view in the database for detailed error messages and stack traces. This can provide insights into what went wrong during testing and help identify any issues with the application or test scripts.

If needed, you can also enable additional debugging in the APEX application by setting the `debug` parameter in the URL to `YES`:

```{apex-workspace-runtime-url}/{app-alias}/?debug=YES&session={session-id}
```

## SQLcl Connection

Always resolve `{connection-name}` from `.github/.copilot-context.md` before connecting to the database.

### Preferred: SQLcl MCP Server

When the SQLcl MCP server is available, use it for all database queries and PL/SQL tests:

1. **Connect** — call `mcp__sqlcl__connect` with the `{connection-name}` from `.github/.copilot-context.md`.
2. **Run queries** — call `mcp__sqlcl__run-sql` with the SQL statement (e.g., `SELECT * FROM APEX_DEBUG_MESSAGES WHERE ...`).

This is the preferred method for ad-hoc debugging queries such as inspecting `APEX_DEBUG_MESSAGES`.

### Fallback: Terminal

If the SQLcl MCP server is unavailable, use SQLcl in the terminal:

```
sql -name {connection-name}
```

Where `{connection-name}` is configured to connect to the {app-schema} schema on the same database. This allows you to run SQL and PL/SQL tests against the same database that the APEX app uses, ensuring consistency between application behavior and database state during testing.
