# 四つ葉倶楽部 会員制サイト

会員専用 Web サイト・PWA。会員ごとの権利状況・配当金・四つ葉ポイントを一元管理する。

## 設計ドキュメント

`docs/` 配下に集約されています。

- [マスター設計書](docs/yotsuba_master_design.md) — まずこれを読む
- [Claude Code 用の指示書](CLAUDE.md)

## 技術スタック

- Next.js 15 (App Router) + TypeScript
- Supabase (PostgreSQL + Auth + Storage + Edge Functions)
- Tailwind CSS + shadcn/ui
- Vercel デプロイ
- PWA (next-pwa)

## セットアップ

### 1. 必要な前提条件

- Node.js 20+
- pnpm or npm
- Supabase CLI (`brew install supabase/tap/supabase`)
- Vercel CLI（デプロイ時のみ）

### 2. リポジトリのクローンと依存インストール

```bash
git clone <repo-url> yotsuba-members-site
cd yotsuba-members-site
pnpm install
```

### 3. Supabase プロジェクト

#### ローカル開発の場合

```bash
supabase init
supabase start
```

ローカル Supabase のキーは `supabase start` の出力に表示されます。

#### 本番 Supabase の場合

[Supabase Dashboard](https://app.supabase.com/) で新しいプロジェクトを作成し、Project URL と anon key を取得。

### 4. 環境変数

`.env.local` を作成：

```env
# クライアント公開可
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
NEXT_PUBLIC_VAPID_PUBLIC_KEY=...

# サーバ専用（絶対に NEXT_PUBLIC_ を付けない）
SUPABASE_SERVICE_ROLE_KEY=...
VAPID_PRIVATE_KEY=...
VAPID_SUBJECT=mailto:admin@yotsuba-club.com
```

VAPID キーは初回のみ生成：

```bash
npx web-push generate-vapid-keys
```

### 5. データベース構築

```bash
supabase db push
```

`supabase/migrations/` の SQL がすべて適用されます。

### 6. TypeScript 型生成

```bash
supabase gen types typescript --local > lib/types/database.ts
```

スキーマを変更したら必ず再実行。

### 7. 1 人目の管理者作成

```bash
SUPABASE_URL=... SUPABASE_SERVICE_ROLE_KEY=... \
  npx tsx scripts/create-first-admin.ts admin@yotsuba-club.com SecurePassword123!
```

その後ログインして `/admin/setup-mfa` で 2FA 設定。

### 8. 開発サーバ起動

```bash
pnpm dev
```

[http://localhost:3000](http://localhost:3000) で起動。

## デプロイ

### Vercel

```bash
vercel link
vercel env add SUPABASE_SERVICE_ROLE_KEY
vercel env add VAPID_PRIVATE_KEY
vercel env add VAPID_SUBJECT
vercel --prod
```

`NEXT_PUBLIC_*` は Vercel ダッシュボードで設定。

### Supabase Edge Functions

```bash
supabase functions deploy send-push --no-verify-jwt
supabase secrets set VAPID_PUBLIC_KEY=... VAPID_PRIVATE_KEY=... VAPID_SUBJECT=...
```

### pg_cron 設定

Supabase ダッシュボードで `pg_cron` 拡張を有効化後、`supabase/migrations/0011_cron_jobs.sql` を適用。

## 開発フロー

### 機能追加

1. マスター設計書 (`docs/yotsuba_master_design.md`) で関連箇所を確認
2. CLAUDE.md のルールに沿って実装
3. 必要なら migration を追加 (`supabase/migrations/`)
4. TypeScript 型再生成
5. テストを書く
6. RLS が効くか手動でも確認

### マイグレーション追加

```bash
supabase migration new add_something
# supabase/migrations/<timestamp>_add_something.sql が作られる
# 編集後...
supabase db push
supabase gen types typescript --local > lib/types/database.ts
```

### よく使うコマンド

```bash
pnpm dev          # 開発サーバ
pnpm build        # 本番ビルド
pnpm lint         # ESLint
pnpm test         # テスト実行
pnpm test:e2e     # E2E テスト (Playwright)

supabase status   # ローカル Supabase の状態確認
supabase db reset # DB を空にして migration を再適用（要注意）
```

## ディレクトリ構成

```
yotsuba-members-site/
├─ app/                 # Next.js App Router
├─ components/          # React コンポーネント
├─ lib/                 # 共通ロジック・型定義
├─ scripts/             # CLI スクリプト（管理者作成など）
├─ supabase/
│  ├─ migrations/       # DB マイグレーション
│  ├─ functions/        # Edge Functions
│  └─ seed.sql          # 初期データ
├─ public/              # 静的ファイル・PWA アセット
├─ tests/               # Playwright E2E テスト
├─ docs/                # 設計ドキュメント
├─ middleware.ts        # 認証ガード
├─ CLAUDE.md            # Claude Code 用指示書
└─ README.md            # このファイル
```

## トラブルシューティング

### RLS で SELECT が 0 件になる

- ログインしているか確認
- 該当ユーザーが本人 or admin か確認
- `is_admin()` が false を返している可能性（2FA 未設定 admin など）
- Supabase の Studio で SQL Editor → `set role authenticated;` → `select auth.uid();` で確認

### マイグレーションが失敗する

- 既存テーブルとの依存関係を確認
- ENUM の追加順を確認（依存先より先に作る）
- ローカルで `supabase db reset` を試す（**本番では絶対やらない**）

### 型エラーが消えない

- `supabase gen types` を再実行
- VS Code を再起動して TypeScript Server をリスタート
- `node_modules/.cache` を削除

### Edge Function がデプロイされない

- `supabase login` で再ログイン
- `supabase link --project-ref <ref>` でプロジェクトを再リンク
- ログ確認: `supabase functions logs send-push`

## ライセンス

Private. 四つ葉倶楽部 専用。

## サポート

設計や実装の質問は CLAUDE.md と設計ドキュメントを参照。それでも不明な点は開発チームへ。
