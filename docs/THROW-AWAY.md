# THROW-AWAY.md — what in the intelligence layer is noise, stale, or lying

**Date:** 2026-11-02 (IST) · **Scope:** the intelligence layer only
**Rule:** every item here is either provably regenerable, provably redundant,
or provably wrong. Nothing is listed merely because it is large.

---

## 1. Pure bloat — throw away, no replacement needed

| Item | Size | Why it is bloat |
|---|---|---|
| `zdash222/_logs/zk/index.sqlite.prev` | **898 MB** | a stale copy from **2026-09-15**, superseded by `index.sqlite` (Oct 14). Pure dead weight — no reader. |
| `efus3/graphify-out/` — 17 of 18 dated dirs | **~4.4 GB** | 18 dated copies of the *same* graph (2026-09-16 → 2026-10-13, 257 MB each). Only the newest has any claim; the rest are historical snapshots of a derived artifact with no consumer. |
| `graphify-out/cache/ast/` under the Flutter SDK scope | **134 MB** | AST cache for 2.2 GB of **vendor Dart source**. The scope itself was retired on 2026-10-31. |
| `graphify-out/` across the other 42 repos | **~3.4 GB** | derived, git-ignored (`.gitignore:96`), regenerable by `graphify update .` — **but see §3: the component itself is in question, not just its copies** |

**Regeneration path** (if the component survives §3):
`graphify update <root>` — tree-sitter, no API, 0 tokens.

---

## 2. Stale — outdated, and quietly so

| Item | Evidence | Action |
|---|---|---|
| `zdash222/_logs/zk/index.sqlite` (2.27 GB) | last built **2026-10-30 19:50**, but the scope list was pruned **2026-10-31** — so the index **predates the prune** and still contains the Flutter SDK and `trash/` scopes it now excludes | rebuild from scratch, do not patch |
| `zdash222/_logs/zk/index.sqlite.prev` | 2026-09-15 | delete outright |
| `zdash222/_logs/zk/embed-cron.log` | ends with `ZK EMBED: ... new=34 changed=14 removed=0` — the index was **nearly static** before the pause | keep as a receipt, not data |
| `zdash222/_logs/zk/graphify.log` | a stream of `RED <scope> 0s` / `RED <scope> 223s` — the build failing fast, repeatedly | keep the *evidence* (it justifies the pause); the log itself is not needed |
| `zexp1/zprotocols/ctx/` — 9 files staged, never committed | `CONTEXT-v1.0.16` … `v1.0.23` + sha256s + `LATEST.md` sit in the **index**, not in a commit | commit them (recovery plan §7) — **do not discard** |

---

## 3. The component in question — not the copies

**`graphify` / the code graph (`graph.json`) may not deserve to exist at all.**

This is the boldest call in this folder, so here is the evidence:

| Question | Answer | Where |
|---|---|---|
| Who reads `graph.json`? | **`zk_graph.sh --check`** — which only asks *"is it newer than 48 h?"* | `zdash222/bin/zk_graph.sh` |
| Is it newer than 48 h achievable? | **Only for a repo actively committed to.** A dormant repo fails forever. | — |
| What did it cost? | **738 s/pass**, 8.0 GB across 43 repos, and the **89–100 °C** thermal event | `graph-cron.log`, `sensors` |
| Did the output ever change a decision? | No evidence it was ever read for content | — |

**Three options, and the seat must choose:**

1. **Delete the component.** If nothing reads `graph.json`, the correct move
   is to remove it — not to rebuild it. This removes ~8 GB, most of the
   738 s, and the entire `ok=3 red=45` failure class.
2. **Keep it with a declared consumer.** If some agent reads `graph.json` for
   code navigation, that consumer should be **named**, and only the scopes it
   needs should be graphed.
3. **Rebuild it properly** per `docs/RECREATE-DESIGN.md` §5 stage 3 — graph only
   scopes with a consumer, per-scope budget, per-scope failure.

**Until this is answered, do not rebuild the code graph.** It is the single
largest cost in the layer and the one with the least evidence of value.

---

## 4. Wrong — actively misleading

| Item | Why it lies |
|---|---|
| `zdash222/_logs/zk/scopes.tsv` **as it was** (pre-2026-10-31) | listed a 2.2 GB Flutter SDK and two `trash/` dirs as *knowledge scopes*. A graph over a Flutter SDK is a graph of someone else's code. |
| The `ok=3 red=45` verdict line | it reads as "45 things are broken". In truth **45 scopes were in the list that could never pass their own DoD**. The number described a **policy** failure, not 45 work items. |
| `~/GRAPHIFY-PAUSED-2026-10-14.md`'s **restore command** | `crontab -l \| sed 's\|^#PAUSED-BY-ADAL-GRAPHIFY: \|\|' \| crontab -` — running it today **re-enables three jobs** (`zk-embed`, `zk-index-zdash222`, `zk-graph`) with no check that the thermal condition still applies. It is a **live footgun**, and the file reads as if it describes the current state. |
| `zsm1/RESUME-HERE.md` (the *document*, not the system) | says Indianization is RED with 130 faults; re-running the checker gives **0 faults**. It says Hindi is RED with 5,589 items; it is **99.3% green**. A reader who trusts it will resume finished work. **Keep the file; add a header pointing at the checkers.** |

---

## 5. Duplicate — the same thing twice

| Item | Note |
|---|---|
| 11 × `claude-memory` scope rows | **cosmetic, not destructive** — `zfts.py` merges same-named roots via `setdefault(scope,[]).append(root)`, so all 7 non-empty memory dirs are walked and indexed under one name. 5 of the 12 were **empty** (0 files, 0 documents) and were dropped 2026-10-31. **Do not rename the scope** — `zk_embed.py` filters on `scope != 'claude-memory' OR path LIKE '%/memory/%'`. |
| `opencode.db` + its salvage | **already resolved** — the 244 GB db was deleted 2026-11-02; the real content (6.9 GB) is on the HDD. Not an intelligence-layer item, but the same lesson: **the bloat was the `event` table, the content was `message`+`part`.** |

---

## 6. What must NOT be thrown away

| Item | Why |
|---|---|
| `zexp1/zprotocols/ctx/` | real content with **six** consumers (`znouns.sh`, `zdev.sh`, `zctx.sh`, `zfts.py`, `claude_home_acceptance.sh`, `write_topology_check.sh`). The **staged-but-uncommitted** files are a cron bug, not garbage — fix the cron (recovery plan §7), don't discard the files. |
| `efus3/scripts/zfts.py` | the `PRUNE_DIRS` / `SPINE_RE` logic is good — it encodes the kc-dc-rc doctrine ("data spines are not knowledge"). Keep the walker; fix the feedback loop structurally. |
| `zdash222/bin/zk_embed.py` | 125 s for 737k chunks is **cheap**. Keep the design; fix the placement (local upstream + preflight). |
| The **scope-prune commit** `c3fa3e7` | it dropped the Flutter SDK + `trash/` + 5 empty memory dirs at the right layer (not `clones.tsv`, which is regenerated every beat). Keep it. |
| `zdash222/_logs/zk/*.log` | the receipts that justify the pause. Small. Keep as evidence. |

---

## 7. The throw-away list, ranked by payoff

| Rank | Action | Frees | Risk |
|---|---|---|---|
| 1 | **Decide the code-graph component** (§3) — delete, or rebuild with a declared consumer | up to **8.0 GB** + most of the 738 s | low if a consumer is named first |
| 2 | delete `index.sqlite.prev` | **898 MB** | none — a stale Sep-15 duplicate |
| 3 | delete 17 of 18 dated `efus3/graphify-out` dirs, keep the newest | **~4.4 GB** | none — derived, git-ignored |
| 4 | rebuild `index.sqlite` from scratch rather than patching it (it predates the prune) | replaces 2.27 GB with a clean one | needs the embed upstream restored (pk31) |
| 5 | prune the other 42 `graphify-out` dirs | **~3.4 GB** | none — regenerable |
| 6 | update `GRAPHIFY-PAUSED-2026-10-14.md` to say the pause is superseded and the restore command is unsafe | 0 | none — **but it is a live footgun** |
| 7 | add a header to `zsm1/RESUME-HERE.md` pointing at the checkers | 0 | none — prevents a whole category of mistake |

**Total: ~16 GB on the SSD, and the removal of an unbounded, thermally
destructive build loop.**

---

*Throw-away list, 2026-11-02 IST. Every size measured; every "why" traced to a
file or a command.*
