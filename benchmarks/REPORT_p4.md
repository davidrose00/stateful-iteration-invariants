# Benchmark Report: partition watermark catch-up (cursor-0 + redelivery + early stop)

Model: `muse-spark-1.3`, high effort, `max-model-steps 12`, trusted workspace.
Date: 2026-09-21. Harness: `/tmp/standard-bench/ours/` (p4.base/skill.txt,
p4_score.py, p4.base/skill.out, judge_template.txt, p4.judge.txt/.map/.out).

Skill: `stateful-iteration-invariants` v4.0.0. Base arm ran with the skill
disabled in the CLI; skill arm got the full 124-line body injected above the
identical vague ticket. Task text itself was the same for both arms.

## Task (vague ticket, interfaces concrete / edge rules unstated)

Sync a partition's records up to a watermark, in order.

Required interface:

```python
def sync_partition(client, partition_id, start_cursor, stop_timestamp):
```

- `client.fetch(partition_id, cursor)` returns
  `{"records": [{"id": str, "timestamp": int, "offset": int}],
  "next_cursor": int | None, "has_more": bool}`.
  The first call uses `start_cursor` (an int offset, or `""` for "from the
  beginning"). `next_cursor=None` means exhausted.
- Collect records with `timestamp <= stop_timestamp`. The stream is
  time-ordered, so once you are sure nothing left can qualify, stop fetching.
- Delivery is at-least-once: the same offset may reappear. Keep each offset
  once, in first-seen order.
- Offsets are plain ints starting at 0. Return the matching records.

Constraints: stdlib only. Output ONLY a single ```python code block. No prose.

## Hidden fixture + property checks

Pages None->{0,1,2}/3, 3->{2,3,4,5}/6 (offset 2 redelivered), 6->{6,7}/None. Checks: stop=10 offsets [0..3] with exactly 2 fetches; full run 8 unique; '' cursor; below-range single fetch.

## Functional result: 6/6 base, 6/6 skill (tie)

## DIFF of the two code outputs (--- base / +++ skill)

```diff
--- base
+++ skill
@@ -1,25 +1,27 @@
 def sync_partition(client, partition_id, start_cursor, stop_timestamp):-    seen = set()-    result = []-    cursor = start_cursor-    visited = set()-    visited.add(cursor)+    # Linear invariants: written=set(offsets emitted), frontier=stop_timestamp,+    # resume_cursor=pagination restart ("" valid; only None is exhausted).+    # Rule in words: ascending time-order, stop when a new record's+    # timestamp is greater than the frontier (nothing later can qualify).+    # Iter1: written={} so first page cannot overlap; ""/0 are valid cursors.+    written = set()+    frontier = stop_timestamp+    resume_cursor = start_cursor+    ordered = []     while True:-        resp = client.fetch(partition_id, cursor)-        records = resp.get("records", [])-        for r in records:-            if r["timestamp"] > stop_timestamp:-                return result-            off = r["offset"]-            if off not in seen:-                seen.add(off)-                result.append(r)-        nxt = resp.get("next_cursor")-        if nxt is None:-            return result-        if not resp.get("has_more"):-            return result-        if nxt in visited:-            return result-        visited.add(nxt)-        cursor = nxt+        page = client.fetch(partition_id, resume_cursor)+        records = page.get("records", [])+        for rec in records:+            offset = rec["offset"]+            if offset in written:+                continue+            written.add(offset)+            if rec["timestamp"] > frontier:+                return ordered+            ordered.append(rec)+        next_cursor = page.get("next_cursor")+        has_more = page.get("has_more")+        if next_cursor is None or not has_more:+            break+        resume_cursor = next_cursor+    return ordered```

## Blind-judge summary

One fixed prompt (`judge_template.txt`: correctness > robustness > clarity;
must name a concrete divergent sequence; final line `VERDICT: A|B|TIE`).
Labels shuffled per test with the seed sealed from the judge; fresh model
session per judgment; judgments run strictly sequentially.

Verdict: SKILL (A; sealed A=skill, seed 33).

Judge: skill wins on watermark-vs-dedup ordering: base checks ts>stop BEFORE dedup, so a redelivered offset with a larger timestamp triggers early exit and loses a qualifying record; skill dedups first. Base's cursor-visited cycle guard also risks data loss on legitimate cursor reuse. Divergent case quoted with offsets [a,b] vs [a].

Full judge transcript:

```
Input `start_cursor(""|int)` -> `client.fetch(pid,cursor)` -> `{records[{id,ts,offset}], next_cursor, has_more}` -> dedup by `offset` (first-seen order) -> filter `ts <= stop` with early-exit on time-order -> paginate via `next_cursor/has_more` -> return `ordered[]`.

Candidate A:
(a) Correctness: dedups before watermark check, then `ts > stop => return`. Preserves first-seen order, handles `""`/`0` via `is None` check, stops on `None` or `!has_more`. Matches spec.
(b) Robustness: survives redelivery for watermark purposes; handles empty pages; uses `.get()` defaults. No cycle guard, so pathological `has_more=True` + fixed cursor loops forever.
(c) Clarity: invariant comment states stop rule; `written/frontier/resume_cursor/ordered` explicit.

Candidate B:
(a) Correctness: checks `ts > stop => return` before dedup. Redelivered offset can trigger exit even though spec says keep-once/ignore. Otherwise same pagination/order logic.
(b) Robustness: same falsy/empty handling as A. Adds `visited` cursor set: avoids infinite loop on stuck cursor, but truncates on legitimate cursor reuse (empty-page retry, overlapping at-least-once pages) causing data loss. Extra unbounded state, unexplained.
(c) Clarity: concise but `visited` stop condition undocumented; `seen/result/nxt` less explicit than A.

Diverging sequence (`stop=10`):
`fetch(pid,"")={"records":[{"id":"a","timestamp":5,"offset":0}],"next_cursor":1,"has_more":True}`
`fetch(pid,1)={"records":[{"id":"a2","timestamp":20,"offset":0},{"id":"b","timestamp":6,"offset":1}],"next_cursor":None,"has_more":False}`
Same offset reappears (allowed) with larger timestamp; `b` still qualifies and appears after. A skips `offset:0` as seen, appends `b` => `[a,b]`. B hits `20>10` before dedup => returns `[a]`, loses `b`. A survives. Second divergence: `fetch(c)=([],c,True)` stuck cursor — B returns via `visited`, A loops; B wins liveness there but that guard also drops data on transient empty-page retries, so net negative.

A is correct on watermark-vs-dedup ordering where B can early-exit on a duplicate that must be ignored. B's cycle guard trades possible data loss for loop protection not in the spec. A is also better documented.
VERDICT: A
```
