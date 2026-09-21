---
name: stateful-iteration-invariants
description: Before writing stop, skip, continue, or descent logic for ANY incremental task — linear (paging, cursor/sync, backfill, retries, "stop when caught up") or hierarchical (trees, ASTs, recursive tool search, graph walks) — enumerate the isolated sets and trace iteration 1. Prevents overloaded-state, wrong-boundary, call-stack, infinite-cycle, and falsy-erasure bugs.
version: 4.0.1
---

# Stateful Iteration Invariants (Linear & Hierarchical)

A unified discipline for stateful loops and deep exploration. The sets, boundaries, and cursors below are **always implied** even when never explicitly named. 

Derive and isolate them as distinct variables *before* writing the first `while`, `for`, `stop`, `skip`, or `recurse` statement.

---

## 1. Rule 1: The Principle of Single-Meaning State

Never let one variable carry two distinct loop semantics. Enumerate your variables according to the structure you are traversing:

### Dialect A: Linear Streams (Paging, Sync, Cursors, Backfills)
- **`written` / `have`** — Already persisted or emitted. Never re-write or re-emit.
- **`frontier`** — The cutoff boundary (watermark, target count, oldest allowed timestamp, "caught up" condition).
- **`resume_cursor`** — Where to restart after a crash, pause, or rate limit (page token, offset, ID).

> **The Canonical Linear Bug:** Reusing a recent-dedup `seen` set as the historical processing `frontier`. They look interchangeable and are not.

### Dialect B: Branching Hierarchies (Trees, ASTs, Graphs, Directory Search)
- **`seen_ids` (Cycle Prevention)** — Set of visited object identities (`id(node)` or `WeakSet`). Prevents circular references. **Never** reuse this as the output/dedup set.
- **`worklist` (Frontier)** — Pending nodes/frames waiting to be entered (`stack` for DFS, `queue` for BFS).
- **`breath_limit`** — Traversal depth ceiling. When hit, you pause and surface rather than descending deeper.
- **`pause_trail`** — LIFO stack of ancestor frames `(node, resume_index, depth)` recorded while unwinding to surface.
- **`finding`** — Contextual target payload. Must strictly preserve valid falsy matches (`false`, `0`, `""`, `null`).

---

## 2. Rule 2: Verbalize the Invariant Before Writing Code

Never write a comparison operator (`<`, `>`, `<=`, `>=`) directly against a raw in-scope variable.
1. **State the rule in words first:**
   * *Linear:* "Stop when the page's oldest item timestamp is older than the sync cutoff."
   * *Hierarchical:* "Pause when the descent depth equals the breath limit."
2. **Bind the verbal rule to the named invariant set.**
3. **Verify Direction:** Ascending vs. descending changes the boundary comparison:
   * Ascending (`oldest → newest`): Advance cursor forward; stop when `item >= frontier`.
   * Descending (`newest → oldest`): Advance cursor backward; stop when `item <= frontier`.

---

## 3. Rule 3: Trace Iteration 1 Concretely (Against Seeded State)

Most loop bugs fire on pass 1. **Never reason about steady-state until you trace iteration 1 against the seeded state:**

### The Linear Overlap Trap
Ask: What does the very first fetch/batch contain, and what is `seen` or `written` seeded to at that exact instant?
* If walking newest→oldest and `seen` is seeded with the latest items, **the first page fully overlaps**. Any naive "stop when we hit seen" check fires immediately and the loop never advances.

### The Tree Root-Suppression Trap
Ask: What happens to the root node at $T=0$?
* If `seen_ids` is seeded with `{id(root)}` for cycle prevention, does the child inspection loop mistake the root for an already-completed branch and skip descending?

### The Zero-Depth / Flat Root Guard
* If the root is flat, empty, or primitive (`null`, `[]`, `{}`): immediately abort and report nothing found.
* If `breath_limit <= 1`: verify pass 1 pauses cleanly without illegally descending into child 0.

---

## 4. Rule 4: The Agentic Exploration Protocol (For Hierarchies)

When exploring deep, unfamiliar composite spaces:

1. **Submersion is Temporary (Anti-Rabbit-Holing):**
   * Descend along a single branch with a clear trail.
   * If you find what you seek OR reach the `breath_limit`, **step back out to the surface immediately**. Never wander horizontally in the depths.
2. **Deepest-Point-First Resumption:**
   * When breath runs out, record checkpoints down the stack into `pause_trail`.
   * Prepend `pause_trail` to the **front** of `worklist` in reverse order (LIFO).
   * Always resume from the deepest paused point first to finish and prune innermost local contexts before working memory degrades.
3. **Deterministic Branch Enumeration:**
   * **Ordered (Arrays):** Iterate sequential indices `resume_index` to `length - 1`.
   * **Unordered (Objects/Maps):** **Sort keys lexicographically** before indexing so index-based cursors never skip or duplicate keys across pauses.
4. **LIFO Stack Order:**
   * When using `stack.pop()`, push children in **reversed order** (`reversed(children)`) to preserve natural left-to-right traversal.
5. **Trust the Void (Falsy Preservation):**
   * If a match evaluates to `false`, `0`, `""`, or `null`, that **is** the finding. Never discard an answer or continue searching because a match is falsy. Describe it in its context and surface.

---

## 5. Red Flags (Stop and Re-derive)

Stop immediately and re-derive your sets if you observe any of the following:

- [ ] One variable is used for both "already written" and "where to stop."
- [ ] A stop or skip condition was added by grabbing whatever variable was nearest in scope.
- [ ] You reasoned about steady-state but never hand-traced pass 1 with seeded values.
- [ ] A global integer counter is used to track depth in a branching traversal (must be frame-bound).
- [ ] Children were pushed $0..N$ to a LIFO stack while expecting $0$ to pop first.
- [ ] Unordered dictionary keys were iterated by index without sorting.
- [ ] A truthiness check (`if node.value:`) was used instead of an explicit presence check (`is not None`).
- [ ] An exception handler matches on attribute shape (`getattr(exc, ...)` instead of `isinstance`) — it retries foreign errors and raises real ones.
- [ ] An identity/dedup key conflates missing with `None`, or assumes hashability — distinct items must never share a key; fall back to `repr`.

---

## 6. Debugging Protocol: Reason Before Telemetry

When any stateful loop or traversal fails:
1. **Never add print statements or fetch remote logs first.**
2. **Reconstruct Iteration 1 on paper:**
   * Write down the seeded values of each named set at $T=0$.
   * Evaluate the first loop boundary condition by hand using those exact seeds.
   * *Over 90% of premature exits, infinite sync loops, and skipped root branches appear here.*
3. **Trace the First Boundary Crossing:** Hand-trace the first item that matches the watermark, hits the cutoff, or reaches the `breath_limit`. Verify that it terminates, unwinds, or records cursor state cleanly.

---

## 7. Pre-Flight Checklist

*Emit this mental checklist before writing or running any stateful loop:*

- [ ] **Sets Named & Isolated:** Every set (`written`, `frontier`, `resume_cursor` OR `seen_ids`, `worklist`, `pause_trail`, `finding`) has a distinct name and exactly one meaning.
- [ ] **Rule Stated in Words:** Stated the stop condition verbally before writing comparison operators.
- [ ] **Iteration 1 Hand-Traced:** Walked pass 1 against seeded values; verified no premature termination on initial batch, seed overlap, or root node.
- [ ] **Direction & Order Verified:**
  - Linear: Sort direction matches step direction and boundary comparisons (`<` vs `>`).
  - Hierarchical: Stack pushes reversed for LIFO; object keys sorted before index access.
- [ ] **Frame-Bound Depth:** Tree depth is enclosed inside frame objects, never a global counter.
- [ ] **Falsy-Safe:** Uses explicit comparisons (`!== undefined`, `is not None`), preserving `0`, `false`, and `""`.
- [ ] **Exact Catch & Keys:** Errors caught by exact type, never attribute shape; dedup keys separate missing/`None`/unhashable.
- [ ] **Surfacing Rule Enforced:** In trees, immediately surface upon hit or breath exhaustion.