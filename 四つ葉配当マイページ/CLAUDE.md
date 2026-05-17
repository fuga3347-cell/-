# CLAUDE.md

このプロジェクトは **四つ葉倶楽部 会員制サイト** です。Next.js + Supabase で構築する PWA 対応の会員ダッシュボードです。

詳細な設計はすべて `docs/yotsuba_master_design.md` に集約されています。**コード変更前にこのファイルを必ず参照してください。**

---

## プロジェクトの本質

**会員ごとの金銭・権利情報を扱うサービスです。** 一般的な Web サービスではなく、以下を常に意識してください：

- 配当金の振込実績を管理する（金額の改ざんは事故）
- 会員間で情報が絶対に漏れない（A会員がB会員のデータを見られない）
- 過去データは書き換えない（監査追跡可能性を維持）

「動けばOK」ではなく「**金銭が絡んでいる前提で、慎重に**」が基本姿勢です。

---

## 技術スタック（固定）

- Next.js 15 (App Router) / TypeScript / Tailwind CSS / shadcn/ui
- Supabase (PostgreSQL + Auth + Storage + Edge Functions)
- next-pwa
- Vercel デプロイ

これら以外のフレームワーク・ライブラリを勝手に追加しないでください。必要なら**事前に相談**。

---

## 絶対に守るべきルール

### セキュリティ

1. **RLS を全テーブルで有効化**。クライアントの表示制御だけでセキュリティを担保しない
2. **`SUPABASE_SERVICE_ROLE_KEY` をクライアントに絶対出さない**。`NEXT_PUBLIC_` プレフィックス禁止
3. **金額・権限の変更は必ず RPC 経由**。直接 UPDATE しない
4. **Storage の private bucket のファイルは signed URL で配信**。public URL を保存・公開しない
5. **`is_admin()` 関数は 2FA 設定済 admin のみ true を返す仕様**。これを変更しない

### データ整合性

6. **金額は integer（円）。小数を絶対に使わない**
7. **過去の `dividend_records`・`transfer_batches` は更新しない**。修正は調整明細で表現
8. **`paid` 状態の `dividend_records` は触らない**。トリガで保護されているが、それに頼らず注意
9. **`transfer_batches.total_amount` はトリガで自動計算**。クライアント側で合計を送らない

### 命名・スタイル

10. **テーブル名・カラム名は snake_case**（既存設計に合わせる）
11. **TypeScript の型名は PascalCase、変数は camelCase**
12. **コンポーネント名は PascalCase、ファイル名も合わせる**
13. **金額表示は `Intl.NumberFormat('ja-JP')` でカンマ区切り**
14. **日付表示は `YYYY/MM/DD` 形式が基本**

---

## ディレクトリ構成

```
app/                    # Next.js App Router
  (public)/             # 未認証アクセス可能
  admin/                # 管理画面（middleware で保護）
  member/               # 会員ページ（middleware で保護）
  api/                  # Route Handlers
components/
  ui/                   # shadcn/ui（自動生成、原則編集しない）
  member/
  admin/
lib/
  supabase/             # client.ts / server.ts / admin.ts
  auth/                 # guards.ts
  types/                # database.ts (supabase gen types で生成)
supabase/
  migrations/           # SQL ファイル（順序: 0001, 0002, ...）
  functions/            # Edge Functions
middleware.ts           # /admin と /member の認証ガード
```

新しいファイルを作る前に、既存の同種ファイルを確認して構造を合わせてください。

---

## Supabase クライアントの使い分け

3 種類あります。**用途を間違えないでください**。

```ts
// lib/supabase/client.ts - ブラウザ用（anon key）
import { createBrowserClient } from '@supabase/ssr';

// lib/supabase/server.ts - RSC・Route Handler 用（anon key + cookie）
import { createServerClient } from '@supabase/ssr';

// lib/supabase/admin.ts - サーバ専用（Service Role Key、RLS バイパス）
import { createClient } from '@supabase/supabase-js';
```

**`admin.ts` は以下の場合のみ使用**：

- DB トリガから呼ばれない、サーバ側で実行する管理者操作
- メール送信などシステム的な処理
- CLI スクリプト

それ以外は `server.ts` を使用。RLS で制御される。

---

## RPC 呼び出しのパターン

すべての金額・権限操作は RPC 経由。直接 UPDATE しないでください。

```ts
// ❌ NG - 直接 UPDATE
await supabase.from('user_point_balances')
  .update({ current_points: 1000 }).eq('user_id', userId);

// ✅ OK - RPC 経由
await supabase.rpc('apply_point_transaction', {
  p_user_id: userId,
  p_type: 'earn',
  p_points: 100,
  p_title: 'イベント参加ボーナス',
});
```

主要 RPC は `docs/yotsuba_master_design.md` §9 と付録 A.1 を参照。

---

## エラーハンドリング

```ts
const { data, error } = await supabase.rpc('...', {...});
if (error) {
  console.error('RPC failed:', error);
  return { success: false, message: error.message };
}
```

- RPC のエラーメッセージは日本語で会員に見せても問題ない設計（admin エラーは英語）
- ただし内部実装の詳細を漏らすメッセージは加工する
- 失敗してもアプリがクラッシュしないように

---

## RLS の動作確認

新しいテーブルやポリシーを追加したら、以下を必ずテスト：

1. 未ログインで SELECT → 0 件
2. 別会員でログインして SELECT → 0 件（本人データのみ見えるか）
3. member ロールで admin 系操作 → 拒否
4. 2FA 未設定 admin で admin 系操作 → 拒否

`supabase/seed.sql` に複数のテストユーザーを用意してあるので、それで切り替えて確認してください。

---

## マイグレーションの作法

1. **既存のマイグレーションを編集しない**。常に新しい番号で追加
2. **CREATE TABLE よりも ALTER TABLE で段階追加**を優先（本番デプロイ後の変更を考慮）
3. **migration を書いたら必ずローカルで apply してエラー確認**

```bash
supabase db push                          # ローカルへ適用
supabase gen types typescript --local > lib/types/database.ts   # 型再生成
```

---

## UI 実装の指針

### shadcn/ui を優先

オリジナル UI を作る前に shadcn/ui に該当コンポーネントがないか確認してください。あれば `npx shadcn@latest add <component>` で追加して使用。

### モバイルファースト

会員サイトは **PWA でスマホから見る前提**。すべての画面を以下で確認：

- 最小幅 360px（iPhone SE）でレイアウト崩れがない
- タップターゲットが 44x44px 以上
- 横スクロールが発生しない

### 金額表示の標準化

```tsx
// utils/format.ts
export const formatYen = (amount: number) =>
  `¥${amount.toLocaleString('ja-JP')}`;

// 使用例
<span>{formatYen(420000)}</span>  // → ¥420,000
```

数字は `font-variant-numeric: tabular-nums` を必ず適用。

### ステータスバッジ

色のルールは固定：

- 緑（`#085041` on `#E1F5EE`）: 完了・成功
- 黄（`#854F0B` on `#FAEEDA`）: 予定・進行中・要対応
- 赤（`#791F1F` on `#FCEBEB`）: 失敗・エラー・キャンセル
- 青（`#0C447C` on `#E6F1FB`）: 情報・付帯
- 紫（`#3C3489` on `#EEEDFE`）: 特殊カテゴリ

---

## テストの方針

MVP では以下を最低限カバー：

1. **セキュリティテスト**（最優先）— RLS が効いているか
2. **RPC のエッジケース** — 残高超過減算、ロック済期間の操作、二重バッチなど
3. **トリガの動作** — total_amount の自動再計算、paid 編集禁止

E2E は Playwright を使用。`/tests/` ディレクトリに配置。

詳細なテスト観点は `docs/yotsuba_master_design.md` §16 を参照。

---

## コミット規約

```
feat: 新機能
fix: バグ修正
refactor: リファクタリング
style: フォーマット変更
docs: ドキュメント変更
test: テスト追加・修正
chore: 依存関係更新など
```

DB スキーマを変更したら必ず `lib/types/database.ts` を再生成してコミット。

---

## やってはいけないこと

- ❌ Service Role Key をクライアントコードに含める
- ❌ 金額計算をクライアント側で完結させる
- ❌ RLS を回避するために admin クライアントを使う（正規の RPC を使う）
- ❌ `dividend_records` や `transfer_batches` を直接 UPDATE
- ❌ 「とりあえず動けばいいだろう」と RLS を一時的に無効化
- ❌ 通知本文にロック画面で見せたくない機密情報を入れる（口座番号、本名フルネームなど）
- ❌ 既存マイグレーションファイルの編集
- ❌ shadcn/ui の `components/ui/` 配下を直接編集
- ❌ 設計書にない新規テーブルや新規 RPC を勝手に追加

迷ったら **聞いてください**。

---

## 参照ドキュメント

- **マスター設計書**: `docs/yotsuba_master_design.md`（必読）
- 配当機能の議論経緯: `docs/yotsuba_dividends_addendum.md`
- 配当率設計: `docs/yotsuba_shares_addendum.md`
- 配当原資・月次運用: `docs/yotsuba_pool_addendum.md`
- 端数処理: `docs/yotsuba_rounding_addendum.md`
- 遡及変更: `docs/yotsuba_retroactive_addendum.md`
- PWA・通知: `docs/yotsuba_pwa_addendum.md`
- 管理者招待・2FA: `docs/yotsuba_admin_invitation_addendum.md`

---

## このプロジェクトでの Claude Code の役割

- **設計の理解者**: マスター設計書をベースに実装する
- **慎重なコーダー**: 金銭が絡むので「動く」より「正しい」を優先
- **質問する人**: 設計書に書かれていないことは、勝手に判断せず聞く
- **テスト書き**: 機能追加と同時にテストも書く

「設計書通りに正確に作る」「曖昧なら聞く」が基本姿勢です。
