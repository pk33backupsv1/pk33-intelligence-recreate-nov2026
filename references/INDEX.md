# references/ — the intelligence seat's source material

| Path | What |
|---|---|
| `ADVERSARY_C.md` | the Architect's brief (argue for recreate) |
| `ADVERSARY_C_FINDINGS.md` | **its output** — the most detailed inventory of the layer, incl. the second index, the vector store's true state, and the local embedder discovery |
| `ADVERSARY_A_FINDINGS.md` | the Skeptic's output — what is refuted, what is overstated, what was missed |
| `ADVERSARY_B_FINDINGS.md` | the Steward's output — core workings, dependencies, re-initiation plan |

The authoritative measurements live in `../docs/INVENTORY.md` (this seat's own
pass). The seat findings are the **inputs**; `../docs/SYNTHESIS.md` in the
sibling folder resolves the conflicts.

## The layer's source code

| Path | What |
|---|---|
| `/home/pk33/zdash22x/zdash222/bin/zk` | scope generation |
| `/home/pk33/zdash22x/zdash222/bin/zk_graph.sh` | the graphify driver — the 738 s offender |
| `/home/pk33/zdash22x/zdash222/bin/zk_embed.py` | the embedder — `ZK_EMBED_URL` → pk31:8095; the `POP_FILTER` denominator bug at ~line 488 |
| `/home/pk33/zdash22x/zdash222/bin/zk_report.sh` | the report (also commits + pushes) |
| `/home/pk33/zdash22x/zdash222/bin/zk_tunnel.sh` | the pk31:8095 tunnel — its header names the two reversal paths |
| `/home/pk33/zexp1/efus3/scripts/zfts.py` | the walker — `PRUNE_DIRS`, `SPINE_RE`, `_load_scopes_tsv()`'s merge |
| `/home/pk33/zexp1/efus3/scripts/zctx.sh` | the context-pack compiler — computed facts, sha seals, token budget |
| `/home/pk33/zexp1/efus3/skills/zk/SKILL.md` | the surface contract ⚠️ **its frontmatter mis-states the schedule** |

## The layer's artefacts

| Path | Size | Note |
|---|---|---|
| `/home/pk33/zdash22x/zdash222/_logs/zk/index.sqlite` | 2.27 GB | **134,812 docs** (not 35,919 — that was the embed input count) |
| `/home/pk33/zdash22x/zdash222/_logs/zk/vec.sqlite` | 3.9 GB | 737,128 chunks, 36,145 paths, bge-m3 1024-dim — **unqueryable** (the query embeds through the same dead server) |
| `/home/pk33/zdash22x/zdash222/_logs/zk/index.sqlite.prev` | 858 MB | stale Sep-15, built against a scope generation that no longer exists |
| `/home/pk33/zexp1/efus3/_logs/zfts/index.sqlite` | 12.3 MB | **a second index**, 1,281 docs, cron still LIVE |
| `graphify-out/` × 51 | 7,705 MB | **97 dated snapshot dirs hold 6,189 MB (80%)** with no reader |
| `/home/pk33/zexp1/zprotocols/ctx/` | 220 KB | 25 context packs + sha256s — the one healthy component |
| `/home/pk33/zexp1/zvec1/data/zvec.sqlite` | 514 MB | 140,808 chunks, 3 months stale — the only **offline** embedding path on the box |

## The offline embedding option

```
/home/pk33/zexp1/zvec1/data/models/models--qdrant--bge-small-en-v1.5-onnx-q/
  snapshots/52398278842ec682c6f32300af41344b1c0b0bb2/
    model_optimized.onnx      (65 MB — complete, cached)
    tokenizer.json
    config.json
```

`fastembed` / `BAAI/bge-small-en-v1.5`, **384-dim ONNX, CPU, no network**.
Build cost logged: `files=18330 chunks=140808 embedded=126468 deleted=0 in
24225.2s`.

⚠️ **Not interchangeable with the remote store.** bge-m3 is 1024-dim, bge-small
is 384-dim. A vector set **cannot be migrated** between them.

## Registers and DoDs that lie

| Path | The lie |
|---|---|
| `/home/pk33/zdash22x/zdash222/docs/reinit/REGISTER.tsv` (row A-16) | asserts graphify produced **ZERO** `graph.json` artifacts machine-wide; today there are 51 dirs and 49 `graph.json` files |
| `/home/pk33/zdash22x/zdash222/docs/dod/WORK.tsv` (ZDOD-M-…-22664) | requires a `fleet-mirror-pk` scope pointing at a **removed disk** |
| `/home/pk33/zdash22x/zdash222/docs/dod/WORK.tsv` (ZDOD-M-…-20048) | first conjunct is `curl -sf http://127.0.0.1:18095/v1/models` — **no listener since pk31 went offline** |
| `/home/pk33/zdash22x/zdash222/docs/protocols/BACKUP-LEGS.tsv` | **no row** for the intelligence layer — 6.9 GB of derived artefacts with no backup leg |

## The plan of record

| Path | What |
|---|---|
| `~/pk33backupsv1/pk33-cleanup/pk-rework-assessment-3-30oct0134hrs.md` | the start point (audit #3) |
| `~/pk33backupsv1/pk33-cleanup/pk-observations-gotchas-audit-4-30oct0134hrs.md` | 20 tagged findings (audit #4) |
| `~/pk33backupsv1/pk33-cleanup/pk-rework-ledger-2026-10-31.md` | the ledger |
| `~/pk33backupsv1/pk33-recovery-nov2026/docs/SYNTHESIS.md` | the three seats, resolved |

---

*Reference index verified 2026-11-02 IST.*
