# Issue

**Title:** `[kv] listagg/string_agg on a CHAR column passes table creation but every write fails`

**Labels:** `component=kv`, `component=server`

---

### Search before asking

- [x] I searched in the [issues](https://github.com/apache/fluss/issues) and found nothing similar.

### Fluss version

main (development)

### Please describe the bug 🐞

Table creation validates aggregation columns with `AggFunctionType#getSupportedDataTypeRoots`, which allows `STRING` **and `CHAR`** for `listagg` / `string_agg`:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-common/src/main/java/org/apache/fluss/metadata/AggFunctionType.java#L166-L168

The docs list `CHAR` too ([aggregation.md listagg](https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/website/docs/table-design/merge-engines/aggregation.md?plain=1#L753), [string_agg](https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/website/docs/table-design/merge-engines/aggregation.md?plain=1#L815)).

However, the server-side factories only accept `StringType`:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-server/src/main/java/org/apache/fluss/server/kv/rowmerger/aggregate/factory/FieldListaggAggFactory.java#L39-L42
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-server/src/main/java/org/apache/fluss/server/kv/rowmerger/aggregate/factory/FieldStringAggFactory.java#L45-L48

The aggregators are created lazily on the first write. So a table such as
```sql
CREATE TABLE t (id INT, v CHAR(10), PRIMARY KEY (id) NOT ENFORCED)
WITH ('table.merge-engine' = 'aggregation', 'fields.v.agg' = 'listagg');
```
is created successfully, and then every upsert fails with:
```
java.lang.IllegalArgumentException: Data type for listagg column must be 'StringType' but was 'CHAR(10)'.
```

I checked all aggregate functions against the types allowed by validation (each function × each supported root, creating the aggregator); `listagg` and `string_agg` on `CHAR` are the only mismatches.

### Solution

Two options:
1. **(Proposed) Reject `CHAR` at DDL time**: remove `CHAR` from the supported roots of `listagg` / `string_agg` and from the docs. `CHAR(n)` is fixed-length, so it cannot hold a concatenation of several values. The `INDEXED` KV format even writes `CHAR(n)` as exactly n bytes. Paimon, which this code is based on, also accepts only `VARCHAR` for `listagg`. Tables created so far with this combination cannot accept writes anyway, so nothing that works today breaks.
2. Accept `CharType` in the factories and `FieldListaggAgg`. That would store values longer than `n` in a `CHAR(n)` column.

### Are you willing to submit a PR?

- [x] I'm willing to submit a PR!
