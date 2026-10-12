# Design principles / 設計の原則

*English first, 日本語は後半にあります。*

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

---

# 日本語版

以下の原則は、先に決めてから守ったものではありません。ブラウザの「戻る」でユーザーがサインアウトされた、
通知がいつの間にか止まっていた、プルダウンに新しい拠点が出てこない――といった個別の出来事から生まれ、
その後ルールとして残したものです。この規模の社内ツールなら同じ原則がそのまま当てはまるはずなので、ここに書き出しておきます。

## 1. インフラを持たない

スプレッドシートがデータベース、スクリプトがサーバー、HTMLサービスがWebサーバー、Drive がバックアップ先、
Gmail がメール送信、時間主導トリガーがスケジューラ。インストール、ホスティング、パッチ適用、支払いのいずれも発生しない。
その代償として、プラットフォームの制約（トランザクションがない、セルが勝手に型変換される、サンドボックスの iframe、メール送信枠）を
受け入れ、それに逆らわず設計で回避する。

## 2. 保守する人は開発者ではない

配布するものはすべて、IT管理者が Apps Script のエディタに貼り付けられる形でなければならない。
つまり、素のファイル4つ、フレームワークなし、バンドラーなし、実行時の npm 依存なし。
アプリの*周辺*のツール（プレビュー生成、テスト実行、マニュアル生成）は Node や PowerShell を使ってよい。
使わなくてもアプリは壊れないからだ。コメントには「その行がなぜ存在するか」を導入した版とともに書き、
次に読む人が「まだ必要か」を判断できるようにする。

## 3. シートが監査証跡である

すべてのレコードは目に見える1行であり、スキーマ変更はすべて目に見える1列である。ダッシュボードの数字が
おかしいと思ったら、運用者はシートを開いて、コードが見ているのと同じデータを見ることができる。
だからコードは、運用者が解釈できないものを一切保存しない。直列化したデータの塊、数式、非表示シート、符号化したIDは使わない。

## 4. サーバーの検証だけを信じ、ページは信じない

チームのユーザーを入れるためのデプロイ設定は、URLを知る誰もが公開関数を呼べる設定でもある。
したがってページは敵対的なものとして扱う。すべての関数がセッショントークンを検証し、管理者用の関数は管理者フラグを検証し、
トリガー用の関数は自分がトリガーから呼ばれたことを検証し、移行用の関数は呼び出し元を検証し、
それ以外は名前の末尾に `_` を付けて外から届かない場所に置く。クライアント側の検証は使い勝手のためにしか存在しない。

## 5. 一つの事実には一つの置き場所

現実の同じものが2つのレコードに現れるときは、どちらが正かを決め、もう一方を派生コピーか明示的な履歴スナップショットにする。
派生コピーは正の側の保存処理が書き換え、スナップショットは画面上でそれと分かるように表示する。
決めないことの代償は数か月後に、気づかないうちに食い違ったレポートという形で支払うことになる。

## 6. すべてを冪等にする

セットアップは無いものだけを作り、あるものには触らない。サンプル投入は既にあるIDを飛ばす。列の追加は最初の読み出し時に自己修復する。
バックアップフォルダは名前ではなくIDで探す。繰り返しても安全な手順は、チェックリストに載せて、デプロイのたびに何も考えずに実行できる手順になる。

## 7. 人が見ている場所で、はっきり失敗する

シートが無ければ報告するのではなく作る。ログは誰も見ていないからだ。バックアップの失敗は状態シートに書き、管理画面に表示する。
そこは誰かが見ているからだ。メール送信の失敗は理由とともに認証ログに書く。
最悪なのは黙って失敗すること、その次に悪いのは誰も読まない場所に失敗を報告することだ。

## 8. まず記録し、構造は後から

ダッシュボードのクイック記録欄は、種別と件名しか受け取らない。電話が鳴っている最中に作られなかった完璧なチケットより、
2項目だけでも存在するチケットのほうがよい。依頼者、資産、優先度、対応内容は、電話が終わってから足せばよい。

## 9. ユーザーに見えるものはすべて版を持つ

コードネーム、ビルド番号、日付は3つの定数として置き、ログイン画面、システム情報ページ、各ファイルのヘッダー、マニュアルの表紙に現れる。
リリース前に grep でずれを検出する。「今どの版ですか」という質問に、助けを求めている本人が一目で答えられなければならない。

## 10. サーバーと同じ規則を課したモックでテストする

このアプリは Apps Script の外では動かないので、プレビュー用のビルドにモックサーバーを注入している。
モックには本物のサーバーと同じ検証（氏名の一意性、最後の管理者の保護、ラックの重複、派生列）を課す。
両者が食い違えばどちらかにバグがあり、サーバー側だったことも一度ではない。
