# Pull request

**Title:** `[common] Use BinaryString order in StartsWith statistics check`

**Branch suggestion:** `fix-startswith-stats-order` · **Patch:** `fix.patch`

---

<!--
Generated-by: Claude Code (Claude Opus 5.5) following [the guidelines](https://github.com/apache/fluss/blob/main/AGENTS.md)
-->

### Purpose

Linked issue: close #xxx

Batch statistics collect string min/max in `BinaryString` (UTF-8 byte) order, but `StartsWith#test(rowCount, min, max, ...)` compares them as `java.lang.String` (UTF-16 order). For strings mixing `U+E000..U+FFFF` characters and supplementary characters (e.g. fullwidth forms and emoji), the check can return false for a batch containing matching rows, and the server skips it.

### Brief change log

- `StartsWith#test(DataType, long, Object, Object, long, Object)`: compare `min`, `max` and the pattern as `BinaryString` instead of converting them to `String`.

### Tests

- Added `PredicateTest#testStartWithStatisticsUseBinaryStringOrder`: batches whose min/max are ordered by `BinaryString#compareTo` (as the collector does) must not be pruned when they contain a matching value. Fails without the fix.
- `PredicateTest` and `PredicateBuilderTest` pass.

### API and Format

No.

### Documentation

No.

---

Generative AI disclosure: Yes — Claude Code (Claude Opus 5.5).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
