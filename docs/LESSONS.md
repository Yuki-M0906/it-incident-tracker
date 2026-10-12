# Platform lessons (Apps Script + Sheets + HtmlService) / プラットフォームの教訓

*English first, 日本語は後半にあります。*

Things that cost real time. Each one is now a rule in the project notes.

## Sheets types what you write

`setValue('TRUE')` stores a boolean. `setValue('2026-01-01')` stores a date.
`setValue('=1+1')` stores a formula. `getValues()` returns `true`, a `Date`, and `2`.
Comparisons like `=== 'TRUE'` silently fail after the first round-trip, and a date written
as text comes back shifted by the sheet's time zone.

Rule: normalise on read (`cellToStr_`) and neutralise on write (`toCell_`). Never compare a
cell against a string literal without normalising.

## `_`-suffixed functions are hidden from the editor too

Suffixing a function with `_` keeps it out of `google.script.run` — but also out of the
editor's "run" dropdown. A migration that was suffixed for safety became unrunnable. The
project notes had asserted the opposite for a while.

Rule: private body `xxx_()` plus a public `runXxx()` that checks the caller is the script
owner acting from the editor (`Session.getActiveUser() === Session.getEffectiveUser()`).

## Triggers must authenticate themselves

A time-driven function is public by necessity. Verify `e.triggerUid` against
`ScriptApp.getProjectTriggers()` and return if it does not match. Mail-sending jobs are the
critical ones: the daily mail quota is shared with the login codes.

## Mail quota is the availability budget

Because login depends on mail, anything that can send mail on request is a denial-of-
service vector. Global rate limits on code issuance are evaluated **before** checking
whether the address is registered. Test-mail buttons have cooldowns. Digest jobs skip users
with nothing to report.

## New Google services need a new consent

Adding a call to a service the script has not used before (Drive export, Gmail, Lock) means
the next deployment silently fails for triggers until the owner re-authorises. The release
checklist runs `setupSheet()` from the editor after every deployment specifically to surface
the consent dialog.

## The sandbox iframe and navigation

`history.pushState` is not available. Use `location.hash` and a `hashchange` listener;
use `location.replace` for the first entry; ignore hash changes you caused yourself
(counter); reset the history on login and logout. Restore by re-opening entities, not by
toggling classes.

## Time zones are three different things

The script project's time zone, the spreadsheet's time zone and the user's browser all
differ in principle. Stamp every timestamp on the server with `Utilities.formatDate(…,
TIMEZONE, …)`; parse stored strings with `Utilities.parseDate` in the same zone; derive
weekday-only logic from the formatted date, not from `Date.getDay()`. Do not let the client
fill timestamps.

## One token, many tabs

The session token lives in `localStorage`, so every tab shares it. An idle-logout in one
tab that invalidates the server token signs out the tab the user is working in. Share a
"last activity" timestamp and only invalidate when no tab has been active.

## A `getValues()` row is a reference

In a loop that both reads and writes rows, keep the in-memory array updated after each
`appendRow`/`setValues`, or a duplicate id later in the same import is appended instead of
merged. Likewise, when a mock server mutates the same object the client holds, "before"
and "after" compare equal — snapshot primitives before the call.

## Allow-lists need coverage tests

A key allow-list that is missing one legitimate key fails *silently* for exactly the
feature that key controls. Every allow-list gets a test that walks the UI's actual keys.

## Idempotent setup, self-healing schema

`setupSheet()` creates what is missing and never touches what exists, so it can be run
after every deployment. Column additions also repair themselves on first read, because
"run the setup function" is the step most often forgotten.

## Build the preview without a server

The four files cannot be opened locally — `google.script.run` does not exist outside Apps
Script. A small script concatenates them into one HTML file and injects a mock
`google.script.run` backed by in-memory sample data. That file is what headless-browser
tests and manual screenshots run against. Keeping the mock honest (same validation rules
as the server) is part of every change; several real bugs were found because the mock
disagreed with the server.

---

# 日本語版

実際に時間を失った事柄です。いずれも今はプロジェクトのメモに規則として書いてあります。

## スプレッドシートは書いた値を型変換する

`setValue('TRUE')` は真偽値として、`setValue('2026-01-01')` は日付として、`setValue('=1+1')` は数式として保存されます。
`getValues()` が返すのは `true`、`Date`、`2` です。`=== 'TRUE'` のような比較は最初の往復以降ずっと静かに失敗し、
テキストとして書いた日付はシートのタイムゾーン分ずれて返ってきます。

規則：読むときに正規化し（`cellToStr_`）、書くときに無害化する（`toCell_`）。正規化せずにセルと文字列リテラルを比較しない。

## `_` 付きの関数はエディタからも隠れる

関数名の末尾に `_` を付けると `google.script.run` から呼べなくなりますが、エディタの「実行」プルダウンからも消えます。
安全のために `_` を付けた移行関数が、実行できなくなっていました。プロジェクトのメモには、しばらくの間その逆が書いてありました。

規則：本体は非公開の `xxx_()`、入口は公開の `runXxx()` とし、入口で「呼び出し元がエディタから実行しているスクリプト所有者であること」
（`Session.getActiveUser() === Session.getEffectiveUser()`）を確認する。

## トリガーは自分で身元を証明する

時間主導の関数はどうしても公開になります。`e.triggerUid` を `ScriptApp.getProjectTriggers()` と照合し、一致しなければ戻ること。
特に重要なのはメールを送るジョブです。1日のメール送信枠はログインコードと共有だからです。

## メール送信枠は可用性の予算

ログインがメールに依存するため、要求に応じてメールを送れるものはすべてサービス妨害の経路になります。
コード発行の全体レート制限は、アドレスが登録済みかを確認する**前**に評価します。テスト送信ボタンにはクールダウンを付け、
サマリーのジョブは報告することのないユーザーを飛ばします。

## 新しい Google サービスには新しい承認が要る

スクリプトがまだ使ったことのないサービス（Drive への書き出し、Gmail、Lock）への呼び出しを追加すると、
所有者が再承認するまで、トリガーからの実行は次のデプロイ後に黙って失敗します。リリースのチェックリストでは、
承認ダイアログを表面化させるためだけに、デプロイのたびにエディタから `setupSheet()` を実行します。

## サンドボックスの iframe と画面遷移

`history.pushState` は使えません。`location.hash` と `hashchange` リスナーを使い、最初のエントリには `location.replace` を使い、
自分で起こしたハッシュ変更はカウンタで無視し、ログインとログアウトで履歴をリセットします。復元はクラスの切り替えではなく、対象を開き直すことで行います。

## タイムゾーンは3つある

スクリプトプロジェクトのタイムゾーン、スプレッドシートのタイムゾーン、利用者のブラウザは、原理的に全部違います。
すべてのタイムスタンプはサーバー側で `Utilities.formatDate(…, TIMEZONE, …)` を使って押し、保存した文字列は同じゾーンで
`Utilities.parseDate` を使って解釈し、平日判定は `Date.getDay()` ではなく整形済みの日付から導きます。クライアントにタイムスタンプを埋めさせないこと。

## トークンは1つ、タブは複数

セッショントークンは `localStorage` にあるので、すべてのタブで共有されます。あるタブの無操作ログアウトがサーバー側のトークンを失効させると、
利用者が作業中のタブまでサインアウトされます。「最終操作時刻」を共有し、どのタブも活動していないときだけ失効させます。

## `getValues()` の行は参照である

行を読みながら書くループでは、`appendRow`／`setValues` のたびにメモリ上の配列も更新しないと、
同じ取込の後ろに出てくる重複IDがマージされずに追記されます。同様に、モックサーバーがクライアントの持つオブジェクトをそのまま書き換えると、
「変更前」と「変更後」が等しくなってしまいます。呼び出し前にプリミティブ値を退避しておくこと。

## 許可リストには網羅テストが要る

正当なキーが1つ欠けた許可リストは、そのキーが制御する機能だけを*静かに*壊します。許可リストにはすべて、画面が実際に使うキーをなめるテストを付けます。

## 冪等なセットアップ、自己修復するスキーマ

`setupSheet()` は無いものだけを作り、あるものには触らないので、デプロイのたびに実行できます。
列の追加も最初の読み出し時に自己修復します。「セットアップ関数を実行する」は最も忘れやすい手順だからです。

## サーバーなしでプレビューを作る

4つのファイルはローカルでは開けません。`google.script.run` は Apps Script の外には存在しないからです。
小さなスクリプトで4ファイルを1つのHTMLに結合し、メモリ上のサンプルデータを返すモックの `google.script.run` を注入します。
ヘッドレスブラウザのテストとマニュアルのスクリーンショットは、このファイルに対して動きます。
モックを正直に保つこと（サーバーと同じ検証規則）はすべての変更の一部であり、モックとサーバーが食い違ったことで見つかった本物のバグがいくつもあります。
