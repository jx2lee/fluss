# Pull request

**Title:** `[server] Restore local max timestamp from the latest non-empty segment`

**Branch suggestion:** `fix-local-max-timestamp-reload` · **Patch:** `fix.patch`

---

<!--
Generated-by: Claude Code (Claude Opus 5.5) following [the guidelines](https://github.com/apache/fluss/blob/main/AGENTS.md)
-->

### Purpose

Linked issue: close #xxx

`LocalLog` initializes `localMaxTimestamp` from the last segment only. When the active segment is empty at load time (e.g. after the TTL cleanup rolled it on an idle bucket, then a restart), it becomes `-1`, and `Replica#getOffsetByTimestamp` returns the latest offset for every timestamp, so consumers starting from a timestamp skip all retained data.

### Brief change log

- `LocalLog`: initialize `localMaxTimestamp` from the latest non-empty segment, scanning backwards and skipping empty segments (`-1` only if all segments are empty, `0` if there are no segments, as before). Only the needed segments are inspected, so no extra time indexes are materialized in the common case.

### Tests

- Added `LogTabletTest#testLocalMaxTimestampIsRestoredWhenActiveSegmentIsEmpty`: append, roll, close and reload; the reloaded `localMaxTimestamp()` equals the one before the reload. Without the fix it is `-1`.
- `LogTabletTest`, `LocalLogTest`, `LogSegmentTest` and `LogManagerTest` pass.

### API and Format

No.

### Documentation

No.

---

Generative AI disclosure: Yes — Claude Code (Claude Opus 5.5).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
