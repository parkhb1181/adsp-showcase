# 1. 서버가 없는 앱에서 결제를 어디서 확정하나

## 상황

이 앱은 `next.config.mjs`가 `output: 'export'`인 정적 빌드입니다. Vercel에 올라가는 건 HTML과 JS뿐이고, Supabase anon key는 빌드 시점에 번들에 구워집니다. 서버가 없습니다.

그런데 결제는 시크릿 키가 필요합니다. PG에 "이 결제가 진짜 됐는지" 물어보는 호출, 그리고 사용자에게 권한을 써 주는 일. 둘 다 클라이언트에서 하면 누구나 자기에게 프리미엄을 줄 수 있습니다.

## 판단

시크릿을 쓰는 일은 전부 **Supabase Edge Function 하나**(`confirm-payment`)로 모았습니다. 클라이언트는 결제창을 열고 결과를 이 함수에 넘기는 것까지만 합니다.

함수가 막아야 하는 것 네 가지를 먼저 적고, 그걸 기준으로 코드를 짰습니다.

1. **금액 위변조** — 클라이언트가 보낸 금액을 믿지 않는다. PG 응답의 금액을 `app_config`의 가격과 대조한다
2. **명의 도용** — 결제할 때 실어 보낸 `customData.userId`가 지금 로그인한 사용자와 같은지 본다. 없으면 남의 주문번호를 알아낸 사람이 자기에게 권한을 붙일 수 있다
3. **중복 부여** — 주문번호로 먼저 걸러내고, 그래도 겹치면 `payments.payment_key`의 UNIQUE 제약에 걸린다. 웹훅·재시도·새로고침 모두 안전하다
4. **권한 위조** — `entitlements` 테이블에 클라이언트 쓰기 정책이 없다. 여기서만 service_role로 쓴다

PG를 토스에서 포트원(KG이니시스)으로 바꾸면서 하나 더 알게 됐습니다. **PG마다 이 함수가 하는 일이 다릅니다.**

- 토스: 결제창을 닫아도 아직 돈이 빠지지 않았다. 여기서 `confirm`을 불러야 승인된다. 이 함수가 실패하면 결제도 안 일어난다
- 포트원: 결제창에서 승인까지 끝났다. 여기서는 조회해서 "정말 결제됐는지"만 확인한다. 이 함수가 실패해도 돈은 이미 빠진 뒤라, **실패 기록이 곧 환불·수동처리 대상**이 된다

그래서 실패도 원장에 남깁니다.

## 코드

`supabase/functions/confirm-payment/index.ts` 본문 흐름

```ts
Deno.serve(async (req) => {
  // 1. 사용자 확인 — Authorization 헤더로 auth.getUser()
  // 2. 입력 — provider, orderId, transactionId
  //    provider 가 없으면 토스로 본다. 캐시된 예전 클라이언트가 남아 있을 수 있다.
  const provider = body.provider ?? 'toss';

  // 3. 기대 금액 — app_config.premium_price_krw. 클라이언트 금액은 참고만
  const expected = Number(cfg.value);

  // 4. 이미 처리된 결제인가 — 주문번호로 본다.
  //    두 PG 모두 시도마다 새로 채번하므로 PG 를 부르기 전에 확인할 수 있다.
  const { data: existing } = await admin.from('payments')
    .select('id, status').eq('order_id', orderId).maybeSingle();
  if (existing) return json({ ok: true, alreadyProcessed: true, status: existing.status });

  // 5. PG 별 확정 — 결과를 Settled | SettleFailure 한 모양으로 맞춘다
  const settled = provider === 'portone'
    ? await settlePortOne({ orderId, expected, userId: user.id })
    : await settleToss({ transactionId, orderId, amount: body.amount, expected });

  if (!settled.ok) {
    // 실패도 원장에 남긴다. 포트원 경로에서는 이미 돈이 빠진 뒤일 수 있어
    // 이 기록이 곧 환불·수동처리의 근거가 된다.
    await admin.from('payments').insert({ user_id: user.id, order_id: orderId,
      payment_key: transactionId ?? orderId, amount: body.amount ?? expected,
      status: 'FAILED', raw: settled.raw ?? { error: settled.error } });
    return json({ error: settled.error, code: settled.code }, settled.httpStatus);
  }

  // 6. 원장 기록 — UNIQUE 위반(23505)이면 그 사이 다른 요청이 먼저 기록한 것.
  //    결제는 확정됐고 권한도 곧 부여되므로 성공으로 처리한다.
  if (payErr?.code === '23505') return json({ ok: true, alreadyProcessed: true });

  // 7. 권한 부여 — 재구매는 이어 붙인다. 남은 기간이 있는데 다시 결제한 사람의
  //    잔여일을 없애면 돈을 내고 손해를 보는 셈이 된다.
  const base = prevExpiry > now ? prevExpiry : now;
  const expiresAt = new Date(base + PREMIUM_DURATION_DAYS * 24 * 60 * 60 * 1000);
  await admin.from('entitlements').upsert({ user_id: user.id, product: 'premium',
    payment_id: payment.id, expires_at: expiresAt.toISOString(), revoked_at: null },
    { onConflict: 'user_id,product' });
});
```

토스 쪽 금액 검증. 토스의 `AMOUNT_MISMATCH`는 결제창 금액과 승인 금액이 다를 때만 잡히므로, 처음부터 100원짜리 결제창을 연 경우는 우리가 걸러야 합니다.

```ts
// 클라이언트가 보낸 금액을 그대로 승인하면 1원 결제로 프리미엄을 살 수 있다.
if (amount !== expected) {
  return { ok: false, httpStatus: 400, error: 'amount_mismatch' };
}
res = await fetch(TOSS_CONFIRM_URL, {
  method: 'POST',
  headers: {
    Authorization: `Basic ${btoa(`${TOSS_SECRET_KEY}:`)}`,
    'Content-Type': 'application/json',
    // 네트워크 재시도 시 중복 승인을 막는다.
    'Idempotency-Key': orderId.slice(0, 50),
  },
  body: JSON.stringify({ paymentKey: transactionId, orderId, amount }),
});
```

클라이언트 쪽 PG 선택. 배포를 나누지 않고 환경변수만 바꿔 전환할 수 있게 해뒀습니다.

```ts
// lib/payment/index.ts
export function getPaymentAdapter(): PaymentAdapter | null {
  const preferred = process.env.NEXT_PUBLIC_PAYMENT_PROVIDER;
  if (preferred) {
    const picked = ADAPTERS[preferred];
    // 지정했는데 키가 없으면 조용히 다른 PG 로 넘어가지 않는다. 설정 실수를
    // 결제 실패로 알아채는 것보다 여기서 드러나는 편이 낫다.
    return picked?.isConfigured() ? picked : null;
  }
  return Object.values(ADAPTERS).find((a) => a.isConfigured()) ?? null;
}
```

## 결과

- 테스트 결제로 결제창 → 확정 → 권한 부여 → 만료 계산까지 흐름을 확인했습니다. 실거래 심사 승인을 기다리는 중입니다
- 토스로 시작해 포트원으로 바꾸는 동안 클라이언트 배포 없이 환경변수로 전환했습니다
- 실패한 확정 시도가 원장에 `FAILED`로 남아, 사례 2의 버그를 화면이 아니라 데이터로 잡을 수 있었습니다

## 한계

- 이용 기간 90일이 Edge Function과 `lib/business-info.ts` 두 곳에 적혀 있습니다. Deno 런타임이라 앱 코드를 가져다 쓸 수 없어서인데, 한쪽만 고치면 화면의 기간과 실제 만료일이 갈립니다. 주석으로 묶어두는 것 이상의 방법은 아직 없습니다
- 웹훅은 받지 않습니다. 클라이언트가 결과를 넘기는 경로 하나뿐이라, 결제 직후 브라우저가 닫히면 확정이 안 된 채 돈만 빠질 수 있습니다. 실거래 전에 웹훅을 붙이는 게 다음 순서입니다
