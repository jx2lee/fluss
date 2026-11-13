# 3. lookup join에서 BYTES 키가 매칭되지 않음

lookup join 키 중 일부는 Fluss 조회 뒤에 클라이언트에서 거르는데, 여기서 `byte[]`를 참조로 비교한다. 그래서 BINARY/VARBINARY/BYTES 키는 한 번도 매칭되지 않는다.

| 항목 | 내용 |
|---|---|
| 모듈 | fluss-flink-common, `LookupNormalizer.RemainingFilter` |
| 테스트 | `LookupNormalizerTest#testRemainingFilterComparesBinaryByContent` |
| 열린 PR | 없음 |

## 언제 생기나

join 키를 전부 Fluss에 보낼 수 없을 때, 나머지 키는 조회 뒤에 `RemainingFilter`가 거른다. 문서에도 "the other join keys are filtered after the lookup"이라고 적혀 있다.

예를 들어 `dim (id BIGINT, payload BYTES, PRIMARY KEY (id))`에 `ON d.id = s.id AND d.payload = s.payload`로 join하면, Fluss는 `id`로만 조회하고 `payload`는 조회 뒤에 거른다. prefix lookup에서 prefix 뒤의 키가 `BYTES`인 경우도 마찬가지다.

## 원인

비교에 `Objects.equals`를 쓴다. Flink 내부값에서 BYTES 계열은 `byte[]`이므로 참조 비교가 된다. probe 쪽 키 행의 배열과 Fluss 행에서 디코딩한 배열은 항상 다른 객체다.

## 영향

Flink는 lookup key로 소스에 넘긴 등호 조건을 다시 검사하지 않는다.

- inner join은 결과가 0건이다.
- left join은 probe 행이 null로 채워져 나온다.

`FlinkLookupFunction`, `FlinkAsyncLookupFunction`, `FlinkFullCacheLookupFunction`이 모두 이 필터를 쓴다. "Flink가 다시 검사하지 않는다"는 부분은 코드를 읽고 내린 결론이고, 테스트로 확인한 건 필터 자체다.

## 재현

lookup key `(id, payload)`, PK와 bucket key는 `id`로 둔다. `createRemainingFilter(GenericRowData.of(1L, new byte[] {1, 2}))`로 필터를 만든 뒤 `isMatch(GenericRowData.of(1L, new byte[] {1, 2}))`를 호출하면 `false`가 나온다.

```
Expecting value to be true but was false
```

## 수정

`Objects.deepEquals`로 비교한다. `byte[]`는 내용으로 비교하고, 다른 내부 타입(`StringData`, `DecimalData`, `TimestampData`, boxed primitive)에서는 `Objects.equals`와 같다.
