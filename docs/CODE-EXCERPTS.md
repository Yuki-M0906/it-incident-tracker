# Code excerpts

Short, rewritten excerpts that show the shape of the important parts. They are not copies
of the production files: names of organisations, people, sites and identifier prefixes
have been replaced, comments rewritten, and surrounding code removed. Each is complete
enough to reuse.

Language throughout: JavaScript on the Apps Script V8 runtime (server) and in the browser
(client); PowerShell 5.1 for the field tools.

---

## 1. One RPC wrapper on the client

Every call to the server goes through this. It appends the session token, turns the
callback API into a Promise, and recognises an expired session anywhere in the app.

```javascript
function callServer(funcName) {
  const args = Array.prototype.slice.call(arguments, 1);
  if (state.sessionToken) args.push(state.sessionToken);   // always the last argument
  return new Promise((resolve, reject) => {
    const r = google.script.run
      .withSuccessHandler(resolve)
      .withFailureHandler(err => {
        const msg = (err && err.message) || String(err);
        if (state.sessionToken && msg.indexOf(SESSION_INVALID_TEXT) >= 0) {
          handleSessionExpired_();        // back to the login screen, with the reason
        }
        reject(new Error(msg));
      });
    r[funcName].apply(r, args);
  });
}
```

For example, a save is `await callServer("saveAsset", asset, "update")` — the token is
never mentioned at call sites, so it cannot be forgotten.

---

## 2. Guards on the server

Every public function starts with one of these. `requireSession_` re-reads the user's row
so that disabling someone takes effect on their next click.

```javascript
function requireSession_(token) {
  const sess = validateSession_(token);                 // cache lookup: sess_<token>
  if (!sess) throw new Error(SESSION_INVALID_TEXT);
  const u = findAuthUser_(sess.email);                  // re-check the sheet every time
  if (!u || (u.status && u.status !== ACTIVE)) {
    CacheService.getScriptCache().remove('sess_' + token);
    throw new Error(SESSION_INVALID_TEXT);
  }
  sess.isAdmin = !!u.isAdmin;                           // refresh, do not trust the cache
  return sess;
}

function requireAdmin_(token) {
  const sess = requireSession_(token);
  if (!sess.isAdmin) throw new Error('Administrators only');
  return sess;
}

// Public API: guard, then delegate to the raw loader. Triggers use loadAssets_() directly.
function loadAssets(token) { requireSession_(token); return loadAssets_(); }
```

---

## 3. Trigger functions prove they were triggered

Time-driven functions cannot carry the `_` suffix (triggers call them by name), so they
check the event object against the project's own triggers.

```javascript
function isFromTrigger_(e) {
  if (!e || !e.triggerUid) return false;
  try {
    return ScriptApp.getProjectTriggers().some(t => t.getUniqueId() === String(e.triggerUid));
  } catch (x) { return false; }
}

function dailyBackup(e) {            // public by necessity
  if (!isFromTrigger_(e)) return;    // anyone else gets nothing
  dailyBackup_();
}

function backupNow(token) {          // the manual button has its own guard
  requireAdmin_(token);
  return dailyBackup_();
}
```

---

## 4. Owner-gated entry points for migrations

A `_`-suffixed function is hidden from the browser **and** from the editor's run menu.
Migrations therefore get a public entry that only works for the script owner running it
from the editor.

```javascript
function requireOwnerContext_() {
  let active = '', effective = '';
  try {
    active    = String(Session.getActiveUser().getEmail()    || '').toLowerCase();
    effective = String(Session.getEffectiveUser().getEmail() || '').toLowerCase();
  } catch (e) { /* fall through */ }
  if (!active || active !== effective) {
    throw new Error('Run this from the Apps Script editor as the script owner.');
  }
}

function runMigrateAssets() { requireOwnerContext_(); return migrateAssets_(); }
```

When called through the web app, the active user is anonymous or someone else and the
effective user is the owner, so the check fails.

---

## 5. Reading and writing cells safely

Sheets interprets written values like typed input. These two helpers wrap every read and
write.

```javascript
// read: booleans and dates come back as strings the rest of the code expects
function cellToStr_(v, dateFormat) {
  if (v == null || v === '') return '';
  if (v === true)  return 'TRUE';
  if (v === false) return 'FALSE';
  if (v instanceof Date) return Utilities.formatDate(v, TIMEZONE, dateFormat || 'yyyy-MM-dd HH:mm');
  return String(v);
}

// write: a leading = or + would be evaluated as a formula; force text
function toCell_(v) {
  if (typeof v === 'string' && /^[=+]/.test(v)) return "'" + v;
  return v;
}
function sanitizeRow_(row) { return row.map(toCell_); }

// a typical loader
function loadHandover_() {
  const sheet = ensureSheet_(SHEET_HANDOVER, setupHandoverSheet_);
  const last = sheet.getLastRow();
  if (last < 2) return [];
  return sheet.getRange(2, 1, last - 1, HANDOVER_COLS.length).getValues().map(r => {
    const obj = {};
    HANDOVER_COLS.forEach((col, j) => { obj[col] = cellToStr_(r[j]); });
    return obj;
  });
}
```

For example, a "closed" flag written as `'TRUE'` comes back as `true`; without
`cellToStr_`, `obj.closed === 'TRUE'` is false forever after the first save.

---

## 6. Sheets that create themselves

```javascript
function ensureSheet_(name, setupFn) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(name);
  if (!sheet && setupFn) { setupFn(ss); sheet = ss.getSheetByName(name); }
  return sheet;
}

function setupRacksSheet_(ss) {          // idempotent: creates only if missing
  if (ss.getSheetByName(SHEET_RACKS)) return;
  const sheet = ss.insertSheet(SHEET_RACKS);
  sheet.getRange(1, 1, 1, RACK_COLS.length).setValues([RACK_COLS]);
  sheet.setFrozenRows(1);
}
```

Column additions follow the same idea: the loader checks the header row and appends the
missing header before reading, so a sheet from an older build keeps working.

---

## 7. Id generation under a lock

```javascript
function withScriptLock_(fn) {
  const lock = LockService.getScriptLock();
  lock.waitLock(15000);
  try { return fn(); } finally { lock.releaseLock(); }
}

function appendTicket(ticket, token) {
  requireSession_(token);
  const sheet = ensureSheet_(SHEET_TICKETS, setupTicketsSheet_);
  ticket.CreatedAt = ticket.CreatedAt || Utilities.formatDate(new Date(), TIMEZONE, 'yyyy-MM-dd HH:mm');
  withScriptLock_(() => {
    ticket.ID = generateNextTicketId_();                        // read max, +1
    sheet.appendRow(sanitizeRow_(TICKET_COLS.map(c => ticket[c] != null ? ticket[c] : '')));
  });
  return ticket;
}
```

---

## 8. Rate limits in the right order

The global limit is evaluated **before** the address is looked up, so an attacker cycling
through addresses cannot drain the mail quota or bloat the auth log.

```javascript
function requestOTP(email) {
  email = normalizeEmail_(email);
  const cache = CacheService.getScriptCache();

  const globalSends = parseInt(cache.get('otprate_global') || '0', 10);
  if (globalSends >= OTP_GLOBAL_LIMIT) return { ok: false, error: BUSY_TEXT };
  cache.put('otprate_global', String(globalSends + 1), OTP_WINDOW_SEC);

  const user = findAuthUser_(email);
  if (!user || (user.status && user.status !== ACTIVE)) { logAuthEvent_(email, 'requestOTP', 'denied'); return { ok: false, error: NOT_REGISTERED_TEXT }; }

  if (cache.get('otpcool_' + email)) return { ok: false, error: WAIT_TEXT };          // 60 s between sends
  const sends = parseInt(cache.get('otprate_' + email) || '0', 10);
  if (sends >= OTP_SEND_LIMIT) return { ok: false, error: LIMIT_TEXT };               // 5 per 10 min

  const code = generateOTP_();                                                        // UUID-derived digits
  cache.put('otp_' + email, code, OTP_TTL_SEC);
  cache.put('otprate_' + email, String(sends + 1), OTP_WINDOW_SEC);
  cache.put('otpcool_' + email, '1', OTP_RESEND_COOLDOWN_SEC);
  cache.remove('otpverify_' + email);          // a new code resets the failure counter
  sendOTPEmail_(email, user.name, code);
  return { ok: true };
}
```

---

## 9. In-app history inside the sandbox

`history.pushState` is unavailable in the Apps Script iframe. The hash is the history.

```javascript
const viewHistory = [];           // snapshots: { view, editingId, viewingAssetId, subtab, ... }
let historyIndex = -1, pendingHashWrites = 0, restoringHistory = false;

function pushViewHistory_() {
  const snap = snapshotView_();
  viewHistory.length = historyIndex + 1;     // drop any "forward" entries
  viewHistory.push(snap);
  historyIndex = viewHistory.length - 1;
  pendingHashWrites++;
  if (historyIndex === 0) location.replace('#v0');   // nothing "before" the app
  else location.hash = '#v' + historyIndex;
}

window.addEventListener('hashchange', () => {
  if (pendingHashWrites > 0) { pendingHashWrites--; return; }   // we caused this one
  if (!state.sessionToken) return;                              // not while logged out
  const m = /^#v(\d+)$/.exec(location.hash || '');
  if (m) restoreViewHistory_(parseInt(m[1], 10));
});

function restoreViewHistory_(idx) {
  const s = viewHistory[idx]; if (!s) return;
  historyIndex = idx;
  restoringHistory = true;                      // showView must not push while restoring
  try {
    if (s.view === 'edit')            openEdit(s.editingId);          // re-open the entity,
    else if (s.view === 'asset-form') openEditAsset(s.editingAssetId); // do not just toggle CSS
    else                              showView(s.view);
  } finally { restoringHistory = false; }
}

function navBack() { history.back(); }        // header button = browser button
```

---

## 10. Rename cascades

When a member's display name changes, every sheet that stores the name is updated in one
pass per column.

```javascript
function cascadeMemberRename_(oldName, newName) {
  const renameColumn = (sheetName, setupFn, cols, colName) => {
    const sh = ensureSheet_(sheetName, setupFn);
    const last = sh.getLastRow(); if (last < 2) return 0;
    const range = sh.getRange(2, cols.indexOf(colName) + 1, last - 1, 1);
    const vals = range.getValues();
    let n = 0;
    vals.forEach(r => { if (String(r[0]).trim() === oldName) { r[0] = newName; n++; } });
    if (n) range.setValues(vals);
    return n;
  };
  renameColumn(SHEET_HANDOVER,  setupHandoverSheet_,  HANDOVER_COLS,  'author');
  renameColumn(SHEET_SCHEDULE,  setupScheduleSheet_,  SCHEDULE_COLS,  'member');
  renameColumn(SHEET_TICKETS,   setupTicketsSheet_,   TICKET_COLS,    'CreatedBy');
  return renameColumn(SHEET_TICKETS, setupTicketsSheet_, TICKET_COLS, 'AssignedTo');
}
```

The same shape is used for site and department renames, and for pushing an employee's
new name, department and site into the assets they use.

---

## 11. Derived columns on save

```javascript
// Assets: the employee id is the truth; name, department and site are copied from the directory
function applyAssetUserFromEmployee_(asset, empMap) {
  const id = String(asset.userEmployeeId || '').trim();
  asset.userEmployeeId = id;
  if (!id) return;                                   // external user: free-text name allowed
  const emp = (empMap || employeeMap_())[id];
  if (!emp) throw new Error('Employee ' + id + ' does not exist');
  asset.userName   = emp.name;
  asset.department = emp.department || asset.department || '';
  asset.site       = emp.site       || asset.site       || '';
}
```

Called from the form save and from the CSV importer, so both paths produce the same
result. The display side prefers the directory's current name and appends "(deleted)"
when the id no longer resolves.

---

## 12. An allow-list auditor for a read-only script (PowerShell)

The directory export tool must never write to Active Directory. Instead of reviewing it,
a separate script parses it and rejects anything not on an allow-list.

```powershell
$ast = [System.Management.Automation.Language.Parser]::ParseFile($Path, [ref]$null, [ref]$errors)

$allowedMethods  = @('FindAll', 'FindOne', 'Add', 'Contains', 'ToString', 'Trim', 'Split', 'Dispose', 'Close')
$explicitlyDenied = @('CommitChanges', 'SetInfo', 'Put', 'PutEx', 'DeleteTree', 'MoveTo', 'Rename',
                      'SetPassword', 'Invoke', 'InvokeMember', 'GetDirectoryEntry', 'Create')

foreach ($m in $ast.FindAll({ param($n) $n -is [System.Management.Automation.Language.InvokeMemberExpressionAst] }, $true)) {
  if ($m.Member -isnot [System.Management.Automation.Language.StringConstantExpressionAst]) {
    Add-Violation $m 'dynamic method name (cannot analyse)'; continue       # $obj.$name()
  }
  $name = $m.Member.Value
  if ($explicitlyDenied -contains $name)     { Add-Violation $m "forbidden .$name()" }
  elseif ($allowedMethods -notcontains $name) { Add-Violation $m "unknown .$name() (not on allow-list)" }
}

foreach ($c in $ast.FindAll({ param($n) $n -is [System.Management.Automation.Language.CommandAst] }, $true)) {
  $raw = $c.GetCommandName()
  if (-not $raw) { Add-Violation $c 'dynamic command (& $var)'; continue }
  $norm = ($raw -split '[\\/]')[-1] -replace '\.(exe|cmd|bat|ps1)$', ''    # Module\Verb-Noun, tool.exe
  if ($localFunctions[$norm]) { continue }                                   # defined in this file: its body is scanned too
  if ($allowedCommands -notcontains $norm) { Add-Violation $c "unknown command $raw" }
}

exit ($(if ($violations.Count) { 1 } else { 0 }))
```

Deny-lists were abandoned because they are defeated by module-qualified names,
`.exe` suffixes, reflection, storing a property collection in a variable, and
`& $variable`. The auditor has nine such bypass samples that must each fail, plus the real
tool that must pass.
