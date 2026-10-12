# Architecture

## Four files

The whole application is four files in one Apps Script project.

| File | Role | Size (build 12.0) |
|---|---|---|
| `Code.gs` | Server: every sheet read/write, auth, notifications, backups, migrations, sample data | ~4,400 lines |
| `index.html` | Page structure for every view, help pages, changelog | ~2,300 lines |
| `styles.html` | All CSS (`<style>` block, included by `index.html`) | ~2,800 lines |
| `script.html` | All client JavaScript (`<script>` block, included by `index.html`) | ~5,000 lines |

`index.html` is served through `HtmlService.createTemplateFromFile('index').evaluate()`.
Template scriptlets pull the other two files in (`<?!= include('styles'); ?>`) and stamp
the version constants into the login footer, so the version is visible before sign-in.

No framework, no bundler, no npm dependency at runtime. Google Fonts is the only external
resource. This was a deliberate choice: the maintainer is an IT manager, not a developer,
and anything that cannot be pasted into the Apps Script editor is a liability.

## Request flow

Every client → server call goes through one wrapper:

```
callServer(functionName, ...args)
  → appends state.sessionToken as the last argument
  → google.script.run.withSuccessHandler / withFailureHandler
  → returns a Promise
  → on a "session invalid" failure, logs the tab out and shows why
```

On the server every public function starts with one of:

| Guard | Used by |
|---|---|
| `requireSession_(token)` | anything that reads or writes data |
| `requireAdmin_(token)` | user management, notification settings, SLA config, CSV import, sample-data seeding |
| `isFromTrigger_(e)` | the three time-driven functions (daily backup, weekly mail, SLA digest) |
| `requireOwnerContext_()` | one-off migrations run by the script owner from the editor |

`requireSession_` validates the cached token **and** re-reads the user's row in the
`AuthUsers` sheet on every call, so disabling a user or removing admin rights takes effect
immediately rather than when the 6-hour session expires.

Functions whose names end in `_` cannot be called from the browser at all (Apps Script
rule). Everything that is not meant to be an API has the suffix.

## Server layout

```
Code.gs
├─ constants: sheet names, column lists (TICKET_COLS, ASSET_COLS …), defaults, OTP limits
├─ helpers:   ensureSheet_, cellToStr_, toCell_/sanitizeRow_, withScriptLock_, requireOwnerContext_
├─ members:   loadITMembers_ (derived from AuthUsers), legacy-member migration, rename cascade
├─ auth:      requestOTP, verifyOTP, validateSession_, requireSession_, requireAdmin_, invalidateSession
├─ users:     loadAuthUsers_ (+ self-healing schema), saveAuthUser, deleteAuthUser
├─ setup:     setupSheet → setupXxxSheet_ for each of the 14 sheets (idempotent)
├─ per-sheet: loadXxx_ (raw) / loadXxx(token) (guarded) / saveXxx / deleteXxx
├─ tickets:   appendTicket, updateTicket (ID generation under a script lock), notifyAssignment_
├─ notifications: global flags in Config, per-user prefs, SLA digest
├─ backups:   dailyBackup(e) / weeklyBackupEmail(e) + manual backupNow(token)
├─ csv:       parseCSV_, escapeCSVField_, preview/apply importers (merge-only)
└─ sample data: seedDummyXxx(token) and run*() entry points for migrations
```

Two conventions make the file navigable despite its size:

- **`loadXxx_()` vs `loadXxx(token)`.** The underscore version is the raw loader used
  internally (and by triggers, which have no session). The public version only adds the
  guard. Nothing else ever reads a sheet directly.
- **`ensureSheet_(name, setupFn)`.** Every sheet access goes through it. If the sheet is
  missing it is created with its header row, so a deleted or renamed sheet never surfaces
  as a cryptic "sheet not found".

## Client layout

```
script.html
├─ state      one object: session, current view, all loaded data, filters, drafts
├─ callServer, toast, showLoading, escapeHTML, $ / $$
├─ history    snapshotView_, pushViewHistory_, restoreViewHistory_, hashchange listener
├─ boot       restore session from localStorage → getInitData → enterApp
├─ idle       30-minute timer, warning modal, multi-tab awareness via localStorage
├─ views      showView(name) → renderXxx(); one render function per screen
├─ forms      ticket (full + quick capture), asset (58 fields, 10 collapsible sections), employee, rack layout (drag & drop)
├─ admin      members / users / notifications / SLA / data I/O / backup status / admin guide
└─ bindEvents all listeners registered once at boot; lists use delegated handlers
```

`getInitData()` returns everything the UI needs in one round trip (tickets, assets,
employees, masters, SLA config, member list …). Subsequent saves re-fetch only the affected
collection. A header "reload" button re-runs `getInitData` and re-renders the current view
without losing scroll position or open panels — deliberately different from a browser
refresh, which users had learned to fear.

## Navigation and the sandbox

Apps Script serves the page inside a sandboxed iframe. `history.pushState` is not usable
there, and a plain browser "back" leaves the app — which, before build 11.15, meant users
were silently signed out several times a day.

The fix is a hash-based history:

- Every `showView` pushes a snapshot (view name, entity ids, sub-tab, filters) onto an
  in-memory array and writes `location.hash = '#v<index>'`.
- A `hashchange` listener maps the index back to the snapshot and **re-opens the entity**
  (`openEdit(id)`, `openEditAsset(id)` …) rather than just toggling CSS classes, so the
  restored screen shows the right record and the right dropdown state.
- The first entry uses `location.replace` so there is nothing "before" the app.
- Login/logout reset the array, so a new user cannot "back" into the previous user's
  screens.
- A header "← back" button simply calls `history.back()`; browser back and in-app back are
  the same thing.

Edge cases that needed explicit handling: restoring a "new ticket" form must keep an
unsent draft but must **discard** a form that still holds another ticket's data (otherwise
"new → edit → back → submit" created a duplicate).

## Time-driven jobs

| Trigger | What it does |
|---|---|
| Daily, early morning | Exports all 14 sheets to CSV into `Drive/IT-Tracker-Backups/yyyy-MM-dd/`, keeps 90 days, records success/failure for the admin screen |
| Weekly, Monday morning | Builds one XLSX of all sheets and mails it to administrators |
| Daily, early morning (weekdays only) | Per-member digest of tickets past their SLA resolution target |

Triggers call public functions by name, so those three cannot carry the `_` suffix.
They verify `e.triggerUid` against the project's registered triggers and return
immediately if called any other way.

## Versioning

Three values define a release: a **codename** (famous diamonds, changed only on a major
version), a **build** (`12.0`), and a **date**. They live in three constants at the top of
`Code.gs`; the login footer, the system-info page, the preview builder and the file
headers all derive from or are checked against them. A grep command in the project notes
detects drift before a release.
