# 2. MultiTableWriter가 Hudi PK 테이블을 PK 인코딩으로 라우팅함

기본 bucket key를 쓰는 PK 테이블에서 `MultiTableWriterImpl`은 PK 인코더를 bucket key 인코더로 그대로 재사용한다. Hudi 테이블에서는 이게 틀리다.

| 항목 | 내용 |
|---|---|
| 모듈 | fluss-client, `MultiTableWriterImpl` |
| 테스트 | `MultiTableWriterLakeFormatITCase#testUpsertPkTableWithDefaultBucketKey`(클러스터 IT) |
| 열린 PR | 없음 |

## 원인

- Hudi의 PK 인코더는 `CompactedKeyEncoder`다. `HudiKeyEncoder`는 정보가 손실되기 때문이다.
- 반면 버킷 계산은 4바이트짜리 `HudiKeyEncoder` 해시를 써야 한다. 이 경우를 위해 `KeyEncoder#ofBucketKeyEncoder(..., isDefaultBucketKey, primaryKeyEncoder)`가 있고, `UpsertWriterImpl`과 `PrimaryKeyLookuper`는 `BucketCalculator`를 거쳐 이걸 쓴다.
- `MultiTableWriterImpl`은 Hudi 예외 처리가 생긴 뒤에 추가됐는데, 예전의 "기본 bucket key면 재사용" 로직을 그대로 가져왔다.

## 영향

`conn.getMultiTable().newMultiTableWrite().createWriter()`로 Hudi PK 테이블에 쓸 때 생긴다.

- **대부분의 키에서 `write()`가 예외로 실패한다.** `HudiBucketingFunction`은 정확히 4바이트를 요구한다.
- **compacted 인코딩이 우연히 4바이트인 키는 엉뚱한 버킷에 들어간다.** 쓰기는 성공하지만 `UpsertWriter`와 `Lookuper`가 계산하는 버킷과 다르다. lookup이 그 행을 찾지 못하고, 같은 키가 두 버킷에 생길 수 있다. 이 부분은 코드를 읽고 내린 결론이다. 테스트로 확인한 건 예외가 나는 경우뿐이다.

## 재현

PK 테이블 `(id INT, name STRING, PRIMARY KEY id)`, `distributedBy(3)`에 `MultiTableWriter`로 20행을 upsert한 뒤, 일반 `Lookuper`로 하나씩 조회한다. 테스트는 `DataLakeFormat` 전체로 파라미터화했다. paimon, lance, iceberg는 통과하고 hudi만 실패한다.

```
java.lang.IllegalArgumentException: bucketKey must be exactly 4 bytes for Hudi bucketing, but got 1 bytes.
```

기존 `ClientToServerITCaseBase`의 클러스터는 `datalake.format = paimon`으로 설정되어 있어 다른 포맷의 테이블을 거부한다. 그래서 별도 클래스로 만들었다.

## 수정

`BucketCalculator`처럼 `KeyEncoder.ofBucketKeyEncoder(rowType, bucketKeys, tableConfig, isDefaultBucketKey, primaryKeyEncoder)`로 bucket key 인코더를 만든다. Hudi가 아닌 포맷은 지금처럼 PK 인코더와 이미 인코딩된 바이트를 재사용한다.
