# Issue

**Title:** `[flink] ALTER TABLE ADD of a computed or metadata column adds a physical column`

**Labels:** `component=connector/flink`

---

### Search before asking

- [x] I searched in the [issues](https://github.com/apache/fluss/issues) and found nothing similar.

### Fluss version

main (development)

### Please describe the bug 🐞

`FlinkConversions#toFlussTableChanges` converts every Flink `TableChange.AddColumn` into a Fluss `TableChange.addColumn`. It never checks whether the column is physical:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-flink/fluss-flink-common/src/main/java/org/apache/fluss/flink/utils/FlinkConversions.java#L443-L452

`FlinkCatalog#alterTable` passes every change through it:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-flink/fluss-flink-common/src/main/java/org/apache/fluss/flink/catalog/FlinkCatalog.java#L664-L668

So `ALTER TABLE t ADD c AS a + 1` succeeds, and `c` becomes a **physical** nullable `INT` column in Fluss. The expression is lost. Existing rows read `c` as `NULL`, and writers now have to provide `c`. Fluss cannot drop columns ([`SchemaUpdate#dropColumn`](https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-server/src/main/java/org/apache/fluss/server/coordinator/SchemaUpdate.java#L101-L103) throws), so the table cannot be repaired.

`ALTER TABLE t ADD m TIMESTAMP_LTZ(3) METADATA FROM 'timestamp'` reaches the same code path. `CREATE TABLE`, by contrast, rejects metadata columns explicitly:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-flink/fluss-flink-common/src/main/java/org/apache/fluss/flink/utils/FlinkConversions.java#L249-L256

(A computed column with a `NOT NULL` result type is rejected by the server, because added columns must be nullable. Every nullable expression gets through.)

**Reproducer** (Flink 1.20, `FlinkCatalogITCase`):
```sql
CREATE TABLE t (id BIGINT, a INT, PRIMARY KEY (id) NOT ENFORCED);
ALTER TABLE t ADD c AS a + 1;   -- succeeds
```
Reading the table back from the catalog gives the columns `[id, a, c]`, with `c` physical, instead of failing or keeping `c` computed:
```
[columns after: ALTER TABLE test_alter_add_non_physical_column ADD c AS a + 1]
Expecting actual:
  ["id", "a", "c"]
to contain exactly (and in same order):
  ["id", "a"]
but some elements were not expected:
  ["c"]
```

### Solution

Reject non-physical columns in the `AddColumn` conversion with an `UnsupportedOperationException`, like other unsupported table changes. Supporting `ALTER TABLE ADD <computed column>` properly (persisting it under `schema.N.expr` like `CREATE TABLE` does) could be a follow-up feature.

### Are you willing to submit a PR?

- [x] I'm willing to submit a PR!
