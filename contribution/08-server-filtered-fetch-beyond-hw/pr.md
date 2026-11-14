# Pull request

**Title:** `[server] Do not read beyond the high watermark in filtered log fetch`

**Branch suggestion:** `fix-filtered-fetch-beyond-hw` · **Patch:** `fix.patch` (apply with `git apply contribution/08-server-filtered-fetch-beyond-hw/fix.patch`)

---

<!--
Generated-by: Claude Code (Claude Opus 5.5) following [the guidelines](https://github.com/apache/fluss/blob/main/AGENTS.md)
-->

### Purpose

Linked issue: close #xxx

`LocalLog#read` with a record batch filter moves to the next segment when every batch of the current segment is filtered out, but only the segment that holds the max offset is bounded by the max offset position. Once the high watermark segment is fully filtered, the next segment is read up to its full size, so records above the high watermark are returned and records between the high watermark and the end of its segment are skipped.

### Brief change log

- `LocalLog#read`: after an all-filtered segment, stop scanning when `readOffset` has reached `maxOffsetMetadata.getMessageOffset()`. The method then returns the existing filtered-empty response with the last filtered end offset, which is bounded by the max offset position.

### Tests

- Added `LocalLogTest#testReadWithFilterDoesNotReadBeyondMaxOffsetInLaterSegment`: the high watermark is inside the first segment, the batches below it are filtered out, and a matching batch exists in the next segment. Fails without the fix (batch `[6, 8]` returned past high watermark 3), passes with it.
- `LocalLogTest` and `LogSegmentTest` pass.

### API and Format

No.

### Documentation

No.

---

Generative AI disclosure: Yes — Claude Code (Claude Opus 5.5).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
