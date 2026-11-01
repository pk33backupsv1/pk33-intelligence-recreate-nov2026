# pk33-intelligence-recreate-nov2026

**The intelligence seat for pk33.** One of two working folders created 2026-11-02
under the `pk33backupsv1` GitHub org.

| | |
|---|---|
| **Purpose** | Decide what the intelligence layer should be, and rebuild it properly |
| **Scope** | The **intelligence layer** — code KG (graphify/`zk`), context packs, project graphs, KG index, embeddings |
| **Method** | **RECREATE** — rebuild from a stated source of truth. **Do not restore.** |
| **Sibling** | [`pk33-recovery-nov2026`](../pk33-recovery-nov2026) — the content workings, **recover** |

---

## Why this is not a recovery

This is the load-bearing distinction of the whole initiative, and it is a
judgement, not a preference.

**The intelligence layer's inputs were rotten, its own output fed back into
itself, and its build could never succeed.** Restore it and you restore the
rot. Four measured facts:

| Fact | Evidence |
|---|---|
| The scope list indexed a **2.2 GB Flutter SDK** and **two `trash/` directories** | `zdash222/_logs/zk/scopes.tsv` (pre-2026-10-31) |
| **11 duplicate `claude-memory` scope rows** — same scope name, different paths | same file |
| The KG index **ingested its own derived output** — 17,701 graphify files leaked in | efus3 commit `fd5fa4a83` (2026-09-16) |
| The build **could never go green** — `ok=3 red=45` on every run, 738 s against a 10-minute cron, forever | `zdash222/_logs/zk/graph-cron.log` |

And the cost was physical: the CPU package reached **89–100 °C** (crit 100).

**Contrast with the content workings** (sibling folder): zsm1 is green with a
real source of truth; zcontent1 holds live pk-directed design work; zncert-w1
is a published product with real consumers. Those get restored.

**The test for this folder:** *can it be rebuilt from a clean checkout, in a
stated time, with a stated verification?* If not, it is not knowledge — it is
folklore, and it does not get restored.

---

## Read in this order

| # | Document | What it is |
|---|---|---|
| 1 | [`AGENTS.md`](AGENTS.md) | What this seat is, the rules, the components |
| 2 | [`docs/INVENTORY.md`](docs/INVENTORY.md) | Every component: what it builds, its input, its output, who consumes the output |
| 3 | [`docs/RECREATE-DESIGN.md`](docs/RECREATE-DESIGN.md) | The target architecture — scope policy, bounded runtime, no self-ingestion, verification gate, DoD |
| 4 | [`docs/THROW-AWAY.md`](docs/THROW-AWAY.md) | What is noise, what is stale, what is lying |
| 5 | [`references/`](references/) | Pointers to the source material |

**Related, elsewhere:**
- [`../pk33-cleanup`](../pk33-cleanup) — the audit set (#1–#4) and the execution ledger
- [`../_agent-out/`](../_agent-out) — the adversarial seat briefs and their findings

---

## Current state

**Everything in this layer is PAUSED.** All six zk crons read `DISABLED` in the
registry (paused 2026-10-31, CPU 97 °C → 45 °C). The pause has held: `heal`
skips `DISABLED` rows.

**Do not re-enable anything here to "just test it".** Two independent reasons:

1. The pause is a **registry state**, not a comment. A raw crontab edit would
   make `heal` restore it within 5 minutes — which is exactly how the
   2026-10-14 pause was undone.
2. The embedding upstream is an **ssh tunnel to pk31:8095**, and **pk31 is
   unreachable**. `curl http://127.0.0.1:18095/v1/models` → `000`, no
   listener. Re-enabling `zk-embed` now produces a RED loop with no cause.

The upstream restoration is Step 4 of the recovery plan — it is a *fleet
service*, not an intelligence-layer design decision, so it lives in the
sibling folder.

---

## The rules that bind this seat

1. **A perpetual RED is a design failure, not a status.** If a scope cannot be
   satisfied, the *scope* is wrong. Do not retry harder — change the design.
2. **Derived output never enters its own index.** That is what `fd5fa4a83`
   had to fix by hand. It must be structural, not a prune job.
3. **A build with no bounded runtime is not a build.** 738 s against a
   10-minute cron is a loop, not a schedule.
4. **A scope must earn its place.** There must be an explicit policy for what
   becomes a scope — currently "whatever `clones.tsv` says", which is how a
   Flutter SDK got in.
5. **Record before you destroy.** See the recovery seat's ledger protocol.
6. **Rebuild from a clean checkout in a stated time, with a stated
   verification gate.** If you cannot, you are restoring folklore.

---

*Created 2026-11-02 IST at the pk33 intelligence seat.*
