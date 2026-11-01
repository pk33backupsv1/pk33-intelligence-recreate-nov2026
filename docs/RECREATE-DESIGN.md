# RECREATE-DESIGN.md — target architecture for the pk33 intelligence layer

**Status:** proposal · **Date:** 2026-11-02 (IST) · **Author:** the intelligence seat

This is a **design**, not an implementation. It states what the layer should
be, and the four failures it must not repeat. Every "must" traces to a
measured failure.

---

## 0. The four failures this design exists to prevent

| # | Failure | Measured | Rule |
|---|---|---|---|
| F1 | A scope that can never be satisfied | `ok=3 red=45` on **every** run | R1 — a perpetual RED is a design failure |
| F2 | Derived output entering its own index | 17,701 graphify files in the index (`fd5fa4a83`) | R2 — structurally impossible, not a prune job |
| F3 | An unbounded build | 738 s/pass against a 10-min cron | R3 — every stage gets a budget |
| F4 | A scope that earns nothing | a 2.2 GB Flutter SDK indexed as knowledge | R4 — a scope must earn its place |

---

## 1. The scope policy (fixes F1, F4)

### 1.1 A scope is a *claim*, and it must be answerable

Today: scopes come from `clones.tsv` — "whatever this box has checked out".
That is why a Flutter SDK qualified.

**Proposed rule:** a scope qualifies only if it passes **all four** gates.

| Gate | Test | Rationale |
|---|---|---|
| **G1 — is it ours?** | the path is under a registered fleet workspace root, **and** is not a vendor SDK / package manager store / cache | the Flutter SDK and `node_modules` fail this |
| **G2 — does it have content?** | ≥ 1 indexable file (`.md`/`.tsv`/`.txt`) after prune rules | an empty scope contributes 0 documents but a scope row |
| **G3 — will it be fresh?** | the scope has a commit within a stated window (default 30 d) | a scope that will never be committed to cannot pass a freshness DoD → **F1** |
| **G4 — is it indexable at all?** | its pruned file count is below a cap (default 20 k files) | bounds the per-scope build time → **F3** |

**G3 is the important one.** The old DoD was *"graph.json younger than 48 h"*.
A repo nobody commits to can never satisfy that — so it is RED forever, and
the loop runs forever to prove it. **A scope that cannot pass its own DoD must
be removed from the scope list, not retried.**

### 1.2 Scope states, and what each means

```
DISCOVERED  → passes G1+G2, not yet built
ELIGIBLE    → passes all four gates
BUILT       → has a current artifact (graph.json + indexed docs)
STALE       → was ELIGIBLE, has not been committed to within the window
RETIRED     → fails G1 (deleted, or moved to trash/fvm/etc.)
```

**A scope is never `RED`.** Red is a *build* verdict, not a *scope* verdict.
This is the structural fix for F1: the old design conflated "this scope could
not be built" with "this scope is in the list", so a permanent failure looked
like a permanent work item.

### 1.3 The scope file becomes a reviewed artefact

`scopes.tsv` becomes **generated but committed**, with the gate verdicts as
columns, so a scope's status is reviewable in git rather than recomputed
forever:

```
scope	root	gates	status	last_commit	built_at	artifact
zsm1	/home/pk33/zexp1/efus3/zsm1	G1,G2,G3,G4	PASS	ELIGIBLE	2026-09-18	—	—
efus3	/home/pk33/zexp1/efus3	G1,G2,G3	G3=FAIL	STALE	2026-10-30	—	—
claude-memory	/home/pk33/.claude/projects/*/memory	G1,G2	G4=PASS	PASS	ELIGIBLE	2026-11-01	—	—
```

**Note `claude-memory` must keep its name.** `zk_embed.py` filters on
`scope != 'claude-memory' OR path LIKE '%/memory/%'` — renaming the scope
would silently break the filter and start embedding non-memory files. This is
a coupling to preserve, not a coincidence.

---

## 2. Structural isolation of derived output (fixes F2)

### 2.1 The rule

> **The indexer must not be able to see the indexer's own output directory.**

The current fix is a name in `PRUNE_DIRS`:
`"graphify-out",  # graphify-out = derived code graph (A-16), never knowledge`.

That is a **repair of a structural feedback loop** — it works until someone
adds a new derived directory. `fd5fa4a83` exists because a name was missing.

### 2.2 The proposed fix — a mount-namespace or path-denominator rule

**Rule:** the indexer computes a **denominator** per scope — the set of
directories it is permitted to descend into — and **excludes every path that
is not under the scope root** *and* **excludes every path matching a
"derived output" registry**.

```python
DERIVED_OUTPUT_REGISTRY = [
    (".git",                "version control metadata"),
    ("node_modules",        "dependency store"),
    (".venv", "venv",       "python env"),
    ("dist", "build",       "build output"),
    (".cache",              "cache"),
    ("graphify-out",        "A-16 derived code graph"),
    ("tool-results",        "claude session artefacts"),
    ("shell-snapshots",     "claude session artefacts"),
    ("ssot_engine_*/{dc,rc,sym}", "data spines — not knowledge"),
]
```

**And the registry is the point** — it is a *named, reviewed* list with a
reason per entry, not a set of strings in a loop. When a new derived
directory is introduced, adding it to the registry is a **reviewable diff**.

### 2.3 A self-check that runs on every build

The build must **assert** that it did not index anything derived:

```bash
# after a build, assert zero derived paths in the index
sqlite3 index.sqlite "
  SELECT count(*) FROM docs
  WHERE path LIKE '%/graphify-out/%'
     OR path LIKE '%/node_modules/%'
     OR path LIKE '%/.git/%'
     OR path LIKE '%/ssot_engine_%/(dc|rc|sym)/%';
"
# must be 0
```

**This is the difference between a repair and a guarantee.** The old system
had the check *implicitly* (a prune list) and *reactively* (17,701 files got
in, then a human noticed). The new system has it **explicitly** and
**assertively**.

---

## 3. Bounded runtime (fixes F3)

### 3.1 Every stage gets a budget

| Stage | Budget | Hard ceiling | On exceed |
|---|---|---|---|
| scope scan | 30 s | 120 s | fail the scope, not the run |
| per-scope walk + index | 60 s | 300 s | fail the scope, record why |
| per-scope graphify | 120 s | 300 s | fail the scope, record why |
| embed pass | 300 s | 900 s | fail the *run*, keep the previous index |
| **total build** | **10 min** | **20 min** | abort; the previous index stays valid |

**Key change: a stage failure is scoped.** In the old design, one bad scope
consumed the whole budget and the run produced `ok=3 red=45` — a *global*
verdict computed from a *partial* success.

### 3.2 The build is incremental and idempotent

- A scope that is `BUILT` and whose content hash is unchanged is **skipped**.
- A scope that fails is **not retried** until its input changes (content hash
  or commit date). **Retry-on-schedule is what made the old loop immortal.**
- The build records `built_at` and the content hash per scope.

### 3.3 The DoD becomes satisfiable

**Old DoD:** `graph.json` younger than 48 h — *unsatisfiable* for a repo
nobody commits to, hence F1.

**New DoD, two parts:**

| Part | Rule | Satisfiable? |
|---|---|---|
| **per-scope** | the scope is `BUILT` (its artifact is newer than its content hash) | yes — always, once built |
| **per-run** | the run completed within budget and the self-check (§2.3) returned 0 derived paths | yes |

**Freshness becomes informational, not a gate.** A scope that has not been
committed to in 30 days becomes `STALE` and is *retired from the build* — it
does not generate a RED, because it is no longer in the build.

---

## 4. Placement (the one thing worth carrying over from the old design)

The old design's **cost structure was fine**; only its control structure was
broken:

```
ZK EMBED: docs=35919 chunks=737128 new=34 changed=14 removed=0 secs=125.1
```

**125 seconds for 737,128 chunks.** The embed pass is cheap. `zk_graph.sh`
was the expensive one (738 s), and its output was consumed by nothing.

### 4.1 The index moves out of `_logs/`

`_logs/zk/index.sqlite` — a vector store under a logs directory. Nothing in
the path distinguishes "important derived data" from "rotted logs".

**Proposed:**

```
zdash222/data/zk/          <- the doc store + vectors (derived, rebuildable)
zdash222/docs/zk/          <- the scope file, the build receipts, the manifests (committed)
```

The distinction: **`data/` is derived and disposable; `docs/` is the record
and is committed.**

### 4.2 The embedding upstream must be local or explicitly healthy

The old design's embedding service is on **pk31:8095**, reached by an ssh
tunnel. pk31 has been offline for 4 days.

**Proposed:** the embedder must be **local to pk33** (or the build must
explicitly declare its upstream and check it before starting). A build that
depends on a remote host that may be down must **fail at the preflight**, not
half-run.

```bash
# preflight — before any scope is touched
curl -sf -m 10 "$ZK_EMBED_URL/v1/models" >/dev/null \
  || { echo "PREFLIGHT FAIL: embed upstream $ZK_EMBED_URL unreachable"; exit 2; }
```

**Today this preflight would fail** — which is the correct behaviour. The
old design ran the build anyway and produced a RED loop.

---

## 5. Build order

Ordered so each stage is verifiable before the next depends on it.

| # | Stage | Produces | Gate to pass |
|---|---|---|---|
| 0 | **Preflight** | — | embed upstream reachable; disk headroom > 20 GB; no build already running |
| 1 | **Scope resolution** | `scopes.tsv` with gate verdicts | every scope is `ELIGIBLE` or explicitly `STALE`; **zero** scopes in `fvm/`, `trash/`, `node_modules/`, or a vendor SDK |
| 2 | **Walk + index** | `index.sqlite` (docs + FTS) | self-check (§2.3) returns 0 derived paths; per-scope budgets respected |
| 3 | **Code graph** | `graph.json` per scope | per-scope budget respected; **only scopes with a real consumer** are graphed |
| 4 | **Embed** | `vec.sqlite` | preflight re-checked; per-run budget respected; **failure keeps the previous index** |
| 5 | **Manifest** | `docs/zk/manifest.tsv` | per-scope counts, hashes, build times — committed |
| 6 | **Report** | `REPORTS.md` | **must not commit or push** (the old one did) |

### Stage 3 is where the old design over-built

`graphify` was run on **all 46 repo scopes**, producing 8 GB of `graphify-out`
— and the output was consumed by nothing but a freshness check.

**Proposed:** graph only scopes with a declared consumer. If nothing reads
`graph.json` for a scope, that scope gets **no graph**. That alone removes
most of the 738 s and most of the 8 GB.

---

## 6. Definition of done

The layer is done when **all** of these hold, each verifiable by a command:

| # | Criterion | Command |
|---|---|---|
| 1 | the whole build runs from a clean checkout | `git clone && ./scripts/zk-build --clean` |
| 2 | it completes within budget | build log shows `total < 10 min` |
| 3 | **no scope is RED** — every scope is `BUILT` or `STALE` | `awk '$status=="RED"' scopes.tsv` → 0 rows |
| 4 | the derived-output self-check returns 0 | §2.3 query → 0 |
| 5 | the index contains no derived path | same query over `docs` → 0 |
| 6 | the scope list has no vendor SDK / trash / package store | §1.1 gates all applied |
| 7 | the index is rebuildable from scratch in ≤ 20 min | `zk-build --clean && time zk-build` |
| 8 | **nothing is consumed by nothing** — every component has a named consumer | §7 table |
| 9 | the upstream dependency is explicit and preflighted | preflight fails cleanly when the upstream is down |
| 10 | the build receipts are committed | `docs/zk/manifest.tsv` in git |

**Criterion 3 is the one that would have caught F1 the first time.**

---

## 7. Every component needs a consumer

| Component | Consumer | If none? |
|---|---|---|
| scope list | the builder | must exist |
| doc store | `zk vsearch` / `zk where` | must exist |
| code graph (`graph.json`) | **currently nothing but the freshness check** | **do not build it** — this is the 8 GB and most of the 738 s |
| embeddings (`vec.sqlite`) | semantic `vsearch` | build only if semantic search is wanted |
| report | a human | build, but **do not commit/push from it** |

**A component with no consumer is not knowledge. It is an expensive habit.**

---

## 8. Open questions

| # | Question | Why it matters |
|---|---|---|
| 1 | Is **semantic** search (embeddings) actually wanted, or is fts5 + ripgrep enough? | If no, the whole embed half of the layer — and the pk31 dependency — can be dropped permanently |
| 2 | Who reads `graph.json`? | If nobody, delete the component rather than recreate it |
| 3 | Is the 48 h freshness DoD wanted at all? | It is the direct cause of F1 |
| 4 | Should the index live on the HDD? | It is 2.27 GB of rebuildable data; the SSD has other claims on it |
| 5 | Does the context-pack corpus (`zprotocols/ctx/`) need to be in the KG at all, given its own cron stages-but-never-commits? | If the KG indexes a stale copy, that is a correctness problem, not a size one |
| 6 | Should `claude-memory` remain one scope with many roots, or become per-project scopes? | The merge is intentional (`setdefault(scope,[]).append(root)`) and `zk_embed.py` depends on the name — changing it has a non-obvious blast radius |

---

*Design proposal, 2026-11-02 IST. Every "must" traces to a measured failure
in `docs/INVENTORY.md` or the audit set.*
