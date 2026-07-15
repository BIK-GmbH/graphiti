# BIK Fork — Operating Model

This is BIK's patch fork of [getzep/graphiti](https://github.com/getzep/graphiti).
Full context (failure modes, bench harness, decision log) lives in
**cdb_tqs → `docs/GRAPHITI.md`**.

## Branch roles

| Ref | Role |
|---|---|
| `main` | untouched upstream mirror (monthly `sync-upstream.yml` PR, no auto-merge) — never commit here |
| `bik/<version>` | patch line on an upstream **release tag** (never on main/prereleases); deliberately small patch set, every commit references a ticket and is an upstreaming candidate |
| `v<version>-bik.N` tags | what cdb_tqs pins in requirements (immutable — reproducible builds & rollbacks) |
| `bik/main` | pointer to the latest validated patch line (repo default branch) |

`git log v<version>..bik/<version>` shows exactly our diff vs. upstream.

## Current patch set (bik/0.29.2)

1. `fix: serialize Neo4j DateTime/Date/Time in to_prompt_json` (cdb_tqs#470)
2. `fix: scope node-dedupe candidates per entity + validate LLM picks` (cdb_tqs#773)
   — env knobs: `NODE_DEDUP_CANDIDATE_LIMIT` (5), `NODE_DEDUP_COSINE_MIN_SCORE`
   (0.75), `NODE_DEDUP_FUZZY_JACCARD` (0.95)

## Upgrade runbook (new upstream release X.Y.Z)

1. `git checkout -b bik/X.Y.Z vX.Y.Z`
2. Cherry-pick the patch set; drop patches upstream has absorbed
3. `pytest tests/utils/` + run the cdb_tqs scaling bench as acceptance test
   (25-doc curve, target <3% incoherent edges)
4. Tag `vX.Y.Z-bik.1`, re-pin cdb_tqs, fast-forward `bik/main` + default branch
