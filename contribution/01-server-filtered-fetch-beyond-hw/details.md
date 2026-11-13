# 1. 필터를 건 로그 fetch가 high watermark를 넘어 읽음

필터(`FilterContext`)를 건 fetch에서 high watermark가 있는 세그먼트의 배치가 전부 걸러지면, `LocalLog#read`가 다음 세그먼트까지 읽는다. 커밋되지 않은 배치가 그대로 클라이언트로 넘어간다.

| 항목 | 내용 |
|---|---|
| 모듈 | fluss-server, `LocalLog` |
| 테스트 | `LocalLogTest#testReadWithFilterDoesNotReadBeyondMaxOffsetInLaterSegment` |
| 열린 PR | 없음. #4965가 같은 루프를 고치지만 이 버그는 그대로 남음 |

## 언제 생기나

필터를 걸어 fetch하는 클라이언트에서 생긴다. 예를 들어 `table.statistics.columns`가 설정된 로그 테이블에 Flink 필터 pushdown을 쓰는 경우다. 리더가 새 세그먼트를 roll했는데 팔로워가 아직 따라오지 못해 high watermark가 이전 세그먼트에 있는 상태는 흔하다.

## 원인

- 읽기 상한 `maxPosition`이 max offset 위치로 잡히는 건 max offset이 든 세그먼트뿐이다. 나머지 세그먼트는 끝까지 읽는다.
- 배치가 전부 걸러진 세그먼트를 지나면 루프는 `readOffset = filteredEndOffset`으로 두고 다음 세그먼트로 넘어간다. 이때 `readOffset`을 `maxOffsetMetadata`와 다시 비교하지 않는다.
- 필터 없는 경로는 `LogSegment#read`가 `null`을 돌려줄 때만 다음 세그먼트로 넘어가므로 영향이 없다.

## 영향

- **커밋되지 않은 행이 반환된다.** 리더가 바뀌면 잘릴 수 있는 행이라, consumer는 이미 사라진 행을 본 셈이 된다.
- **행이 영영 누락될 수 있다.** high watermark부터 그 세그먼트 끝까지의 배치는 검사하지 않았는데, 다음 fetch offset은 그 뒤로 넘어간다. 그 배치가 커밋된 뒤 필터에 맞더라도 consumer는 읽지 못한다.

## 재현

1. 세그먼트 1: 배치 `[0, 2]`(a = 1, 2, 3), `[3, 5]`(a = 6, 7, 8)
2. roll 후 세그먼트 2: 배치 `[6, 8]`(a = 7, 8, 9)
3. high watermark = 3, 필터 `a > 5`로 offset 0부터 읽기

빈 결과와 `filteredEndOffset = 3`이 나와야 하는데, 세그먼트 2의 `[6, 8]`이 반환된다.

```
[batch [6, 8] read beyond high watermark 3]
Expecting actual:
  8L
to be less than:
  3L
```

## 수정

걸러진 세그먼트를 지난 뒤 `readOffset >= maxOffsetMetadata.getMessageOffset()`이면 루프를 멈춘다. high watermark 세그먼트의 `filteredEndOffset`은 `maxPosition`으로 제한되므로, 응답은 high watermark 이하의 offset을 돌려주고 클라이언트는 거기서부터 이어 읽는다.
