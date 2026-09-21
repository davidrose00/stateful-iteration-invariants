# Benchmark Report: TTL cache + singleflight + errors + falsy + LRU + invalidate

Model: `muse-spark-1.3`, high effort, `max-model-steps 12`, trusted workspace.
Date: 2026-09-21. Harness: `/tmp/standard-bench/ours/` (p2.base/skill.txt,
p2_score.py, p2.base/skill.out, judge_template.txt, p2.judge.txt/.map/.out).

Skill: `stateful-iteration-invariants` v4.0.0. Base arm ran with the skill
disabled in the CLI; skill arm got the full 124-line body injected above the
identical vague ticket. Task text itself was the same for both arms.

## Task (vague ticket, interfaces concrete / edge rules unstated)

Put a cache in front of `load_user(id)`.

Required interface:

```python
def make_loader_cache(loader, ttl=60, maxsize=128, clock=None):
    """loader(key) -> value. clock() -> float seconds, defaults to time.monotonic.
    Returns `cached(key)` with extra methods `cached.invalidate(key)` and
    `cached.clear()`."""
```

Rules: entries expire after `ttl` seconds; concurrent requests for the same key
share a single load; failed loads leave nothing cached; memory is bounded —
past `maxsize` entries, evict least-recently-used; invalidation drops one key,
clear drops everything. Different keys are independent.

Constraints: stdlib only. Output ONLY a single ```python code block. No prose.

## Hidden fixture + property checks

Fake clock; expiry valid@59s gone@61s; 10-thread barrier stampede = 1 load; error retry; falsy 0 cached; maxsize=2 LRU eviction order; invalidate/clear reload counts.

## Functional result: 6/6 base, 6/6 skill (tie)

## DIFF of the two code outputs (--- base / +++ skill)

```diff
--- base
+++ skill
@@ -1,5 +1,5 @@
-import collections import threading import time+from collections import OrderedDict  @@ -7,63 +7,81 @@
     if clock is None:         clock = time.monotonic-    cache = collections.OrderedDict()+    lock = threading.RLock()+    cache = OrderedDict()     inflight = {}-    lock = threading.Lock()++    class _Inflight:+        __slots__ = ("event", "value", "exc", "owner")+        def __init__(self, owner):+            self.event = threading.Event()+            self.value = None+            self.exc = None+            self.owner = owner      def cached(key):+        now = clock()         with lock:-            now = clock()             if key in cache:-                val, ts = cache[key]-                if ttl is None or now - ts < ttl:+                val, exp = cache[key]+                if exp is None or now < exp:                     cache.move_to_end(key)                     return val                 else:                     del cache[key]-            rec = inflight.get(key)-            if rec is None:-                rec = {"event": threading.Event(), "value": None, "exc": None, "ok": False}-                inflight[key] = rec-                is_owner = True+            call = inflight.get(key)+            if call is not None:+                if call.owner == threading.get_ident():+                    # Re-entrant load on same thread: bypass dedup to avoid deadlock.+                    is_loader = False+                    is_follower = False+                else:+                    is_loader = False+                    is_follower = True             else:-                is_owner = False-        if not is_owner:-            rec["event"].wait()-            if rec["ok"]:-                return rec["value"]-            raise rec["exc"]+                call = _Inflight(threading.get_ident())+                inflight[key] = call+                is_loader = True+                is_follower = False++        if is_follower:+            call.event.wait()+            if call.exc is not None:+                raise call.exc+            return call.value++        if not is_loader:+            # Same-thread re-entry: direct load, no shared state touched.+            return loader(key)+         try:             value = loader(key)         except BaseException as e:             with lock:-                rec["exc"] = e-                if inflight.get(key) is rec:-                    del inflight[key]-                rec["event"].set()+                call.exc = e+                inflight.pop(key, None)+                call.event.set()             raise-        stored_ts = clock()+         with lock:-            if maxsize is None or maxsize > 0:-                if ttl is not None:-                    for k2, (_v2, t2) in list(cache.items()):-                        if stored_ts - t2 >= ttl:-                            try:-                                del cache[k2]-                            except KeyError:-                                pass-                if key in cache:-                    try:-                        del cache[key]-                    except KeyError:-                        pass-                cache[key] = (value, stored_ts)-                if maxsize is not None:-                    while len(cache) > maxsize:-                        cache.popitem(last=False)-            rec["value"] = value-            rec["ok"] = True-            if inflight.get(key) is rec:-                del inflight[key]-            rec["event"].set()+            try:+                exp = None if ttl is None else clock() + ttl+            except BaseException:+                call.value = value+                inflight.pop(key, None)+                call.event.set()+                return value+            if maxsize is not None and maxsize <= 0:+                call.value = value+                inflight.pop(key, None)+                call.event.set()+                return value+            cache[key] = (value, exp)+            cache.move_to_end(key)+            if maxsize is not None:+                while len(cache) > maxsize:+                    cache.popitem(last=False)+            call.value = value+            inflight.pop(key, None)+            call.event.set()         return value ```

## Blind-judge summary

One fixed prompt (`judge_template.txt`: correctness > robustness > clarity;
must name a concrete divergent sequence; final line `VERDICT: A|B|TIE`).
Labels shuffled per test with the seed sealed from the judge; fresh model
session per judgment; judgments run strictly sequentially.

Verdict: SKILL (A; sealed A=skill, seed 22).

Judge: skill wins. Both meet the contract, but base self-deadlocks on same-thread re-entrant load and hangs followers if store-time clock() raises (leaked inflight, never-set Event); skill uses owner tracking + RLock + clock-outside-lock and degrades gracefully. No case found where base survives and skill fails.

Full judge transcript:

```
Assumptions: `key` hashable, `value` arbitrary (including falsy/`None`), `ttl` numeric/`None`/`0`, `maxsize` int/`None`, `clock` monotonic float source, `loader` may raise, may re-enter, may be slow.

Data flow, both candidates:

1. `cached(key)` -> acquire `lock` -> `clock()` lookup -> `OrderedDict` hit with TTL check + `move_to_end` -> return.
2. Miss -> register/join per-key `inflight` record -> release `lock`.
3. Owner calls `loader(key)` outside `lock`; followers `Event.wait()`.
4. Owner re-acquires `lock` -> `clock()` store timestamp -> `cache[key]=...`, `move_to_end`, `popitem(last=False)` while over `maxsize` -> publish to `inflight` record -> `Event.set()` -> return/raise.
5. `invalidate/clear` -> `lock` -> `cache.pop/clear` only.

(a) Correctness vs stated rules:

- Both: TTL expiry, single-flight dedup for distinct threads, no cache on failure, LRU via `OrderedDict` + `move_to_end`/`popitem(last=False)`, `invalidate`/`clear`, key independence, falsy-value safe (`key in cache`, `is None`/`ok` flag checks).
- Both leave in-flight value to be cached even if `invalidate`/`clear` interleaves; spec silent, tie.

(b) Robustness:

- Re-entrancy: A records `owner=get_ident()` and same-thread re-entry bypasses dedup with direct `loader(key)`. B uses plain `Lock` + no owner check; a `loader` that calls `cached(same-key)` becomes a follower on its own `Event` -> self-deadlock.
- `clock()` failure on store path: A wraps `clock()+ttl` in try/except, wakes followers with value uncached and returns value. B does `stored_ts=clock()` outside any try with no `Event.set()`/cleanup; owner raises clock error despite successful load, `inflight` entry leaks, any follower/second caller blocks forever.
- `clock` inside `lock`: B calls user `clock()` while holding non-reentrant `Lock` on every lookup plus an O(n) expiry sweep on every store; a blocking/re-entering `clock` deadlocks. A takes lookup `now` outside `lock` and uses `RLock`, no sweep.
- `ttl=None`/`maxsize=None`/`maxsize<=0`: both handle (never-expire/unbounded/coalesce-without-store).

(c) Clarity:

- A: explicit `_Inflight` slots class, `is_loader/is_follower` flags, comment on re-entrancy, O(1) lazy expiry. B: dict record with string keys, mixed timestamp vs expiry representation, unexplained full-cache sweep under lock.

Concrete divergence:

Sequence 1 - same-thread re-entry survives in A, hangs in B:

```python
cached = make_loader_cache(loader, ttl=60, maxsize=128)
state = {"re": False}
def loader(k):
    if k == "a" and not state["re"]:
        state["re"] = True
        return cached("a")  # re-entrant same-key load
    return 42
cached_inner = cached  # loader closes over cached
cached("a")
```

A: outer registers owner, inner sees `owner==get_ident()`, direct `loader("a")` -> `42`, outer caches/returns `42`. B: inner sees existing `rec`, `is_owner=False`, waits on its own unfinished `Event` -> deadlock/timeout.

Sequence 2 - store-time `clock` raises: fake `clock` returning `0` then raising once with two threads on same key. A returns loader value to owner+followers uncached; B owner raises clock error and followers hang on never-set `Event`, next call hangs.

I cannot find a case where B survives and A fails; A strictly dominates on hanging bugs.

Verdict justification: Both meet the happy-path contract, but B self-deadlocks on same-thread re-entry and leaks/hangs `inflight` followers if the store-time `clock()` raises, while A explicitly avoids the deadlock and degrades to uncached shared return. B's in-lock `clock()` calls and per-store sweep add contention/complexity without correctness benefit. Correctness then robustness both favor A; clarity slightly favors A.

VERDICT: A
```
