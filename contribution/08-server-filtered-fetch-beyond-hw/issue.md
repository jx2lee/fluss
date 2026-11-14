# Issue

**Title:** `[server] Filtered log fetch reads beyond the high watermark when the high watermark segment is fully filtered out`

**Labels:** `component=server`, `component=log`

---

### Search before asking

- [x] I searched in the [issues](https://github.com/apache/fluss/issues) and found nothing similar.

### Fluss version

main (development)

### Please describe the bug 🐞

With a record batch filter (`FilterContext`), `LocalLog#read` keeps scanning newer segments while every batch of the current segment is filtered out.

The read limit `maxPosition` is the position of the max offset (the high watermark for client fetches) **only for the segment that contains the max offset**. Every other segment is read up to its full size:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-server/src/main/java/org/apache/fluss/server/log/LocalLog.java#L436-L442

After an "all filtered" segment, the loop sets `readOffset = filteredEndOffset` and moves on to the next segment. It never checks `readOffset` against `maxOffsetMetadata` again:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-server/src/main/java/org/apache/fluss/server/log/LocalLog.java#L453-L476

So when the high watermark segment is fully filtered out and the leader has already rolled a newer segment, batches above the high watermark in the newer segment are returned. The high watermark still being in an older segment is the normal state while followers catch up after a roll.

The unfiltered path is not affected, because it only moves to the next segment when `LogSegment#read` returns `null`.

Impact for clients that fetch with a filter (e.g. Flink filter pushdown on log tables with `table.statistics.columns`), which use `FetchIsolation.HIGH_WATERMARK`:
- **Uncommitted records are returned.** They can be truncated after a leader change, so the consumer has seen rows that no longer exist.
- **Records can be skipped permanently.** Batches between the high watermark and the end of the high watermark segment are never scanned, but the client's next fetch offset jumps past them. If they match the filter once committed, the consumer never reads them.

**Reproducer** (unit test, `LocalLogTest`):
1. Segment 1: batch `[0, 2]` with `a` = 1, 2, 3, and batch `[3, 5]` with `a` = 6, 7, 8.
2. Roll. Segment 2: batch `[6, 8]` with `a` = 7, 8, 9.
3. High watermark = offset 3, inside segment 1 right after the first batch.
4. `localLog.read(0, ..., maxOffsetMetadata = highWatermark, filter = a > 5)`.

Expected: empty records, `filteredEndOffset = 3`.
Actual: batch `[6, 8]` from segment 2 is returned.

```
java.lang.AssertionError:
[batch [6, 8] read beyond high watermark 3]
Expecting actual:
  8L
to be less than:
  3L
	at org.apache.fluss.server.log.LocalLogTest.testReadWithFilterDoesNotReadBeyondMaxOffsetInLaterSegment
```

### Solution

After an all-filtered segment, stop scanning once `readOffset >= maxOffsetMetadata.getMessageOffset()`. The filtered end offset of the high watermark segment is bounded by `maxPosition`, so the response returns `filteredEndOffset` ≤ the high watermark, and the client resumes from there.

Note: open PR #4965 changes this loop to support a client-provided stopping offset, but does not bound the scan by the max offset.

### Are you willing to submit a PR?

- [x] I'm willing to submit a PR!
