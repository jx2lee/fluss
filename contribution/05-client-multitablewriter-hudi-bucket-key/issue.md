# Issue

**Title:** `[client] MultiTableWriter uses the primary key encoding to route Hudi primary key tables`

**Labels:** `component=client`, `component=lake/hudi`

---

### Search before asking

- [x] I searched in the [issues](https://github.com/apache/fluss/issues) and found nothing similar.

### Fluss version

main (development)

### Please describe the bug 🐞

For a primary key table with the default bucket key, `MultiTableWriterImpl` reuses the primary key encoder as the bucket key encoder:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-client/src/main/java/org/apache/fluss/client/table/writer/MultiTableWriterImpl.java#L438-L444

That is wrong for Hudi. The Hudi primary key encoder is `CompactedKeyEncoder`, because `HudiKeyEncoder` is lossy:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-common/src/main/java/org/apache/fluss/row/encode/KeyEncoder.java#L83-L88

Bucketing, however, must use the 4-byte `HudiKeyEncoder` hash. `KeyEncoder#ofBucketKeyEncoder(..., isDefaultBucketKey, primaryKeyEncoder)` exists for exactly this case, and `UpsertWriterImpl` / `PrimaryKeyLookuper` use it through `BucketCalculator`:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-common/src/main/java/org/apache/fluss/row/encode/KeyEncoder.java#L133-L144

`MultiTableWriterImpl` was added after the Hudi exception and kept the older "reuse when default bucket key" logic.

Effect when writing a Hudi primary key table through `conn.getMultiTable().newMultiTableWrite().createWriter()`:
- For most keys, `write()` throws, because `HudiBucketingFunction` requires exactly 4 bytes:
  ```
  org.apache.fluss.exception.FlussRuntimeException: Failed to send record to table test_mt_writer_lake_db.pk_hudi. Writer state: running
  Caused by: java.lang.IllegalArgumentException: bucketKey must be exactly 4 bytes for Hudi bucketing, but got 1 bytes. The bucket key bytes are expected to be produced by HudiKeyEncoder.
  ```
- For keys whose compacted encoding happens to be 4 bytes long, the write succeeds but the row is sent to a different bucket than the one `UpsertWriter` and `Lookuper` compute. Lookups then miss it, and the same key can exist in two buckets. (This follows from the code; the test below only covers the exception.)

**Reproducer** (IT): a primary key table `(id INT, name STRING, PRIMARY KEY id)`, `distributedBy(3)`, `table.datalake.format = <format>`. Upsert 20 rows with `MultiTableWriter`, then look each one up with `table.newLookup().createLookuper()`. The test passes for `paimon`, `lance` and `iceberg` and fails for `hudi` with the exception above.

### Solution

Build the bucket key encoder with `KeyEncoder.ofBucketKeyEncoder(rowType, bucketKeys, tableConfig, isDefaultBucketKey, primaryKeyEncoder)`, as `BucketCalculator` does. It still reuses the primary key encoder (and the already encoded bytes) for every format except Hudi.

### Are you willing to submit a PR?

- [x] I'm willing to submit a PR!
