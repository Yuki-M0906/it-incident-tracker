# IT Incident Tracker — design notes

A small-team IT operations tool built entirely on **Google Apps Script + Google Sheets**:
incident / request tickets, IT asset inventory, employee directory, team handover notes,
schedule, SLA tracking, reports, daily backups, and e-mail notifications — in one web app,
with no server to run and no database to host.

> **This repository publishes the design only.** The source code, sample data, manuals and
> anything that describes a real organisation (names, addresses, sites, directory layout)
> are intentionally not included. What is here is the architecture, the data model, the
> security model, and the engineering lessons — the parts that transfer to any team that
> wants to build something similar.

## これは何か（日本語）

3〜4名のIT部門が日々の業務を一元管理するために作った社内ツールの**設計資料**です。
電話やチャットで飛んでくる依頼をその場で10秒で起票し、資産・社員と紐付けて後から追跡でき、
月次でレポート化する、という運用を Google Workspace だけで成立させています。

| 領域 | 機能 |
|---|---|
| チケット | インシデント／問い合わせ／サービスリクエスト／改善要望／時間外記録の5種別。優先度別SLAの超過監視、担当者・依頼者・対象資産の相互リンク |
| マスタ | 資産（58項目・10セクション）、社員、部署、拠点、サーバーラック（ドラッグ＆ドロップ配置）、拠点棚卸（SNMPスキャナ取込→精査→反映） |
| チーム | 引継ぎノート（スレッド）、休暇・外出のスケジュール表 |
| レポート | 担当者別ワークロード、拠点×種別などのクロス集計、故障多発端末ランキング |
| 管理 | ログインユーザー（＝担当者）、通知設定、SLA基準、CSV入出力、日次バックアップ状況 |
| 認証・保護 | メールのワンタイムコード、30分無操作ログアウト、全API呼び出しのサーバー側セッション検証、数式インジェクション対策 |

## Why Apps Script

- **Zero infrastructure.** The spreadsheet is the database, the script is the server, the
  HTML service is the front end. Nothing to patch, nothing to pay for, backups are a Drive
  folder.
- **The team already lives in Google Workspace.** Sign-in, mail (for one-time codes and
  notifications), Drive (for backups) and time-driven triggers come for free.
- **It is auditable by non-developers.** Every record is a visible row; every schema change
  is a visible column. When something looks wrong the operator can open the sheet.

The trade-offs — no transactions, auto-typed cells, a sandboxed iframe, a per-script URL
callable by anyone who has it — shaped most of the design decisions below.

## Languages and stack

| Layer | What | Why |
|---|---|---|
| Server | JavaScript on the Apps Script V8 runtime, one file | runs inside Google Workspace; no hosting |
| Client | Plain HTML / CSS / JavaScript, no framework, no bundler | must be readable and pasteable by a non-developer |
| Data | Google Sheets, 14 sheets, no formulas | visible, auditable, backed up by Drive |
| Runtime dependencies | Google Fonts only | nothing to install or update |
| Field tools | PowerShell 5.1 (SNMP inventory scanner, read-only directory export, its AST auditor) | runs on any Windows PC without installing anything |
| Developer tooling | Node.js: preview builder, headless-browser regression suites, Word manual generator | optional; never shipped |

See [docs/PRINCIPLES.md](docs/PRINCIPLES.md) for the reasoning and
[docs/CODE-EXCERPTS.md](docs/CODE-EXCERPTS.md) for short, sanitised excerpts of the parts
that matter.

## Documents

| File | Contents |
|---|---|
| [docs/PRINCIPLES.md](docs/PRINCIPLES.md) | Ten design principles and the incidents that produced them |
| [docs/CODE-EXCERPTS.md](docs/CODE-EXCERPTS.md) | Sanitised excerpts: RPC wrapper, guards, trigger check, cell helpers, locks, rate limits, hash history, cascades, derived columns, the allow-list auditor |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | The four-file layout, request flow, client state model, hash-based navigation inside the Apps Script iframe |
| [docs/DATA-MODEL.md](docs/DATA-MODEL.md) | The 14 sheets, their columns, which columns are keys, which are derived, and how renames cascade |
| [docs/SECURITY.md](docs/SECURITY.md) | Threat model for a "anyone with the URL can call any public function" deployment, OTP login, sessions, rate limits, trigger guards, formula injection |
| [docs/INTEGRITY.md](docs/INTEGRITY.md) | "Same thing stored twice" problems we found and how they were fixed — the most reusable part of this repo |
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
