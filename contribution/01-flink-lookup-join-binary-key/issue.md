# Issue

**Title:** `[flink] Lookup join drops matches when a join key filtered after the lookup is BINARY/BYTES`

**Labels:** `component=connector/flink`

---

### Search before asking

- [x] I searched in the [issues](https://github.com/apache/fluss/issues) and found nothing similar.

### Fluss version

main (development)

### Please describe the bug 🐞

When not all join keys of a lookup join can be sent to Fluss, the remaining ones are checked on the client by `LookupNormalizer.RemainingFilter`. As the docs put it, "the other join keys are filtered after the lookup" ([lookups.md](https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/website/docs/engine-flink/lookups.md?plain=1#L298)). The check uses `Objects.equals`:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-flink/fluss-flink-common/src/main/java/org/apache/fluss/flink/source/lookup/LookupNormalizer.java#L163-L166

Flink's internal value for `BINARY` / `VARBINARY` / `BYTES` is a `byte[]`, so `Objects.equals` compares references. The array from the probe-side key row and the array decoded from the Fluss row are never the same object, so the filter never matches.

Flink does not re-check equality conditions it passed to the source as lookup keys, so:
- an inner join returns **no rows**,
- a left join returns the probe rows **padded with nulls**.

This affects `FlinkLookupFunction`, `FlinkAsyncLookupFunction` and `FlinkFullCacheLookupFunction`, which all use this filter.

Example: dim table `dim (id BIGINT, payload BYTES, PRIMARY KEY (id))` and
`... JOIN dim FOR SYSTEM_TIME AS OF s.proc AS d ON d.id = s.id AND d.payload = s.payload`.
Fluss is queried by `id` only; `payload` is filtered after the lookup. The same happens for prefix lookups where a key after the prefix is `BYTES`.

**Reproducer** (unit test, `LookupNormalizerTest`): lookup keys `(id, payload)`, primary/bucket key `id`. `createRemainingFilter(GenericRowData.of(1L, new byte[] {1, 2}))` then `isMatch(GenericRowData.of(1L, new byte[] {1, 2}))` returns `false`:
```
Expecting value to be true but was false
	at org.apache.fluss.flink.source.lookup.LookupNormalizerTest.testRemainingFilterComparesBinaryByContent
```

### Solution

Compare with `Objects.deepEquals`, which compares `byte[]` by content and behaves like `Objects.equals` for every other internal type (`StringData`, `DecimalData`, `TimestampData`, boxed primitives).

### Are you willing to submit a PR?

- [x] I'm willing to submit a PR!
