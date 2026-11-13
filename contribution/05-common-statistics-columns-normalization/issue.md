# Issue

**Title:** `[common] table.statistics.columns values accepted at table creation make every append fail`

**Labels:** `component=common`, `component=client`

---

### Search before asking

- [x] I searched in the [issues](https://github.com/apache/fluss/issues) and found nothing similar.

### Fluss version

main (development)

### Please describe the bug 🐞

Table creation validates `table.statistics.columns` with `StatisticsConfigUtils#validateStatisticsConfig`, but writers parse it with `TableConfig#getStatisticsColumns`. The two disagree, so some values pass validation and then break every append to the table.

**1. Wildcard with surrounding whitespace (`' *'`, `'* '`).** Validation trims before checking for the wildcard:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-common/src/main/java/org/apache/fluss/config/StatisticsConfigUtils.java#L69

`TableConfig` does not trim, so it falls through to the column list and returns `SPECIFIED ["*"]`:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-common/src/main/java/org/apache/fluss/config/TableConfig.java#L230-L237

When the client creates the statistics collector for a batch (`RecordAccumulator`), `TableInfo#getStatsIndexMapping` throws:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-common/src/main/java/org/apache/fluss/metadata/TableInfo.java#L349-L356
```
java.lang.IllegalArgumentException: Statistics column '*' not found in table schema
```

**2. Duplicated columns (`'id,id'`, `'id, name, id'`).** Validation only checks that each name exists and has a supported type. `TableConfig` keeps the duplicates, and `LogRecordBatchStatisticsWriter` builds the statistics row type from them:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-common/src/main/java/org/apache/fluss/record/LogRecordBatchStatisticsWriter.java#L93-L98
```
java.lang.IllegalArgumentException: Field names must be unique. Found duplicates: [id]
```

In both cases the server accepts the table, but no client can append to it:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-client/src/main/java/org/apache/fluss/client/write/RecordAccumulator.java#L1138-L1142

**Reproducer** (unit test, `StatisticsConfigUtilsTest`): build a descriptor with the property, call `validateStatisticsConfig` (passes), then check `TableInfo.of(...).getTableConfig().getStatisticsColumns()`:
```
expected: ALL
 but was: SPECIFIED
	at ...StatisticsConfigUtilsTest.testWildcardWithSurroundingWhitespaceIsTreatedAsAllColumns
```

### Solution

Normalize in `TableConfig#getStatisticsColumns`, which both validation and writers use: trim before comparing with `*`, and drop duplicate column names. Fixing the parser instead of only tightening validation also makes tables that were already created with such values writable again.

### Are you willing to submit a PR?

- [x] I'm willing to submit a PR!
