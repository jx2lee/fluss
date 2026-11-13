# Pull request

**Title:** `[client] Use the Hudi bucket key encoder in MultiTableWriter`

**Branch suggestion:** `fix-multitable-writer-hudi-bucket-key` · **Patch:** `fix.patch`

---

<!--
Generated-by: Claude Code (Claude Opus 5.5) following [the guidelines](https://github.com/apache/fluss/blob/main/AGENTS.md)
-->

### Purpose

Linked issue: close #xxx

`MultiTableWriterImpl` reuses the primary key encoder for bucketing whenever the bucket key is the default one. For Hudi tables the primary key encoder is `CompactedKeyEncoder`, but bucketing needs `HudiKeyEncoder`, so upserts either fail (`bucketKey must be exactly 4 bytes for Hudi bucketing`) or go to a bucket that `UpsertWriter` and `Lookuper` do not use.

### Brief change log

- `MultiTableWriterImpl.PkTableState`: create the bucket key encoder with `KeyEncoder.ofBucketKeyEncoder(rowType, bucketKeys, tableConfig, isDefaultBucketKey, primaryKeyEncoder)`, the same factory `BucketCalculator` uses for `UpsertWriterImpl`. It returns the primary key encoder itself for every format except Hudi, so `encodeBucketKey` keeps reusing the encoded primary key bytes for those formats.

### Tests

- Added `MultiTableWriterLakeFormatITCase#testUpsertPkTableWithDefaultBucketKey`, parameterized over all `DataLakeFormat`s: writes through `MultiTableWriter` and looks every key up with the regular `Lookuper`. Without the fix the Hudi case fails with `IllegalArgumentException: bucketKey must be exactly 4 bytes for Hudi bucketing, but got 1 bytes`; with the fix all formats pass.
- It is a separate class because `ClientToServerITCaseBase` configures the cluster with `datalake.format = paimon`, which rejects tables of other lake formats.
- `MultiTableWriterITCase` and `BucketCalculatorTest` pass.

### API and Format

No.

### Documentation

No.

---

Generative AI disclosure: Yes — Claude Code (Claude Opus 5.5).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
