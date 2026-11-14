# 7. 재시작 후 localMaxTimestamp가 -1이 됨

활성 세그먼트가 빈 상태에서 재시작하면 버킷의 `localMaxTimestamp`가 -1이 된다. 그러면 timestamp로 offset을 찾을 때 항상 최신 offset이 나오고 timestamp부터 읽기 시작한 consumer는 남아 있는 데이터를 전부 건너뛴다.

| 항목 | 내용 |
|---|---|
| 모듈 | fluss-server, `LocalLog` |
| 테스트 | `LogTabletTest#testLocalMaxTimestampIsRestoredWhenActiveSegmentIsEmpty` |
| 열린 PR | 없음 |

## 원인

로그를 로드할 때 `LocalLog`는 `localMaxTimestamp`를 마지막 세그먼트에서만 가져온다. 마지막(활성) 세그먼트가 비어 있으면 `maxTimestampSoFar()`가 -1이다. 이 값은 append할 때만 갱신되기 때문에 다음 쓰기 전까지 -1로 남는다.

빈 활성 세그먼트는 흔하다. 예를 들어 TTL 정리 작업이 쓰기가 없는 버킷의 만료된 활성 세그먼트를 roll한다. 그 뒤에 재시작하면 마지막 세그먼트가 빈 채로 로드된다.

## 영향

`Replica#getOffsetByTimestamp`는 요청한 timestamp가 `localMaxTimestamp`보다 크고 현재 시각 이하이면 최신 offset을 바로 돌려준다. 값이 -1이면 0부터 현재까지의 모든 timestamp가 이 분기를 탄다.

- timestamp 기준 list offsets(예: Flink `timestamp` startup mode)가 high watermark를 돌려준다.
- consumer는 로컬과 원격에 남은 데이터를 모두 건너뛴다.
- timestamp lag 메트릭 값도 틀린다.

## 재현

레코드를 쓰고 `roll()`, flush, close를 한 뒤 `LogTablet`을 다시 로드한다. 데이터는 여전히 찾을 수 있지만(`lookupOffsetForTimestamp(maxTimestamp) == 0`), 값은 -1이다.

```
expected: 1794536215266L
 but was: -1L
```

## 수정

로드할 때 세그먼트를 뒤에서부터 보면서 빈 세그먼트는 건너뛰고, 처음 만나는 비어 있지 않은 세그먼트의 max timestamp를 쓴다. 보통 마지막 한두 개만 보기 때문에 시작할 때 time index를 더 읽지 않는다. 세그먼트가 없으면 0, 전부 비어 있으면 -1로 기존과 같다.

## 남은 문제

로컬 세그먼트가 전부 빈 경우, 즉 데이터가 로컬에서 만료되고 원격에만 있는 경우는 재시작 뒤에도 -1이라서 같은 문제가 생긴다. 고치려면 `Replica#getOffsetByTimestamp`가 원격 로그의 max timestamp를 써야 한다. 별도 변경으로 다루는 편이 낫다.
