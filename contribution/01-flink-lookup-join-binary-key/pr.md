# Pull request

**Title:** `[flink] Compare binary join keys by content in lookup remaining filter`

**Branch suggestion:** `fix-lookup-remaining-filter-binary` · **Patch:** `fix.patch`

---

<!--
Generated-by: Claude Code (Claude Opus 5.5) following [the guidelines](https://github.com/apache/fluss/blob/main/AGENTS.md)
-->

### Purpose

Linked issue: close #xxx

Join keys that are not sent to Fluss are checked after the lookup by `LookupNormalizer.RemainingFilter`, which uses `Objects.equals`. For `BINARY`/`BYTES` keys the internal value is a `byte[]`, so the check compares references and never matches: inner lookup joins return nothing, left joins return null-padded rows.

### Brief change log

- `LookupNormalizer.FieldCondition#fieldMatches`: use `Objects.deepEquals`, so `byte[]` values are compared by content. Other internal types compare as before.

### Tests

- Added `LookupNormalizerTest#testRemainingFilterComparesBinaryByContent`: primary key `id`, lookup keys `(id, payload BYTES)`; a looked-up row with an equal but distinct `byte[]` must match and a different one must not. Fails without the fix, passes with it.
- `LookupNormalizerTest` passes.

### API and Format

No.

### Documentation

No.

---

Generative AI disclosure: Yes — Claude Code (Claude Opus 5.5).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
