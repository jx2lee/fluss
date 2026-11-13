# 5. table.statistics.columns 값이 DDL은 통과하고 append는 전부 실패함

`table.statistics.columns`를 `' *'`나 `'id,id'`로 지정하면 테이블은 만들어지지만, 그 뒤로는 어떤 클라이언트도 append할 수 없다.

| 항목 | 내용 |
|---|---|
| 모듈 | fluss-common, `TableConfig` |
| 테스트 | `StatisticsConfigUtilsTest`의 새 테스트 2개 |
| 열린 PR | 없음 |

## 원인

테이블 생성 때의 검증은 `StatisticsConfigUtils#validateStatisticsConfig`가 하고, writer의 파싱은 `TableConfig#getStatisticsColumns`가 한다. 둘의 해석이 다르다.

1. **앞뒤에 공백이 붙은 와일드카드(`' *'`, `'* '`)**: 검증은 trim한 뒤 `*`와 비교한다. `TableConfig`는 trim하지 않으므로 컬럼 목록으로 보고 `SPECIFIED ["*"]`를 돌려준다. 클라이언트(`RecordAccumulator`)가 배치용 통계 collector를 만들 때 `TableInfo#getStatsIndexMapping`이 실패한다.
   ```
   java.lang.IllegalArgumentException: Statistics column '*' not found in table schema
   ```
2. **중복된 컬럼(`'id,id'`, `'id, name, id'`)**: 검증은 컬럼이 있는지와 타입만 확인한다. `TableConfig`는 중복을 그대로 두고, `LogRecordBatchStatisticsWriter`가 이 목록으로 통계 row type을 만들다가 실패한다.
   ```
   java.lang.IllegalArgumentException: Field names must be unique. Found duplicates: [id]
   ```

## 재현

- `#testWildcardWithSurroundingWhitespaceIsTreatedAsAllColumns`(`" *"`, `"* "`, `" * "`): 검증은 통과하지만 모드가 `ALL`이 아니라 `SPECIFIED`로 나온다.
- `#testDuplicateColumnsAreDeduplicated`(`"id, name, id"`): 통계 collector 생성이 실패한다.

## 수정

검증과 writer가 함께 쓰는 `TableConfig#getStatisticsColumns`에서 정규화한다. `*`와 비교하기 전에 trim하고, 중복 컬럼은 뺀다.

검증만 엄격하게 하지 않고 파서를 고쳤으므로, 이런 값으로 이미 만들어진 테이블에도 다시 쓸 수 있다.
