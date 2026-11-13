# 4. ALTER TABLE ADD로 계산 컬럼을 추가하면 물리 컬럼이 됨

Flink에서 `ALTER TABLE t ADD c AS a + 1`을 실행하면 성공하고, `c`가 Fluss에 물리 컬럼(nullable `INT`)으로 추가된다. 식은 사라진다.

| 항목 | 내용 |
|---|---|
| 모듈 | fluss-flink-common, `FlinkConversions` |
| 테스트 | `FlinkConversionsTest#testAddNonPhysicalColumnIsNotConvertedToPhysicalColumn`(단위), `FlinkCatalogITCase#testAlterTableAddNonPhysicalColumn`(Flink 1.20 end-to-end) |
| 열린 PR | 없음 |

## 원인

- `FlinkConversions#toFlussTableChanges`는 Flink의 `TableChange.AddColumn`을 전부 Fluss의 `TableChange.addColumn`으로 변환하고 물리 컬럼인지는 확인하지 않는다.
- `FlinkCatalog#alterTable`은 모든 변경을 이 메서드에 넘긴다.
- `ALTER TABLE t ADD m TIMESTAMP_LTZ(3) METADATA FROM 'timestamp'`도 같은 경로를 탄다. 반면 `CREATE TABLE`은 metadata 컬럼을 명시적으로 거부한다.

## 영향

- 기존 행은 `c`를 `NULL`로 읽고, writer는 이제 `c`를 채워야 한다.
- Fluss는 컬럼 삭제를 지원하지 않기 때문에(`SchemaUpdate#dropColumn`이 예외를 던짐) **테이블을 되돌릴 수 없다.**
- 결과 타입이 `NOT NULL`인 계산 컬럼은 서버가 거부한다. 추가하는 컬럼은 nullable이어야 하기 때문이다. nullable인 식은 모두 통과한다.

## 재현

```sql
CREATE TABLE t (id BIGINT, a INT, PRIMARY KEY (id) NOT ENFORCED);
ALTER TABLE t ADD c AS a + 1;   -- 성공함
```

카탈로그에서 테이블을 다시 읽으면 컬럼이 `[id, a, c]`이고 `c`는 물리 컬럼이다. 원래는 ALTER가 실패하거나 `c`가 계산 컬럼으로 남아야 한다. IT는 `Flink120CatalogITCase`로 실행했다.

## 수정

`AddColumn` 변환에서 물리 컬럼이 아니면 `UnsupportedOperationException`을 던진다. 지원하지 않는 다른 테이블 변경과 같은 방식이다. 수정 후에는 두 ALTER 모두 실패하고 스키마는 `[id, a]`로 남는다.

`ALTER TABLE ADD <계산 컬럼>`을 제대로 지원하려면 `CREATE TABLE`처럼 `schema.N.expr`에 저장하면 된다. 별도 기능으로 다루는 편이 낫다.
