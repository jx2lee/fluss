# Issue

**Title:** `[common] StartsWith statistics check compares in UTF-16 order and wrongly skips batches`

**Labels:** `component=common`, `component=log`

---

### Search before asking

- [x] I searched in the [issues](https://github.com/apache/fluss/issues) and found nothing similar.

### Fluss version

main (development)

### Please describe the bug 🐞

Log batch statistics collect string min/max with `BinaryString#compareTo`, which compares UTF-8 bytes, i.e. code point order:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-common/src/main/java/org/apache/fluss/record/LogRecordBatchStatisticsCollector.java#L168-L173

`StartsWith#test(rowCount, min, max, ...)` converts them to `java.lang.String` and uses `String#compareTo`, which compares UTF-16 code units:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-common/src/main/java/org/apache/fluss/predicate/StartsWith.java#L56-L60

The two orders disagree between BMP characters in `U+E000..U+FFFF` and supplementary characters (`U+10000+`). The first group includes fullwidth forms, CJK compatibility ideographs, Arabic presentation forms and U+FFFD; the second includes emoji and CJK Extension B.

The server skips a batch when this check returns false:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-server/src/main/java/org/apache/fluss/server/log/LogSegment.java#L656-L664

Flink turns `LIKE 'abc%'` into `startsWith`:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-flink/fluss-flink-common/src/main/java/org/apache/fluss/flink/utils/PredicateConverter.java#L162

So `WHERE s LIKE 'Ａ%'` on a log table with statistics can silently miss matching rows.

**Reproducer** (unit test, `PredicateTest`): a batch with the values `"a"`, `"ＡＢ"`, `"😀"` has min `"a"` and max `"😀"` in `BinaryString` order. The row `"ＡＢ"` matches `startsWith("Ａ")`, but the statistics check says the batch cannot match, because `"😀".compareTo("Ａ") < 0` in UTF-16:
```
Expecting value to be true but was false
	at org.apache.fluss.predicate.PredicateTest.testStartWithStatisticsUseBinaryStringOrder
```
The same happens for an emoji prefix: min `"Ａ"`, max `"😀🎉"`, `startsWith("😀")`.

The other comparison predicates (`<`, `>`, `BETWEEN`, ...) compare the `BinaryString` objects directly and are not affected.

### Solution

Compare `min`, `max` and the pattern as `BinaryString` (`compareTo` / `startsWith`), the same order the statistics are collected in. This also avoids decoding three strings per batch.

### Are you willing to submit a PR?

- [x] I'm willing to submit a PR!
