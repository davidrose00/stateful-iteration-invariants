# stateful-iteration-invariants

A skill for coding agents: a unified discipline for stateful loops and deep
exploration. Before writing any `while`, `for`, stop/skip/continue, or
recurse statement, it forces agents to name isolated invariant sets, state the
stop rule in words, hand-trace iteration 1 against seeded state, and verify
direction, ordering, depth-bounding, and falsy-safety.

Covers two dialects: **linear streams** (paging, sync cursors, backfills,
retries — `written` / `frontier` / `resume_cursor`) and **branching
hierarchies** (trees, ASTs, graphs — `seen_ids` / `worklist` /
`breath_limit` / `pause_trail` / `finding`).

The skill itself is [`SKILL.md`](SKILL.md) — drop-in for any
SKILL.md-compatible agent.

## Install

Claude Code — copy into your skills directory:

```bash
mkdir -p ~/.claude/skills/stateful-iteration-invariants
cp SKILL.md ~/.claude/skills/stateful-iteration-invariants/SKILL.md
```

Muse Code — import from your Claude skills:

```bash
muse skills import --from claude
```

## Does it help? Blind-judged benchmark

Tested on `muse-spark-1.3`, which scores high on instruction-following. 
Five realistic tickets (paginated ingestion, TTL cache, watermark
catch-up, tree flattener, crash-resumable sync), solved with and without the
skill by `muse-spark-1.3` at high effort, then scored by a second model
session that is blind to everything: same fixed rubric every test
(correctness > robustness > clarity, must name a concrete divergent input,
verdict `A`/`B`/`TIE`), labels shuffled per test. Full reports with diffs,
fixtures, and judge transcripts live in [`benchmarks/`](benchmarks/)
(`REPORT_p1`/`p2`/`p3`/`p4`/`p5` — p3 is the resume-sync task).

| Task | Blind judge |
|---|---|
| p1 paginator + redelivery | **skill** (exact-type catch, `None`-safe identity) |
| p2 cache + LRU + stampede | **skill** (no re-entry deadlock, no hung followers on clock failure) |
| p4 watermark catch-up | **skill** (dedup-before-watermark; base exits early on redelivered offsets) |
| p5 tree flattener | **skill** (symlink-alias semantics + type guards) |
| resume-sync (earlier) | **skill** |

Score: skill 5, base 0.
