# 6. CHAR 컬럼에 listagg/string_agg를 지정하면 쓰기가 전부 실패함

aggregation merge engine 테이블에서 CHAR 컬럼에 `listagg`나 `string_agg`를 지정하면 테이블은 만들어지지만, 그 뒤의 모든 쓰기가 실패한다.

| 항목 | 내용 |
|---|---|
| 모듈 | fluss-common `AggFunctionType`, 문서 `aggregation.md` |
| 테스트 | `TableDescriptorTest#testValidateAggFunctionWithDataType`, `FieldAggregatorFactoryTest#testCreateAggregatorForEverySupportedDataType`(fluss-server, 새로 추가) |
| 열린 PR | 없음 |

## 원인

- 테이블 생성 시 검증(`AggFunctionType#getSupportedDataTypeRoots`)은 `listagg`와 `string_agg`에 `STRING`과 `CHAR`를 허용한다. 문서에도 `CHAR`가 적혀 있다.
- 그런데 서버 쪽 factory(`FieldListaggAggFactory`, `FieldStringAggFactory`)는 `StringType`만 받는다.
- aggregator는 첫 쓰기 때 만들어지므로 실패도 그때 난다.

## 재현

```sql
CREATE TABLE t (id INT, v CHAR(10), PRIMARY KEY (id) NOT ENFORCED)
WITH ('table.merge-engine' = 'aggregation', 'fields.v.agg' = 'listagg');
```

생성은 성공하고, 이후 upsert할 때마다 아래 오류가 난다.

```
java.lang.IllegalArgumentException: Data type for listagg column must be 'StringType' but was 'CHAR(10)'.
```

모든 aggregate 함수와 검증이 허용하는 타입의 조합으로 aggregator를 만들어 봤고, 어긋나는 건 이 두 개뿐이었다.

## 수정

DDL 단계에서 거부한다. `listagg`/`string_agg`의 허용 타입과 문서(`aggregation.md` 753행, 815행)에서 `CHAR`를 뺀다.

- `CHAR(n)`은 고정 길이라서 여러 값을 이어 붙인 결과를 담을 수 없다. `INDEXED` KV 포맷은 `CHAR(n)`을 정확히 n바이트로 쓴다.
- 이 코드의 원본인 Paimon도 `listagg`에 `VARCHAR`만 받는다.
- 이 조합으로 만든 테이블은 지금도 쓰기가 되지 않으므로, 현재 동작하는 것 중 깨지는 건 없다.

다른 방법은 factory가 `CharType`도 받게 하는 것이다. 하지만 그러면 `CHAR(n)` 컬럼에 n보다 긴 값이 저장된다.

새로 추가한 factory 테스트는 검증이 허용하는 모든 함수·타입 조합으로 aggregator 생성이 성공하는지 확인한다. 앞으로 검증과 factory가 어긋나면 이 테스트에서 잡힌다.
