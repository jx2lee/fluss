# FIP-62: Support tiering Fluss data to Delta Lake

> Draft. FIP-61 is the highest number on the wiki as of 2026-11-12 (FIP-60 is Schema Evolution), so 62 is the next free one. Check again before copying this to Confluence.

Document the state by adding a label to the FIP page with one of "discussion", "accepted", "released", "rejected".

| | |
|---|---|
| Status | Draft |
| Discussion thread | Not started |
| Vote thread | Not started |
| ISSUE | https://github.com/apache/fluss/issues/967 |
| Release | To be determined |

Please keep the discussion on the mailing list rather than commenting on the wiki (wiki discussions get unwieldy fast).

## Motivation

Currently Fluss can tier data to Paimon, Iceberg, Lance and Hudi, but not to Delta Lake. Delta Lake is the default format on Databricks, and almost all query engines (Spark, Trino, Flink, DuckDB, ...) can read it. Users who already have a Delta Lake based lakehouse have asked for it in #967.

Today such user has two options: tier to Paimon or Iceberg and copy the result to Delta with another job, or do not put that data into Fluss. Both are not good. This FIP adds `delta` as a new `datalake.format`. Fluss tables are tiered directly into Delta tables, and the result is a normal Delta table which any Delta reader can read.

We split the work into three phases, same as FIP-24 did for Hudi. This document describes phase 1 in detail. For phase 2 and 3 it only defines the scope and the decisions that are hard to change later, the details will come with their PRs. If phase 2 or 3 needs to change the shared tiering interfaces, we will open a separate FIP for it.

| Phase | Scope | Status |
|---|---|---|
| 1 | Log tables. Delta Kernel public APIs only. | PR https://github.com/apache/fluss/pull/4944 |
| 2 | Primary key tables (merge-on-read with deletion vectors), union read | not started |
| 3 | Commits through a catalog (Unity Catalog etc.) | not started |

## Public Interfaces

- `DataLakeFormat` gets a new value `DELTA("delta")`. Enable it with `datalake.format: delta` on the cluster and `'table.datalake.enabled' = 'true'` on the table. Other than this, the usage is same as other formats.
- New module `fluss-lake/fluss-lake-delta`, shipped as `plugins/delta/` in the distribution. It contains `DeltaLakeStoragePlugin` (identifier `delta`), `DeltaLakeStorage`, `DeltaLakeCatalog` and `DeltaLakeTieringFactory`.
- New cluster options under `datalake.delta.` prefix:
  - `datalake.delta.warehouse` (required). Root path of the Delta tables. Fluss table `db.tbl` becomes the path-based Delta table `<warehouse>/db/tbl`.
  - `datalake.delta.write.batch-size` (default `100000`). How many rows the writer buffers before writing a Parquet file. Can be overridden per table with the custom property `write.batch-size`. This option is only used by Fluss, it is not written to the Delta table (different from the `delta.*` properties below).
  - All other options under `datalake.delta.*` go to the Hadoop `Configuration` of the Kernel engine. File system credentials (`fs.s3a.*` etc.) are set here.
- Table properties follow the same two rules as Paimon, Iceberg and Hudi. Custom properties starting with `delta.` are passed to the Delta table as they are, e.g. `delta.logRetentionDuration`. Delta's own properties already use this prefix, so nothing is stripped (for Paimon and Iceberg the `paimon.` / `iceberg.` prefix is removed). All other Fluss table properties are written to the Delta table with a `fluss.` prefix, e.g. `table.datalake.enabled` becomes `fluss.table.datalake.enabled`. They are only informational, Fluss never reads them back, but `DESCRIBE DETAIL` shows how the table is configured in Fluss.
- `write.batch-size` has no prefix because it is a Fluss writer option and not a Delta table property. This follows Lance, which uses the unprefixed `batch_size` for the same purpose.
- No changes to existing public APIs or storage formats.

## Proposed Changes

### Design principles

Some decisions affect the whole design, so we list them first.

1. The written tables must be standard Delta tables. Any Delta reader should be able to read them without Fluss jar. So the only Delta dependency is Delta Kernel (`delta-kernel-api` + `delta-kernel-defaults`, 4.4.1), and phase 1 only uses its public API. Kernel is the recommended way for new connectors in Delta project, it has no Spark dependency, and it is already used by the Flink and Trino Delta connectors.
2. Bucketing stays on Fluss side, same as Lance. Delta has no bucketing spec, so there is nothing we can reuse. See the Bucketing section below.
3. Tiering progress is stored in the Delta commit itself. The bucket offsets of one round go into the same commit as the data files, so they become visible together and recovery can read the offsets back from the log.
4. Fail fast. Anything that Delta cannot express, or phase 1 does not handle, fails when the Fluss table is created with datalake enabled, not later in the tiering job after several hours.

### Bucketing

Iceberg and Paimon make Fluss use the bucket function of the lake, so one Fluss bucket is also one bucket on the lake side and the reader can seek into it. Delta has no bucket transform and no partition transform, only plain partition columns and liquid clustering, and clustering is only a hint to the writer. So there is nothing on the Delta side a Fluss bucket could map to.

Fluss therefore keeps its default bucketing for Delta tables, same as Lance:

- `FlussBucketingFunction` and the default key encoder are used. Setting `datalake.format: delta` does not change how bucket keys are encoded. The datalake encoder described in `table.datalake.format` only exists for Paimon and Iceberg.
- Tables with no bucket key, one bucket key or several bucket keys all work the same way. Iceberg has to reject several bucket keys, Delta does not.
- Because the bucketing does not depend on the lake, `table.datalake.enabled` can be turned on later for an existing table without rewriting data. This is different from Iceberg, where a table that was created with the Fluss bucketing cannot enable datalake afterwards.
- A Delta tiered table is partitioned only by the Fluss partition keys. The bucket is not visible in the Delta layout. One Parquet file always contains data of one bucket, because one writer handles one bucket, but Delta readers do not know this. Phase 2 may record the bucket in the `add` action tags, see below.

### Module layout

```
fluss-lake/fluss-lake-delta
├── DeltaConfig                      options, table path, Kernel Engine factory
├── DeltaLakeStoragePlugin / DeltaLakeStorage
├── DeltaLakeCatalog                 LakeCatalog: create / alter Delta tables
├── tiering/
│   ├── DeltaLakeTieringFactory      LakeTieringFactory
│   ├── DeltaLakeWriter              LakeWriter: rows -> Parquet files
│   ├── DeltaLakeCommitter           LakeCommitter: files -> one Delta commit
│   ├── DeltaDataFile, DeltaWriteResult, DeltaCommittable (+ serializers)
└── utils/
    ├── FlussDataTypeToDeltaDataType type mapping
    ├── FlussRowToDeltaValues        row conversion
    └── DeltaColumnarBatch           Kernel ColumnarBatch backed by Java objects
```

The module requires Java 11 because Kernel requires it. Same as `fluss-lake-lance` and `fluss-lake-hudi`, it is skipped on JDK 8 via `${skip.on.java8}` and runs in the `lake` CI stage.

### DeltaLakeCatalog

`createTable` is called when a Fluss table is created, or altered, with `table.datalake.enabled = true`.

Phase 1 rejects the following:

- Primary key tables. `InvalidTableException` with "Currently, we don't support tiering a primary key table to Delta Lake". Phase 2 removes this.
- Partition keys that are not `STRING`. Delta stores partition values as string in the log, and Fluss partition names are string too, so STRING maps 1:1 without any conversion. Other types need a parse step and we want to avoid it.
- Fluss types that Delta does not have. Currently only `TIME`.
- `delta.*` properties that enable column mapping directly or indirectly (`delta.enableIcebergCompatV2` enables it implicitly). Kernel cannot write to column-mapped tables, so tiering will fail at the first round anyway.

The created Delta table:

- The Delta schema is the Fluss row type, mapped as the table below. It only contains user columns. Following FIP-27 there are no `__bucket` / `__offset` / `__timestamp` columns, Delta only has the clean layout, same as Hudi. The three names are still reserved, but the coordinator already rejects them for every table no matter which lake format (`TableDescriptorValidation`), so Delta catalog does not need to check it again.
- Partition keys become Delta partition columns, one to one, via `DataLayoutSpec.partitioned(...)`.
- The table is created with `TableManager.buildCreateTableTransaction(path, schema, "Apache-Fluss")`. We enable the `domainMetadata` writer feature (`delta.feature.domainMetadata = supported`) and add every `delta.*` custom property of the Fluss table, and the other Fluss properties with the `fluss.` prefix as described in Public Interfaces.
- If a Delta table already exists at that path (usually a retry of same create request), we accept it when schema, partition columns and the `domainMetadata` feature match, otherwise throw `TableAlreadyExistException`. So `createTable` can be retried safely, and it will not silently reuse a table created by someone else.

`alterTable` supports very few things on purpose. Fluss side options (`table.datalake.freshness` etc.) can be set and reset, the mirrored `fluss.*` property is updated in one metadata-only Delta commit. A `delta.*` property can only be set when the value is same as the current value in the Delta table. Schema change, bucket count change and lake path change are rejected with "not supported yet". Schema evolution for Delta side is future work.

### Type mapping

| Fluss | Delta | Note |
|---|---|---|
| CHAR, STRING | STRING | |
| BOOLEAN | BOOLEAN | |
| BINARY, BYTES | BINARY | |
| DECIMAL(p, s) | DECIMAL(p, s) | |
| TINYINT / SMALLINT / INT / BIGINT | BYTE / SHORT / INTEGER / LONG | |
| FLOAT / DOUBLE | FLOAT / DOUBLE | |
| DATE | DATE | |
| TIME | not supported | Delta has no time type, rejected at create |
| TIMESTAMP (without time zone) | TIMESTAMP_NTZ | microsecond precision |
| TIMESTAMP_LTZ | TIMESTAMP | microsecond precision, UTC adjusted |
| ARRAY / MAP / ROW | ARRAY / MAP / STRUCT | nested types map recursively |

Nullable follows the Fluss column. Timestamps with precision above 6 are truncated to microseconds, because Parquet and Delta only store microseconds.

### DeltaLakeWriter

One writer per bucket (and partition) per tiering round.

- On open, the writer loads the latest snapshot and builds an update transaction. This transaction is never committed, we only use it to get the table state and physical schema from Kernel, so Kernel can tell us how to layout the files. The writer also checks that the Delta column names still match the Fluss row type, and fails if they do not match.
- `write(LogRecord)` converts the `InternalRow` to plain Java objects (`FlussRowToDeltaValues`) and appends them to an in-memory buffer.
- Every `write.batch-size` rows, and once more in `complete()`, the buffer is converted to a `DeltaColumnarBatch` (our implementation of Kernel `ColumnarBatch` / `ColumnVector` on top of the buffered objects). It goes through `Transaction.transformLogicalData` with the partition values of the writer, and is written with `ParquetHandler.writeParquetFiles` into the directory from `Transaction.getWriteContext`. Kernel computes the file statistics for the configured stats columns.
- `complete()` returns a `DeltaWriteResult` with one `DeltaDataFile` per Parquet file: path, size, modification time, stats JSON, partition values. The files exist on storage at this point, but they are not visible until the committer adds them to the log.

The writer does not know about Delta versions, it only produces files. So writers of different buckets never conflict with each other.

### DeltaLakeCommitter

`toCommittable` concatenates the data files of all write results in the round into one `DeltaCommittable`.

`commit` makes exactly one Delta commit from it:

1. Load the latest snapshot, build an update transaction with `Operation.WRITE` and engine info `Apache-Fluss`.
2. Group the files by partition values, get a `DataWriteContext` per partition, generate the `add` actions with `Transaction.generateAppendActions`.
3. Take the snapshot properties of the round (the bucket offsets Fluss passes via `CommitContext.snapshotProperties()`), serialize them to JSON, attach them with `transaction.addDomainMetadata` under the domain `fluss.tiering`.
4. `transaction.commit`. Kernel handles the optimistic concurrency control with other writers on `_delta_log`, including retry.
5. Run the post-commit hooks returned by Kernel (checkpoint, checksum). They only make the log faster to read, so if one of them fails we just log a warning and continue, the commit is already done.
6. Return `LakeCommitResult.committedIsReadable(version)`. Fluss records the Delta version as the lake snapshot id.

`abort` deletes the files of a committable that will not be committed.

`getMissingLakeSnapshot(latestLakeSnapshotIdOfFluss)` handles the case that the job crashes between "committed to Delta" and "reported to Fluss". It loads the latest snapshot, reads the `fluss.tiering` domain metadata, then goes back version by version until the domain metadata changes. We need this because another Delta writer may commit after our last commit, and we should not treat its version as ours. If the found version is newer than what Fluss has, it is returned as `CommittedLakeSnapshot(version, properties)`, otherwise `null`.

Why we use domain metadata instead of `commitInfo`: `commitInfo` is only informational. Other tools may drop or rewrite it, and snapshot API does not expose it. Domain metadata is part of the protocol, survives checkpoints, and can be read with `Snapshot.getDomainMetadata`. Hudi stores the same offsets in the extra metadata of the timeline, domain metadata is the same thing in Delta.

### Union read (phase 1)

Not supported in phase 1. Delta has no bucketing spec, so a Fluss client cannot ask for "bucket 3 of this Delta table". A Delta tiered table is expected to be read by Delta reader directly, and Fluss serves the latest data. A Fluss client that asks lake data from a Delta tiered table gets the same behavior as Lance today.

### Phase 2: primary key tables and union read

- A primary key table maps to a Delta table with the Fluss primary key recorded in table metadata. Updates and deletes are merge-on-read: new Parquet files for the upserts, deletion vectors (`delta.enableDeletionVectors`) for the old rows in existing files. Tiering should not rewrite whole files.
- Currently writing deletion vectors is only possible through Kernel internal APIs. The Kernel-based Flink sink (delta-io/delta#6964, delta-io/delta#6977) does the same. So phase 2 will depend on `delta-kernel-defaults` internals, pinned to one exact version, and will switch to public API when it is available.
- Union read follows the FIP-27 clean-layout model: per-bucket end offsets come from the `fluss.tiering` domain metadata of the Delta version, not from row columns, and `LakeSource` / `RecordReader` are built on Kernel `Scan` API. The open question is how the reader finds files of one Fluss bucket, because Delta has no bucket transform. Two options: (a) record the Fluss bucket of each file in the `add` action tags, Kernel exposes them per file and other readers ignore them, or (b) read the whole snapshot and bucket on Fluss side as Lance does. If we need an optional system column at the end (for example an offset column for reading in offset order), it follows FIP-27 rules: off by default, on only through an explicit table option.

### Phase 3: catalog-managed tables

Phase 1 tables are path-based and can be registered as external tables in any metastore. Phase 3 uses Kernel catalog-managed commits (the `unitycatalog` module and the `catalogManaged` table feature) so that commits go through a catalog such as Unity Catalog. This gives managed tables and access control at catalog level. It adds a catalog type option (for example `datalake.delta.catalog.type`) and credentials. The writer and committer above do not change.

## Compatibility, Deprecation, and Migration Plan

- Nothing changes for existing users. `delta` is a new `datalake.format` value and an optional plugin.
- Switching `datalake.format` on a running cluster is not supported, same as the other formats today.
- `DeltaCommittable` and `DeltaWriteResult` have versioned serializers (`SimpleVersionedSerializer`), so in-flight tiering state is not broken by plugin upgrade.
- Delta tables only have the FIP-27 clean layout, so the legacy table handling that Paimon and Iceberg needed is not needed here. Phase 2 must not require re-creating phase 1 tables. If we adopt per-file bucket tags, we can add them only to new files.

## Test Plan

- Unit tests in `fluss-lake-delta`: type mapping (`FlussDataTypeToDeltaDataTypeTest`), columnar batch conversion (`DeltaColumnarBatchTest`), data file serialization (`DeltaDataFileTest`), writer/committer round trips for partitioned and non-partitioned log tables including `getMissingLakeSnapshot` recovery (`DeltaTieringTest`).
- `DeltaTieringITCase`: runs the Flink tiering job against a Fluss cluster, tiers a log table and a partitioned log table, reads the Delta table back with Kernel and checks the rows, the Delta version and the `fluss.tiering` domain metadata (bucket offsets).
- Create/alter failure cases: primary key table, non-STRING partition key, `TIME` column, column-mapping properties, mismatched pre-existing Delta table, altering `delta.*` properties.
- Manually check that the tables can be read by external readers (Spark with delta-spark, DuckDB) without any Fluss jar.
- CI: `lake` stage, skipped on JDK 8.

## Rejected Alternatives

- Delta Standalone instead of Kernel. This was the original suggestion in #967. Standalone is in maintenance mode, does not support the newer protocol features (deletion vectors, domain metadata, catalog-managed tables), and the Delta project recommends Kernel for new connectors. Kernel covers all three phases with one dependency.
- Delta Spark or the Flink Delta connector. Both bring a full engine runtime into the tiering plugin (Spark, or Flink Table API plus Hadoop). `delta-flink` is also tied to one Flink version, but Fluss tiering needs to support multiple Flink versions. Kernel has no engine dependency.
- `__bucket`, `__offset`, `__timestamp` system columns. That was the layout of Paimon and Iceberg tables before FIP-27. FIP-27 made clean layout the default and requires explicit opt-in for any system column, and Hudi (FIP-24) only has clean layout. Delta follows the same, the tiering progress is stored in the commit instead.
- Bucket offsets in `commitInfo` or in a side file. `commitInfo` is not guaranteed to be preserved by other tools and cannot be queried through snapshot API. A side file loses the atomicity between data and progress. Domain metadata solves both.
- Mapping Fluss buckets to liquid clustering or to a partition column. Delta has no bucket transform and no way to read one bucket, clustering is only a hint to the writer. Keeping bucketing on Fluss side (same as Lance) is simpler. If phase 2 needs per-bucket read, tags on the `add` actions can provide it without system column.
- One FIP per phase. Discussed in #967. Phase 1 adds nothing more than a format value and a module, so one FIP that fixes the direction and scopes the phases is better. Phase 2 should review its internal API usage before it starts.
