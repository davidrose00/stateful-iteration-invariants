# Benchmark Report: crash-resumable paginated sync (complex + vague)

Model: `muse-spark-1.3`, high effort, `max-model-steps 12`, trusted workspace.
Date: 2026-09-21. Artifacts: `/tmp/standard-bench/ours/`
(resume.base.txt, resume.skill.txt, resume_score.py, resume.*.out,
judge_template.txt, resume.judge.txt/.map/.out).

Skill: `stateful-iteration-invariants` v4.0.0. Base arm ran with the skill
disabled in the CLI (no description in context); skill arm got the full
124-line body injected above the identical task text.

## Task prompt (identical for both arms, vague ticket style)

Copy every user record from a paged API into `store` (dict id->record).
Worker gets killed mid-run, restarted with the same `store`; restarts must
finish with no gaps/duplicates. Interface: `run_worker(api, store,
max_pages=None)`. Pages may be empty while data remains. `RateLimitError`
(with `retry_after_sec`) is provided — sleep then retry. Bookkeeping keys
under `"_"` allowed. Nothing else specified: no resume protocol, no retry
budget semantics, no completion-marker convention.

## Hidden fixture (model never sees this)

`None -> [u1,u2]/t1; t1 -> []/t2 (tombstone); t2 -> RateLimitError once,
then [u3]/t3; t3 -> [u4]/None.`
Kill simulated by `run_worker(api, store, max_pages=2)`, then restart
uncapped. `time.sleep` patched/recording.

## Property scorer (6 checks)

S1 progress before kill == {u1,u2} | S2 complete after restart == {u1..u4} |
S3 restart fetches only within {t2,t3} (no refetch) | S4 final page reached |
S5 total fetches <= 10 | S6 rate limit slept.

## Functional result: 6/6 vs 6/6 (tie)

Both arms: phase-2 fetch sequence exactly `[t2, t2, t3]`, 5 total fetches,
one sleep of 0.01. Pass/fail cannot separate them.

## DIFF of the two code outputs (base 47 lines vs skill 58 lines)

```diff
--- base
+++ skill
@@
-# flow: cursor(store) -> api.get_users(cursor) -> items -> store[id] -> next_cursor(store) -> done
 def run_worker(api, store, max_pages=None):
@@
-    cursor = store.get("_next_page_token")
-    fetched = 0
+    cursor = store.get("_cursor", None)
+    pages_fetched = 0
     while True:
-        if max_pages is not None and fetched >= max_pages:
+        if max_pages is not None and pages_fetched >= max_pages:
             return
-        try:
-            if cursor is None:
-                try:
-                    page = api.get_users(None)
-                except TypeError:
-                    page = api.get_users()
-            else:
-                page = api.get_users(cursor)
-        except Exception as exc:
-            retry_after = getattr(exc, "retry_after_sec", None)
-            is_rate_limit = type(exc).__name__ == "RateLimitError" or retry_after is not None
-            if not is_rate_limit:
-                raise
-            time.sleep(retry_after if retry_after else 0)
-            continue
-        items = page.get("items", []) if isinstance(page, dict) else []
+        while True:
+            try:
+                result = api.get_users(cursor)
+                break
+            except Exception as exc:
+                if exc.__class__.__name__ != "RateLimitError":
+                    raise
+                delay = getattr(exc, "retry_after_sec", 0)
+                try:
+                    delay = float(delay)
+                except (TypeError, ValueError):
+                    delay = 0
+                if delay < 0:
+                    delay = 0
+                time.sleep(delay)
+        pages_fetched += 1
+        items = result.get("items", []) if isinstance(result, dict) else []
+        if items is None:
+            items = []
         for item in items:
@@
                     uid = item["user_id"]
-                elif "userId" in item:
-                    uid = item["userId"]
+                elif "uid" in item:
+                    uid = item["uid"]
                 else:
                     continue
-            store[uid] = item
-        nxt = page.get("next_page_token") if isinstance(page, dict) else None
-        fetched += 1
-        store["_next_page_token"] = nxt
-        if nxt is None:
+                rec = item
+            elif isinstance(item, (list, tuple)) and len(item) == 2:
+                uid, rec = item
+            else:
+                continue
+            try:
+                present = uid in store
+            except TypeError:
+                uid = str(uid)
+                present = uid in store
+            if not present:
+                store[uid] = rec
+        next_token = result.get("next_page_token") if isinstance(result, dict) else None
+        if next_token is None:
             store["_done"] = True
             return
-        cursor = nxt
+        store["_cursor"] = next_token
+        cursor = next_token
```

Reading: same architecture (token checkpoint under `_` keys, done flag,
sleep-then-retry same cursor). Base is shorter with a zero-arg-call fallback
and multi-spelling id lookup, but a broad retry predicate and raw
`sleep(retry_after or 0)`. Skill adds an inner retry loop, delay
coercion/clamping, `None`-items normalization, unhashable-uid fallback, and
no-overwrite-if-present.

## Blind-judge summary (methodology + verdict)

One fixed prompt (`judge_template.txt`, slots TASK/A/B) for every test:
correctness first, robustness to implied-but-unstated edges second, clarity
third; must name a concrete divergent input sequence; final line
`VERDICT: A|B|TIE`. Labels shuffled per test (seed 3 here: A=skill, B=base),
mapping sealed from the judge; judge model = fresh `muse-spark-1.3` session.

Verdict: **A (skill arm)**. Core loops judged tied; A wins on surviving
`items=None`, malformed `retry_after_sec`, and unhashable uids where B
crashes. B credited for the zero-arg fallback and brevity; judged not enough
to outweigh crash paths. Decisive divergent case quoted: page
`{"items": None, "next_page_token": None}` — A normalizes and finishes,
B raises `TypeError`.

## Conclusion

Functional scoring: tie (6/6, 6/6). Blind robustness judging: skill wins.
The arms differ observably, and the difference is hardening depth.
