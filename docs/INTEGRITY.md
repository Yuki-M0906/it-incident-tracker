# Integrity: when one thing is stored as two / 整合性：一つのものが二か所にあるとき

*English first, 日本語は後半にあります。*

The most valuable lesson from two years of running this app is not about Apps Script. It
is that **the concepts users have in their heads and the records the system keeps drift
apart**, quietly, whenever the same real-world thing is represented twice. This document
lists every instance found in a deliberate audit (builds 11.19 and 12.0) and what was done.
The pattern is general; the fixes are examples.

## The pattern

Two records that "obviously" describe the same thing, linked only by a string that both
happen to contain. Editing one does not touch the other. Nothing breaks on the day of the
edit. Weeks later a report is wrong, a notification is not sent, or a dropdown is empty,
and the cause is three steps removed.

Three questions catch it:

1. For every entity, where else is its name or id written?
2. When this entity is renamed, deactivated or deleted, which of those places change?
3. If none of them do, is the copy a *snapshot* (history, fine) or a *reference* (must
   follow)?

## Cases found

### IT members vs. login users — HIGH, fixed in 11.19

Two lists: a "member" list in the config sheet (used for the assignee dropdown) and the
login-user sheet (used for authentication). Linked by display-name equality. Add a login
user and forget the member list → the person can log in but cannot be assigned anything
and "my tickets" is always empty. Rename on one side → that person's notifications stop,
because the mail address is looked up by name.

**Fix:** one list. A login user *is* a member when a member flag is on. Renaming cascades
to every sheet that stores the name. Names are unique. The old list is migrated once; names
that do not match any user are shown as "unlinked legacy members" for the administrator to
link or delete.

### Login users vs. the employee directory — HIGH, fixed in 12.0

The same four people exist as login users ("Smith") and as employees ("Jane Smith,
E0001"). No link. Reports show the member and the employee as different people when an IT
member is also a requester; the directory marks someone as left while their login stays
active.

**Fix:** an optional employee-id column on the login user, validated to exist and to be
unique. It enables a "me" button on the ticket form and a warning badge when the linked
employee has left.

### Asset user vs. the employee directory — HIGH, fixed in 12.0

Assets stored the user's name, id, department and site as free text copied at save time.
Nothing refreshed them. With directory sync from Active Directory the directory moved
forward and every asset stayed where it was; assets stayed "in use" by people who had left.

**Fix:** the employee id is the truth. Name, department and site are derived on save and
on import (an unknown id is an error); saving an employee rewrites the derived columns on
that employee's assets. Display prefers the directory's current name.

### Ticket requester — MEDIUM, partly fixed

Requester is stored as free text (snapshot) and, when picked from the directory, as an
id. Lists resolved the id; search, mail bodies and CSV used the text, so a renamed employee
could not be found by their new name.

**Fix:** search and mail resolve the id too. The text remains as the snapshot it was
always meant to be. Quick-capture tickets (no id) remain a known gap.

### Two site columns on one asset — MEDIUM, open

"Belongs to site" and "site name" exist side by side; only one is used by every screen.
Whether they mean different things (user's site vs. physical location) is a business
question, so the fix waits for the answer.

### Rename cascades with gaps — LOW, fixed

Site rename reached employees and assets but not racks or staging rows; member rename
reached tickets but not the asset "updated by" column. Each was a one-line addition once
the inventory above existed.

### Vocabulary defined in several places — MEDIUM, fixed

Seed lists for departments and sites existed in three variants (setup defaults, demo data,
an external sync tool), so a standard "set up, then load demo data" produced a mixed
master. Dropdowns on the asset form were built only from values already present in assets,
so a newly added site could not be chosen. The sample asset set used a vocabulary that
matched nothing else.

**Fix:** one constant for each list, referenced everywhere; dropdowns merge the master
sheets; sample data regenerated from one consistent source.

### Numbers that should agree but did not — LOW, fixed

- "After-hours this month" counted a flag; its "see all" link filtered by ticket type;
  the manual said both. Now one predicate, used by the card, the link and the form.
- The dashboard "overdue" tile excluded on-hold tickets; the list tab and the digest
  included them.
- Closed-at was stamped with browser time, created-at with server time.
- The login-user count on the system page was 0 for non-admins because the list was only
  fetched on the admin screen.
- A "recommended trigger time" appeared with two different values on one screen.

### A regression hidden by a safety feature — HIGH, fixed

A key allow-list added to stop arbitrary writes to the config sheet omitted the global
notification switch. For two builds the switch could not be turned on from the UI, so no
notification mail went out. Caught only by a line-by-line consistency audit. Lesson:
allow-lists need a test that exercises every legitimate key.

## How the audit was run

Three passes over the whole code base, each with a single question:

1. *Vocabulary*: every enumerated value (statuses, types, priorities, device kinds,
   departments, sites) — where is it defined, where is it compared, do the sets match?
2. *References*: every id or name that appears in more than one sheet — which saves and
   deletes touch each copy?
3. *Semantics*: every number shown in two places, every rule described in help text or
   the manual — is it computed from the same inputs by the same predicate?

Each finding was listed with the consequence a user would see, then approved or deferred
one by one before anything was changed.

---

# 日本語版

このアプリを2年間運用して得た最も価値のある教訓は、Apps Script の話ではありません。
**利用者が頭の中で持っている概念と、システムが保持しているレコードは、同じ現実のものを二重に表現した瞬間から、
静かにずれていく**ということです。この文書は、意図的に行った監査（ビルド 11.19 と 12.0）で見つかった事例と対処を
すべて列挙したものです。パターンは一般的で、対処はその一例です。

## パターン

「どう見ても同じもの」を表す2つのレコードが、たまたま両方に含まれている文字列だけで結びついている。
片方を編集してももう片方には触れない。編集した当日には何も壊れない。数週間後にレポートが間違い、
通知が届かず、プルダウンが空になり、原因は3段階離れたところにある。

見つけるための問いは3つです。

1. 各エンティティについて、その名前やIDは他のどこに書かれているか。
2. そのエンティティを改名・無効化・削除したとき、そのうちのどこが変わるか。
3. どこも変わらないなら、そのコピーは*スナップショット*（履歴として正しい）か、*参照*（追随すべき）か。

## 見つかった事例

### IT担当者とログインユーザー — 重大、11.19 で修正

リストが2つありました。設定シート上の「担当者」一覧（担当プルダウン用）と、ログインユーザーのシート（認証用）です。
結びつけているのは表示名の一致だけ。ログインユーザーを追加して担当者一覧を忘れると、その人はログインできるのに担当に割り当てられず、
「自分の担当」は常に0件。片方だけ改名すると、メールアドレスを名前で引いているため、その人への通知が止まる。

**対処：** リストを1つにしました。ログインユーザーは担当者フラグがONなら*担当者である*、という定義です。
改名は名前を保存しているすべてのシートに伝播し、名前は一意にしました。旧リストは一度だけ移行し、
どのユーザーにも一致しなかった名前は「未紐付けの旧担当者」として管理者に提示し、紐付けか削除をしてもらいます。

### ログインユーザーと社員名簿 — 重大、12.0 で修正

同じ4人が、ログインユーザー（「山田」）としても社員（「山田 花子、E0001」）としても存在するのに、リンクがありませんでした。
IT担当者が依頼者にもなる案件で、レポートは担当者と社員を別人として扱い、名簿では退職扱いなのにログインは有効のまま、ということが起きます。

**対処：** ログインユーザーに任意の社員番号列を追加し、存在することと一意であることを検証します。
これにより起票フォームの「自分」ボタンと、紐付いた社員が退職済みのときの警告バッジが可能になりました。

### 資産の利用者と社員名簿 — 重大、12.0 で修正

資産は利用者の氏名・社員番号・部署・拠点を、保存時にコピーした自由入力として持っていました。何も更新しません。
Active Directory からの名簿同期を入れると名簿だけが進み、資産は置き去りになります。退職した人が「使用中」の資産がそのまま残りました。

**対処：** 社員番号を正とします。氏名・部署・拠点は保存時と取込時に派生させ（未知のIDはエラー）、
社員を保存すると、その社員の資産の派生列を書き直します。表示は名簿の現在の氏名を優先します。

### チケットの依頼者 — 中、一部修正

依頼者は自由入力（スナップショット）として保存され、名簿から選んだ場合はIDも保存されます。一覧はIDを解決していましたが、
検索・メール本文・CSVはテキストを使っていたので、改名した社員を新しい名前で検索できませんでした。

**対処：** 検索とメールもIDを解決するようにしました。テキストは本来の意図どおりスナップショットとして残します。
クイック記録のチケット（IDなし）は既知の未対応事項です。

### 1つの資産に拠点列が2つ — 中、未対応

「所属拠点」と「拠点名」が並んでいて、すべての画面が片方だけを使っています。意味が違うのか（利用者の拠点と物理的な設置場所）は
業務上の判断なので、答えが出るまで修正を待っています。

### 伝播の漏れ — 軽微、修正

拠点の改名は社員と資産には届いていましたが、ラックと棚卸行には届いていませんでした。担当者の改名はチケットには届いていましたが、
資産の「更新者」列には届いていませんでした。上の棚卸表ができてしまえば、どれも1行の追加で済みました。

### 語彙が複数の場所で定義されている — 中、修正

部署と拠点の初期リストが3通り（セットアップの既定値、デモデータ、外部の同期ツール）あったため、
「セットアップしてからデモデータを入れる」という標準手順で混ざったマスタができていました。資産フォームのプルダウンは
既存の資産にある値からしか作られておらず、新しく追加した拠点を選べませんでした。サンプル資産の語彙は他の何とも一致していませんでした。

**対処：** リストごとに定数を1つにしてすべてがそれを参照するようにし、プルダウンはマスタシートをマージし、
サンプルデータは整合した1つの元から再生成しました。

### 一致すべき数字が一致していなかった — 軽微、修正

- 「今月の時間外」はフラグで数え、その「一覧へ」はチケット種別で絞り、マニュアルは両方だと書いていた。
  今は1つの述語をカード・リンク・フォームが共有しています。
- ダッシュボードの「期限超過」は保留中を除外し、一覧のタブとサマリーは含めていた。
- クローズ日時はブラウザ時刻、起票日時はサーバー時刻で押していた。
- システム情報の「ログインユーザー数」は、一覧を管理画面でしか取得しないため非管理者には0と出ていた。
- 「推奨トリガー時刻」が同じ画面に2つの値で出ていた。

### 安全機能に隠れていた退行 — 重大、修正

設定シートへの任意の書き込みを防ぐために追加したキーの許可リストに、通知の全体スイッチが入っていませんでした。
2つのビルドの間、画面からスイッチをONにできず、通知メールが一通も出ていませんでした。見つけたのは、行ごとの整合性監査だけです。
教訓：許可リストには、正当なキーをすべて通すテストが必要です。

## 監査の進め方

コード全体を3回通し、それぞれ1つの問いだけを立てました。

1. *語彙*：列挙値（状態、種別、優先度、機器種別、部署、拠点）はどこで定義され、どこで比較され、集合は一致しているか。
2. *参照*：複数のシートに現れるIDや名前について、どの保存・削除がどのコピーに触れるか。
3. *意味*：2か所に表示される数字、ヘルプやマニュアルに書かれた規則は、同じ入力から同じ述語で計算されているか。

見つかったものは、利用者に見える影響とともに一覧にし、何かを変える前に1件ずつ承認または保留の判断を仰ぎました。
