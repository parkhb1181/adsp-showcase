# 3. 클라이언트를 못 믿는 구조에서 권한과 한도를 어떻게 세나

## 상황

무료 사용자는 하루에 풀 수 있는 문항 수에 한도가 있고, 프리미엄은 한도가 없습니다. 이 둘을 판정하려면 세 가지가 필요합니다.

- 이 사람이 프리미엄인가 (그리고 아직 만료 전인가)
- 오늘 몇 문제 풀었나
- 오늘이 언제인가

처음엔 전부 클라이언트에 있었습니다. `localStorage`에 카운트를 세고, 날짜도 기기 시계로 봤습니다. 캐시를 지우면 초기화되고, 시계를 되돌리면 만료된 권한이 살아나는 구조였습니다.

정적 빌드라 anon key가 번들에 들어가니, 테이블 쓰기를 열어두면 누구나 자기에게 프리미엄을 `INSERT`할 수 있습니다.

## 판단

**클라이언트는 읽기만 한다.** 판정은 전부 서버 함수가 하고, 쓰기는 사례 1의 Edge Function만 합니다.

| 무엇 | 어디서 | 왜 |
|---|---|---|
| 프리미엄 여부 | `is_premium()` RPC | 만료 비교를 클라이언트에서 하면 기기 시계를 되돌리는 것만으로 끝난 권한이 되살아난다 |
| 오늘 날짜 | `kst_today()` | 클라이언트 시간을 쓰면 한도를 무한히 리셋할 수 있다. UTC로 두면 한국 오전 9시에 초기화돼 사용자가 혼란스럽다 |
| 사용량 증가 | `increment_usage()` RPC, upsert 한 문장 | `select` 후 `update`로 나누면 두 탭에서 동시에 풀 때 어긋난다. `security definer`지만 인자로 user_id를 받지 않고 `auth.uid()`를 쓴다 — 받으면 남의 카운터를 올릴 수 있다 |
| `entitlements` · `payments` 쓰기 | 정책 없음 (service_role 전용) | RLS를 열어두면 누구나 자기에게 권한을 준다 |

비로그인 사용자는 이 구조를 안 탑니다. 계정이 없으면 카운트를 귀속시킬 곳이 없어서, 종전대로 로컬에서 셉니다. 기기 시계를 신뢰할 수밖에 없는데, 그건 비로그인 한도(15문제)가 감수하는 비용입니다.

## 코드

`docs/supabase-payments.sql`

```sql
-- 무료 한도가 일일제라 "오늘"의 기준이 필요하다. 클라이언트 시간을 쓰면
-- 기기 시계를 바꿔 한도를 무한히 리셋할 수 있으므로 반드시 서버에서 구한다.
-- UTC 로 두면 한국 시간 오전 9시에 한도가 초기화돼 사용자가 혼란스러워한다.
create or replace function public.kst_today()
returns date language sql stable
as $$ select (now() at time zone 'Asia/Seoul')::date; $$;

-- 만료 판정을 서버가 한다. 클라이언트에서 expires_at 을 받아 비교하면 기기
-- 시계를 되돌리는 것만으로 만료된 권한이 되살아난다.
create or replace function public.is_premium()
returns boolean language sql security definer set search_path = public stable
as $$
  select exists (
    select 1 from public.entitlements
    where user_id = auth.uid()
      and product = 'premium'
      and revoked_at is null
      and (expires_at is null or expires_at > now())
  );
$$;

-- 본인 결제 내역만 조회 가능. 쓰기 정책은 만들지 않는다(= service_role 전용).
create policy "payments: read own"
  on public.payments for select to authenticated
  using (user_id = auth.uid());

-- upsert 한 문장으로 처리해야 원자적이다. select 후 update 로 나누면 같은
-- 사용자가 두 탭에서 동시에 풀 때 카운트가 어긋난다.
-- security definer 이지만 p_user_id 를 인자로 받지 않고 auth.uid() 를 쓴다.
-- 인자로 받으면 남의 카운터를 올릴 수 있는 구멍이 된다.
create or replace function public.increment_usage()
returns table (usage_date date, solved_count integer)
language plpgsql security definer set search_path = public
as $$
declare v_user uuid := auth.uid();
begin
  if v_user is null then raise exception 'not authenticated'; end if;
  return query
  insert into public.usage_counters as u (user_id, usage_date, solved_count)
  values (v_user, public.kst_today(), 1)
  on conflict (user_id, usage_date) do update
    set solved_count = u.solved_count + 1, updated_at = now()
  returning u.usage_date, u.solved_count;
end; $$;
```

`lib/entitlement.ts` — 실패했을 때 어느 쪽으로 넘어지는지가 중요합니다.

```ts
export async function fetchIsPremium(): Promise<boolean> {
  const { data, error } = await supabase.rpc('is_premium');
  if (error) {
    // 조회 실패 시 프리미엄이 아닌 것으로 본다. 반대로 두면 오류가 곧 무료 개방이다.
    return false;
  }
  return data === true;
}

/**
 * 사용량 1 증가. 문항을 "푼" 시점(정답 제출)에 부른다.
 * 실패해도 학습 흐름은 끊지 않는다 — 카운트가 한 번 덜 오르는 편이,
 * 네트워크 오류로 문제를 못 푸는 것보다 낫다.
 */
export async function incrementUsage(): Promise<number | null> { /* rpc('increment_usage') */ }
```

`lib/hooks/useDailyUsage.ts` — 화면은 낙관적으로 먼저 올리고 서버 값으로 교정합니다.

```ts
const registerSolved = useCallback(() => {
  // 화면은 즉시 반영한다.
  setSolvedToday((n) => n + 1);
  if (!isLoggedIn) { incrementFreeUsage(); return; }
  incrementUsage().then((serverCount) => {
    // 서버가 준 값이 진실이다. 다른 탭에서 푼 것까지 반영된다.
    if (serverCount !== null && aliveRef.current) setSolvedToday(serverCount);
  });
}, [isLoggedIn]);
```

## 한도 값을 어디에 둘지 — 한 번 틀렸던 것

처음엔 한도를 서버 `app_config`에도 두고 코드 상수와 둘 중 작은 값을 썼습니다. 재배포 없이 조절하려는 의도였습니다.

그런데 서버에 남은 옛날 값이 코드를 이겼습니다. 상수를 20에서 50으로 올렸는데 한도가 그대로였고, 원인을 찾는 데 시간이 갔습니다. "둘 중 작은 값"이라는 규칙이 문제였습니다. 조절하려고 만든 장치가 조절을 막았습니다.

지금은 **한도는 코드 상수만 봅니다.** 가격만 서버에 남겼습니다.

```ts
// lib/hooks/useDailyUsage.ts
// 한도 값은 코드 상수만 본다. 예전에는 서버 app_config 도 같이 읽어 둘 중
// 작은 값을 썼는데, 재배포 없이 조절하려던 의도와 달리 서버에 남은 옛날
// 값이 코드를 이겨서 상수를 올려도 한도가 그대로인 사고가 났다.
```

50으로 올린 근거도 코드에 적어뒀습니다. 지금 단계에서는 전환율보다 유입이 먼저고, 20은 하루치 학습으로도 빠듯해 결제가 아니라 이탈을 부른다고 봤습니다. 50이면 하루 20~30문제 푸는 사람은 안 걸리고, 시험이 임박해 몰아서 푸는 사람만 닿는데 그 구간이 결제 의사가 가장 높은 자리이기도 합니다.

그리고 **결제 수단이 없는 동안에는 로그인 사용자를 아예 막지 않습니다.** 살 방법이 없는데 벽만 세우면 막다른 길이라서요. PG 키가 설정되는 순간 회원 한도가 저절로 켜집니다.

```ts
const paymentReady = isPaymentAvailable();
const limit = isPremium
  ? Number.POSITIVE_INFINITY
  : isLoggedIn
    ? paymentReady ? FALLBACK_LIMITS.member : Number.POSITIVE_INFINITY
    : FREE_DAILY_LIMIT_GUEST;
```

## 결과

- 비로그인 15 / 로그인(결제 전) 무제한 / 로그인(결제 후) 50 / 프리미엄 무제한. 열한 가지 조합(로그인 여부 × 결제 가능 여부 × 프리미엄 여부 × 한도 도달)을 확인했습니다
- 기기 시계를 바꿔도 한도와 만료가 움직이지 않습니다

## 한계

- 비로그인 카운트는 여전히 로컬입니다. 캐시를 지우면 15문제가 다시 열립니다. 계정 없이 막을 방법은 없다고 보고 두었습니다
- 가격은 아직 `app_config`에 있습니다. 한도에서 겪은 같은 문제가 가격에서 생길 수 있는데, 가격은 Edge Function도 같은 곳을 읽어야 해서 코드 상수로 못 옮겼습니다
