---
description: "Use when testing the APEX application via Playwright, browser automation, or E2E testing. Covers the APEX app URL, login flow, and session handling for the APEX app."
---
# APEX Application Testing

## Application URL

The Oracle APEX application is hosted at: {app-url}

- **App ID**: {appid}
- **App Alias**: {app-alias}
- **Schema**: {app-schema}
- **Database Version**: {database-version}

## Login Flow

The APEX app requires authentication. When testing via Playwright or browser automation:

1. Navigate to the application URL
2. The app will redirect to the APEX login page if no active session exists
3. **Do NOT attempt to automate login**. Instead, pause and ask the user to log in manually
4. After the user confirms login, take a snapshot to verify the authenticated page loaded
5. Proceed with automated testing

## SQLcl Connection

When running database queries or PL/SQL tests, use:

```
sql -name {connection-name}
```

Where `connection-name` is configured to connect to the {app-schema} schema on the same database. This allows you to run SQL and PL/SQL tests against the same database that the APEX app uses, ensuring consistency between application behavior and database state during testing.
