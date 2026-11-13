# Pull request

**Title:** `[flink] Reject adding computed and metadata columns via ALTER TABLE`

**Branch suggestion:** `fix-alter-add-non-physical-column` · **Patch:** `fix.patch`

---

<!--
Generated-by: Claude Code (Claude Opus 5.5) following [the guidelines](https://github.com/apache/fluss/blob/main/AGENTS.md)
-->

### Purpose

Linked issue: close #xxx

`ALTER TABLE t ADD c AS a + 1` (and `ADD m ... METADATA`) is converted into a Fluss `AddColumn` for a physical column. The table silently gains a physical, always-NULL column that writers must provide, and since Fluss cannot drop columns it cannot be undone.

### Brief change log

- `FlinkConversions#toFlussTableChanges`: throw `UnsupportedOperationException("Adding non-physical column ... is not supported yet.")` when the added column is not physical. `CREATE TABLE` already rejects metadata columns, and other unsupported table changes already throw `UnsupportedOperationException`.

### Tests

- Added `FlinkConversionsTest#testAddNonPhysicalColumnIsNotConvertedToPhysicalColumn` (unit): metadata and computed `AddColumn` changes are rejected.
- Added `FlinkCatalogITCase#testAlterTableAddNonPhysicalColumn` (end to end, run via `Flink120CatalogITCase`): both `ALTER TABLE ... ADD c AS a + 1` and `ALTER TABLE ... ADD m TIMESTAMP_LTZ(3) METADATA FROM 'timestamp'` fail and the schema stays `[id, a]`. Without the fix the first statement succeeds and the schema becomes `[id, a, c]`.
- `FlinkConversionsTest`, `LookupNormalizerTest`, `PredicateConverterTest` and `Flink120CatalogITCase#testAlterTableWatermark` pass.

### API and Format

No.

### Documentation

No. (Computed/metadata columns via `ALTER TABLE` were never documented as supported.)

---

Generative AI disclosure: Yes — Claude Code (Claude Opus 5.5).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
