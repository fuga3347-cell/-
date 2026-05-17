## 10. Supabase Storage

### 10.1 Bucket 一覧

| Bucket 名 | 公開設定 | 用途 |
|---|---|---|
| `resources` | private | 会員限定資料 |
| `right-completion-notices` | **private** | 完了通知書 PDF |
| `dividend-period-sources` | **private** | 配当原資の根拠ファイル（admin専用） |
| `transfer-statements` | **private** | 振込明細書 PDF |

すべて private。public bucket は使わない。

### 10.2 ファイルパス規約

```
right-completion-notices/{user_id}/{notice_id}.pdf
resources/{resource_id}/{filename}
dividend-period-sources/{project_id}/{period_id}_{YYYYMM}.{ext}
transfer-statements/{user_id}/{batch_id}.pdf
```

`user_id` をパスに含めることで、Storage RLS でフォルダ名と `auth.uid()` を照合可能。

### 10.3 Storage RLS

```sql
-- 完了通知書
create policy "rcn_storage_read_owner_or_admin"
  on storage.objects for select using (
    bucket_id = 'right-completion-notices'
    and (
      (storage.foldername(name))[1] = auth.uid()::text
      or public.is_admin()
    )
  );
create policy "rcn_storage_write_admin"
  on storage.objects for insert
  with check (bucket_id = 'right-completion-notices' and public.is_admin());
create policy "rcn_storage_update_admin"
  on storage.objects for update
  using (bucket_id = 'right-completion-notices' and public.is_admin());
create policy "rcn_storage_delete_admin"
  on storage.objects for delete
  using (bucket_id = 'right-completion-notices' and public.is_admin());

-- 会員限定資料
create policy "resources_storage_read_member"
  on storage.objects for select
  using (bucket_id = 'resources' and public.is_approved_member());
create policy "resources_storage_write_admin"
  on storage.objects for insert
  with check (bucket_id = 'resources' and public.is_admin());

-- 配当原資根拠（admin 専用、会員は完全に見えない）
create policy "dps_storage_admin_only"
  on storage.objects for all
  using (bucket_id = 'dividend-period-sources' and public.is_admin())
  with check (bucket_id = 'dividend-period-sources' and public.is_admin());

-- 振込明細書
create policy "ts_storage_read_owner_or_admin"
  on storage.objects for select using (
    bucket_id = 'transfer-statements'
    and (
      (storage.foldername(name))[1] = auth.uid()::text
      or public.is_admin()
    )
  );
create policy "ts_storage_write_admin"
  on storage.objects for insert
  with check (bucket_id = 'transfer-statements' and public.is_admin());
```

### 10.4 ファイル配信フロー（共通）

1. クライアントから「PDF を見たい」リクエスト
2. サーバ側 Route Handler でログインユーザーを確認
3. DB で本人 or admin であることを検証
4. `supabase.storage.from(bucket).createSignedUrl(file_path, 60)` で 60 秒の signed URL 発行
5. クライアントに URL を返す（DB には保存しない）

完了通知書の場合、合わせて `mark_notice_confirmed` RPC で `confirmed_at` を更新する。

---

## 11. 認証・認可・管理者招待

### 11.1 認証フロー

#### 会員

1. `/register` で会員登録申請（profiles に `pending` で作成）
2. admin が `/admin/members` で承認 → `status = 'approved'`
3. 会員が `/login` でログイン
4. `/member` 以下にアクセス可能

#### 管理者

1. 既存 admin が `/admin/admins` で「招待」を発行
2. メールで招待リンクが送付される（有効期限 7 日）
3. 招待された人がリンクをクリック
4. ログイン or 新規登録
5. `accept_admin_invitation` RPC が呼ばれ、`role = 'admin'` に昇格
6. `/admin/setup-mfa` で 2FA 必須設定
7. `mark_mfa_enrolled` 呼出後、`is_admin()` が true を返すようになり管理機能を使える

### 11.2 1 人目の管理者

CLI スクリプトで作成（chicken-and-egg 問題の解決）。

```bash
SUPABASE_URL=... SUPABASE_SERVICE_ROLE_KEY=... \
  npx tsx scripts/create-first-admin.ts admin@yotsuba-club.com SecurePassword123!
```

### 11.3 ミドルウェア

```ts
// middleware.ts
export async function middleware(req) {
  const res = NextResponse.next();
  const supabase = createMiddlewareClient({ req, res });
  const { data: { session } } = await supabase.auth.getSession();
  const pathname = req.nextUrl.pathname;

  if (pathname.startsWith('/admin')) {
    if (!session) {
      return NextResponse.redirect(new URL('/login?redirect=' + pathname, req.url));
    }
    const { data: profile } = await supabase
      .from('profiles')
      .select('role, mfa_enrolled_at, status')
      .eq('id', session.user.id).single();

    if (!profile || profile.role !== 'admin' || profile.status !== 'approved') {
      return NextResponse.redirect(new URL('/member', req.url));
    }
    if (!profile.mfa_enrolled_at && pathname !== '/admin/setup-mfa') {
      return NextResponse.redirect(new URL('/admin/setup-mfa', req.url));
    }
  }

  if (pathname.startsWith('/member')) {
    if (!session) {
      return NextResponse.redirect(new URL('/login', req.url));
    }
    const { data: profile } = await supabase
      .from('profiles').select('status').eq('id', session.user.id).single();

    if (!profile || profile.status !== 'approved') {
      return NextResponse.redirect(new URL('/login?error=not_approved', req.url));
    }
  }

  return res;
}

export const config = {
  matcher: ['/admin/:path*', '/member/:path*'],
};
```

### 11.4 認可の三層

| 層 | 役割 | 漏れたら |
|---|---|---|
| middleware.ts | 未ログイン・権限不足を入口で弾く | UX 悪化 |
| RSC ガード | ページごとに `requireApprovedMember()` / `requireAdmin()` | 個別画面のアクセス漏れ |
| **RLS（DB）** | 本人・admin 以外のデータを絶対返さない | **データ漏洩** |

クライアント側の表示制御だけでセキュリティを担保しない。RLS が最後の砦。

### 11.5 セキュリティヘッダ

`next.config.js` で `/admin/*` に追加：

```js
headers: async () => [
  {
    source: '/admin/:path*',
    headers: [
      { key: 'X-Frame-Options', value: 'DENY' },
      { key: 'X-Robots-Tag', value: 'noindex, nofollow' },
      { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
    ],
  },
],
```

`public/robots.txt`：

```
User-agent: *
Disallow: /admin
Disallow: /login
```

---

## 12. PWA とプッシュ通知

### 12.1 PWA 化（next-pwa）

#### 追加ファイル

```
public/
  manifest.json
  icon-72.png / 96 / 128 / 144 / 152 / 192 / 384 / 512.png
  apple-touch-icon.png (180x180)
  icon-maskable-512.png
```

`manifest.json` と `next.config.js` の設定は `yotsuba_pwa_addendum.md` §1 を参照。

### 12.2 通知 5 種類

| カテゴリ | トリガ | 通知例 |
|---|---|---|
| `transfer` | `transfer_batches.status` が `transferred` に変化 | 「¥420,000 の配当金が振込されました」 |
| `certificate` | `right_completion_notices` に新規 INSERT | 「権利の完了通知書が発行されました」 |
| `point` | `yotsuba_point_transactions` に `earn` で INSERT | 「500 pt が付与されました」 |
| `notice` | `notices.is_published` が true になる | 「新しいお知らせがあります」 |
| `event` | `events.is_published` が true / 開催前日 | 「明日のイベントのリマインダー」 |

### 12.3 アーキテクチャ

```
[クライアント]              [DB トリガ]              [Edge Function]
   購読登録                   状態変化検知              Web Push 送信
       ↓                         ↓                         ↑
push_subscriptions       notifications (pending)    pg_cron 1分ごと
```

### 12.4 イベントリマインダー

```sql
select cron.schedule('event-reminder-daily', '0 9 * * *',
  $$select public.send_event_reminders()$$);
```

毎日朝 9 時に、翌日開催のイベント申込者へ通知。

### 12.5 プライバシー配慮

通知本文には機密情報を入れすぎない（ロック画面に表示されるため）。

- ✅ 「¥420,000 の配当金が振込されました」
- ❌ 「口座末尾 1234 に振込」

---

## 13. Next.js ディレクトリ構成

```
yotsuba-members-site/
├─ app/
│  ├─ (public)/
│  │  ├─ page.tsx                                # /
│  │  ├─ login/page.tsx
│  │  ├─ register/page.tsx
│  │  └─ contact/page.tsx
│  ├─ admin/
│  │  ├─ accept-invitation/page.tsx              # 招待承認
│  │  ├─ setup-mfa/page.tsx                      # MFA 設定強制誘導
│  │  ├─ layout.tsx                              # admin ガード
│  │  ├─ page.tsx                                # /admin ダッシュボード
│  │  ├─ admins/page.tsx                         # 管理者管理
│  │  ├─ members/
│  │  │  ├─ page.tsx
│  │  │  └─ [id]/page.tsx
│  │  ├─ notices/page.tsx
│  │  ├─ events/page.tsx
│  │  ├─ applications/page.tsx
│  │  ├─ resources/page.tsx
│  │  ├─ rights/page.tsx
│  │  ├─ certificates/page.tsx
│  │  ├─ points/page.tsx
│  │  ├─ projects/
│  │  │  ├─ page.tsx
│  │  │  └─ [id]/page.tsx
│  │  ├─ dividends/page.tsx
│  │  └─ transfers/
│  │     ├─ page.tsx
│  │     └─ [id]/page.tsx
│  ├─ member/
│  │  ├─ layout.tsx                              # member ガード
│  │  ├─ page.tsx                                # /member ダッシュボード
│  │  ├─ notices/...
│  │  ├─ events/...
│  │  ├─ resources/page.tsx
│  │  ├─ notifications/page.tsx
│  │  └─ mypage/
│  │     ├─ page.tsx                             # 5 カード集約
│  │     ├─ profile/page.tsx
│  │     ├─ rights/page.tsx
│  │     ├─ certificates/
│  │     │  ├─ page.tsx
│  │     │  └─ [id]/page.tsx
│  │     ├─ points/page.tsx
│  │     ├─ dividends/
│  │     │  ├─ page.tsx
│  │     │  └─ [batchId]/page.tsx
│  │     └─ notifications/page.tsx               # 通知設定
│  └─ api/
│     ├─ member/transfers/[id]/confirm/route.ts
│     ├─ member/certificates/[id]/signed-url/route.ts
│     ├─ member/push/subscribe/route.ts
│     ├─ admin/transfers/route.ts
│     ├─ admin/transfers/[id]/mark-paid/route.ts
│     ├─ admin/dividends/generate/route.ts
│     └─ admin/admins/invite/route.ts
├─ components/
│  ├─ ui/                                        # shadcn/ui
│  ├─ member/
│  │  ├─ DashboardCards/
│  │  │  ├─ ProfileCard.tsx
│  │  │  ├─ RightsCard.tsx
│  │  │  ├─ CertificatesCard.tsx
│  │  │  ├─ PointsCard.tsx
│  │  │  └─ DividendsCard.tsx
│  │  └─ ...
│  └─ admin/
│     └─ ...
├─ lib/
│  ├─ supabase/
│  │  ├─ client.ts
│  │  ├─ server.ts
│  │  └─ admin.ts                                # Service Role
│  ├─ auth/
│  │  └─ guards.ts
│  ├─ pwa/
│  │  └─ pushSubscribe.ts
│  └─ types/
│     └─ database.ts                             # supabase gen types
├─ scripts/
│  └─ create-first-admin.ts
├─ supabase/
│  ├─ functions/
│  │  └─ send-push/
│  │     ├─ index.ts
│  │     └─ web-push.ts
│  ├─ migrations/
│  │  ├─ 0001_enums.sql
│  │  ├─ 0002_tables_core.sql
│  │  ├─ 0003_tables_rights.sql
│  │  ├─ 0004_tables_points.sql
│  │  ├─ 0005_tables_projects_dividends.sql
│  │  ├─ 0006_tables_notifications.sql
│  │  ├─ 0007_triggers.sql
│  │  ├─ 0008_rls.sql
│  │  ├─ 0009_rpc.sql
│  │  ├─ 0010_storage_policies.sql
│  │  └─ 0011_cron_jobs.sql
│  └─ seed.sql
├─ public/
│  ├─ manifest.json
│  ├─ icon-*.png
│  ├─ robots.txt
│  └─ sw.js
├─ middleware.ts
├─ next.config.js
├─ package.json
└─ tsconfig.json
```

---

## 14. ページ別実装ガイド

### 14.1 /member/mypage（ダッシュボード）

5 カードを並列取得：

```ts
const [profile, rights, certs, points, dividendSummary, nextTransfer] = await Promise.all([
  supabase.from('profiles').select('*').eq('id', user.id).single(),
  supabase.from('user_rights')
    .select('*, rights(name, description)')
    .eq('user_id', user.id)
    .order('updated_at', { ascending: false }).limit(3),
  supabase.from('right_completion_notices')
    .select('id, title, status, issued_at')
    .eq('user_id', user.id)
    .order('issued_at', { ascending: false }),
  supabase.from('user_point_balances')
    .select('current_points, total_earned_points, total_used_points')
    .eq('user_id', user.id).maybeSingle(),
  supabase.from('v_user_dividend_summary').select('*').eq('user_id', user.id).maybeSingle(),
  supabase.from('v_user_next_transfer').select('*').eq('user_id', user.id).maybeSingle(),
]);
```

### 14.2 /member/mypage/dividends（配当・振込確認）

- サマリーカード（4 数値）
- 振込確認一覧（カード型、確認ボタン + PDF DL）
- 案件別配当明細（テーブル）

「確認しました」ボタンは `status = 'transferred' && confirmation_status = 'unconfirmed'` の振込にのみ表示。

### 14.3 /admin/transfers（振込管理）

- KPI 4 カード（予定 / 振込済 / 未確認 / 失敗）
- フィルタ・検索
- 一括選択 + 一括操作バー
- テーブル + 詳細プレビュー

### 14.4 /admin/projects/[id] の配当期間タブ

月次運用のメイン作業画面。

- 期間一覧
- 「2026 年 5 月分を作成」ボタン → `ensure_monthly_periods`
- 各期間に対し：
  - `total_rate_pool` / `unit_price` 入力
  - 根拠ファイルアップロード
  - 「前月コピー」ボタン
  - 「計算実行」ボタン → `generate_dividend_records_for_period`
  - 「ロック」ボタン → `lock_dividend_period`

---

## 15. 運用フロー

### 15.1 月次配当の運用フロー

```
[毎月初]
  1. /admin → 「月次運用を開始」ボタン
     → ensure_monthly_periods(2026, 5)
     → 全アクティブ案件に draft 状態の期間を作成

[案件ごと]
  2. /admin/projects/[id] の配当期間タブ
     a. 「前月コピー」or 手入力で total_rate_pool / unit_price
     b. 根拠ファイル（Excel/PDF）をアップロード
     c. source_note にメモ
     d. 「計算実行」→ generate_dividend_records_for_period
     e. 配当明細プレビュー、必要なら手動調整
     f. 「ロック」→ lock_dividend_period（根拠必須）

[月末・振込前]
  3. /admin/dividends で全員分の明細を確認
  4. 会員ごとに複数明細を選択 → 「振込バッチ作成」
     → create_transfer_batch
  5. /admin/transfers で振込予定リスト確認
  6. 銀行で実際の振込実施
  7. 「振込済に変更」→ mark_transfer_as_transferred
     → 会員に push 通知が自動送信
     → 会員がマイページで「確認しました」を押す
```

### 15.2 会員入会フロー

```
1. /register で本人が申請（profiles: pending）
2. admin にメール通知（オプション）
3. /admin/members で承認 → status = 'approved'
4. 会員にメール通知（オプション）
5. /admin/projects/[id] で案件への紐付け（user_projects 追加）
6. 配当率設定（apply_share_change）
7. 翌月の月次運用から自動的に配当発生
```

### 15.3 管理者追加フロー

```
1. 既存 admin が /admin/admins → 「招待」
2. 招待先メールアドレス入力 → create_admin_invitation
3. メール送信（Edge Function）
4. 招待者がリンククリック → /admin/accept-invitation?token=xxx
5. ログイン（既存ユーザー）or 新規登録
6. accept_admin_invitation 自動実行 → role = 'admin'
7. /admin/setup-mfa に強制誘導
8. TOTP 設定 → mark_mfa_enrolled
9. /admin ダッシュボードへ
```

---

## 16. テスト観点

### 16.1 セキュリティ（最重要）

- [ ] 未ログインで `/member/*` → `/login`
- [ ] `pending` 会員で `/member/*` → 拒否
- [ ] member で `/admin/*` → `/member` リダイレクト
- [ ] 2FA 未設定 admin で `/admin/*` → `/admin/setup-mfa`
- [ ] 会員 A が会員 B の `dividend_records` を SELECT → 0 件
- [ ] 会員 A が会員 B の `transfer_batches` を SELECT → 0 件
- [ ] 会員 A が会員 B の `user_project_shares` を SELECT → 0 件
- [ ] 会員 A が会員 B の `notifications` を SELECT → 0 件
- [ ] 会員 A が会員 B の通知書 PDF を直 GET → 403
- [ ] 会員が `transfer_batches` を直接 UPDATE → 拒否
- [ ] 会員が `dividend_records` に INSERT/UPDATE/DELETE → 拒否
- [ ] member が `is_admin()` を呼ぶ → false
- [ ] 2FA 未設定 admin が `is_admin()` を呼ぶ → false
- [ ] `confirm_transfer_by_member(他人の batchId)` → 例外
- [ ] Service Role Key がクライアントバンドルに含まれていない（grep 確認）

### 16.2 整合性

- [ ] `transfer_batch_items` の INSERT/UPDATE/DELETE で `total_amount` 自動再計算
- [ ] 異なる user_id の dividend を同一バッチに → 例外
- [ ] 同じ dividend を 2 バッチに → UNIQUE 違反
- [ ] cancelled dividend をバッチに → 例外
- [ ] paid dividend の amount を変更 → 例外
- [ ] 通常明細でマイナス amount → CHECK 違反
- [ ] 調整明細でマイナス amount → OK
- [ ] `apply_point_transaction` で残高超過減算 → 例外
- [ ] share_type=rate で value=1.5 → CHECK 違反
- [ ] share_type=unit で value=30.5 → CHECK 違反
- [ ] locked 期間に対し `generate_dividend_records_for_period` → 例外
- [ ] 同一 user×project×type で `effective_to is null` の行が 2 つ → UNIQUE 違反

### 16.3 業務ロジック

- [ ] 端数発生時、`rounding_residual` に正しい値が記録される
- [ ] rate / unit / amount 混在で全員分が正しく計算される
- [ ] `should_pay_amount_share` が count 終了後に false を返す
- [ ] `get_active_share` に過去日付を渡すと当時の値が返る
- [ ] `apply_share_change` の遡及で note に履歴が残る
- [ ] `create_adjustment_dividend` で調整明細が作成される
- [ ] 期間再計算で `occurrences_used` が二重カウントされない
- [ ] イベント前日に申込者に通知が届く（pg_cron）

### 16.4 管理者

- [ ] 自分自身を `demote_admin` → 例外
- [ ] 最後の admin を `demote_admin` → 例外
- [ ] 期限切れ token で `accept_admin_invitation` → 例外
- [ ] 別ユーザーのログインで他人の招待を承諾 → 例外
- [ ] revoke 後の招待を承諾 → 例外

### 16.5 PWA・通知

- [ ] スマホで「ホーム画面に追加」表示
- [ ] アプリ起動が standalone モード
- [ ] オフラインで最後のページ表示
- [ ] 振込完了で push 通知到達
- [ ] 通知書発行で push 通知到達
- [ ] ポイント付与で push 通知到達
- [ ] お知らせ・イベント公開で全承認会員に通知
- [ ] カテゴリ OFF 設定時に push 送信されない
- [ ] 通知タップで該当ページに遷移
- [ ] 410 エンドポイントで `is_active = false` 化

---

## 17. 開発フェーズ

| Phase | 内容 | 期間目安 |
|---|---|---|
| **P0** | 環境構築（Next.js / Supabase / Vercel / Claude Code） | 2-3 日 |
| **P1** | DB スキーマ構築（migration 一式の適用） | 3-5 日 |
| **P2** | 認証フロー・ミドルウェア・1 人目 admin 作成 | 3-4 日 |
| **P3** | 会員ページ実装（マイページ・ダッシュボード・お知らせ・イベント） | 2 週間 |
| **P4.1** | 管理画面基盤（TOP・会員管理・お知らせ・イベント） | 1 週間 |
| **P4.2** | 権利・通知書・ポイント管理 | 1 週間 |
| **P4.5** | 案件・配当・振込（最重要・最複雑） | 2 週間 |
| **P5** | PWA・通知システム | 2 週間 |
| **P6** | セキュリティ監査・実データテスト | 1-2 週間 |
| **P7** | 法務確認・利用規約・運用準備 | 並行 |
| **合計** | エンジニア 1 名フルタイム | 約 2.5-3 ヶ月 |

---

## 18. 未決事項と将来拡張

### 18.1 未決事項（リリース前に決定が必要）

| # | 項目 | 影響範囲 |
|---|---|---|
| 1 | 会員番号の採番ルール（連番 / 年度プレフィックス） | profiles.member_number |
| 2 | 通知書の「再発行」の扱い（旧を reissued / 上書き） | right_completion_notices |
| 3 | 配当原資 P3 案（自動算出）への移行時期 | project_dividend_periods |
| 4 | rate 合計 100% 超の事故防止策 | apply_share_change / RPC |
| 5 | 振込済後の取消フロー（返金処理用レコード設計） | dividend_records |
| 6 | adjust 型ポイントの符号付き運用 vs 別関数 | apply_point_transaction |
| 7 | share 遡及変更時の会員通知 | 業務ポリシー |
| 8 | イベント定員管理（DB 側 / アプリ側） | event_applications |

### 18.2 将来拡張（MVP 後）

| 拡張 | 内容 |
|---|---|
| 端数処理 R2 → R3 | 最大持分者集約 / 案件プール繰越 |
| 配当原資 P3 案 | 売上・費用から自動算出 |
| 操作監査ログ | admin の全操作を記録 |
| クリティカル操作の再認証 | 振込実行時にパスワード再入力 |
| 管理者サブドメイン分離（B 案） | admin.yotsuba-club.com |
| IP 制限 | オフィス IP のみ admin アクセス |
| ロール細分化 | super_admin / accounting / support |
| メール通知 | push と並行でメール送信 |
| LINE 通知 | LINE Messaging API |
| 月次まとめ通知 | 「先月の配当合計は XXX 円でした」 |
| App Store 配信 | React Native への移行 |
| CSV 一括取込 | 配当原資・会員紐付け |
| PDF 自動生成 | 振込明細書を自動生成 |
| 承認フロー | 入力者と承認者を分ける |
| 多言語対応 | 英語版 UI |

### 18.3 設計上残された議論

詳細は元の追補書を参照：

- `yotsuba_db_design.md` §10 — 基本設計の残課題
- `yotsuba_dividends_addendum.md` §10 — 配当機能の残課題
- `yotsuba_shares_addendum.md` §11 — share 設計の残課題
- `yotsuba_pool_addendum.md` §10 — 配当原資の残課題
- `yotsuba_rounding_addendum.md` §7 — 端数処理の将来拡張
- `yotsuba_retroactive_addendum.md` §9 — 遡及変更の残課題
- `yotsuba_pwa_addendum.md` §13 — PWA の将来拡張
- `yotsuba_admin_invitation_addendum.md` §10 — 管理者管理の将来拡張

---

## 付録 A: クイックリファレンス

### A.1 主要 RPC 一覧

| RPC | 用途 | 呼び元 |
|---|---|---|
| `apply_point_transaction` | ポイント増減 | admin |
| `apply_share_change` | 配当率変更 | admin |
| `get_active_share` | 過去日時点の share 取得 | admin |
| `ensure_monthly_periods` | 月次期間一括作成 | admin |
| `copy_from_previous_period` | 前月設定コピー | admin |
| `generate_dividend_records_for_period` | 配当明細生成 | admin |
| `lock_dividend_period` | 期間ロック | admin |
| `create_transfer_batch` | 振込バッチ作成 | admin |
| `mark_transfer_as_transferred` | 振込済化 | admin |
| `confirm_transfer_by_member` | 会員確認 | 会員 |
| `create_adjustment_dividend` | 調整明細作成 | admin |
| `calculate_period_adjustment_diff` | 差額計算 | admin |
| `create_admin_invitation` | 管理者招待 | admin |
| `accept_admin_invitation` | 招待承認 | 招待者 |
| `revoke_admin_invitation` | 招待取消 | admin |
| `demote_admin` | 権限剥奪 | admin |
| `mark_mfa_enrolled` | MFA 設定完了 | admin（自分） |
| `register_push_subscription` | Push 購読登録 | 会員 |
| `mark_notification_as_read` | 通知既読化 | 会員 |
| `enqueue_notification` | 通知作成 | DB トリガ |

### A.2 主要トリガ一覧

| トリガ | テーブル | 用途 |
|---|---|---|
| `tbi_recalc_total_aiud` | transfer_batch_items | total_amount 再計算 |
| `tbi_check_user_match_biu` | transfer_batch_items | user 整合性検証 |
| `dr_protect_paid_bu` | dividend_records | paid 編集禁止 |
| `transfer_completed_notify` | transfer_batches | 振込完了通知 |
| `certificate_issued_notify` | right_completion_notices | 通知書発行通知 |
| `point_earned_notify` | yotsuba_point_transactions | ポイント付与通知 |
| `notice_published_notify` | notices | お知らせ公開通知 |
| `event_published_notify` | events | イベント公開通知 |

### A.3 主要 VIEW

| VIEW | 用途 |
|---|---|
| `v_user_dividend_summary` | 会員ごとの配当サマリ |
| `v_user_next_transfer` | 会員ごとの次回振込予定 |

---

**このドキュメントについて**

本書は四つ葉倶楽部 会員制サイトのデータ構造マスター設計書です。
詳細な議論経緯は個別の追補書を参照してください。
Claude Code で実装を進める際は、このマスター設計書を参照してください。

