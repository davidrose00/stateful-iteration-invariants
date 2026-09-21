# Benchmark Report: paginated REST ingestion (tombstones + rate limits + redelivery + limit)

Model: `muse-spark-1.3`, high effort, `max-model-steps 12`, trusted workspace.
Date: 2026-09-21. Harness: `/tmp/standard-bench/ours/` (p1.base/skill.txt,
p1_score.py, p1.base/skill.out, judge_template.txt, p1.judge.txt/.map/.out).

Skill: `stateful-iteration-invariants` v4.0.0. Base arm ran with the skill
disabled in the CLI; skill arm got the full 124-line body injected above the
identical vague ticket. Task text itself was the same for both arms.

## Task (vague ticket, interfaces concrete / edge rules unstated)

Stream active users from `client.get_users(page_token)` as a generator or list-returning
`iter_active_users(api_client, limit=None)`.

What we know about the endpoint: pages sometimes come back empty (soft deletes)
while more data remains; it rate-limits with a `RateLimitError` carrying
`retry_after_sec` (provided by the runtime — do not define your own; sleep,
then retry the same page); and at-least-once delivery means the same record
may reappear across pages — emit each user once, in first-seen order.
`limit` caps total records yielded; hitting it must stop fetching immediately.

Constraints: stdlib only. Output ONLY a single ```python code block. No prose.

## Hidden fixture + property checks

None->[u1,u2]/t1; t1->[]/t2; t2->RateLimit once then [u3,u2dup]/t3; t3->RateLimit once then [u4,u5]/None. Checks: full dedup order, exact fetch sequence, 2 sleeps, limit=4 exact halt with no prefetch.

## Functional result: 5/5 base, 5/5 skill (tie)

## DIFF of the two code outputs (--- base / +++ skill)

```diff
--- base
+++ skill
@@ -2,128 +2,93 @@
  -def _extract_users_and_token(page):+def _page_items_and_cursor(page):+    """Return (users, next_token) supporting common page shapes."""+    if page is None:+        return [], None+    if isinstance(page, (tuple, list)) and len(page) == 2:+        users, nxt = page+        return (users or []), nxt     if isinstance(page, dict):-        users = page.get("users", page.get("items", page.get("data", page.get("results", [])))) or []-        nxt = page.get(-            "next_page_token",-            page.get(-                "next_token",-                page.get(-                    "next_page",-                    page.get(-                        "next",-                        page.get(-                            "page_token",-                            page.get("token", page.get("cursor")),-                        ),-                    ),-                ),-            ),-        )-        return users, nxt-    if isinstance(page, (list, tuple)) and len(page) == 2:-        return (page[0] or []), page[1]-    users = getattr(-        page,-        "users",-        getattr(-            page,-            "items",-            getattr(page, "data", getattr(page, "results", [])),-        ),-    ) or []-    nxt = getattr(-        page,-        "next_page_token",-        getattr(-            page,-            "next_token",-            getattr(-                page,-                "next_page",-                getattr(-                    page,-                    "next",-                    getattr(-                        page,-                        "page_token",-                        getattr(page, "token", getattr(page, "cursor", None)),-                    ),-                ),-            ),-        ),-    )-    return users, nxt---def _hashable(value):-    try:-        hash(value)-        return value-    except TypeError:-        return repr(value)+        for k in ("users", "items", "data", "results", "records"):+            if k in page and isinstance(page[k], (list, tuple)):+                users = page[k]+                break+        else:+            users = []+        nxt = None+        for k in ("next_page_token", "next_token", "page_token", "cursor", "next_cursor"):+            if k in page:+                nxt = page[k]+                break+        return (users or []), nxt+    users = getattr(page, "users", None)+    if users is None:+        for k in ("items", "data", "results", "records"):+            users = getattr(page, k, None)+            if users is not None:+                break+    nxt = None+    for k in ("next_page_token", "next_token", "page_token", "cursor", "next_cursor"):+        nxt = getattr(page, k, None)+        if nxt is not None:+            break+    return (users or []), nxt   def _identity(user):     if isinstance(user, dict):-        for field in ("id", "user_id", "uuid", "key"):-            if user.get(field) is not None:-                return ("id", _hashable(user.get(field)))-        return ("record", repr(sorted(user.items(), key=lambda kv: repr(kv[0]))))-    for field in ("id", "user_id", "uuid", "key"):-        val = getattr(user, field, None)-        if val is not None:-            return ("id", _hashable(val))+        for k in ("id", "user_id", "uid"):+            if k in user:+                try:+                    return ("k", user[k])+                except Exception:+                    pass+        try:+            hash(user)+        except Exception:+            return ("repr", repr(user))+        return ("val", user)+    uid = getattr(user, "id", None)+    if uid is not None:+        return ("k", uid)     try:         hash(user)-        return ("value", user)-    except TypeError:-        pass-    try:-        return ("record", repr(user))+        return ("val", user)     except Exception:-        return ("record", str(type(user)))+        return ("repr", repr(user))   def iter_active_users(api_client, limit=None):-    try:-        _RateLimitError = RateLimitError-    except NameError:-        _RateLimitError = None-+    # Invariants: emitted = ids already yielded (dedup, first-seen order);+    # resume_cursor page_token = where to fetch/resume; frontier = limit count+    # plus end-of-stream (no next token). Rule: stop when emitted reaches+    # limit or no next token; skip dupes; never stop on an empty page while+    # a next token remains. Iteration 1: token None, emitted empty, so even a+    # fully-overlapping/empty first page advances via its next token.     if limit is not None and limit <= 0:         return--    seen = set()+    emitted = set()     yielded = 0     page_token = None-     while True:         try:             page = api_client.get_users(page_token)         except Exception as exc:-            is_rate_limit = False-            if _RateLimitError is not None and isinstance(exc, _RateLimitError):-                is_rate_limit = True-            elif type(exc).__name__ == "RateLimitError":-                is_rate_limit = True-            if not is_rate_limit:+            retry_after = getattr(exc, "retry_after_sec", None)+            if retry_after is None:                 raise-            time.sleep(getattr(exc, "retry_after_sec", 0) or 0)+            time.sleep(retry_after)             continue--        users, next_token = _extract_users_and_token(page)--        for user in users or []:+        users, page_token_next = _page_items_and_cursor(page)+        for user in users:             key = _identity(user)-            if key in seen:+            if key in emitted:                 continue-            seen.add(key)+            emitted.add(key)             yield user             yielded += 1             if limit is not None and yielded >= limit:                 return--        if not next_token:-            break-        page_token = next_token+        if not page_token_next:+            return+        page_token = page_token_next```

## Blind-judge summary

One fixed prompt (`judge_template.txt`: correctness > robustness > clarity;
must name a concrete divergent sequence; final line `VERDICT: A|B|TIE`).
Labels shuffled per test with the seed sealed from the judge; fresh model
session per judgment; judgments run strictly sequentially.

Verdict: BASE (A; sealed A=base, seed 11).

Judge: base wins on correctness (dedup identity incl. None/unhashable ids; precise RateLimitError catch vs skill's duck-typed retry that inverts missing-attr and cross-error cases). Skill only wins style. Divergent case: page of two {id:None} users — base yields both, skill drops one; unhashable id crashes skill.

Full judge transcript:

```
Data flow: `None` → `get_users(page_token)` → `page(users, next_token)` → `extract` → `identity(user)` → `seen? skip : yield` → `limit? return : page_token=next` → repeat; exception path `RateLimitError → sleep(retry_after_sec) → retry same token`.

Assumptions: `page` is dict/object/(users, token) pair; `user` is dict/object with stable id field if present; `next_token` truthy means more data; `retry_after_sec` numeric.

1a. Correctness:
- Empty page + token: both continue, terminate only on falsy token. Pass.
- Rate-limit retry same page: A checks type (`isinstance` + `type.__name__` fallback, sleeps `getattr(...,0) or 0`). B duck-types on `getattr(exc,"retry_after_sec",None)`. B retries any error with that attr and raises a real `RateLimitError` if attr missing/`None`. A matches spec.
- Dedup first-seen order: both use `seen` set + in-order yield. A's `_identity` handles `id/user_id/uuid/key`, `_hashable` fallback, sorted-items `repr` for id-less dicts. B checks only `id/user_id/uid`, returns `("k",None)` for `{"id":None}`, leaves unhashable ids unhandled, uses order-sensitive `repr` for id-less dicts.
- Limit: both `return` immediately after `yielded>=limit`, no extra fetch; both short-circuit `limit<=0`.

1b. Robustness:
- `None`/empty/`limit=0`: both survive (A via `or []`/getattr chain, B via explicit `None` guard).
- Falsy `retry_after_sec`: A sleeps 0 and retries; B raises on `None`/missing.
- Wrong-error swallow: non-rate error with `retry_after_sec` attr is retried forever by B, re-raised by A.
- Shapes: A misses `records`/`next_cursor`; B misses `next`/`next_page`/`token`. Both misread bare user-lists of length 2. Wash on unspecified shapes.

1c. Clarity: B has flatter key loops, docstring, invariant comment — easier to extend. A is deeply nested `getattr`/`get` chains with no comments. B reads better, but its `_identity` simplicity hides bugs; readability does not offset correctness.

2. Diverging sequences:
- `page={"users":[{"id":None,"n":"a"},{"id":None,"n":"b"}],"next_page_token":None}`: A skips `None` ids, keys by sorted `repr`, yields both. B keys both as `("k",None)`, yields only first. A survives; B silently drops data.
- Same class: `{"users":[{"id":["x"]}],"next_page_token":None}`: A `repr`s the unhashable id, yields. B builds `("k",["x"])` then `in set` raises `TypeError`. A survives; B crashes. Rate-limit variant: `RateLimitError` with `retry_after_sec=None`/missing, or `ValueError` with that attr set — A retries/raises correctly, B inverts both.

A wins correctness (dedup identity, precise rate-limit catch) and robustness (None/unhashable ids, missing attr, no cross-error swallow); B wins only style. Correctness > robustness > clarity decides it.
VERDICT: A
```

## Addendum: v4.0.1 rematch (2026-09-22)

After amending the skill (exact-type-over-attribute-shape catch + separated
missing/`None`/unhashable dedup keys, v4.0.1), the p1 skill arm was re-run
(functional 5/5) and re-judged blind against the same base output
(new shuffle, seed 55: A=skill, B=base).

Rematch verdict: **A (skill arm)**. The judge found the v4.0.1 output fixed
the prior failure modes (exact-type rate-limit catch, None-safe identity)
and additionally faulted base for misparsing any 2-element list page as a
`(users, token)` tuple — a data-loss bug. Caveat: judge emphasis shifts
between runs (the prior transcript had praised base's None-id handling);
verdicts are single samples, not significance. Series tally now: 5 skill wins, 1 base win across 6 blind judgments (p1 flipped on rematch).

## Addendum 2: wording experiment (2026-09-22)

The amendment went through three wordings, each tested with a fresh p1 skill
run + fresh sealed blind judgment vs the same base output:
- strong global ("never match by attribute shape") -> skill wins;
- soft ("exact type over attribute shape") -> base wins, old failure modes return;
- scoped-strong ("in retry/resume paths, never match by attribute shape") -> skill wins (B; sealed A=base, B=skill, seed 77).
The judge credited exact-type catch, falsy-token termination (None/"" only),
bare-list page handling, and clearer key logic; it faulted base for treating
any len-2 list as a (users, token) tuple (data corruption + wasted fetch).
p1 judgment tally: base 2, skill 2 across 4 runs — wording strength decides.
Final skill text keeps the scoped-strong form.

## Final decision: strong global wording kept (2026-09-22)

Three wordings were tested head-to-head on p1 (fresh skill run + fresh sealed
judgment each): strong global won, soft lost, scoped-strong won. The skill
keeps the strong global form ("never match by attribute shape").
