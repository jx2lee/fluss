# Pull request

**Title:** `[kv] Reject CHAR columns for listagg and string_agg at table creation`

**Branch suggestion:** `fix-listagg-char-validation` · **Patch:** `fix.patch`

---

<!--
Generated-by: Claude Code (Claude Opus 5.5) following [the guidelines](https://github.com/apache/fluss/blob/main/AGENTS.md)
-->

### Purpose

Linked issue: close #xxx

`listagg` / `string_agg` on a `CHAR(n)` column passes table creation, because `AggFunctionType` lists `CHAR` as supported, but the server-side factories only accept `StringType`. Every write to such a table fails with `Data type for listagg column must be 'StringType' but was 'CHAR(10)'`. A fixed-length `CHAR(n)` cannot hold a concatenated result, so this PR rejects the combination at DDL time instead.

### Brief change log

- `AggFunctionType#getSupportedDataTypeRoots`: `listagg` / `string_agg` support `STRING` only.
- `aggregation.md`: list `STRING` as the only supported type for `listagg` and `string_agg`.

### Tests

- `TableDescriptorTest#testValidateAggFunctionWithDataType`: `listagg` and `string_agg` reject `CHAR(10)`. Fails without the fix.
- Added `FieldAggregatorFactoryTest#testCreateAggregatorForEverySupportedDataType` (fluss-server): for every `AggFunctionType` and every data type root it declares as supported, creating the aggregator must succeed. This keeps validation and the server-side factories in sync in the future. Without the fix it fails for exactly `listagg` and `string_agg` on `CHAR(10)`.
- `FieldAggregatorParameterizedTest`, `AggregateRowMergerTest` and `TableDescriptorValidationTest` pass.

### API and Format

No storage format change. DDL that used `listagg` / `string_agg` on `CHAR` is now rejected; such tables could not accept writes before.

### Documentation

Updated `website/docs/table-design/merge-engines/aggregation.md`.

---

Generative AI disclosure: Yes — Claude Code (Claude Opus 5.5).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
