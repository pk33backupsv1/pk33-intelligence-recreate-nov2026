# AGENTS.md — pk33 intelligence seat (Nov 2026)

**You are working on the intelligence layer of pk33.** Read this completely
before you touch anything.

This seat decides **what the intelligence layer should be** and rebuilds it.
It does not restore the old one.

---

## 0. The canonical home — `pk33backupsv1` (pk's ruling, 2026-11-02)

> **`pk33backupsv1` is the canonical GitHub home for pk33's seat work.
> `awgbackups` is SKIPPED — do not create, push to, or migrate anything there.**

**Why:** `awgbackups` has **zero members** and `pk-szt6` is not one of them, so
it cannot be written from this box (`CreateRepository` is refused). It was
created on 2026-10-30T21:14:14Z — 16 minutes before the first attempt against
it — and has never been populated.

**What this means operationally:**

| | |
|---|---|
| **Canonical org** | **`pk33backupsv1`** — all three seat repos live here |
| Canonical local root | `~/pk33backupsv1/` |
| Local repos | `pk33-cleanup` · `pk33-recovery-nov2026` · `pk33-intelligence-recreate-nov2026` |
| `awgbackups` | **skipped** — nothing to do there, do not retry against it |
| `pk-szt6/pk33-cleanup` | a **stale mirror** from 2026-10-30/31 — superseded by `pk33backupsv1/pk33-cleanup`; do not push to it |

**Audit trail:** the original repo was at `~/awgbackups/pk33-cleanup` with
remote `pk-szt6/pk33-cleanup`. It was **shifted** (moved, not copied) to
`~/pk33backupsv1/pk33-cleanup` with remote `pk33backupsv1/pk33-cleanup` on
2026-11-02. Ledger row `4.12`.


---

## 1. The one distinction that governs this seat

> **Recover the content workings. Recreate the intelligence layer.**

The content workings (zsm1, zcontent1, zncert-w1, the zcons kernel, the
Gitea seat) live in
[`pk33-recovery-nov2026`](../pk33-recovery-nov2026). They have real content,
a real source of truth, and real consumers. Restore them.

**The intelligence layer — this seat — does not.** Its scope list indexed a
2.2 GB Flutter SDK and two `trash/` directories; its index ate its own derived
output (17,701 files, `fd5fa4a83`); and its build could never succeed —
`ok=3 red=45` on every run, 738 s against a 10-minute cron, with the CPU
package at **89–100 °C**.

**Restore it and you restore the rot.**

**The test:** can it be rebuilt from a clean checkout, in a stated time, with a
stated verification? If not, it is folklore.

---

## 2. Current state — everything is PAUSED

All six zk crons read `DISABLED` in the registry (`efus3/_logs/agent-crons/registry.tsv`).
Paused 2026-10-31; CPU 97 °C → 45 °C. The pause has held because `heal` skips
`DISABLED` rows.

**Do not re-enable anything here to "just test it."**

1. A hand-edited crontab line is `DEAD`-to-`heal` — `warden.sh` calls `heal`
   every 5 minutes and will restore it. **That is how the 2026-10-14 pause was
   undone.** Use `pcl_selfcron.sh disable` only.
2. The embedding upstream is an ssh tunnel to **pk31:8095**, and pk31 is
   **unreachable**. `curl http://127.0.0.1:18095/v1/models` → `000`, no
   listener. `zk-embed` would fail even if re-enabled. Restoring the upstream
   is Step 4 of the *recovery* plan — a fleet service, not a design decision.

---

## 3. The components

Full detail in [`docs/INVENTORY.md`](docs/INVENTORY.md).

| # | Component | Verdict |
|---|---|---|
| C1 | scope generation (`bin/zk`) | recreate the **policy** |
| C2 | the walker (`zfts.py`) | **keep** — the exclusion logic is good |
| C3 | code graph (graphify) | **recreate** — output consumed by nothing |
| C4 | document store (`index.sqlite`) | **recreate** — self-ingested once |
| C5 | embeddings (`zk_embed.py`) | keep design, **recreate placement** |
| C6 | context packs (`zprotocols/ctx/`) | **recover** — real consumers; tracked in the sibling seat |
| C7 | report (`zk_report.sh`) | **recreate** — a report should not commit and push |

---

## 4. Design rules this seat must satisfy

These are not preferences; each one is a specific failure that already
happened.

### R1 — A perpetual RED is a design failure, not a status
The old build was `ok=3 red=45` forever, at 738 s per run. A scope that cannot
be satisfied means **the scope is wrong**. Do not retry harder — change the
scope policy.

### R2 — Derived output never enters its own index
That is `fd5fa4a83`: 17,701 graphify files leaked into the zk index. The
current fix is a prune entry (`graphify-out` in `PRUNE_DIRS`) — **a repair of
a structural feedback loop**. It must be structural: the indexer must not be
able to see its own output directory, by construction.

### R3 — A build with no bounded runtime is not a build
738 s against a 10-minute cron is a loop. Every stage needs a stated budget,
and exceeding it must be a *failure with a reason*, not an overlap.

### R4 — A scope must earn its place
There must be an explicit policy for what becomes a scope. "Whatever
`clones.tsv` says" is how a Flutter SDK got indexed.

### R5 — Rebuildable from a clean checkout, in a stated time, with a stated gate
If you cannot state the time and the gate, you are restoring folklore.

### R6 — A wrong artifact is worse than no artifact
The `zncert-w1` house rule, and it applies here: a scope that cannot be
satisfied should be RED and **excluded**, not half-built. Absence is a valid
state.

---

## 5. Where the design lives

| Document | What it is |
|---|---|
| [`docs/INVENTORY.md`](docs/INVENTORY.md) | every component: input, output, consumers, verdict |
| [`docs/RECREATE-DESIGN.md`](docs/RECREATE-DESIGN.md) | the target architecture — schema, build order, verification gate, DoD |
| [`docs/THROW-AWAY.md`](docs/THROW-AWAY.md) | what is noise, what is stale, what is lying |
| [`README.md`](README.md) | the seat's purpose and rules |
| [`references/`](references/) | pointers to source material |

---

## 6. Cross-seat rules

- **Ledger rows** for every delete/shift/pause/prune. The ledger lives in
  [`../pk33-cleanup/pk-rework-ledger-2026-10-31.md`](../pk33-cleanup/pk-rework-ledger-2026-10-31.md).
  Use the row shape in the sibling seat's protocols doc.
- **`rm -f` is unavailable on this seat** — use `unlink`, or move to
  `~/.trash-<date>/` first.
- **Two seats must disagree** before an irreversible action. Pattern:
  `references/tmux-adversarial-review.md` in the sibling seat.
- **Re-verify inherited numbers.** Every number in `docs/INVENTORY.md` was
  measured on 2026-11-02 — but this box changes.

---

## 7. Honest limits

The 2026-10-31 ledger **partially answered** one of the open questions from
audit #4 ("could the 11 `claude-memory` rows lose data?"): **no** —
`zfts.py` merges same-named roots via `setdefault(scope, []).append(root)`, so
all memory dirs are walked and indexed under one scope name. The collision was
**cosmetic**, not destructive. But `zk_embed.py` filters on that exact scope
name, so **renaming the scope would silently break the filter** — a
non-obvious coupling to preserve.

---

*Established 2026-11-02 IST at the pk33 intelligence seat.*
