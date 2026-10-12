# Architecture / アーキテクチャ

*English first, 日本語は後半にあります。*

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
| Daily, early morning | Exports all 14 sheets to CSV into `Drive/<backup folder>/yyyy-MM-dd/`, keeps 90 days, records success/failure for the admin screen |
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

---

# 日本語版

## 4つのファイル

アプリ全体は、1つの Apps Script プロジェクト内の4ファイルで構成されています。

| ファイル | 役割 | 規模（ビルド 12.0） |
|---|---|---|
| `Code.gs` | サーバー：全シートの読み書き、認証、通知、バックアップ、移行、サンプルデータ | 約4,400行 |
| `index.html` | 全画面の構造、ヘルプページ、更新履歴 | 約2,300行 |
| `styles.html` | 全CSS（`<style>` ブロック。`index.html` から取り込む） | 約2,800行 |
| `script.html` | クライアント側の全JavaScript（`<script>` ブロック。`index.html` から取り込む） | 約5,000行 |

`index.html` は `HtmlService.createTemplateFromFile('index').evaluate()` で配信します。
テンプレートのスクリプトレット（`<?!= include('styles'); ?>`）が残り2ファイルを取り込み、
版番号の定数をログイン画面のフッターに埋め込むので、サインイン前でも版が分かります。

フレームワークなし、バンドラーなし、実行時の npm 依存なし。外部リソースは Google Fonts だけです。
これは意図的な選択です。保守するのは開発者ではなくIT管理者であり、Apps Script のエディタに貼り付けられないものは負債になります。

## 呼び出しの流れ

クライアントからサーバーへの呼び出しは、すべて1つのラッパーを通ります。

```
callServer(関数名, ...引数)
  → state.sessionToken を最後の引数として付加
  → google.script.run.withSuccessHandler / withFailureHandler
  → Promise を返す
  → 「セッションが無効」で失敗したら、そのタブをログアウトして理由を表示
```

サーバー側では、すべての公開関数が次のいずれかのガードで始まります。

| ガード | 使う関数 |
|---|---|
| `requireSession_(token)` | データを読む・書くすべての関数 |
| `requireAdmin_(token)` | ユーザー管理、通知設定、SLA設定、CSV取込、サンプルデータ投入 |
| `isFromTrigger_(e)` | 時間主導の3関数（日次バックアップ、週次メール、SLA超過サマリー） |
| `requireOwnerContext_()` | スクリプト所有者がエディタから実行する一回限りの移行 |

`requireSession_` はキャッシュ上のトークンを検証するだけでなく、呼び出しのたびに `AuthUsers` シートのその人の行を
読み直します。そのため、ユーザーの無効化や管理者権限の剥奪は、6時間のセッション期限を待たずに即座に効きます。

名前が `_` で終わる関数はブラウザからまったく呼べません（Apps Script の仕様）。API として公開する意図のないものには、すべてこの接尾辞を付けています。

## サーバー側の構成

```
Code.gs
├─ 定数:       シート名、列定義（TICKET_COLS, ASSET_COLS …）、既定値、OTPの上限
├─ ヘルパー:   ensureSheet_, cellToStr_, toCell_/sanitizeRow_, withScriptLock_, requireOwnerContext_
├─ 担当者:     loadITMembers_（AuthUsers から導出）、旧担当者の移行、名称変更の伝播
├─ 認証:       requestOTP, verifyOTP, validateSession_, requireSession_, requireAdmin_, invalidateSession
├─ ユーザー:   loadAuthUsers_（スキーマの自己修復付き）, saveAuthUser, deleteAuthUser
├─ セットアップ: setupSheet → 14シートそれぞれの setupXxxSheet_（冪等）
├─ シート別:   loadXxx_（生）/ loadXxx(token)（ガード付き）/ saveXxx / deleteXxx
├─ チケット:   appendTicket, updateTicket（スクリプトロック下で採番）, notifyAssignment_
├─ 通知:       Config の全体フラグ、ユーザー別設定、SLA超過サマリー
├─ バックアップ: dailyBackup(e) / weeklyBackupEmail(e) ＋ 手動の backupNow(token)
├─ CSV:        parseCSV_, escapeCSVField_, プレビュー／適用の取込（マージのみ）
└─ サンプル:   seedDummyXxx(token) と、移行用の run*() 入口
```

この規模でも迷わずに読めるのは、2つの約束事のおかげです。

- **`loadXxx_()` と `loadXxx(token)`。** アンダースコア付きが内部用（セッションを持たないトリガーからも使う）の生のローダー、
  公開版はそれにガードを足しただけ。それ以外の場所でシートを直接読むことはありません。
- **`ensureSheet_(name, setupFn)`。** シートへのアクセスは必ずここを通ります。シートが無ければヘッダー行付きで作るので、
  削除や改名されたシートが「シートが見つかりません」という分かりにくいエラーになることはありません。

## クライアント側の構成

```
script.html
├─ state       1つのオブジェクト：セッション、現在の画面、読み込んだ全データ、フィルタ、下書き
├─ callServer, toast, showLoading, escapeHTML, $ / $$
├─ 履歴        snapshotView_, pushViewHistory_, restoreViewHistory_, hashchange リスナー
├─ 起動        localStorage からセッション復元 → getInitData → enterApp
├─ 無操作      30分タイマー、警告モーダル、localStorage による複数タブの把握
├─ 画面        showView(名前) → renderXxx()。画面ごとに描画関数1つ
├─ フォーム    チケット（詳細＋クイック記録）、資産（58項目・折りたたみ10セクション）、社員、ラック配置（ドラッグ＆ドロップ）
├─ 管理        担当者 / ユーザー / 通知 / SLA / データ入出力 / バックアップ状況 / 管理者ガイド
└─ bindEvents  リスナーは起動時に一度だけ登録。一覧は委譲ハンドラ
```

`getInitData()` は画面に必要なすべて（チケット、資産、社員、マスタ、SLA設定、担当者一覧など）を1往復で返します。
以降の保存では影響のあるコレクションだけを取り直します。ヘッダーの「再読込」ボタンは `getInitData` を再実行して
現在の画面だけを描き直し、スクロール位置や開いているパネルを保ちます。ユーザーが怖がるようになっていたブラウザの更新とは、
意図的に違う動きにしています。

## 画面遷移とサンドボックス

Apps Script はページをサンドボックスの iframe の中で配信します。そこでは `history.pushState` が使えず、
ブラウザの「戻る」を押すとアプリから出てしまいます。ビルド 11.15 より前は、これが原因で1日に何度もユーザーが黙ってサインアウトされていました。

対策はハッシュによる履歴です。

- `showView` のたびにスナップショット（画面名、対象のID、サブタブ、フィルタ）をメモリ上の配列に積み、`location.hash = '#v<番号>'` を書く。
- `hashchange` リスナーが番号からスナップショットを引き、CSSクラスを切り替えるのではなく**対象を開き直す**（`openEdit(id)`、`openEditAsset(id)` など）。
  復元後の画面が正しいレコードと正しいプルダウンの状態を示すためです。
- 最初のエントリは `location.replace` を使い、アプリの「前」が存在しないようにする。
- ログインとログアウトで配列をリセットし、次のユーザーが前のユーザーの画面に「戻れ」ないようにする。
- ヘッダーの「← 戻る」ボタンは単に `history.back()` を呼ぶ。ブラウザの戻るとアプリ内の戻るは同じものです。

明示的な対処が必要だった端のケース：「新規起票」フォームの復元では未送信の下書きは残さなければならない一方で、
別のチケットのデータを抱えたままのフォームは**破棄**しなければなりません（そうしないと「新規 → 編集 → 戻る → 送信」で重複チケットができていました）。

## 時間主導のジョブ

| トリガー | 処理 |
|---|---|
| 毎日 早朝 | 14シートすべてを CSV にして `Drive/<バックアップフォルダ>/yyyy-MM-dd/` へ書き出し、90日保持。成否を管理画面向けに記録 |
| 毎週 月曜早朝 | 全シートを1つの XLSX にまとめ、管理者へメール |
| 毎日 早朝（平日のみ） | SLA の解決目標を過ぎたチケットを担当者ごとにまとめてメール |

トリガーは公開関数を名前で呼ぶため、この3つには `_` を付けられません。代わりに `e.triggerUid` をプロジェクトに登録済みのトリガーと照合し、
それ以外の経路で呼ばれたら何もせず戻ります。

## バージョン

リリースを定義するのは3つの値です。**コードネーム**（著名なダイヤモンドの名。メジャー版でのみ変更）、**ビルド番号**（`12.0`）、**日付**。
`Code.gs` 冒頭の3つの定数が正で、ログイン画面のフッター、システム情報ページ、プレビュー生成ツール、各ファイルのヘッダーはすべてここから導出するか、
ここと照合します。プロジェクトのメモにある grep コマンドで、リリース前にずれを検出します。
