# IT Incident Tracker — design notes / 設計資料

*English first, 日本語は後半にあります。*

A small-team IT operations tool built entirely on **Google Apps Script + Google Sheets**:
incident / request tickets, IT asset inventory, employee directory, team handover notes,
schedule, SLA tracking, reports, daily backups, and e-mail notifications — in one web app,
with no server to run and no database to host.

> **This repository publishes the design only.** The source code, sample data, manuals and
> anything that describes a real organisation (names, addresses, sites, directory layout)
> are intentionally not included. What is here is the architecture, the data model, the
> security model, sanitised code excerpts and the engineering lessons — the parts that
> transfer to any team that wants to build something similar.

## What it does

| Area | Features |
|---|---|
| Tickets | Five types (incident, inquiry, service request, improvement, after-hours record). Priority-based SLA tracking, cross-links to assignee, requester and affected asset |
| Masters | Assets (58 fields in 10 sections), employees, departments, sites, server racks (drag-and-drop placement), site inventory (SNMP scanner import → review → promote) |
| Team | Threaded handover notes, leave / travel schedule |
| Reports | Workload per member, cross-tabs such as site × type, most-failing devices |
| Admin | Login users (= members), notification settings, SLA targets, CSV in/out, daily backup status |
| Auth / protection | E-mail one-time codes, 30-minute idle logout, server-side session check on every call, formula-injection defence |

## Why Apps Script

- **Zero infrastructure.** The spreadsheet is the database, the script is the server, the
  HTML service is the front end. Nothing to patch, nothing to pay for, backups are a Drive
  folder.
- **The team already lives in Google Workspace.** Sign-in, mail (for one-time codes and
  notifications), Drive (for backups) and time-driven triggers come for free.
- **It is auditable by non-developers.** Every record is a visible row; every schema change
  is a visible column. When something looks wrong the operator can open the sheet.

The trade-offs — no transactions, auto-typed cells, a sandboxed iframe, a per-script URL
callable by anyone who has it — shaped most of the design decisions documented here.

## Languages and stack

| Layer | What | Why |
|---|---|---|
| Server | JavaScript on the Apps Script V8 runtime, one file | runs inside Google Workspace; no hosting |
| Client | Plain HTML / CSS / JavaScript, no framework, no bundler | must be readable and pasteable by a non-developer |
| Data | Google Sheets, 14 sheets, no formulas | visible, auditable, backed up by Drive |
| Runtime dependencies | Google Fonts only | nothing to install or update |
| Field tools | PowerShell 5.1 (SNMP inventory scanner, read-only directory export, its AST auditor) | runs on any Windows PC without installing anything |
| Developer tooling | Node.js: preview builder, headless-browser regression suites, Word manual generator | optional; never shipped |

## Documents

| File | Contents |
|---|---|
| [docs/PRINCIPLES.md](docs/PRINCIPLES.md) | Ten design principles and the incidents that produced them |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | The four-file layout, request flow, client state model, hash-based navigation inside the Apps Script iframe |
| [docs/DATA-MODEL.md](docs/DATA-MODEL.md) | The 14 sheets, their columns, which columns are keys, which are derived, and how renames cascade |
| [docs/SECURITY.md](docs/SECURITY.md) | Threat model for an "anyone with the URL can call any public function" deployment, OTP login, sessions, rate limits, trigger guards, formula injection |
| [docs/INTEGRITY.md](docs/INTEGRITY.md) | "Same thing stored twice" problems we found and how they were fixed — the most reusable part of this repo |
| [docs/CODE-EXCERPTS.md](docs/CODE-EXCERPTS.md) | Sanitised excerpts: RPC wrapper, guards, trigger check, cell helpers, locks, rate limits, hash history, cascades, derived columns, the allow-list auditor |
| [docs/LESSONS.md](docs/LESSONS.md) | Platform gotchas: Sheets type coercion, `_`-suffixed functions, `history.pushState` in the sandbox, time zones, multi-tab sessions |
| [docs/TOOLING.md](docs/TOOLING.md) | How the app is tested and documented without a dev server: a mock-injected preview build, headless-browser regression suites, generated Word manuals, a read-only AD export tool with an AST-based auditor |
| [docs/ROADMAP.md](docs/ROADMAP.md) | What was deliberately left out and what would come next |
| [LICENSING.md](LICENSING.md) | Status of this publication |

## At a glance

```
Browser (one HTML page, vanilla JS)          Apps Script project (one .gs file)
┌──────────────────────────────┐             ┌──────────────────────────────┐
│ login (e-mail OTP)           │ ──RPC──▶    │ requestOTP / verifyOTP        │
│ state object + render fns    │ ◀─JSON──    │ requireSession_ on every call │
│ hash history (#v0, #v1 …)    │             │ loaders / savers per sheet    │
│ idle timer, multi-tab aware  │             │ cascades, validators          │
└──────────────────────────────┘             │ time-driven triggers:         │
                                             │   daily backup → Drive CSV    │
                                             │   weekly XLSX by mail         │
                                             │   SLA digest (weekdays)       │
                                             └───────────────┬───────────────┘
                                                             │
                                             Google Sheets (14 sheets = tables)
```

## Status

Developed and operated since 2026 for a real IT team. Version numbering follows a
codename / build / date scheme (e.g. `Jonker · Build 12.0 · 2026-09-22`). The design here
reflects build 12.0.

---

# 日本語版

**Google Apps Script と Google スプレッドシートだけで作った、少人数IT部門向けの業務管理ツール**の設計資料です。
インシデントや依頼のチケット、IT資産台帳、社員名簿、引継ぎノート、スケジュール、SLA管理、レポート、日次バックアップ、
メール通知を1つのWebアプリにまとめています。サーバーもデータベースも用意する必要がありません。

> **このリポジトリで公開しているのは設計のみです。** ソースコード本体、サンプルデータ、マニュアル、
> および実在の組織を特定できる情報（社名、人名、拠点名、ディレクトリ構成など）は意図的に含めていません。
> ここにあるのは、アーキテクチャ、データモデル、セキュリティモデル、匿名化したコード抜粋、そして運用から得た教訓です。
> いずれも、同じようなものを作ろうとする別のチームにそのまま持ち込める部分です。

## 何ができるか

| 領域 | 機能 |
|---|---|
| チケット | インシデント／問い合わせ／サービスリクエスト／改善要望／時間外記録の5種別。優先度ごとのSLA超過監視。担当者・依頼者・対象資産との相互リンク |
| マスタ | 資産（58項目・10セクション）、社員、部署、拠点、サーバーラック（ドラッグ＆ドロップで配置）、拠点棚卸（SNMPスキャナの結果を取り込み、精査してから資産台帳へ反映） |
| チーム | スレッド形式の引継ぎノート、休暇・外出のスケジュール表 |
| レポート | 担当者別の作業負荷、拠点×種別などのクロス集計、故障の多い機器のランキング |
| 管理 | ログインユーザー（＝担当者）、通知設定、SLA基準、CSVの入出力、日次バックアップの状況 |
| 認証・保護 | メールで届くワンタイムコードによるログイン、30分無操作で自動ログアウト、すべてのAPI呼び出しをサーバー側でセッション検証、数式インジェクション対策 |

## なぜ Apps Script なのか

- **インフラが要らない。** スプレッドシートがデータベース、スクリプトがサーバー、HTMLサービスが画面。パッチを当てる対象も、支払う費用もなく、バックアップは Drive のフォルダに溜まる。
- **チームが既に Google Workspace で仕事をしている。** サインイン、メール（ワンタイムコードと通知）、Drive（バックアップ）、時間主導トリガーが最初から揃っている。
- **開発者でなくても監査できる。** すべてのレコードが見える行であり、スキーマ変更はすべて見える列である。何かおかしいと思ったら、運用者はシートを開けばよい。

その代わりに引き受けた制約（トランザクションがない、セルの値が自動的に型変換される、サンドボックスの iframe の中で動く、URLを知っていれば誰でも関数を呼べる）が、ここに記した設計判断のほとんどを形作っています。

## 言語と構成

| 層 | 実体 | 理由 |
|---|---|---|
| サーバー | Apps Script V8 ランタイム上の JavaScript、1ファイル | Google Workspace の中で動くため、ホスティングが不要 |
| クライアント | 素の HTML / CSS / JavaScript。フレームワークもバンドラーも使わない | 開発者ではない保守担当者が読めて、そのまま貼り付けられること |
| データ | Google スプレッドシート14枚。数式は使わない | 目で見えて監査でき、Drive がバックアップしてくれる |
| 実行時の依存 | Google Fonts のみ | インストールも更新も不要 |
| 現場ツール | PowerShell 5.1（SNMP棚卸スキャナ、読み取り専用のディレクトリ出力ツール、その AST 監査ツール） | どの Windows PC でも追加インストールなしで動く |
| 開発ツール | Node.js（プレビュー生成、ヘッドレスブラウザによる回帰テスト、Word マニュアル生成） | あれば便利だが、配布物には含まれない |

## 資料一覧

| ファイル | 内容 |
|---|---|
| [docs/PRINCIPLES.md](docs/PRINCIPLES.md) | 設計の原則10項目と、それぞれが生まれたきっかけ |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | 4ファイル構成、呼び出しの流れ、クライアントの状態管理、Apps Script の iframe 内でのハッシュによる画面遷移 |
| [docs/DATA-MODEL.md](docs/DATA-MODEL.md) | 14シートと列、どの列がキーでどの列が派生値か、名称変更がどう伝播するか |
| [docs/SECURITY.md](docs/SECURITY.md) | 「URLを知る人は誰でも公開関数を呼べる」という前提での脅威モデル、OTPログイン、セッション、レート制限、トリガーの自衛、数式インジェクション |
| [docs/INTEGRITY.md](docs/INTEGRITY.md) | 「同じものが二か所に保存されている」問題の発見と対処。このリポジトリで最も再利用しやすい部分 |
| [docs/CODE-EXCERPTS.md](docs/CODE-EXCERPTS.md) | 匿名化したコード抜粋：RPCラッパー、ガード、トリガー検証、セルの変換、ロック、レート制限、ハッシュ履歴、名称変更の伝播、派生列、許可リスト方式の監査ツール |
| [docs/LESSONS.md](docs/LESSONS.md) | プラットフォーム固有の落とし穴：セルの型変換、`_` 付き関数、サンドボックス内の `history.pushState`、タイムゾーン、複数タブのセッション |
| [docs/TOOLING.md](docs/TOOLING.md) | 開発サーバーなしでテストと文書化を回す方法：モックを注入したプレビュー、ヘッドレスブラウザの回帰テスト、Word マニュアルの自動生成、読み取り専用のAD出力ツールと AST 監査 |
| [docs/ROADMAP.md](docs/ROADMAP.md) | 意図的にやらなかったことと、次にやること |
| [LICENSING.md](LICENSING.md) | この公開の位置づけ |

## 全体像

```
ブラウザ（HTML 1ページ・素のJS）              Apps Script プロジェクト（.gs 1ファイル）
┌──────────────────────────────┐             ┌──────────────────────────────┐
│ ログイン（メールOTP）        │ ──RPC──▶    │ requestOTP / verifyOTP        │
│ state オブジェクト＋描画関数 │ ◀─JSON──    │ 全呼び出しで requireSession_  │
│ ハッシュ履歴（#v0, #v1 …）  │             │ シートごとの読み書き          │
│ 無操作タイマー・複数タブ対応 │             │ 伝播処理・検証                │
└──────────────────────────────┘             │ 時間主導トリガー：            │
                                             │   日次バックアップ → Drive CSV│
                                             │   週次 XLSX をメール送付      │
                                             │   SLA超過サマリー（平日）     │
                                             └───────────────┬───────────────┘
                                                             │
                                             Google スプレッドシート（14シート＝テーブル）
```

## 現状

2026年から実際のIT部門で開発・運用しています。版は「コードネーム／ビルド番号／日付」の3つで表します
（例：`Jonker · Build 12.0 · 2026-09-22`）。本資料はビルド 12.0 時点の設計を反映しています。
