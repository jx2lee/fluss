# Fluss 기여 초안

**먼저 할 일:** [1번 details.md](01-flink-lookup-join-binary-key/details.md)를 읽고 아래 「올리는 방법」대로 등록한다. 번호는 쉬운 순서다.

`apache/fluss` main `67bc81c1ee4fe98c86da558a64236a6fda694528`(2026-11-12)에서 찾은 버그 8개다.

- 8개 모두 main에서 실패하는 테스트로 재현했고, 수정 후에는 통과한다(red → green).
- 열린 PR 450개의 변경 파일을 모두 확인했고, 같은 파일을 건드리는 PR은 diff까지 읽었다. 이 버그를 고치는 PR은 없다.
- 열린·닫힌 이슈와 PR을 키워드로 검색했고 기존 보고는 없었다.

## 폴더 구성

- `details.md`: 버그 설명(원인·영향·재현·수정), 한국어
- `issue.md`: 이슈 제목과 본문(`.github/ISSUE_TEMPLATE/bug.yml` 형식), 영어
- `pr.md`: PR 제목과 본문(`.github/PULL_REQUEST_TEMPLATE.md` 형식), 영어
- `fix.patch`: 수정과 테스트. 위 커밋에 `git apply`로 단독으로도, 다른 패치와 함께도 적용된다

`issue.md`와 `pr.md`는 apache/fluss에 올릴 글이라 영어로 두었다.

이 브랜치(`search-known-issues`)에서 빌드하면 RAT가 라이선스 헤더 없는 `fix.patch`를 잡는다. `-Drat.skip=true`를 붙인다.

## 난이도 순 목록

| # | 난이도 | 버그 | 영향 | 난이도 근거 |
|---|---|---|---|---|
| [1](01-flink-lookup-join-binary-key/details.md) | 쉬움 | lookup join이 조회 뒤 `byte[]` 키를 참조로 비교 | BYTES 키 inner join 결과 0건 | 수정 1줄, 단위 테스트 1개 |
| [2](02-common-startswith-stats-order/details.md) | 쉬움 | `LIKE 'x%'` 통계 검사는 UTF-16 순서, collector는 UTF-8 순서 | 행이 조용히 누락(전각·이모지 혼합) | 수정 5줄, 단위 테스트 1개 |
| [3](03-common-statistics-columns-normalization/details.md) | 쉬움 | `table.statistics.columns=' *'`/`'id,id'`가 DDL을 통과한 뒤 append 전부 실패 | 테이블은 생기는데 쓸 수 없음 | 수정 4줄, 단위 테스트 2개 |
| [4](04-flink-alter-add-non-physical-column/details.md) | 중간 | `ALTER TABLE ADD c AS a + 1`이 물리 컬럼을 추가 | 되돌릴 수 없는 스키마 손상(DROP COLUMN 없음) | Flink IT 필요, "거부 대신 지원" 요청 가능 |
| [5](05-client-multitablewriter-hudi-bucket-key/details.md) | 중간 | `MultiTableWriter`가 Hudi PK 테이블을 PK 인코딩으로 라우팅 | upsert 예외, 또는 엉뚱한 버킷에 저장 | 클러스터 IT가 무거움 |
| [6](06-listagg-string-agg-char-column/details.md) | 중간 | CHAR 컬럼의 `listagg`/`string_agg`가 DDL을 통과한 뒤 쓰기 전부 실패 | 테이블은 생기는데 쓸 수 없음 | 설계 선택(DDL 거부)과 DDL 동작 변경 |
| [7](07-server-local-max-timestamp-reload/details.md) | 어려움 | 활성 세그먼트가 빈 채로 재시작하면 `localMaxTimestamp`가 -1 | timestamp 기준으로 시작하면 과거 데이터 전부 건너뜀 | 서버 시작 경로, 원격 전용 경우가 남음 |
| [8](08-server-filtered-fetch-beyond-hw/details.md) | 어려움 | 필터를 건 로그 fetch가 high watermark를 넘어 다음 세그먼트까지 읽음 | 커밋 안 된 행 반환, 일부 행 영구 누락 | 핵심 fetch 경로, 열린 PR #4965와 겹침 |

올리는 데 걸리는 시간은 1~3번이 각각 약 20분, 4·5번은 IT 때문에 30~40분이다.

## 검증 결과(패치 8개를 함께 적용한 상태)

- fluss-common, fluss-server, fluss-client, fluss-flink-common: `spotless:check`·`checkstyle:check` 통과, 위반 0건.
- 관련 기존 테스트와 새 테스트: common 110, server 262, client 29(`MultiTableWriterITCase` 전체 포함), flink 55개. 실패 0건.
- `Flink120CatalogITCase#testAlterTableAddNonPhysicalColumn`: #4 수정 없이는 실패하고(스키마가 `[id, a, c]`가 됨), 수정 후 통과.
- 돌리지 않은 것: 전체 `./mvnw clean verify`.

## 올리는 방법

```bash
git fetch upstream main && git fetch origin search-known-issues
git switch -c fix-lookup-remaining-filter-binary upstream/main   # 브랜치 이름은 pr.md에 있음
git show origin/search-known-issues:contribution/01-flink-lookup-join-binary-key/fix.patch | git apply
./mvnw test -pl fluss-flink/fluss-flink-common -Dtest=LookupNormalizerTest   # 테스트 이름은 pr.md에 있음
```

1. `issue.md`로 이슈를 등록하고 번호를 적어 둔다.
2. `pr.md`의 `close #xxx`를 그 번호로 바꾼다.
3. 커밋하고 fork(`origin`)에 push한 뒤 `gh pr create --web`을 실행해 `pr.md`를 붙여 넣는다.

이 초안은 Claude Code(AI)로 작성했고 아직 사람이 검토하지 않았다. 올리기 전에 diff와 근거를 직접 읽어야 한다.
