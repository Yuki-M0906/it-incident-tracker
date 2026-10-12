# Data model

Google Sheets is the only store. One sheet per entity, header row frozen, one record per
row, no formulas. Column lists are constants in the server file and are the single source
of truth for both the header row and the object keys exchanged with the client.

## The 14 sheets

| Sheet | Cols | Purpose | Key |
|---|---|---|---|
| Tickets | 18 | Incident / inquiry / service request / improvement / after-hours record | `ID` (`yyyy-NNNN`, per-year sequence) |
| Config | 2 | Key/value: system list for tickets, global notification switches | `Category` |
| Assets | 58 | IT assets in 10 sections (basic, accounting, user, location, OS, network, maintenance, lifecycle, admin, rack) | `資産管理番号` (asset id, `PREFIX-00001`) |
| Employees | 12 | Employee directory (id, name, kana, department, site, title, dates, status, manager id, extension, note) | `社員番号` (employee id) |
| Departments | 1 | Department master | name |
| Sites | 1 | Site master | name |
| AuthUsers | 7 | Login users = IT members: e-mail, display name, admin flag, status, note, **member flag**, **employee id** | `email` |
| AuthLog | 5 | Authentication events (timestamp, e-mail, event, result, note) | append-only |
| Handover | 9 | Team handover notes with threaded replies (`ParentID`) | `ID` |
| Schedule | 6 | Member × date entries (leave, remote, trip …) | `ID` |
| NotificationPrefs | 3 | Per-user opt-in flags for assignment mail and SLA digest | `email` |
| Racks | 7 | Server racks (site, room, name, height in U, installed date, note) | `RackID` |
| InventoryStaging | 15 | Rows imported from the SNMP scanner, reviewed before promotion to Assets | `StagingID` |
| SLAConfig | 5 | Per-priority response / resolution targets and descriptions | `priority` |

Schema changes are versioned in the project notes. Adding a column never reorders
existing ones; loaders read `COLS.length` columns and the setup function appends missing
headers to an existing sheet (migration branch), so an older sheet keeps working after an
upgrade. For the two most recent column additions the loader itself repairs the schema on
first read, so forgetting to run the setup function after deploying is harmless.

## Keys, references and derived columns

The important design question in a spreadsheet-backed app is **which column is the truth
when the same fact appears twice**. The current rules:

| Fact | Truth | Derived copies | Kept in sync by |
|---|---|---|---|
| Who the IT members are | `AuthUsers` rows with member flag = TRUE and status active | — (member list is computed, not stored) | n/a |
| A member's display name | `AuthUsers.氏名` (unique) | `Tickets.CreatedBy/AssignedTo`, `Handover.投稿者`, `Schedule.メンバー`, `Assets.更新者`, `InventoryStaging.ImportedBy` | rename cascade on save |
| A member as an employee | `AuthUsers.社員番号` → `Employees` | — | validated on save (must exist, one user per employee) |
| Ticket requester | `Tickets.RequesterID` → `Employees` | `Tickets.Requester` (free text, snapshot at creation) | display/search/mail resolve the id first and fall back to the text; text is history |
| Asset user | `Assets.利用者社員番号` → `Employees` | `Assets.利用者氏名/所属部門/所属拠点` | written from the employee on asset save and import; re-written when the employee changes |
| Department / site names | `Departments` / `Sites` | columns in Employees, Assets, Racks, InventoryStaging | rename cascade on save; delete reports reference counts |
| Rack placement | `Assets.ラックID/起点U/占有U` → `Racks` | — | validated (rack exists, within height, no overlap) on form save, CSV import and layout save |
| SLA targets | `SLAConfig` sheet | hard-coded defaults only as fallback | save requires exactly P1–P4 |

Free-text snapshots are kept on purpose: a ticket should still show who asked for it
after the employee leaves. Everything that is displayed, searched or mailed goes through
a resolver that prefers the id and appends "(deleted)" when the id no longer resolves.

## Reading and writing cells

Two helpers wrap every read and write, because Sheets interprets what you write the way
it would interpret typed input:

- `cellToStr_(v)` — booleans back to `'TRUE'`/`'FALSE'`, `Date` objects to
  `yyyy-MM-dd[ HH:mm]` in the project time zone, everything else to a string. Without it,
  a `'TRUE'` written by the app comes back as `true`, and `=== 'TRUE'` comparisons fail
  forever after the first round-trip.
- `toCell_(v)` / `sanitizeRow_(row)` — prefixes a `'` to strings starting with `=` or `+`
  so they are stored as text, not evaluated as formulas.

Writes that generate an id (`tickets`, handover notes, schedule entries, racks, staging
rows) take a script lock around "read max id → append", so two people saving at the same
moment cannot get the same id.

## Backups

- **Daily CSV**: every sheet, UTF-8 with BOM, one folder per day, 90-day retention. The
  backup folder is remembered by id (not looked up by name) so a trashed folder with the
  same name is never reused.
- **Weekly XLSX**: one workbook, mailed to administrators, giving an off-Drive copy.
- **Restore** is documented as a runbook: single-sheet restore from CSV, whole-spreadsheet
  restore, and rebuild from the weekly XLSX.

## Sample data

A set of mutually consistent sample records (departments, sites, employees, assets,
tickets) can be loaded from the admin screen for evaluation. They share one id space and
one vocabulary with the default masters, so demo screens show cross-links that actually
resolve. Real deployments never load them; the buttons skip ids that already exist.
