# Benchmark Report: directory flattener (ternary predicate + symlink cycles + dangling + 20k depth)

Model: `muse-spark-1.3`, high effort, `max-model-steps 12`, trusted workspace.
Date: 2026-09-21. Harness: `/tmp/standard-bench/ours/` (p5.base/skill.txt,
p5_score.py, p5.base/skill.out, judge_template.txt, p5.judge.txt/.map/.out).

Skill: `stateful-iteration-invariants` v4.0.0. Base arm ran with the skill
disabled in the CLI; skill arm got the full 124-line body injected above the
identical vague ticket. Task text itself was the same for both arms.

## Task (vague ticket, interfaces concrete / edge rules unstated)

Flatten a virtual directory tree into retained node ids.

Required interface:

```python
def flatten_tree(root_node, prune_predicate):
```

- Nodes: `{"id": str, "is_dir": bool, "children": [...] | None,
  "target": str | None}`. A non-null `target` is a symlink: visit the node
  with that id. `children` is present when `is_dir` is true.
- `prune_predicate(node)` is someone else's code (source below — read its
  docstring carefully and honor every distinct return value):

```python
def prune_predicate(node):
    """Return True to discard the node and its whole subtree,
    False to keep the node and descend, 0 to keep the node
    but NOT descend into it."""
    nid = node["id"]
    if nid.startswith("skip_"):
        return True
    if nid == "bundle":
        return 0
    return False
```

- Depth-first, pre-order, children in listed order. Retain each visited node
  once (first visit wins) — links can point back at ancestors, and a link to
  a missing id is kept as a bare entry without descending.
- Branches can run tens of thousands deep with no fan-out: do not rely on
  the call stack. Return the retained ids in visit order.

Constraints: stdlib only. Output ONLY a single ```python code block. No prose.

## Hidden fixture + property checks

Fixture: skip_ subtree, bundle (0-keep-no-descend), docs with readme + up-link cycle, loop-link cycle, dangling ghost-link, file; plus 20k linear chain. Checks: exact visit order, 0-vs-False, pruning, 20002-node depth without RecursionError.

## Functional result: 4/4 base, 4/4 skill (tie)

## DIFF of the two code outputs (--- base / +++ skill)

```diff
--- base
+++ skill
@@ -1,52 +1,64 @@
 def flatten_tree(root_node, prune_predicate):-    # in: root dict -> index -> iterative pre-order -> predicate -> retained ids     if not isinstance(root_node, dict):         return []-    # id -> first object (iterative, no recursion)-    id_to_node = {}+    rid = root_node.get("id")+    if not isinstance(rid, str):+        return []+    # index: id str -> node dict (first wins, preorder); seen_objs: python-object cycle guard+    index = {}+    seen_objs = set()     stack = [root_node]     while stack:-        cur = stack.pop()-        if not isinstance(cur, dict):+        n = stack.pop()+        if not isinstance(n, dict):             continue-        nid = cur.get("id")-        if nid is None or nid in id_to_node:+        if id(n) in seen_objs:             continue-        id_to_node[nid] = cur-        kids = cur.get("children")-        if isinstance(kids, list):-            for ch in reversed(kids):-                if isinstance(ch, dict) and ch.get("id") not in id_to_node:-                    stack.append(ch)-    # depth-first pre-order, first visit wins, iterative+        seen_objs.add(id(n))+        nid = n.get("id")+        if isinstance(nid, str) and nid not in index:+            index[nid] = n+        ch = n.get("children")+        if isinstance(ch, list):+            for c in reversed(ch):+                if isinstance(c, dict) and id(c) not in seen_objs:+                    stack.append(c)+    # visited: string ids already processed (retained or pruned); retained: visit order+    visited = set()     retained = []-    seen = set()-    stack = [root_node]-    while stack:-        node = stack.pop()+    worklist = [root_node]+    while worklist:+        node = worklist.pop()         if not isinstance(node, dict):             continue         nid = node.get("id")-        if nid is None or nid in seen:+        if not isinstance(nid, str):             continue-        seen.add(nid)-        r = prune_predicate(node)-        if r is True:+        if nid in visited:             continue-        if r is False:-            retained.append(nid)-            kids = node.get("children") if node.get("is_dir") else None-            if isinstance(kids, list) and kids:-                for ch in reversed(kids):-                    if isinstance(ch, dict):-                        chid = ch.get("id")-                        if chid is None or chid not in seen:-                            stack.append(ch)-            tgt = node.get("target")-            if tgt is not None and tgt not in seen:-                tn = id_to_node.get(tgt)-                if tn is not None:-                    stack.append(tn)+        visited.add(nid)+        res = prune_predicate(node)+        if res is True:+            continue+        retained.append(nid)+        # stop rule in words: descend only when predicate says keep-and-descend+        if res is not False:+            continue+        target = node.get("target")+        if target is not None:+            if isinstance(target, str) and target not in visited:+                tnode = index.get(target)+                if tnode is not None:+                    worklist.append(tnode)         else:-            retained.append(nid)+            if node.get("is_dir"):+                children = node.get("children")+                if isinstance(children, list):+                    for child in reversed(children):+                        if not isinstance(child, dict):+                            continue+                        cid = child.get("id")+                        if isinstance(cid, str) and cid in visited:+                            continue+                        worklist.append(child)     return retained```

## Blind-judge summary

One fixed prompt (`judge_template.txt`: correctness > robustness > clarity;
must name a concrete divergent sequence; final line `VERDICT: A|B|TIE`).
Labels shuffled per test with the seed sealed from the judge; fresh model
session per judgment; judgments run strictly sequentially.

Verdict: SKILL (B; sealed B=skill, seed 44).

Judge: skill wins on symlink-alias semantics (target present means alias — ignore own children; base descends into both) plus type validation (non-string id/target guards) and id-less-subtree indexing. Two divergent cases: link-with-children (base retains shadowed kid), non-string id (base crashes in predicate).

Full judge transcript:

```
Input `root_node` -> index `id->node` -> iterative LIFO stack pre-order -> `prune_predicate(node)` gate -> `retained[]` in visit order.

Flow per node: pop, dedup by `id` (first wins), call predicate, `True`=drop+no descend, `False`=keep+descend, `0`=keep+no descend. `target!=None` means resolve via index; missing target means keep symlink bare. Both use explicit stacks, so 10k+ depth does not use call stack. Both distinguish `0` vs `False` with `is`, which is required since `0==False`.

1a. Correctness:
- A: `True` drops, `False` keeps and pushes children (gated on `is_dir`) plus `target`, `else` keeps with no descend. Keeps `0` correctly, DFS order correct.
- B: `True` drops, append then `is not False` stops descent for `0`, otherwise follows `target` xor `children` (gated on `is_dir`). Keeps `0` correctly, DFS order correct.
- Difference: A descends into both `children` and `target` when both present, visiting target first. B treats non-null `target` as symlink alias and ignores its own `children`. B matches "is a symlink: visit the node with that id" model; A retains extra `kid` subtree that should be shadowed.

1b. Robustness:
- Empty/non-dict root: both return `[]`.
- Non-string `id`: A allows e.g. `123` through to `prune_predicate`, where `nid.startswith` raises `AttributeError`. B checks `isinstance(nid,str)` before predicate and skips, returns `["root"]`.
- Non-string `target`, non-dict children, `None` children: B validates `isinstance(target,str)`; A does `tgt not in seen` + `index.get(tgt)` without type check.
- Indexing: A skips indexing subtree under `None`-id or duplicate-id node (`continue` before pushing kids). B uses `id(obj)` guard and still indexes descendants, so symlink to node under id-less wrapper resolves in B, misses in A.

1c. Clarity:
- A compact but `else: retained.append` hides the `0` case.
- B names `index/visited/worklist`, comments stop rule, makes `if res is not False: continue` and `target`-xor-`children` explicit.

2. Diverging input:
```python
root={"id":"root","is_dir":True,"children":[
 {"id":"link","is_dir":True,"children":[{"id":"kid","is_dir":False}],"target":"other"},
 {"id":"other","is_dir":False}],"target":None}
```
A retains `["root","link","other","kid"]`; B retains `["root","link","other"]`. B survives under symlink-alias semantics.

Second:
```python
root={"id":"root","is_dir":True,"children":[{"id":123,"is_dir":False}],"target":None}
```
A crashes in predicate; B returns `["root"]`. B survives.

B wins on correctness for symlink exclusivity, on robustness for type validation and indexing, and is more explicit. A is shorter but conflates alias with container and is unsafe on unexpected id shapes.
VERDICT: B
```
