# 2. 정상 결제가 전부 owner_mismatch로 튕겼다

## 상황

포트원(KG이니시스) 연동을 붙이고 테스트 결제를 돌렸습니다. 결제창에서 승인까지 갔는데 권한이 붙지 않았습니다.

화면에는 `owner_mismatch`, 원장(`payments`)에는 `FAILED`가 남았습니다. 그런데 원장에 같이 저장해둔 PG 응답 원본(`raw`)을 열어보니 이상했습니다.

- `status`: `PAID`
- `amount.total`: `2900` — 맞음
- `customData` 안의 `userId`: 로그인한 사용자와 같은 값

전부 맞는데 판정만 틀렸습니다.

## 판단

`owner_mismatch`는 사례 1의 두 번째 방어선입니다. 결제창을 열 때 `customData: JSON.stringify({ userId })`를 실어 보내고, 확정할 때 이걸 풀어서 지금 로그인한 사용자와 비교합니다.

원인은 인코딩이었습니다. **우리가 `JSON.stringify`한 문자열을 포트원이 다시 JSON 문자열로 감싸서 돌려줍니다.** 한 번만 `JSON.parse`하면 객체가 아니라 문자열이 나오고, `.userId`는 `undefined`가 됩니다. `undefined !== userId`라서 전부 튕긴 겁니다.

두 가지 선택지가 있었습니다.

1. `JSON.parse`를 두 번 한다 — 지금은 맞지만 포트원이 나중에 한 겹만 주도록 바꾸면 반대로 깨진다
2. **객체가 될 때까지 푼다** — 몇 겹이 오든 동작한다. 상한을 둬서 무한 루프는 막는다

2번으로 갔습니다. 외부 API의 응답 형식은 우리가 통제할 수 없으니, 형식이 바뀌어도 깨지지 않는 쪽이 맞다고 봤습니다.

## 코드

`supabase/functions/confirm-payment/index.ts` — `settlePortOne` 안

```ts
// before
const parsed = JSON.parse(String(payment.customData ?? '{}'));
customUserId = parsed?.userId;
```

```ts
// after
// 결제창을 열 때 실어 보낸 값.
//
// 두 번 감싸여 돌아온다. 우리가 JSON.stringify 한 문자열을 포트원이 다시 JSON
// 문자열로 감싸서 주기 때문에, 한 번만 풀면 객체가 아니라 문자열이 나오고
// .userId 가 undefined 가 된다. 실제로 이것 때문에 정상 결제가 전부
// owner_mismatch 로 튕겼다.
//
// 포트원이 나중에 한 겹만 주도록 바꿔도 깨지지 않게, 객체가 될 때까지 푼다.
let customUserId: string | undefined;
try {
  let parsed: unknown = payment.customData ?? '{}';
  for (let i = 0; i < 3 && typeof parsed === 'string'; i++) {
    parsed = JSON.parse(parsed);
  }
  if (parsed && typeof parsed === 'object') {
    customUserId = (parsed as { userId?: string }).userId;
  }
} catch {
  customUserId = undefined;
}
if (customUserId !== userId) {
  console.warn(`결제 소유자 불일치 orderId=${orderId} custom=${customUserId} user=${userId}`);
  return { ok: false, httpStatus: 403, error: 'owner_mismatch', raw: payment };
}
```

보내는 쪽은 그대로 뒀습니다. 이 값이 없으면 남의 주문번호를 알아낸 사람이 자기에게 권한을 붙일 수 있어서, 빼는 선택지는 없었습니다.

```ts
// lib/payment/portone.ts
// 서버가 "이 결제가 정말 이 사람 것인가" 를 확인할 근거. 이게 없으면 남의
// 주문번호를 알아낸 사람이 자기에게 권한을 붙일 수 있다.
customData: JSON.stringify({ userId }),
```

## 결과

- 같은 테스트 결제를 다시 돌려 `PAID` → 원장 기록 → 권한 부여까지 통과했습니다
- 원인을 찾는 데 걸린 시간의 대부분은 "코드가 아니라 데이터를 보는 것"이었습니다. 실패한 확정 시도도 `raw`째로 원장에 남기기로 한 결정(사례 1)이 없었으면, 화면의 `owner_mismatch` 한 줄만 보고 우리 쪽 userId 전달을 의심하며 헤맸을 겁니다

## 한계

- 상한 3겹은 임의값입니다. 포트원이 세 겹 이상으로 감쌀 이유는 없다고 봤지만, 근거는 "그럴 리 없다"뿐입니다
- 이 버그는 테스트 결제에서 잡혔습니다. 실거래에서 먼저 터졌으면 돈은 빠지고 권한은 안 붙은 사용자가 생겼을 텐데, 그때 대응할 환불 절차는 아직 문서로만 있습니다
