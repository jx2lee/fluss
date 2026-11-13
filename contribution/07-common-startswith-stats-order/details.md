# 7. startsWith 통계 검사가 다른 문자열 순서로 비교함

`LIKE 'x%'` 조건으로 배치를 건너뛸지 판단할 때 문자열을 UTF-16 순서로 비교한다. 통계의 min/max는 UTF-8 순서로 모으기 때문에, 맞는 행이 들어 있는 배치를 건너뛸 수 있다.

| 항목 | 내용 |
|---|---|
| 모듈 | fluss-common, `StartsWith` |
| 테스트 | `PredicateTest#testStartWithStatisticsUseBinaryStringOrder` |
| 열린 PR | 없음 |

## 원인

- 로그 배치 통계는 문자열 min/max를 `BinaryString#compareTo`로 모은다. UTF-8 바이트 비교, 즉 code point 순서다.
- `StartsWith#test(rowCount, min, max, ...)`는 이 값을 `java.lang.String`으로 바꿔서 `String#compareTo`로 비교한다. UTF-16 code unit 순서다.
- 두 순서는 `U+E000..U+FFFF` 구간의 BMP 문자와 보조 문자(`U+10000` 이상) 사이에서 어긋난다. 앞쪽에는 전각 문자, CJK 호환 한자, 아랍어 표시형, U+FFFD가 있고 뒤쪽에는 이모지와 CJK 확장 B가 있다.
- 서버(`LogSegment`)는 이 검사가 false면 배치를 건너뛰고, Flink는 `LIKE 'abc%'`를 `startsWith`로 바꾼다.

## 영향

통계가 켜진 로그 테이블에서 `WHERE s LIKE 'Ａ%'`가 맞는 행을 조용히 빠뜨릴 수 있다. `<`, `>`, `BETWEEN` 같은 다른 비교 조건은 `BinaryString`끼리 직접 비교하므로 영향이 없다.

## 재현

값이 `"a"`, `"ＡＢ"`, `"😀"`인 배치의 min은 `"a"`, max는 `"😀"`이다(`BinaryString` 순서). `"ＡＢ"`는 `startsWith("Ａ")`에 맞는다. 그런데 UTF-16에서는 `"😀".compareTo("Ａ") < 0`이므로, 검사는 이 배치에 맞는 행이 없다고 판단한다.

```
Expecting value to be true but was false
```

이모지 prefix에서도 똑같다(min `"Ａ"`, max `"😀🎉"`, `startsWith("😀")`).

## 수정

`min`, `max`, 패턴을 `BinaryString`으로 비교한다(`compareTo`, `startsWith`). 통계를 모을 때와 같은 순서다. 배치마다 문자열 3개를 디코딩하던 비용도 없어진다.
