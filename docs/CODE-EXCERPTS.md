# Code excerpts / コード抜粋

*Each section has an English explanation, a Japanese one, then the shared code. / 各節は英語の説明、日本語の説明、共通のコードの順です。*

Short, rewritten excerpts that show the shape of the important parts. They are not copies
of the production files: names of organisations, people, sites and identifier prefixes
have been replaced, comments rewritten, and surrounding code removed. Each is complete
enough to reuse.

Language throughout: JavaScript on the Apps Script V8 runtime (server) and in the browser
(client); PowerShell 5.1 for the field tools.

重要な部分の「形」を示すための、短く書き直した抜粋です。本番ファイルのコピーではありません。組織・人・拠点の名前や識別子の接頭辞は置き換え、
コメントは書き直し、周辺のコードは省いています。それぞれ、そのまま再利用できる程度には完結しています。

言語はサーバー・クライアントとも JavaScript（Apps Script V8 ランタイムとブラウザ）、現場ツールは PowerShell 5.1 です。

---

## 1. One RPC wrapper on the client / クライアント側の唯一のRPCラッパー

Every call to the server goes through this. It appends the session token, turns the
callback API into a Promise, and recognises an expired session anywhere in the app.

サーバーへの呼び出しはすべてここを通ります。セッショントークンを付加し、コールバック形式のAPIを Promise に変換し、
アプリのどこで起きたセッション切れも検知します。

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

例えば保存は `await callServer("saveAsset", asset, "update")` と書きます。呼び出し側にトークンは一切現れないので、付け忘れが起きません。

---

## 2. Guards on the server / サーバー側のガード

Every public function starts with one of these. `requireSession_` re-reads the user's row
so that disabling someone takes effect on their next click.

公開関数はすべて、これらのいずれかで始まります。`requireSession_` はユーザーの行を読み直すので、無効化は次のクリックから効きます。

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

## 3. Trigger functions prove they were triggered / トリガー関数は自分がトリガーから呼ばれたことを証明する

Time-driven functions cannot carry the `_` suffix (triggers call them by name), so they
check the event object against the project's own triggers.

時間主導の関数には `_` を付けられない（トリガーが名前で呼ぶ）ので、イベントオブジェクトをプロジェクト自身のトリガーと照合します。

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

## 4. Owner-gated entry points for migrations / 所有者だけが通れる移行の入口

A `_`-suffixed function is hidden from the browser **and** from the editor's run menu.
Migrations therefore get a public entry that only works for the script owner running it
from the editor.

`_` 付きの関数はブラウザからもエディタの実行メニューからも見えません。そこで移行処理には、
スクリプト所有者がエディタから実行したときだけ動く公開の入口を用意します。

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

Webアプリ経由で呼ばれた場合、アクティブユーザーは匿名か別人で、実効ユーザーは所有者になるため、この検査は失敗します。

---

## 5. Reading and writing cells safely / セルを安全に読み書きする

Sheets interprets written values like typed input. These two helpers wrap every read and
write.

スプレッドシートは書き込んだ値を手入力と同じように解釈します。この2つのヘルパーで、すべての読み書きを包みます。

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

例えば `'TRUE'` と書いた「クローズ」フラグは `true` として返ってきます。`cellToStr_` が無ければ、`obj.closed === 'TRUE'` は最初の保存以降ずっと偽です。

---

## 6. Sheets that create themselves / 自分で自分を作るシート

Every sheet access goes through this helper; a missing sheet is created with its header row, so "sheet not found" never surfaces.

シートへのアクセスは必ずこの関数を通ります。無ければヘッダー付きで作るので、「シートが見つかりません」というエラーは出ません。

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

列の追加も同じ考え方です。ローダーがヘッダー行を確認し、足りないヘッダーを足してから読むので、古いビルドのシートもそのまま動きます。

---

## 7. Id generation under a lock / ロック下での採番

"Read the max id, add one, append" runs inside a script lock so two simultaneous saves cannot receive the same id.

「最大IDを読んで+1して追記する」をスクリプトロックで囲み、同時保存でIDが重複しないようにします。

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

## 8. Rate limits in the right order / 正しい順序のレート制限

The global limit is evaluated **before** the address is looked up, so an attacker cycling
through addresses cannot drain the mail quota or bloat the auth log.

全体の上限はアドレスを照会する前に評価します。アドレスを変えながら連打する攻撃者に、メール送信枠を枯渇させたり認証ログを膨らませたりさせないためです。

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

## 9. In-app history inside the sandbox / サンドボックス内のアプリ内履歴

`history.pushState` is unavailable in the Apps Script iframe. The hash is the history.

Apps Script の iframe では `history.pushState` が使えません。ハッシュそのものを履歴として使います。

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

## 10. Rename cascades / 名称変更の伝播

When a member's display name changes, every sheet that stores the name is updated in one
pass per column.

担当者の表示名が変わると、その名前を保存しているすべてのシートを、列ごとに1回の走査で更新します。

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

拠点や部署の改名、社員の新しい氏名・部署・拠点をその人が使う資産へ反映する処理も、同じ形です。

---

## 11. Derived columns on save / 保存時の派生列

The employee id is the truth for an asset's user; name, department and site are copied from the directory.

資産の利用者は社員番号が正で、氏名・部署・拠点は名簿から複写します。

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

フォームの保存と CSV 取込の両方から呼ぶので、どちらの経路でも結果は同じです。表示側は名簿の現在の氏名を優先し、IDが解決できなくなったら「（削除済）」を付けます。

---

## 12. An allow-list auditor for a read-only script (PowerShell) / 読み取り専用スクリプトの許可リスト監査

The directory export tool must never write to Active Directory. Instead of reviewing it,
a separate script parses it and rejects anything not on an allow-list.

名簿出力ツールは Active Directory に決して書き込んではいけません。目で確認する代わりに、別のスクリプトが構文解析し、許可リストに無いものをすべて拒否します。

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

拒否リストは捨てました。モジュール修飾名、`.exe` の接尾辞、リフレクション、プロパティ集合の変数への退避、`& $variable` ですり抜けられるからです。
監査ツールには、必ず不合格にならなければならない9つのすり抜けサンプルと、必ず合格しなければならない本物のツールが付いています。
