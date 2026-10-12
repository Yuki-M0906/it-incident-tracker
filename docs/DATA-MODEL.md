# Data model / データモデル

*English first, 日本語は後半にあります。*

Google Sheets is the only store. One sheet per entity, header row frozen, one record per
row, no formulas. Column lists are constants in the server file and are the single source
of truth for both the header row and the object keys exchanged with the client.

## The 14 sheets

| Sheet | Cols | Purpose | Key |
|---|---|---|---|
| Tickets | 18 | Incident / inquiry / service request / improvement / after-hours record | `ID` (`yyyy-NNNN`, per-year sequence) |
| Config | 2 | Key/value: system list for tickets, global notification switches | `Category` |
| Assets | 58 | IT assets in 10 sections (basic, accounting, user, location, OS, network, maintenance, lifecycle, admin, rack) | asset id (`PREFIX-00001`) |
| Employees | 12 | Employee directory (id, name, kana, department, site, title, dates, status, manager id, extension, note) | employee id |
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
| A member's display name | `AuthUsers` display name (unique) | `CreatedBy` / `AssignedTo` on tickets, author on handover notes, member on schedule entries, "updated by" on assets, "imported by" on staging rows | rename cascade on save |
| A member as an employee | `AuthUsers` employee id → `Employees` | — | validated on save (must exist, one user per employee) |
| Ticket requester | `Tickets.RequesterID` → `Employees` | `Tickets.Requester` (free text, snapshot at creation) | display/search/mail resolve the id first and fall back to the text; text is history |
| Asset user | asset's employee id → `Employees` | asset's user name, department, site | written from the employee on asset save and import; re-written when the employee changes |
| Department / site names | `Departments` / `Sites` | columns in Employees, Assets, Racks, InventoryStaging | rename cascade on save; delete reports reference counts |
| Rack placement | asset's rack id, start U, size U → `Racks` | — | validated (rack exists, within height, no overlap) on form save, CSV import and layout save |
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

Writes that generate an id (tickets, handover notes, schedule entries, racks, staging
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

---

# 日本語版

データの置き場所は Google スプレッドシートだけです。エンティティごとに1シート、ヘッダー行は固定、1行1レコード、数式は使いません。
列の一覧はサーバー側ファイルの定数として持ち、ヘッダー行とクライアントとやり取りするオブジェクトのキーの両方について、これが唯一の正です。

## 14のシート

| シート | 列数 | 用途 | キー |
|---|---|---|---|
| Tickets | 18 | インシデント／問い合わせ／サービスリクエスト／改善要望／時間外記録 | `ID`（`yyyy-NNNN`、年ごとの連番） |
| Config | 2 | キー／値：チケットの対象システム一覧、通知の全体スイッチ | `Category` |
| Assets | 58 | IT資産。10セクション（基本、会計、利用者、設置、OS、ネットワーク、保守、ライフサイクル、管理補足、ラック） | 資産管理番号（`PREFIX-00001`） |
| Employees | 12 | 社員名簿（社員番号、氏名、カナ、部署、拠点、役職、入退社日、ステータス、上長、内線、備考） | 社員番号 |
| Departments | 1 | 部署マスタ | 名称 |
| Sites | 1 | 拠点マスタ | 名称 |
| AuthUsers | 7 | ログインユーザー＝IT担当者：メール、表示名、管理者フラグ、ステータス、備考、**担当者フラグ**、**社員番号** | `email` |
| AuthLog | 5 | 認証イベント（日時、メール、イベント、結果、備考） | 追記のみ |
| Handover | 9 | スレッド形式の引継ぎノート（返信は `ParentID` で紐付け） | `ID` |
| Schedule | 6 | 担当者×日付の予定（休暇、在宅、出張など） | `ID` |
| NotificationPrefs | 3 | ユーザーごとの通知オプトイン（アサイン通知、SLA超過サマリー） | `email` |
| Racks | 7 | サーバーラック（拠点、部屋、名前、高さU、設置日、備考） | `RackID` |
| InventoryStaging | 15 | SNMPスキャナから取り込んだ行。精査してから Assets に昇格 | `StagingID` |
| SLAConfig | 5 | 優先度ごとの初動・解決目標と説明 | `priority` |

スキーマの変更はプロジェクトのメモで版管理しています。列の追加で既存の列順を変えることはありません。ローダーは `COLS.length` 列を読み、
セットアップ関数は既存シートに足りないヘッダーを追記する（移行分岐）ので、古いシートもアップグレード後にそのまま動きます。
直近2回の列追加では、ローダー自身が最初の読み出し時にスキーマを修復するので、デプロイ後にセットアップ関数を実行し忘れても害はありません。

## キー・参照・派生列

スプレッドシートを土台にしたアプリで本当に重要な設計上の問いは、**同じ事実が2か所にあるとき、どちらの列が正か**です。現在の規則：

| 事実 | 正 | 派生コピー | 同期の方法 |
|---|---|---|---|
| 誰がIT担当者か | `AuthUsers` のうち担当者フラグが TRUE で有効な行 | なし（担当者一覧は保存せず計算する） | 不要 |
| 担当者の表示名 | `AuthUsers` の表示名（一意） | チケットの `CreatedBy` / `AssignedTo`、引継ぎノートの投稿者、スケジュールのメンバー、資産の更新者、棚卸行の取込者 | 保存時に名称変更を伝播 |
| 担当者と社員の対応 | `AuthUsers` の社員番号 → `Employees` | なし | 保存時に検証（存在すること、1社員につき1ユーザー） |
| チケットの依頼者 | `Tickets.RequesterID` → `Employees` | `Tickets.Requester`（自由入力。起票時のスナップショット） | 表示・検索・メールはまずIDを解決し、無ければテキストにフォールバック。テキストは履歴 |
| 資産の利用者 | 資産の社員番号 → `Employees` | 資産の利用者氏名、部署、拠点 | 資産の保存・取込時に社員から書き込み、社員が変わったら書き直す |
| 部署名・拠点名 | `Departments` / `Sites` | Employees、Assets、Racks、InventoryStaging の各列 | 保存時に名称変更を伝播。削除時は参照件数を提示 |
| ラックへの搭載 | 資産のラックID・起点U・占有U → `Racks` | なし | フォーム保存、CSV取込、配置保存のいずれでも検証（ラックの存在、高さの範囲内、重複なし） |
| SLA目標 | `SLAConfig` シート | コード内の既定値はフォールバックのみ | 保存時に P1〜P4 がちょうど揃っていることを要求 |

自由入力のスナップショットは意図して残しています。社員が退職した後も、チケットには誰が依頼したかが表示されるべきだからです。
表示・検索・メールに出るものはすべて、IDを優先して解決し、IDが解決できなくなったら「（削除済）」を付けるリゾルバを通ります。

## セルの読み書き

書いた値をスプレッドシートが「手で入力したもの」と同じように解釈するため、読み書きはすべて2つのヘルパーで包んでいます。

- `cellToStr_(v)` — 真偽値を `'TRUE'`/`'FALSE'` に、`Date` をプロジェクトのタイムゾーンで `yyyy-MM-dd[ HH:mm]` に、それ以外を文字列に戻す。
  これが無いと、アプリが書いた `'TRUE'` は `true` として返ってきて、`=== 'TRUE'` の比較は最初の往復以降ずっと失敗します。
- `toCell_(v)` / `sanitizeRow_(row)` — `=` や `+` で始まる文字列に `'` を前置し、数式として評価されずにテキストとして保存されるようにする。

IDを採番する書き込み（チケット、引継ぎノート、スケジュール、ラック、棚卸行）は「最大IDを読む → 追記する」をスクリプトロックで囲み、
同時に保存した2人が同じIDを取らないようにしています。

## バックアップ

- **日次 CSV**：全シートを UTF-8（BOM付き）で、日付ごとのフォルダに90日分保持。バックアップフォルダは名前で探さずIDで覚えておくので、
  ゴミ箱にある同名フォルダを誤って使うことがありません。
- **週次 XLSX**：全シートを1冊にまとめて管理者へメール。Drive の外にもコピーが残ります。
- **復旧**はランブックとして文書化：CSV からの単一シート復元、スプレッドシート全体の復元、週次 XLSX からの再構築。

## サンプルデータ

評価用に、互いに整合したサンプル（部署、拠点、社員、資産、チケット）を管理画面から投入できます。既定のマスタと同じID空間・同じ語彙を共有しているので、
デモ画面の相互リンクが実際に解決します。本番環境で投入することはありませんし、ボタンは既に存在するIDを飛ばします。
