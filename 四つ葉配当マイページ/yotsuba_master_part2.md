## 5. テーブル一覧と ENUM 定義

### 5.1 ENUM 完全リスト

```sql
-- 認証・会員
create type user_role as enum ('member', 'admin');
create type user_status as enum ('pending', 'approved', 'suspended', 'withdrawn');

-- 権利
create type right_status as enum (
  'not_acquired', 'applying', 'acquired', 'completed', 'suspended', 'expired'
);

-- 完了通知書
create type notice_confirm_status as enum (
  'unconfirmed', 'confirmed', 'reissued', 'invalid'
);

-- ポイント
create type point_transaction_type as enum (
  'earn', 'use', 'adjust', 'expire'
);

-- 案件・配当
create type project_status as enum ('draft', 'active', 'closed', 'suspended');
create type user_project_status as enum ('active', 'paused', 'ended');
create type dividend_status as enum ('pending', 'scheduled', 'paid', 'cancelled');
create type transfer_status as enum ('scheduled', 'transferred', 'failed', 'cancelled');
create type transfer_confirmation_status as enum ('unconfirmed', 'confirmed');

-- 配当率（share）
create type share_type as enum ('rate', 'unit', 'amount');
create type termination_type as enum ('none', 'date', 'count', 'conditional');

-- 配当明細の調整
create type adjustment_reason as enum (
  'correction', 'retroactive_contract', 'retroactive_join', 'manual'
);

-- 通知
create type notification_category as enum (
  'transfer', 'certificate', 'point', 'notice', 'event'
);
create type notification_status as enum ('pending', 'sent', 'failed', 'read');
```

---

## 6. テーブル DDL（完全版）

このセクションでは、すべての追補で議論した最終形のテーブル定義をまとめる。

### 6.1 認証・プロフィール

#### profiles

```sql
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  email text not null,
  name text not null,
  phone text,
  member_number text unique,
  birth_date date,
  role user_role not null default 'member',
  status user_status not null default 'pending',
  mfa_enrolled_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index profiles_status_idx on public.profiles(status);
create index profiles_role_idx on public.profiles(role);
```

#### admin_invitations

```sql
create table public.admin_invitations (
  id uuid primary key default gen_random_uuid(),
  email text not null,
  token text not null unique,
  invited_by uuid not null references auth.users(id) on delete restrict,
  expires_at timestamptz not null default (now() + interval '7 days'),
  used_at timestamptz,
  revoked_at timestamptz,
  note text,
  created_at timestamptz not null default now()
);

create unique index ai_active_email_uniq
  on public.admin_invitations(email)
  where used_at is null and revoked_at is null;

create index ai_token_active_idx
  on public.admin_invitations(token)
  where used_at is null and revoked_at is null;
```

### 6.2 コンテンツ系

#### notices

```sql
create table public.notices (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  body text not null,
  category text not null default 'その他',
  is_published boolean not null default false,
  published_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

#### events / event_applications

```sql
create table public.events (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  description text not null,
  event_date timestamptz not null,
  location text,
  capacity integer,
  fee integer not null default 0,
  application_deadline timestamptz,
  is_published boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.event_applications (
  id uuid primary key default gen_random_uuid(),
  event_id uuid not null references public.events(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  status text not null default 'applied',
  message text,
  created_at timestamptz not null default now(),
  unique (event_id, user_id)
);
```

#### resources / inquiries

```sql
create table public.resources (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  description text,
  file_path text not null,
  visibility text not null default 'members',
  is_published boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.inquiries (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  email text not null,
  message text not null,
  status text not null default 'new',
  created_at timestamptz not null default now()
);
```

### 6.3 権利・通知書

#### rights / user_rights

```sql
create table public.rights (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  description text,
  is_active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.user_rights (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  right_id uuid not null references public.rights(id) on delete cascade,
  status right_status not null default 'not_acquired',
  acquired_at timestamptz,
  completed_at timestamptz,
  expires_at timestamptz,
  note text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (user_id, right_id)
);

create index user_rights_user_idx on public.user_rights(user_id);
create index user_rights_status_idx on public.user_rights(status);
```

#### right_completion_notices

```sql
create table public.right_completion_notices (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  right_id uuid not null references public.rights(id) on delete cascade,
  title text not null,
  file_path text not null,
  status notice_confirm_status not null default 'unconfirmed',
  issued_at timestamptz not null default now(),
  confirmed_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index rcn_user_idx on public.right_completion_notices(user_id);
create index rcn_status_idx on public.right_completion_notices(status);
```

### 6.4 ポイント

```sql
create table public.user_point_balances (
  user_id uuid primary key references auth.users(id) on delete cascade,
  current_points integer not null default 0,
  total_earned_points integer not null default 0,
  total_used_points integer not null default 0,
  updated_at timestamptz not null default now(),
  constraint current_points_nonneg check (current_points >= 0)
);

create table public.yotsuba_point_transactions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  type point_transaction_type not null,
  points integer not null,
  balance_after integer not null,
  title text not null,
  description text,
  occurred_at timestamptz not null default now(),
  created_at timestamptz not null default now(),
  constraint balance_after_nonneg check (balance_after >= 0)
);

create index ypt_user_idx on public.yotsuba_point_transactions(user_id, occurred_at desc);
```

### 6.5 案件・配当・振込

#### projects / user_projects

```sql
create table public.projects (
  id uuid primary key default gen_random_uuid(),
  code text unique not null,
  name text not null,
  description text,
  status project_status not null default 'active',
  start_date date,
  end_date date,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  constraint projects_date_order check (
    start_date is null or end_date is null or start_date <= end_date
  )
);

create index projects_status_idx on public.projects(status);

create table public.user_projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  project_id uuid not null references public.projects(id) on delete cascade,
  status user_project_status not null default 'active',
  joined_at timestamptz,
  left_at timestamptz,
  note text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (user_id, project_id)
);

create index user_projects_user_idx on public.user_projects(user_id);
create index user_projects_project_idx on public.user_projects(project_id);
```

#### user_project_shares（配当率の履歴）

```sql
create table public.user_project_shares (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  project_id uuid not null references public.projects(id) on delete cascade,
  share_type share_type not null,
  share_value numeric(14, 6) not null,
  effective_from date not null,
  effective_to date,
  termination_type termination_type not null default 'none',
  max_occurrences integer,
  occurrences_used integer not null default 0,
  termination_condition jsonb,
  note text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  constraint share_value_nonneg check (share_value >= 0),
  constraint rate_max check (share_type <> 'rate' or share_value <= 1),
  constraint amount_integer check (share_type <> 'amount' or share_value = floor(share_value)),
  constraint unit_integer check (share_type <> 'unit' or share_value = floor(share_value)),
  constraint effective_order check (effective_to is null or effective_from < effective_to),

  constraint termination_consistency check (
    (termination_type = 'none' and max_occurrences is null and termination_condition is null)
    or (termination_type = 'date' and max_occurrences is null and termination_condition is null)
    or (termination_type = 'count' and max_occurrences is not null and max_occurrences > 0)
    or (termination_type = 'conditional' and termination_condition is not null)
  ),
  constraint occurrences_within_max check (
    max_occurrences is null or occurrences_used <= max_occurrences
  ),
  constraint termination_only_for_amount check (
    share_type = 'amount' or termination_type = 'none'
  )
);

create unique index user_project_shares_active_uniq
  on public.user_project_shares (user_id, project_id, share_type)
  where effective_to is null;

create index ups_lookup_idx
  on public.user_project_shares (user_id, project_id, share_type, effective_from desc);
```

#### project_dividend_periods

```sql
create table public.project_dividend_periods (
  id uuid primary key default gen_random_uuid(),
  project_id uuid not null references public.projects(id) on delete cascade,
  period_start date not null,
  period_end date not null,
  total_rate_pool integer,
  unit_price integer,
  status text not null default 'draft',
  calculated_at timestamptz,
  source_file_path text,
  source_note text,
  locked_at timestamptz,
  locked_by uuid references auth.users(id) on delete set null,
  rounding_residual integer not null default 0,
  rounding_strategy text not null default 'truncate',
  note text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  constraint period_order check (period_start <= period_end),
  constraint pool_nonneg check (total_rate_pool is null or total_rate_pool >= 0),
  constraint unit_price_nonneg check (unit_price is null or unit_price >= 0),
  unique (project_id, period_start, period_end)
);

create index pdp_project_period_idx
  on public.project_dividend_periods(project_id, period_end desc);
```

#### dividend_records

```sql
create table public.dividend_records (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  project_id uuid not null references public.projects(id) on delete cascade,
  period_start date not null,
  period_end date not null,
  amount integer not null,
  status dividend_status not null default 'pending',
  scheduled_payment_date date,
  paid_at timestamptz,
  is_adjustment boolean not null default false,
  target_period_id uuid references public.project_dividend_periods(id),
  adjustment_reason adjustment_reason,
  note text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  constraint dr_period_order check (period_start <= period_end),
  constraint dr_amount_check check (
    (is_adjustment = true) or (amount >= 0)
  )
);

create index dr_user_idx on public.dividend_records(user_id, status);
create index dr_project_idx on public.dividend_records(project_id);
create index dr_user_period_idx on public.dividend_records(user_id, period_end desc);
create index dr_user_period_status_idx
  on public.dividend_records(user_id, period_end desc, status);
create index dr_adjustment_idx
  on public.dividend_records(is_adjustment, target_period_id)
  where is_adjustment = true;
```

#### transfer_batches

```sql
create table public.transfer_batches (
  id uuid primary key default gen_random_uuid(),
  transfer_number text unique not null,
  user_id uuid not null references auth.users(id) on delete cascade,
  total_amount integer not null default 0,
  status transfer_status not null default 'scheduled',
  scheduled_transfer_date date,
  transferred_at timestamptz,
  confirmation_status transfer_confirmation_status not null default 'unconfirmed',
  confirmed_at timestamptz,
  confirmed_by uuid references auth.users(id) on delete set null,
  pdf_file_path text,
  note text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  constraint tb_total_nonneg check (total_amount >= 0)
);

create index tb_user_idx on public.transfer_batches(user_id, status);
create index tb_scheduled_date_idx
  on public.transfer_batches(scheduled_transfer_date);
```

#### transfer_batch_items

```sql
create table public.transfer_batch_items (
  id uuid primary key default gen_random_uuid(),
  transfer_batch_id uuid not null references public.transfer_batches(id) on delete cascade,
  dividend_record_id uuid not null references public.dividend_records(id) on delete restrict,
  amount integer not null,
  created_at timestamptz not null default now(),
  unique (transfer_batch_id, dividend_record_id),
  unique (dividend_record_id)
);

create index tbi_batch_idx on public.transfer_batch_items(transfer_batch_id);
```

注: `transfer_batch_items.amount` の `>= 0` 制約は意図的に外している。マイナス調整明細を含めるため。`transfer_batches.total_amount >= 0` で合計マイナスは防止。

### 6.6 通知

```sql
create table public.push_subscriptions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  endpoint text not null,
  p256dh_key text not null,
  auth_key text not null,
  user_agent text,
  device_label text,
  is_active boolean not null default true,
  last_used_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (user_id, endpoint)
);

create index ps_user_active_idx
  on public.push_subscriptions(user_id, is_active)
  where is_active = true;

create table public.notification_preferences (
  user_id uuid not null references auth.users(id) on delete cascade,
  category notification_category not null,
  push_enabled boolean not null default true,
  email_enabled boolean not null default false,
  updated_at timestamptz not null default now(),
  primary key (user_id, category)
);

create table public.notifications (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  category notification_category not null,
  title text not null,
  body text not null,
  link_url text,
  related_resource_type text,
  related_resource_id uuid,
  status notification_status not null default 'pending',
  push_sent_at timestamptz,
  read_at timestamptz,
  created_at timestamptz not null default now()
);

create index n_user_status_idx
  on public.notifications(user_id, status, created_at desc);
create index n_user_unread_idx
  on public.notifications(user_id, created_at desc)
  where read_at is null;
```

### 6.7 ダッシュボード集計用 VIEW

```sql
create or replace view public.v_user_dividend_summary
with (security_invoker = on) as
select
  user_id,
  coalesce(sum(amount) filter (where status in ('pending', 'scheduled', 'paid')), 0)::int
    as total_scheduled_amount,
  coalesce(sum(amount) filter (where status = 'paid'), 0)::int
    as total_paid_amount,
  coalesce(sum(amount) filter (where status in ('pending', 'scheduled')), 0)::int
    as total_unpaid_amount,
  count(distinct project_id) as project_count
from public.dividend_records
group by user_id;

create or replace view public.v_user_next_transfer
with (security_invoker = on) as
select distinct on (user_id)
  user_id,
  id as next_transfer_batch_id,
  transfer_number,
  scheduled_transfer_date,
  total_amount
from public.transfer_batches
where status = 'scheduled' and scheduled_transfer_date is not null
order by user_id, scheduled_transfer_date asc;
```

---

