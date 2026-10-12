# Roadmap and deliberate omissions / ロードマップと意図的な省略

*English first, 日本語は後半にあります。*

## Done, in order of appearance

| Version | Theme |
|---|---|
| v2–v5 | tickets → assets → employees → reports |
| v6 | visual redesign |
| v7 | CSV in/out, in-app help, member management |
| v8 | e-mail one-time-code login, login-user management, auth log |
| v9 | department and site masters, admin tab, admin-only enforcement, session/admin guard split |
| v10 | idle logout, admin guide, sample-data buttons |
| v11.0–11.8 | team tab (handover threads, schedule), daily backup + weekly mail, assignment and SLA notifications, cross-entity links, rack layout, site inventory via SNMP, SLA config sheet |
| v11.9–11.18 | codename versioning, systematic security hardening (session checks, trigger guards, owner-gated migrations, rate limits, formula injection), in-app back navigation, pre-production audit |
| v11.19 | members unified with login users |
| v12.0 | integrity audit: asset users and login users linked to the employee directory, one vocabulary, consistent counts |

## Open

- **Asset site columns**: decide whether "belongs to" and "located at" are one concept or
  two, then remove or rename.
- **Delete actions**: either expose asset/employee deletion in the UI with reference
  counts, or close the server functions. Today deletion happens in the sheet directly,
  without the orphan check the code already implements.
- **Directory sync and "on leave"**: the export tool currently marks every enabled account
  as active, overwriting a manually set "on leave" status on each import.
- **Response-time SLA**: targets are configured and displayed but only resolution time is
  monitored. Either monitor both or stop showing the first.
- **Quick capture requester**: free text only; add the directory suggester used by the
  full form.

## Deliberately not planned

- **Remote monitoring of sites** (an earlier "v12" idea). Management protocols do not
  traverse the site firewalls, and opening them is not within the team's remit. Replaced by
  the on-demand inventory scanner run on site.
- **Per-record permissions, approval workflows, SLA pause/clock rules.** The team is three
  or four people; the cost of modelling these exceeds their value at this size.
- **A framework or a build step for the app itself.** The maintainer must be able to read
  and paste the source. The tooling around the app may use Node; the app may not.

---

# 日本語版

## 実装済み（登場順）

| 版 | テーマ |
|---|---|
| v2〜v5 | チケット → 資産 → 社員 → レポート |
| v6 | 画面デザインの刷新 |
| v7 | CSV入出力、アプリ内ヘルプ、担当者管理 |
| v8 | メールのワンタイムコードによるログイン、ログインユーザー管理、認証ログ |
| v9 | 部署・拠点マスタ、管理タブ、管理者限定操作の強制、セッション／管理者ガードの分離 |
| v10 | 無操作ログアウト、管理者ガイド、サンプルデータ投入ボタン |
| v11.0〜11.8 | チームタブ（引継ぎノートのスレッド、スケジュール）、日次バックアップ＋週次メール、アサイン通知とSLA通知、相互リンク、ラック配置、SNMPによる拠点棚卸、SLA設定シート |
| v11.9〜11.18 | コードネームによる版管理、体系的なセキュリティ強化（セッション検証、トリガーの自衛、所有者限定の移行、レート制限、数式インジェクション対策）、アプリ内の戻る、本番前監査 |
| v11.19 | 担当者とログインユーザーの統合 |
| v12.0 | 整合性監査：資産の利用者とログインユーザーを社員名簿に紐付け、語彙の統一、数字の一致 |

## 未着手

- **資産の拠点列**：「所属拠点」と「設置場所」が1つの概念か2つかを決め、片方を削除するか改名する。
- **削除操作**：資産・社員の削除を参照件数の表示付きで画面に出すか、サーバー関数を閉じるか。現状はシート上で直接削除されており、
  コードに実装済みの孤立参照チェックが働いていない。
- **名簿同期と「休職」**：出力ツールは有効なアカウントをすべて在籍扱いにするため、手で設定した「休職」が取込のたびに上書きされる。
- **初動目標のSLA**：目標値は設定・表示されるが、監視しているのは解決時間だけ。両方を監視するか、初動の表示をやめるか。
- **クイック記録の依頼者**：自由入力のみ。詳細フォームと同じ名簿サジェストを付ける。

## 意図的に計画しないこと

- **拠点の遠隔監視**（以前の「v12」構想）。管理用プロトコルは拠点のファイアウォールを通らず、それを開けるのはチームの権限外。
  現地で実行するオンデマンドの棚卸スキャナに置き換えた。
- **レコード単位の権限、承認ワークフロー、SLAの一時停止やクロック規則。** チームは3〜4人で、この規模ではモデル化の費用が価値を上回る。
- **アプリ本体のフレームワークやビルド工程。** 保守担当者がソースを読めて貼り付けられなければならない。周辺ツールは Node を使ってよいが、アプリは使わない。
