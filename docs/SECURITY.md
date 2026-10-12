# Security model

## The deployment and what it implies

The web app is deployed as **"anyone, even anonymous, can access" / "execute as the
owner"**. That is the only mode in which users without accounts in the hosting Google
Workspace can open the page, and it has one consequence that drives everything else:

> Anyone who knows the URL can call **any** global function in the script, with the
> owner's permissions, by name, from the browser console.

So the server cannot trust the page. It can only trust what it checks itself.

## Rules

1. **Every data function takes a session token as its last argument and validates it.**
   The client wrapper appends the token automatically; the server checks it before doing
   anything. Admin-only functions check the admin flag as well.
2. **Anything that must not be callable has a `_` suffix.** Apps Script refuses to expose
   such functions to `google.script.run`. Internal loaders, cascades, mail senders,
   migrations and seeders are all suffixed.
3. **Time-driven functions verify they were started by a trigger.** They cannot be
   suffixed (triggers call them by name), so they compare the event's `triggerUid` with the
   project's registered triggers and bail out otherwise. Without this, anyone with the URL
   could fire the backup mail or the SLA digest in a loop and exhaust the daily mail quota
   — which would also lock everyone out, because login codes go by mail.
4. **Manual migrations are owner-gated.** `_`-suffixed functions do not appear in the
   editor's run menu either, so each has a public `run*()` entry that first requires the
   active user to equal the effective user — true only for the script owner running it from
   the editor.
5. **Clickjacking off.** `XFrameOptionsMode.DEFAULT`, not `ALLOWALL`.

## Authentication

E-mail one-time codes, no passwords.

```
enter e-mail ──▶ is it in AuthUsers and active? ──▶ 8-digit code (UUID-derived, not Math.random)
                                                      stored in CacheService, 10 min TTL
                                                      sent by mail
enter code   ──▶ matches? ──▶ session token (UUID) in CacheService, 6 h
                              client keeps it in localStorage
```

Limits, all enforced server-side:

| Limit | Value | Why |
|---|---|---|
| Codes per address | 5 per 10 min | ordinary brute-force protection |
| Codes overall | 30 per 10 min, checked **before** the address is looked up | an attacker cycling addresses must not drain the mail quota, and must not fill the auth log with lookups |
| Re-send interval per address | 60 s | double-clicks and mail bombing |
| Wrong attempts per code | 8, then the code is destroyed | brute force on a known-issued code |
| Counter reset | issuing a new code resets the failure counter | otherwise a third party could lock a user out by burning attempts against their address |

Every attempt — sent, denied, failed, success — is appended to `AuthLog` with a reason.

## Sessions

- 6-hour hard limit (cache TTL). 30-minute idle logout on the client with a 2-minute
  warning.
- `requireSession_` re-reads the user's `AuthUsers` row on every call: disabling, deleting
  or de-admin-ing a user applies to their next click, not their next login.
- Logout invalidates the server-side token.
- Multiple tabs share one token. An idle tab therefore checks a shared "last activity"
  timestamp before invalidating; if another tab is active it only returns itself to the
  login screen.
- Any server response containing the session-invalid message sends the tab to the login
  screen with the reason, from any view.

## Data hygiene

- Formula injection: every string written to a sheet or a CSV export that begins with
  `=`, `+`, `-`, `@`, tab or CR is prefixed so spreadsheets treat it as text.
- HTML injection: every user-provided string rendered in the page goes through one
  escape function. Lists are built from templates that escape every field.
- Notification keys written to the config sheet are checked against an allow-list.
- The "last administrator" and the "last active member" cannot be disabled or deleted.
- Display names of login users are unique, because they are the key other sheets store.

## Things that are deliberately not done

- No per-record ACLs. Everyone who can log in can read and edit every ticket, asset and
  employee; only user management, notifications, SLA config and bulk import are admin-only.
  The team is three or four people who already share this responsibility.
- No audit trail of data edits (only of authentication). Last-write-wins on concurrent
  edits of the same record.
- No secrets in the repository. The only configuration that is environment-specific is
  the spreadsheet the script is bound to and the trigger schedule.
