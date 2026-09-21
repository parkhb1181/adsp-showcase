# 4. 두 기기의 진도를 합치면 숫자가 부풀었다

## 상황

진도(푼 문제, 북마크, 버린 문제, 마지막 세션)는 원래 `localStorage`에만 있었습니다. 폰에서 풀다가 PC에서 열면 처음부터였습니다.

Supabase Auth를 붙이면서 진도를 계정에 묶었습니다. 그러면 두 기기의 진도를 합쳐야 합니다. 폰에서 30문제, PC에서 20문제를 풀었으면 어느 쪽도 버리면 안 됩니다.

## 판단

병합 규칙을 필드마다 따로 정했습니다. 기준은 하나입니다. **같은 데이터를 몇 번 병합해도 결과가 같아야 한다(멱등).** 동기화는 여러 번 일어나고, 같은 기기가 자기 자신과 합쳐지는 경우도 있습니다.

| 필드 | 규칙 | 이유 |
|---|---|---|
| `answers` | questionId별로 `answeredAt`이 늦은 쪽. `attemptCount`는 **max** | 합으로 두면 동기화할 때마다 시도 횟수가 부풀어 오른다 |
| `bookmarks` · `userDiscarded` | 합집합 | 어느 기기에서 표시했든 남긴다 |
| `freeUsage.solvedCount` | max | 같은 이유 |
| `lastSession` | `startedAt`이 늦은 쪽 | |
| `createdAt` | 이른 쪽 | 최초 사용 시점 보존 |

`attemptCount`가 제일 헷갈리는 자리였습니다. "폰에서 3번, PC에서 2번 시도했으면 5번"이 직관적인데, 그러면 같은 진도를 다시 병합할 때 5+5가 됩니다. 병합은 몇 번 일어날지 모르니 합은 쓸 수 없고, max로 두면 "적어도 이만큼은 시도했다"는 값이 흔들리지 않습니다. 정확한 총합을 잃는 대신 부풀지 않는 쪽을 골랐습니다.

## 코드

`lib/sync.ts`

```ts
// 잘못된 날짜 문자열이 섞여도 비교가 깨지지 않도록 0으로 떨어뜨린다
function ts(value: string | null | undefined): number {
  if (!value) return 0;
  const parsed = Date.parse(value);
  return Number.isNaN(parsed) ? 0 : parsed;
}

export function mergeProgress(local: UserProgress, remote: UserProgress): UserProgress {
  const byId = new Map<string, AnswerRecord>();
  for (const a of remote.answers) byId.set(a.questionId, a);

  for (const a of local.answers) {
    const other = byId.get(a.questionId);
    if (!other) { byId.set(a.questionId, a); continue; }
    const winner = ts(a.answeredAt) >= ts(other.answeredAt) ? a : other;
    byId.set(a.questionId, {
      ...winner,
      attemptCount: Math.max(a.attemptCount, other.attemptCount),
    });
  }

  return {
    version: local.version,
    answers: Array.from(byId.values()),
    bookmarks: union(local.bookmarks, remote.bookmarks),
    userDiscarded: union(local.userDiscarded, remote.userDiscarded),
    freeUsage: {
      solvedCount: Math.max(local.freeUsage.solvedCount, remote.freeUsage.solvedCount),
      lastSolvedAt: ts(local.freeUsage.lastSolvedAt) >= ts(remote.freeUsage.lastSolvedAt)
        ? local.freeUsage.lastSolvedAt : remote.freeUsage.lastSolvedAt,
    },
    lastSession: laterSession(local.lastSession, remote.lastSession),
    createdAt: ts(local.createdAt) <= ts(remote.createdAt) && ts(local.createdAt) > 0
      ? local.createdAt : remote.createdAt || local.createdAt,
    updatedAt: new Date().toISOString(),
  };
}

// DB에서 읽은 jsonb는 무엇이든 들어올 수 있으므로 최소한의 형태만 확인한다
export function isUserProgress(value: unknown): value is UserProgress {
  if (typeof value !== 'object' || value === null) return false;
  const v = value as Record<string, unknown>;
  return Array.isArray(v.answers) && Array.isArray(v.bookmarks)
    && Array.isArray(v.userDiscarded) && typeof v.freeUsage === 'object' && v.freeUsage !== null;
}
```

## 결과

- 로그인하면 로컬 진도와 서버 진도를 병합해 양쪽에 다시 씁니다. 어느 기기에서 열어도 같은 진도가 보입니다
- 같은 진도를 반복 병합해도 `attemptCount`와 `solvedCount`가 변하지 않습니다

## 한계: 알고 있는 것

**합집합이라 "해제"가 다른 기기로 전파되지 않습니다.** A 기기에서 북마크를 풀어도 B 기기가 다음 동기화에서 되살립니다. 북마크와 버린 문제 둘 다 그렇습니다.

제대로 고치려면 `{ id, active, updatedAt }` 형태의 툼스톤이 필요합니다. 지운 사실을 지우지 말고 "지웠다"는 기록으로 남겨서, 늦은 쪽이 이기게 하는 방식입니다. 지금 스키마는 id 배열이라 그 정보가 없습니다.

사용자 규모에서 아직 이 문제가 접수된 적이 없어서 뒤로 미뤘습니다. 코드에는 한계와 고칠 방향을 적어뒀습니다. 모르고 두는 것과 알고 미루는 건 다르다고 봤습니다.

```ts
/**
 * 알려진 한계: bookmarks/userDiscarded가 합집합이라 "해제"가 다른 기기로 전파되지
 * 않는다. A기기에서 북마크를 풀어도 B기기가 다음 동기화에서 되살린다.
 * 제대로 고치려면 {id, active, updatedAt} 형태의 툼스톤 스키마가 필요하다.
 */
```

그리고 `freeUsage.solvedCount`는 이제 로그인 사용자에게는 쓰지 않습니다. 클라이언트가 값을 정하는 구조라 캐시를 지우면 초기화됐고, 사례 3에서 서버 카운터(`usage_counters`)로 옮겼습니다. 이 필드는 비로그인 사용자 몫으로만 남아 있습니다.
