# Design principles

The decisions below were not written down first and followed later. They emerged from
specific incidents — a user signed out by the browser's back button, a notification that
silently stopped, a dropdown that could not offer a new site — and were then kept as rules.
They are listed here because the same rules would apply to any small-team internal tool,
not just this one.

## 1. No infrastructure

The spreadsheet is the database, the script is the server, the HTML service is the web
server, Drive is the backup target, Gmail is the mail relay, time-driven triggers are the
scheduler. Nothing is installed, hosted, patched or paid for. The price is accepting the
platform's limits (no transactions, typed cells, a sandboxed iframe, mail quotas) and
designing around them rather than fighting them.

## 2. The maintainer is not a developer

Everything that ships must be pasteable into the Apps Script editor by an IT manager:
four plain files, no framework, no bundler, no npm at runtime. Tooling *around* the app
(preview builder, test runner, manual generator) may use Node and PowerShell, because it
can be skipped without breaking anything. Comments explain *why* a line exists, with the
build that introduced it, so the next reader can decide whether it still applies.

## 3. The sheet is the audit trail

Every record is a visible row and every schema change is a visible column. When a number
on a dashboard looks wrong, the operator can open the sheet and see the data the way the
code sees it. The code therefore never stores anything the operator could not interpret:
no serialised blobs, no formulas, no hidden sheets, no encoded ids.

## 4. Trust the server's checks, never the page

The deployment mode that lets the team's users in also lets anyone with the URL call any
public function. So the page is treated as hostile: every function validates a session
token, admin functions validate the admin flag, trigger functions validate their trigger,
migrations validate their caller, and anything else is suffixed out of reach. Client-side
checks exist for usability only.

## 5. One fact, one home

Whenever the same real-world thing appears in two records, decide which one is the truth
and make the other a derived copy or an explicit historical snapshot. Derived copies are
rewritten by the save path of the truth; snapshots are labelled as such in the UI. The
cost of not deciding is paid months later, in reports that quietly disagree.

## 6. Idempotent everything

Setup creates what is missing and never touches what exists. Seeders skip ids that are
already there. Column additions repair themselves on first read. Backup folders are found
by id, not by name. A step that is safe to repeat is a step that can be put on a checklist
and run after every deployment without thinking.

## 7. Fail loudly where a person is looking

A missing sheet is created rather than reported, because nobody is looking at the log.
A failed backup is written to a status sheet and shown on the admin screen, because
somebody is looking there. A failed mail send is written to the auth log with the reason.
Silent failure is the worst outcome; the second worst is a failure reported where nobody
reads.

## 8. Capture first, structure later

The quick-capture box on the dashboard takes a type and a title and nothing else. A
ticket that exists with two fields is better than a perfect ticket that was never created
because the phone was ringing. Everything else — requester, asset, priority, resolution —
can be added when the call is over.

## 9. Version everything the user can see

Codename, build and date live in three constants and appear on the login screen, the
system page, every file header and the manual's cover. A grep catches drift before a
release. "Which version are you on?" has to be answerable in one glance by the person
asking for help.

## 10. Test against a mock that is held to the server's rules

The app cannot run outside Apps Script, so a preview build injects a mock server. The
mock is required to enforce the same validation the real server does (unique names, last
administrator, rack overlap, derived columns). When they disagree, one of them has a bug —
and it has been the server more than once.
