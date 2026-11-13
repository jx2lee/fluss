# Issue

**Title:** `[server] Local max timestamp is -1 after reload when the active segment is empty, so list offsets by timestamp returns the latest offset`

**Labels:** `component=server`, `component=log`

---

### Search before asking

- [x] I searched in the [issues](https://github.com/apache/fluss/issues) and found nothing similar.

### Fluss version

main (development)

### Please describe the bug 🐞

When a log is loaded, `LocalLog` initializes `localMaxTimestamp` from the **last** segment only:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-server/src/main/java/org/apache/fluss/server/log/LocalLog.java#L119-L120

If the last (active) segment is empty, `maxTimestampSoFar()` is `-1`, even when older segments still hold data. The field is otherwise only updated on append, so it stays `-1` until the next write.

An empty active segment is a normal state. For example, the TTL cleanup rolls the expired active segment of an idle bucket:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-server/src/main/java/org/apache/fluss/server/log/LogTablet.java#L1428-L1432

A restart afterwards loads the log with an empty last segment.

`Replica#getOffsetByTimestamp` treats `localMaxTimestamp` as the newest timestamp of the bucket and short-circuits to the latest offset:
https://github.com/apache/fluss/blob/67bc81c1ee4fe98c86da558a64236a6fda694528/fluss-server/src/main/java/org/apache/fluss/server/replica/Replica.java#L2032-L2033

With `localMaxTimestamp = -1`, every timestamp between 0 and now takes this branch. So list offsets by timestamp (e.g. the Flink `timestamp` startup mode) returns the high watermark, and consumers silently skip all retained data, both local and remote. The timestamp lag metric is also wrong.

**Reproducer** (unit test, `LogTabletTest`): append records, `roll()`, flush, close, and reload the `LogTablet`. The data can still be found (`lookupOffsetForTimestamp(maxTimestamp) == 0`), but:
```
expected: 1794536215266L
 but was: -1L
	at org.apache.fluss.server.log.LogTabletTest.testLocalMaxTimestampIsRestoredWhenActiveSegmentIsEmpty
```

### Solution

On load, use the max timestamp of the latest **non-empty** segment, scanning backwards and skipping empty segments. Usually only the last one or two segments are inspected, so startup does not materialize more time indexes than needed.

Follow-up worth discussing: if *all* local segments are empty (all data expired locally and only exists in remote storage), the value stays `-1` after a restart and the same short-circuit applies. Covering that needs the remote log's max timestamp in `Replica#getOffsetByTimestamp`, which seems like a separate change.

### Are you willing to submit a PR?

- [x] I'm willing to submit a PR!
