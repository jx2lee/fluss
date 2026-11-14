# Pull request

**Title:** `[common] Normalize table.statistics.columns consistently with validation`

**Branch suggestion:** `fix-statistics-columns-normalization` · **Patch:** `fix.patch`

---

<!--
Generated-by: Claude Code (Claude Opus 5.5) following [the guidelines](https://github.com/apache/fluss/blob/main/AGENTS.md)
-->

### Purpose

Linked issue: close #xxx

`table.statistics.columns = ' *'` or `'id,id'` passes table creation validation, but `TableConfig#getStatisticsColumns` parses it as `SPECIFIED ["*"]` or `["id", "id"]`. Creating the batch statistics collector then fails on every append (`Statistics column '*' not found in table schema` / `Field names must be unique`).

### Brief change log

- `TableConfig#getStatisticsColumns`: trim the value before comparing it with the `*` wildcard, matching `StatisticsConfigUtils#validateStatisticsConfig`, and drop duplicate column names (keeping the first occurrence order).

Because the fix is in the parser used by both validation and the write path, tables already created with such values become writable after upgrading the client.

### Tests

- Added `StatisticsConfigUtilsTest#testWildcardWithSurroundingWhitespaceIsTreatedAsAllColumns` (`" *"`, `"* "`, `" * "`): validation passes, mode is `ALL`, the stats index mapping covers the supported columns, and the statistics collector can be created. Fails without the fix.
- Added `StatisticsConfigUtilsTest#testDuplicateColumnsAreDeduplicated` (`"id, name, id"`): columns are `[id, name]` and the collector can be created. Fails without the fix.
- `StatisticsConfigUtilsTest`, `StatisticsColumnsConfigTest`, `TableConfigTest` and `LogRecordBatchStatistics*Test` pass.

### API and Format

No.

### Documentation

No.

---

Generative AI disclosure: Yes — Claude Code (Claude Opus 5.5).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
