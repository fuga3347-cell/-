## 7. 整合性トリガ

### 7.1 ヘルパー関数

すべての RLS・トリガで使う共通関数。

```sql
create or replace function public.is_admin()
returns boolean
language sql
security definer
set search_path = public
stable
as $$
  select exists (
    select 1 from public.profiles
    where id = auth.uid()
      and role = 'admin'
      and mfa_enrolled_at is not null
  );
$$;

create or replace function public.is_admin_pending_mfa()
returns boolean
language sql
security definer
set search_path = public
stable
as $$
  select exists (
    select 1 from public.profiles
    where id = auth.uid()
      and role = 'admin'
      and mfa_enrolled_at is null
  );
$$;

create or replace function public.is_approved_member()
returns boolean
language sql
security definer
set search_path = public
stable
as $$
  select exists (
    select 1 from public.profiles
    where id = auth.uid() and status = 'approved'
  );
$$;

grant execute on function public.is_admin() to authenticated;
grant execute on function public.is_admin_pending_mfa() to authenticated;
grant execute on function public.is_approved_member() to authenticated;
```

**重要**: `is_admin()` は **2FA 設定済の admin のみ true を返す**。2FA 未設定 admin は管理データにアクセス不可。

### 7.2 transfer_batches.total_amount 自動再計算

```sql
create or replace function public.recalc_transfer_batch_total()
returns trigger
language plpgsql
as $$
declare
  v_batch_id uuid;
begin
  v_batch_id := coalesce(new.transfer_batch_id, old.transfer_batch_id);
  update public.transfer_batches t
    set total_amount = coalesce((
      select sum(amount)::int
      from public.transfer_batch_items
      where transfer_batch_id = v_batch_id
    ), 0),
    updated_at = now()
  where t.id = v_batch_id;
  return null;
end;
$$;

create trigger tbi_recalc_total_aiud
  after insert or update or delete on public.transfer_batch_items
  for each row execute function public.recalc_transfer_batch_total();
```

### 7.3 transfer_batch_items の user 整合性チェック

```sql
create or replace function public.check_tbi_user_match()
returns trigger
language plpgsql
as $$
declare
  v_batch_user uuid;
  v_dr_user uuid;
  v_dr_status dividend_status;
  v_dr_amount integer;
begin
  select user_id into v_batch_user from public.transfer_batches where id = new.transfer_batch_id;
  select user_id, status, amount
    into v_dr_user, v_dr_status, v_dr_amount
    from public.dividend_records where id = new.dividend_record_id;

  if v_batch_user is null or v_dr_user is null then
    raise exception 'invalid batch or dividend record';
  end if;
  if v_batch_user <> v_dr_user then
    raise exception 'user mismatch: batch user=% vs dividend user=%', v_batch_user, v_dr_user;
  end if;
  if v_dr_status = 'cancelled' then
    raise exception 'cancelled dividend cannot be included in transfer batch';
  end if;
  if new.amount <> v_dr_amount then
    raise exception 'transfer_batch_item.amount (%) must equal dividend_record.amount (%)', new.amount, v_dr_amount;
  end if;
  return new;
end;
$$;

create trigger tbi_check_user_match_biu
  before insert or update on public.transfer_batch_items
  for each row execute function public.check_tbi_user_match();
```

### 7.4 paid 配当明細の編集禁止

```sql
create or replace function public.protect_paid_dividend()
returns trigger
language plpgsql
as $$
begin
  if old.status = 'paid' then
    if new.amount is distinct from old.amount
       or new.user_id is distinct from old.user_id
       or new.project_id is distinct from old.project_id
       or new.period_start is distinct from old.period_start
       or new.period_end is distinct from old.period_end then
      raise exception 'paid dividend record cannot be modified; create an adjustment record instead';
    end if;
  end if;
  new.updated_at := now();
  return new;
end;
$$;

create trigger dr_protect_paid_bu
  before update on public.dividend_records
  for each row execute function public.protect_paid_dividend();
```

---

## 8. RLS ポリシー

すべてのテーブルで RLS 有効化。基本パターン: 「本人のみ閲覧、admin は全権、書き込みは RPC 経由」。

### 8.1 profiles

```sql
alter table public.profiles enable row level security;

create policy "profiles_select_self"
  on public.profiles for select using (id = auth.uid());

create policy "profiles_select_admin"
  on public.profiles for select using (public.is_admin());

create policy "profiles_update_self"
  on public.profiles for update
  using (id = auth.uid()) with check (id = auth.uid());

create policy "profiles_update_admin"
  on public.profiles for update
  using (public.is_admin()) with check (public.is_admin());

create policy "profiles_delete_admin"
  on public.profiles for delete using (public.is_admin());
```

### 8.2 admin_invitations

```sql
alter table public.admin_invitations enable row level security;

create policy "ai_admin_all"
  on public.admin_invitations for all
  using (public.is_admin()) with check (public.is_admin());
```

注: 招待トークンの検証は `accept_admin_invitation` RPC（security definer）で行うため、未認証ユーザー向けの SELECT ポリシーは作らない。

### 8.3 公開系（notices / events / resources）

```sql
alter table public.notices enable row level security;
create policy "notices_select_members"
  on public.notices for select
  using (is_published = true and public.is_approved_member());
create policy "notices_admin_all"
  on public.notices for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.events enable row level security;
create policy "events_select_members"
  on public.events for select
  using (is_published = true and public.is_approved_member());
create policy "events_admin_all"
  on public.events for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.resources enable row level security;
create policy "resources_select_members"
  on public.resources for select
  using (is_published = true and visibility = 'members' and public.is_approved_member());
create policy "resources_admin_all"
  on public.resources for all
  using (public.is_admin()) with check (public.is_admin());
```

### 8.4 event_applications

```sql
alter table public.event_applications enable row level security;

create policy "ea_select_self"
  on public.event_applications for select using (user_id = auth.uid());

create policy "ea_insert_self"
  on public.event_applications for insert
  with check (user_id = auth.uid() and public.is_approved_member());

create policy "ea_delete_self"
  on public.event_applications for delete using (user_id = auth.uid());

create policy "ea_admin_all"
  on public.event_applications for all
  using (public.is_admin()) with check (public.is_admin());
```

### 8.5 権利・通知書

```sql
alter table public.rights enable row level security;
create policy "rights_select_members"
  on public.rights for select
  using (is_active = true and public.is_approved_member());
create policy "rights_admin_all"
  on public.rights for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.user_rights enable row level security;
create policy "ur_select_self"
  on public.user_rights for select using (user_id = auth.uid());
create policy "ur_admin_all"
  on public.user_rights for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.right_completion_notices enable row level security;
create policy "rcn_select_self"
  on public.right_completion_notices for select using (user_id = auth.uid());
create policy "rcn_admin_all"
  on public.right_completion_notices for all
  using (public.is_admin()) with check (public.is_admin());
```

通知書の `confirmed_at` 更新は RPC `mark_notice_confirmed` 経由のみ。

### 8.6 ポイント

```sql
alter table public.user_point_balances enable row level security;
create policy "upb_select_self"
  on public.user_point_balances for select using (user_id = auth.uid());
create policy "upb_admin_all"
  on public.user_point_balances for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.yotsuba_point_transactions enable row level security;
create policy "ypt_select_self"
  on public.yotsuba_point_transactions for select using (user_id = auth.uid());
create policy "ypt_admin_all"
  on public.yotsuba_point_transactions for all
  using (public.is_admin()) with check (public.is_admin());
```

### 8.7 案件・配当・振込

```sql
alter table public.projects enable row level security;
create policy "projects_select_member_joined"
  on public.projects for select
  using (
    public.is_approved_member()
    and exists (
      select 1 from public.user_projects up
      where up.project_id = projects.id
        and up.user_id = auth.uid()
        and up.status <> 'ended'
    )
  );
create policy "projects_admin_all"
  on public.projects for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.user_projects enable row level security;
create policy "up_select_self"
  on public.user_projects for select using (user_id = auth.uid());
create policy "up_admin_all"
  on public.user_projects for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.user_project_shares enable row level security;
create policy "ups_select_self"
  on public.user_project_shares for select using (user_id = auth.uid());
create policy "ups_admin_all"
  on public.user_project_shares for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.project_dividend_periods enable row level security;
create policy "pdp_admin_all"
  on public.project_dividend_periods for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.dividend_records enable row level security;
create policy "dr_select_self"
  on public.dividend_records for select using (user_id = auth.uid());
create policy "dr_admin_all"
  on public.dividend_records for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.transfer_batches enable row level security;
create policy "tb_select_self"
  on public.transfer_batches for select using (user_id = auth.uid());
create policy "tb_admin_all"
  on public.transfer_batches for all
  using (public.is_admin()) with check (public.is_admin());

alter table public.transfer_batch_items enable row level security;
create policy "tbi_select_self"
  on public.transfer_batch_items for select
  using (exists (
    select 1 from public.transfer_batches tb
    where tb.id = transfer_batch_items.transfer_batch_id
      and tb.user_id = auth.uid()
  ));
create policy "tbi_admin_all"
  on public.transfer_batch_items for all
  using (public.is_admin()) with check (public.is_admin());
```

**重要**: `transfer_batches` には会員 UPDATE ポリシーを作らない。会員の確認操作は `confirm_transfer_by_member` RPC のみで実行する（RLS ではカラム単位の更新制限ができないため）。

### 8.8 通知

```sql
alter table public.push_subscriptions enable row level security;
create policy "ps_select_self"
  on public.push_subscriptions for select using (user_id = auth.uid());
create policy "ps_modify_self"
  on public.push_subscriptions for all
  using (user_id = auth.uid()) with check (user_id = auth.uid());
create policy "ps_admin_select"
  on public.push_subscriptions for select using (public.is_admin());

alter table public.notification_preferences enable row level security;
create policy "np_modify_self"
  on public.notification_preferences for all
  using (user_id = auth.uid()) with check (user_id = auth.uid());

alter table public.notifications enable row level security;
create policy "n_select_self"
  on public.notifications for select using (user_id = auth.uid());
create policy "n_update_read_self"
  on public.notifications for update
  using (user_id = auth.uid()) with check (user_id = auth.uid());
create policy "n_admin_all"
  on public.notifications for all
  using (public.is_admin()) with check (public.is_admin());
```

### 8.9 inquiries

```sql
alter table public.inquiries enable row level security;
create policy "inq_insert_anyone"
  on public.inquiries for insert with check (true);
create policy "inq_admin_all"
  on public.inquiries for select using (public.is_admin());
create policy "inq_admin_update"
  on public.inquiries for update
  using (public.is_admin()) with check (public.is_admin());
```

---

## 9. RPC 関数（業務ロジック）

すべての金額操作・状態遷移は RPC 経由。`security definer` で実行されるため、RLS をバイパスする代わりに関数内で権限チェックを行う。

### 9.1 ポイント

#### apply_point_transaction

```sql
create or replace function public.apply_point_transaction(
  p_user_id uuid,
  p_type point_transaction_type,
  p_points integer,
  p_title text,
  p_description text default null
) returns public.yotsuba_point_transactions
language plpgsql
security definer
set search_path = public
as $$
declare
  v_current integer;
  v_new integer;
  v_delta integer;
  v_tx public.yotsuba_point_transactions;
begin
  if p_points <= 0 then
    raise exception 'points must be positive';
  end if;

  v_delta := case p_type
    when 'earn' then p_points
    when 'use' then -p_points
    when 'expire' then -p_points
    when 'adjust' then p_points
  end;

  insert into public.user_point_balances(user_id)
    values (p_user_id) on conflict (user_id) do nothing;

  select current_points into v_current
    from public.user_point_balances
    where user_id = p_user_id for update;

  v_new := v_current + v_delta;
  if v_new < 0 then
    raise exception 'insufficient points: current=%, delta=%', v_current, v_delta;
  end if;

  update public.user_point_balances
    set current_points = v_new,
        total_earned_points = total_earned_points + greatest(v_delta, 0),
        total_used_points = total_used_points + greatest(-v_delta, 0),
        updated_at = now()
  where user_id = p_user_id;

  insert into public.yotsuba_point_transactions(
    user_id, type, points, balance_after, title, description
  ) values (
    p_user_id, p_type, p_points, v_new, p_title, p_description
  ) returning * into v_tx;

  return v_tx;
end;
$$;
```

### 9.2 配当率（share）

#### get_active_share

```sql
create or replace function public.get_active_share(
  p_user_id uuid,
  p_project_id uuid,
  p_as_of date
) returns public.user_project_shares
language sql
stable
security invoker
as $$
  select * from public.user_project_shares
  where user_id = p_user_id and project_id = p_project_id
    and effective_from <= p_as_of
    and (effective_to is null or effective_to > p_as_of)
  order by effective_from desc limit 1;
$$;
```

#### apply_share_change

```sql
create or replace function public.apply_share_change(
  p_user_id uuid, p_project_id uuid, p_share_type share_type,
  p_new_value numeric, p_effective_from date,
  p_note text default null,
  p_allow_retroactive boolean default false,
  p_termination_type termination_type default 'none',
  p_max_occurrences integer default null,
  p_termination_condition jsonb default null
) returns public.user_project_shares
language plpgsql
security definer
set search_path = public
as $$
declare
  v_new public.user_project_shares;
  v_old_id uuid;
  v_locked_count integer;
begin
  if not public.is_admin() then
    raise exception 'admin only';
  end if;
  if p_share_type <> 'amount' and p_termination_type <> 'none' then
    raise exception 'termination_type is only allowed for share_type=amount';
  end if;

  select count(*) into v_locked_count
    from public.project_dividend_periods
    where project_id = p_project_id and status = 'locked'
      and period_end >= p_effective_from;
  if v_locked_count > 0 and not p_allow_retroactive then
    raise exception 'retroactive change affects % locked period(s); set p_allow_retroactive=true to proceed', v_locked_count;
  end if;

  select id into v_old_id
    from public.user_project_shares
    where user_id = p_user_id and project_id = p_project_id
      and share_type = p_share_type and effective_to is null;
  if v_old_id is not null then
    update public.user_project_shares
      set effective_to = p_effective_from, updated_at = now()
      where id = v_old_id;
  end if;

  insert into public.user_project_shares(
    user_id, project_id, share_type, share_value, effective_from,
    termination_type, max_occurrences, termination_condition, note
  ) values (
    p_user_id, p_project_id, p_share_type, p_new_value, p_effective_from,
    p_termination_type, p_max_occurrences, p_termination_condition,
    case when p_allow_retroactive
         then coalesce(p_note || ' | ', '') || format('[retroactive by %s at %s]', auth.uid(), now())
         else p_note end
  ) returning * into v_new;

  return v_new;
end;
$$;
```

### 9.3 配当期間と明細生成

#### ensure_monthly_periods

```sql
create or replace function public.ensure_monthly_periods(
  p_year integer, p_month integer
) returns setof public.project_dividend_periods
language plpgsql
security definer
set search_path = public
as $$
declare
  v_period_start date;
  v_period_end date;
  v_project record;
  v_existing public.project_dividend_periods;
  v_new public.project_dividend_periods;
begin
  if not public.is_admin() then
    raise exception 'admin only';
  end if;

  v_period_start := make_date(p_year, p_month, 1);
  v_period_end := (v_period_start + interval '1 month' - interval '1 day')::date;

  for v_project in
    select id from public.projects where status = 'active'
  loop
    select * into v_existing
      from public.project_dividend_periods
      where project_id = v_project.id
        and period_start = v_period_start and period_end = v_period_end;

    if v_existing.id is not null then
      return next v_existing;
    else
      insert into public.project_dividend_periods(
        project_id, period_start, period_end, status
      ) values (
        v_project.id, v_period_start, v_period_end, 'draft'
      ) returning * into v_new;
      return next v_new;
    end if;
  end loop;
end;
$$;
```

#### generate_dividend_records_for_period（中核）

```sql
create or replace function public.generate_dividend_records_for_period(
  p_period_id uuid
) returns setof public.dividend_records
language plpgsql
security definer
set search_path = public
as $$
declare
  v_period public.project_dividend_periods;
  v_share public.user_project_shares;
  v_amount integer;
  v_user record;
  v_record public.dividend_records;
  v_total_distributed integer := 0;
  v_total_rate_share_value numeric := 0;
  v_residual integer := 0;
begin
  if not public.is_admin() then
    raise exception 'admin only';
  end if;

  select * into v_period
    from public.project_dividend_periods
    where id = p_period_id for update;

  if v_period.id is null then
    raise exception 'period not found';
  end if;
  if v_period.status = 'locked' then
    raise exception 'period is locked; cannot regenerate';
  end if;

  -- count型 amount share の occurrences_used を巻き戻す
  update public.user_project_shares ups
    set occurrences_used = occurrences_used - 1,
        effective_to = case when occurrences_used - 1 < max_occurrences then null else effective_to end,
        updated_at = now()
    where ups.share_type = 'amount'
      and ups.termination_type = 'count'
      and exists (
        select 1 from public.dividend_records dr
        where dr.user_id = ups.user_id and dr.project_id = ups.project_id
          and dr.period_start = v_period.period_start
          and dr.period_end = v_period.period_end and dr.status = 'pending'
      );

  delete from public.dividend_records
    where project_id = v_period.project_id
      and period_start = v_period.period_start
      and period_end = v_period.period_end
      and status = 'pending';

  for v_user in
    select distinct up.user_id from public.user_projects up
    where up.project_id = v_period.project_id and up.status = 'active'
  loop
    v_share := public.get_active_share(v_user.user_id, v_period.project_id, v_period.period_end);
    if v_share.id is null then continue; end if;

    if v_share.share_type = 'rate' then
      if v_period.total_rate_pool is null then
        raise exception 'total_rate_pool is required for rate-type shares (period %)', p_period_id;
      end if;
      v_amount := floor(v_period.total_rate_pool * v_share.share_value)::integer;
      v_total_rate_share_value := v_total_rate_share_value + v_share.share_value;
      v_total_distributed := v_total_distributed + v_amount;

    elsif v_share.share_type = 'unit' then
      if v_period.unit_price is null then
        raise exception 'unit_price is required for unit-type shares (period %)', p_period_id;
      end if;
      v_amount := (v_period.unit_price * v_share.share_value)::integer;

    elsif v_share.share_type = 'amount' then
      if not public.should_pay_amount_share(v_share.id, v_period.period_end) then
        continue;
      end if;
      v_amount := v_share.share_value::integer;

      if v_share.termination_type = 'count' then
        update public.user_project_shares
          set occurrences_used = occurrences_used + 1, updated_at = now()
          where id = v_share.id;

        if v_share.occurrences_used + 1 >= v_share.max_occurrences then
          update public.user_project_shares
            set effective_to = v_period.period_end + interval '1 day', updated_at = now()
            where id = v_share.id;
        end if;
      end if;
    end if;

    if v_amount > 0 then
      insert into public.dividend_records(
        user_id, project_id, period_start, period_end, amount, status, note
      ) values (
        v_user.user_id, v_period.project_id, v_period.period_start, v_period.period_end,
        v_amount, 'pending',
        format('auto-generated: share_type=%s, share_value=%s', v_share.share_type, v_share.share_value)
      ) returning * into v_record;
      return next v_record;
    end if;
  end loop;

  -- 端数記録（R1: 切り捨て）
  if v_period.total_rate_pool is not null then
    v_residual := floor(v_period.total_rate_pool * v_total_rate_share_value)::integer - v_total_distributed;
    if v_residual < 0 then v_residual := 0; end if;
  end if;

  update public.project_dividend_periods
    set status = 'calculated', calculated_at = now(),
        rounding_residual = v_residual,
        rounding_strategy = 'truncate', updated_at = now()
    where id = p_period_id;

  return;
end;
$$;
```

#### should_pay_amount_share

```sql
create or replace function public.should_pay_amount_share(
  p_share_id uuid, p_period_end date
) returns boolean
language plpgsql
stable
security definer
set search_path = public
as $$
declare
  v_share public.user_project_shares;
begin
  select * into v_share from public.user_project_shares where id = p_share_id;

  if v_share.id is null or v_share.share_type <> 'amount' then return false; end if;
  if v_share.effective_from > p_period_end then return false; end if;
  if v_share.effective_to is not null and v_share.effective_to <= p_period_end then
    return false;
  end if;

  if v_share.termination_type = 'count' then
    if v_share.occurrences_used >= v_share.max_occurrences then return false; end if;
  end if;

  if v_share.termination_type = 'conditional' then
    if not public.check_termination_condition(
      v_share.user_id, v_share.termination_condition, p_period_end
    ) then return false; end if;
  end if;

  return true;
end;
$$;
```

#### lock_dividend_period

```sql
create or replace function public.lock_dividend_period(p_period_id uuid)
returns public.project_dividend_periods
language plpgsql
security definer
set search_path = public
as $$
declare
  v_period public.project_dividend_periods;
begin
  if not public.is_admin() then
    raise exception 'admin only';
  end if;

  select * into v_period
    from public.project_dividend_periods
    where id = p_period_id for update;
  if v_period.id is null then raise exception 'period not found'; end if;
  if v_period.status = 'locked' then raise exception 'already locked'; end if;
  if v_period.status <> 'calculated' then
    raise exception 'period must be calculated before locking';
  end if;
  if v_period.source_file_path is null and v_period.source_note is null then
    raise exception 'source_file_path or source_note is required to lock the period';
  end if;

  update public.project_dividend_periods
    set status = 'locked', locked_at = now(),
        locked_by = auth.uid(), updated_at = now()
    where id = p_period_id returning * into v_period;
  return v_period;
end;
$$;
```

### 9.4 振込

#### create_transfer_batch

```sql
create or replace function public.create_transfer_batch(
  p_user_id uuid, p_dividend_ids uuid[],
  p_scheduled_transfer_date date default null,
  p_note text default null
) returns public.transfer_batches
language plpgsql
security definer
set search_path = public
as $$
declare
  v_batch public.transfer_batches;
  v_transfer_number text;
begin
  if not public.is_admin() then raise exception 'admin only'; end if;
  if p_user_id is null or array_length(p_dividend_ids, 1) is null then
    raise exception 'user_id and dividend_ids required';
  end if;

  if exists (
    select 1 from public.dividend_records
    where id = any(p_dividend_ids) and user_id <> p_user_id
  ) then raise exception 'some dividend records belong to other users'; end if;

  if exists (
    select 1 from public.dividend_records
    where id = any(p_dividend_ids) and status = 'cancelled'
  ) then raise exception 'cancelled dividends cannot be batched'; end if;

  if exists (
    select 1 from public.transfer_batch_items
    where dividend_record_id = any(p_dividend_ids)
  ) then raise exception 'some dividends already belong to a transfer batch'; end if;

  v_transfer_number := 'TR-' || to_char(now(), 'YYYYMMDD') || '-' ||
                       lpad((floor(random() * 10000))::text, 4, '0');

  insert into public.transfer_batches(
    transfer_number, user_id, scheduled_transfer_date, note
  ) values (
    v_transfer_number, p_user_id, p_scheduled_transfer_date, p_note
  ) returning * into v_batch;

  insert into public.transfer_batch_items(transfer_batch_id, dividend_record_id, amount)
  select v_batch.id, dr.id, dr.amount
    from public.dividend_records dr where dr.id = any(p_dividend_ids);

  update public.dividend_records
    set status = 'scheduled',
        scheduled_payment_date = coalesce(scheduled_payment_date, p_scheduled_transfer_date),
        updated_at = now()
    where id = any(p_dividend_ids);

  select * into v_batch from public.transfer_batches where id = v_batch.id;
  return v_batch;
end;
$$;
```

#### mark_transfer_as_transferred

```sql
create or replace function public.mark_transfer_as_transferred(
  p_batch_id uuid, p_transferred_at timestamptz default now()
) returns public.transfer_batches
language plpgsql
security definer
set search_path = public
as $$
declare
  v_batch public.transfer_batches;
begin
  if not public.is_admin() then raise exception 'admin only'; end if;

  update public.transfer_batches
    set status = 'transferred', transferred_at = p_transferred_at,
        updated_at = now()
    where id = p_batch_id and status = 'scheduled'
    returning * into v_batch;
  if v_batch.id is null then
    raise exception 'batch not found or not in scheduled state';
  end if;

  update public.dividend_records
    set status = 'paid', paid_at = p_transferred_at, updated_at = now()
    where id in (
      select dividend_record_id from public.transfer_batch_items
      where transfer_batch_id = p_batch_id
    );

  return v_batch;
end;
$$;
```

#### confirm_transfer_by_member（会員専用）

```sql
create or replace function public.confirm_transfer_by_member(p_batch_id uuid)
returns public.transfer_batches
language plpgsql
security definer
set search_path = public
as $$
declare
  v_batch public.transfer_batches;
  v_uid uuid;
begin
  v_uid := auth.uid();
  if v_uid is null then raise exception 'not authenticated'; end if;

  update public.transfer_batches
    set confirmation_status = 'confirmed', confirmed_at = now(),
        confirmed_by = v_uid, updated_at = now()
    where id = p_batch_id and user_id = v_uid
      and status = 'transferred' and confirmation_status = 'unconfirmed'
    returning * into v_batch;

  if v_batch.id is null then
    raise exception 'batch not found, not yours, or not eligible';
  end if;
  return v_batch;
end;
$$;

grant execute on function public.confirm_transfer_by_member(uuid) to authenticated;
```

### 9.5 管理者招待

#### create_admin_invitation / accept_admin_invitation / revoke_admin_invitation / demote_admin / mark_mfa_enrolled

仕様は `yotsuba_admin_invitation_addendum.md` §4 参照。

### 9.6 通知

#### register_push_subscription / mark_notification_as_read / enqueue_notification

仕様は `yotsuba_pwa_addendum.md` §5 参照。

### 9.7 調整明細

#### create_adjustment_dividend / calculate_period_adjustment_diff

仕様は `yotsuba_retroactive_addendum.md` §4 参照。

---

