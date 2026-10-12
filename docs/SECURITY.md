# Security model / セキュリティモデル

*English first, 日本語は後半にあります。*

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

---

# 日本語版

## デプロイ設定とその帰結

このWebアプリは **「全員（匿名を含む）がアクセス可」／「所有者として実行」** でデプロイしています。
配信元の Google Workspace にアカウントを持たないユーザーがページを開ける唯一の方式ですが、他のすべてを決める帰結が1つあります。

> URLを知っている人は誰でも、スクリプト内の**どの**グローバル関数でも、所有者の権限で、名前を指定してブラウザのコンソールから呼べる。

したがってサーバーはページを信用できません。自分で検証したものだけを信用します。

## 規則

1. **データを扱う関数はすべて最後の引数にセッショントークンを取り、検証する。** クライアントのラッパーが自動で付加し、
   サーバーは何かをする前に必ず確認します。管理者専用の関数は管理者フラグも確認します。
2. **呼ばれてはいけないものには `_` を付ける。** Apps Script は末尾 `_` の関数を `google.script.run` に公開しません。
   内部用のローダー、伝播処理、メール送信、移行、サンプル投入はすべてこの接尾辞付きです。
3. **時間主導の関数は、トリガーから起動されたことを自分で確認する。** これらは接尾辞を付けられない（トリガーが名前で呼ぶ）ので、
   イベントの `triggerUid` をプロジェクトに登録済みのトリガーと照合し、違えば何もせずに戻ります。これが無いと、
   URLを知る人がバックアップメールやSLAサマリーをループで起動して1日のメール送信枠を使い切れます。ログインコードもメールで届くので、
   全員がログインできなくなります。
4. **手動の移行は所有者だけが実行できる。** `_` 付きの関数はエディタの実行メニューにも出ないため、それぞれに公開の `run*()` 入口を用意し、
   先頭で「アクティブユーザー＝実効ユーザー」を要求します。これが成り立つのは、スクリプト所有者がエディタから実行したときだけです。
5. **クリックジャッキング対策。** `XFrameOptionsMode` は `ALLOWALL` ではなく `DEFAULT`。

## 認証

メールで届くワンタイムコード方式で、パスワードはありません。

```
メール入力 ──▶ AuthUsers に登録済みで有効か？ ──▶ 8桁コード（UUID由来。Math.random は使わない）
                                                     CacheService に保存、有効期限10分
                                                     メールで送信
コード入力 ──▶ 一致するか？ ──▶ セッショントークン（UUID）を CacheService に6時間保存
                                 クライアントは localStorage に保持
```

上限はすべてサーバー側で強制します。

| 上限 | 値 | 理由 |
|---|---|---|
| アドレスごとの発行回数 | 10分に5回 | 通常の総当たり対策 |
| 全体の発行回数 | 10分に30回。アドレスの登録確認**より前**に評価 | アドレスを変えながら連打する攻撃者にメール送信枠を枯渇させない。認証ログを照会で埋めさせない |
| 同一アドレスへの再送間隔 | 60秒 | 二重クリックとメール爆撃 |
| コードごとの誤入力回数 | 8回で失効 | 発行済みコードへの総当たり |
| カウンタのリセット | 新しいコードを発行すると失敗回数をリセット | そうしないと、第三者が他人のアドレスに失敗を積むだけで締め出せる |

送信・拒否・失敗・成功のすべての試行を、理由とともに `AuthLog` に追記します。

## セッション

- 6時間の絶対上限（キャッシュのTTL）。クライアント側では30分無操作で自動ログアウト、2分前に警告。
- `requireSession_` は呼び出しのたびに `AuthUsers` の行を読み直すので、無効化・削除・権限剥奪は次のログイン時ではなく次のクリックから効きます。
- ログアウトでサーバー側のトークンを失効させます。
- 複数タブは1つのトークンを共有します。無操作になったタブは、失効させる前に共有の「最終操作時刻」を確認し、
  他のタブが活動中なら自分だけをログイン画面に戻します。
- サーバーの応答に「セッションが無効」のメッセージが含まれていれば、どの画面からでも理由付きでログイン画面に戻ります。

## データの衛生

- 数式インジェクション：シートや CSV に書くすべての文字列について、`=`、`+`、`-`、`@`、タブ、CR で始まるものには接頭辞を付け、
  表計算ソフトにテキストとして扱わせます。
- HTMLインジェクション：画面に描画するユーザー入力はすべて1つのエスケープ関数を通します。一覧は全項目をエスケープするテンプレートから組み立てます。
- 設定シートに書く通知キーは許可リストと照合します。
- 「最後の管理者」と「最後の有効な担当者」は無効化も削除もできません。
- ログインユーザーの表示名は一意です。他のシートが保存するキーだからです。

## 意図的にやっていないこと

- レコード単位のアクセス制御はありません。ログインできる人は全員、すべてのチケット・資産・社員を読めて編集できます。
  管理者専用なのはユーザー管理、通知、SLA設定、一括取込だけです。チームは3〜4人で、もともとこの責任を共有しています。
- データ編集の監査証跡はありません（認証のみ）。同じレコードを同時に編集した場合は後勝ちです。
- リポジトリに秘密情報はありません。環境固有の設定は、スクリプトが紐付くスプレッドシートとトリガーのスケジュールだけです。
