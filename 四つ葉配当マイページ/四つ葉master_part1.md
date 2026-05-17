# 四つ葉倶楽部 会員制サイト データ構造設計書

**統合版 v1.0**
最終更新: 2026-05-17

---

## 目次

1. [プロジェクト概要](#1-プロジェクト概要)
2. [技術スタック](#2-技術スタック)
3. [MVP 機能と画面構成](#3-mvp-機能と画面構成)
4. [データモデル概観](#4-データモデル概観)
5. [テーブル一覧と ENUM 定義](#5-テーブル一覧と-enum-定義)
6. [テーブル DDL（完全版）](#6-テーブル-ddl完全版)
7. [整合性トリガ](#7-整合性トリガ)
8. [RLS ポリシー](#8-rls-ポリシー)
9. [RPC 関数（業務ロジック）](#9-rpc-関数業務ロジック)
10. [Supabase Storage](#10-supabase-storage)
11. [認証・認可・管理者招待](#11-認証認可管理者招待)
12. [PWA とプッシュ通知](#12-pwa-とプッシュ通知)
13. [Next.js ディレクトリ構成](#13-nextjs-ディレクトリ構成)
14. [ページ別実装ガイド](#14-ページ別実装ガイド)
15. [運用フロー](#15-運用フロー)
16. [テスト観点](#16-テスト観点)
17. [開発フェーズ](#17-開発フェーズ)
18. [未決事項と将来拡張](#18-未決事項と将来拡張)

---

## 1. プロジェクト概要

### 1.1 目的

四つ葉倶楽部の会員専用 Web サイト・PWA を構築する。会員ごとの権利状況・配当金・四つ葉ポイントを一元管理する「会員ダッシュボード」として機能する。

### 1.2 3 つの中核機能

仕様で「最重要方針」と指定されたもの。後付けではなく最初から組み込む。

1. **権利の完了通知書の確認** — PDF を private bucket で配信、本人のみ閲覧可能
2. **権利取得確認** — 会員の権利状況の表示と管理
3. **四つ葉ポイント** — 履歴ベースで管理、改ざん防止のため RPC 経由のみで操作

### 1.3 追加された主要機能（設計議論で決定）

- **複数案件配当金・合計振込確認** — 会員 1 名が複数案件に紐づき、配当金と振込履歴を確認可能
- **配当率（share）の柔軟な管理** — rate / unit / amount の 3 タイプ対応、履歴管理
- **月次運用** — 毎月の配当原資入力 → 計算 → 振込のワークフロー
- **PWA + プッシュ通知** — 会員のスマホでアプリのように使える
- **管理者招待制** — 自己登録不可、既存 admin の招待 + 2FA 必須

### 1.4 設計の核となる原則

| # | 原則 | 理由 |
|---|---|---|
| 1 | RLS で全データを保護 | クライアントの表示制御だけに依存しない |
| 2 | 金額は integer（円） | 小数を排除、丸め誤差を防ぐ |
| 3 | 過去データは絶対に書き換えない | 監査追跡可能性、訂正は調整明細で |
| 4 | クライアント計算を信用しない | 合計金額は DB トリガで自動再計算 |
| 5 | 機密ファイルは Storage の private bucket | DB には file_path のみ、signed URL で配信 |
| 6 | 金額操作は RPC のみ経由 | 直接 UPDATE を許さない、整合性を保証 |
| 7 | 段階的拡張を見越した設計 | MVP の制約と将来拡張のバランス |

---

## 2. 技術スタック

| カテゴリ | 採用技術 |
|---|---|
| フレームワーク | Next.js 15（App Router） |
| 言語 | TypeScript |
| データベース | Supabase PostgreSQL |
| 認証 | Supabase Auth（MFA / TOTP 対応） |
| ストレージ | Supabase Storage |
| Edge Functions | Supabase Edge Functions（Deno） |
| 定期実行 | pg_cron |
| スタイリング | Tailwind CSS |
| UI コンポーネント | shadcn/ui |
| ホスティング | Vercel |
| PWA | next-pwa |
| 開発支援 | Claude Code |

### 2.1 環境変数

```env
# クライアント公開可
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
NEXT_PUBLIC_VAPID_PUBLIC_KEY=...      # Web Push 用公開鍵

# サーバ専用（NEXT_PUBLIC_ を絶対に付けない）
SUPABASE_SERVICE_ROLE_KEY=...
VAPID_PRIVATE_KEY=...
VAPID_SUBJECT=mailto:admin@yotsuba-club.com
```

---

## 3. MVP 機能と画面構成

### 3.1 機能一覧

**認証・会員管理**
- ログイン / ログアウト
- 会員登録申請
- 管理者による会員承認

**会員機能**
- 会員ダッシュボード（マイページ TOP、5 つのカード）
- お知らせ一覧 / 詳細
- イベント一覧 / 詳細 / 申込
- 会員限定資料一覧
- マイページ
  - 会員情報
  - 権利取得確認
  - 権利の完了通知書確認（PDF ダウンロード可）
  - 四つ葉ポイント確認
  - 配当金・振込確認（PDF ダウンロード可）
  - 通知設定

**管理機能**
- 管理ダッシュボード（TOP）
- 会員管理
- お知らせ管理
- イベント管理 / イベント申込管理
- 限定資料管理
- 権利管理
- 権利の完了通知書管理
- 四つ葉ポイント管理
- 案件管理
- 配当金明細管理
- 振込管理
- 管理者管理（招待・権限剥奪）

### 3.2 ページ構成

#### 一般ページ

| Path | 内容 |
|---|---|
| `/` | ランディング |
| `/login` | ログイン |
| `/register` | 会員登録申請 |
| `/contact` | お問い合わせ |

#### 会員ページ

| Path | 内容 |
|---|---|
| `/member` | ダッシュボード（5 カード集約） |
| `/member/notices` | お知らせ一覧 |
| `/member/notices/[id]` | お知らせ詳細 |
| `/member/events` | イベント一覧 |
| `/member/events/[id]` | イベント詳細・申込 |
| `/member/resources` | 限定資料 |
| `/member/mypage` | マイページ TOP |
| `/member/mypage/profile` | 会員情報 |
| `/member/mypage/rights` | 権利取得確認 |
| `/member/mypage/certificates` | 完了通知書 |
| `/member/mypage/certificates/[id]` | 通知書詳細 |
| `/member/mypage/points` | ポイント履歴 |
| `/member/mypage/dividends` | 配当・振込確認 |
| `/member/mypage/dividends/[batchId]` | 振込詳細 |
| `/member/mypage/notifications` | 通知設定 |
| `/member/notifications` | 通知履歴 |

#### 管理ページ

| Path | 内容 |
|---|---|
| `/admin` | 管理ダッシュボード |
| `/admin/setup-mfa` | 2FA 初期設定 |
| `/admin/admins` | 管理者管理 |
| `/admin/members` | 会員一覧 |
| `/admin/members/[id]` | 会員詳細 |
| `/admin/notices` | お知らせ管理 |
| `/admin/events` | イベント管理 |
| `/admin/applications` | イベント申込管理 |
| `/admin/resources` | 限定資料管理 |
| `/admin/rights` | 権利管理 |
| `/admin/certificates` | 完了通知書管理 |
| `/admin/points` | ポイント管理 |
| `/admin/projects` | 案件管理 |
| `/admin/projects/[id]` | 案件詳細 |
| `/admin/dividends` | 配当明細管理 |
| `/admin/transfers` | 振込管理 |
| `/admin/transfers/[id]` | 振込詳細 |

### 3.3 アクセス制御

`profiles.role`: `member` / `admin`
`profiles.status`: `pending` / `approved` / `suspended` / `withdrawn`

| ユーザー状態 | `/member/*` | `/admin/*` |
|---|---|---|
| 未ログイン | `/login` リダイレクト | `/login` リダイレクト |
| `pending` | 拒否 | 拒否 |
| `approved` member | 許可 | `/member` リダイレクト |
| `approved` admin（2FA 未設定） | 許可 | `/admin/setup-mfa` のみ |
| `approved` admin（2FA 設定済） | 許可 | 許可 |
| `suspended` / `withdrawn` | 拒否 | 拒否 |

---

## 4. データモデル概観

### 4.1 主要テーブル群と役割

```
[認証・プロフィール]
  auth.users           ← Supabase Auth 管理
  profiles             ← 会員情報・ロール・ステータス
  admin_invitations    ← 管理者招待

[コンテンツ系]
  notices              ← お知らせ
  events               ← イベント
  event_applications   ← イベント申込
  resources            ← 会員限定資料
  inquiries            ← 問い合わせ

[権利・通知書]
  rights                       ← 権利マスタ
  user_rights                  ← 会員の権利取得状況
  right_completion_notices     ← 完了通知書

[ポイント]
  user_point_balances          ← 現在残高（キャッシュ）
  yotsuba_point_transactions   ← 増減履歴（真実の源）

[案件・配当・振込]
  projects                     ← 案件マスタ
  user_projects                ← 会員 ⇔ 案件の紐付け
  user_project_shares          ← 配当率の履歴
  project_dividend_periods     ← 案件×期間ごとの配当原資
  dividend_records             ← 配当明細
  transfer_batches             ← 振込バッチ
  transfer_batch_items         ← 振込 ⇔ 明細の紐付け

[通知]
  push_subscriptions           ← Push 購読情報
  notification_preferences     ← 通知 ON/OFF 設定
  notifications                ← 通知履歴・既読管理
```

### 4.2 配当計算の流れ

```
1. project_dividend_periods に当月分の期間を作成
   ↓
2. total_rate_pool / unit_price と根拠ファイルを admin が入力
   ↓
3. generate_dividend_records_for_period() RPC を実行
   ↓ user_project_shares から各会員の share を取得
   ↓ rate / unit / amount 型ごとに金額を計算
   ↓ 端数は切り捨て、残差を rounding_residual に記録
   ↓
4. dividend_records が pending 状態で作成される
   ↓
5. lock_dividend_period() で期間ロック（根拠必須）
   ↓
6. create_transfer_batch() で振込バッチ作成
   ↓ 複数の dividend_records を集約
   ↓ total_amount はトリガで自動計算
   ↓
7. mark_transfer_as_transferred() で振込済化
   ↓
8. 会員がマイページで「確認しました」を押す
   ↓ confirm_transfer_by_member() RPC
```

### 4.3 重要な制約関係

- `transfer_batch_items.dividend_record_id` は UNIQUE ← 二重振込防止
- `transfer_batches.total_amount` は items の合計と一致 ← トリガで保証
- `transfer_batch_items.amount` は `dividend_records.amount` と一致 ← トリガで検証
- `dividend_records` で `status = 'paid'` のレコードは金額変更不可 ← トリガで保護
- 通常の `dividend_records.amount >= 0`、調整明細のみマイナス可

### 4.4 過去再現性

配当率を遡って変更しても、過去の配当・振込データは**絶対に書き換えない**。修正は「調整明細」（`dividend_records.is_adjustment = true`）として新規追加する。

`get_active_share(user_id, project_id, as_of)` に過去日付を渡せば、当時の率が取得できる。

---

