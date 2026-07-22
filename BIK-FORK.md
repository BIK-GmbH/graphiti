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
2. `fix: scope node-dedupe candidates per entity + validate LLM picks` (cdb_tqs#773 v1)
   — env knobs: `NODE_DEDUP_CANDIDATE_LIMIT` (5), `NODE_DEDUP_COSINE_MIN_SCORE`
   (0.75), `NODE_DEDUP_FUZZY_JACCARD` (0.95)
3. `fix: reject family/variant name-extension merges` (cdb_tqs#773 v2)
   — `NODE_DEDUP_REJECT_NAME_EXTENSION` (default on; conversational-memory
   deployments should disable)
4. `feat: exact-only dedup for identifier-like entity types` (cdb_tqs#773 v3)
   — `NODE_DEDUP_EXACT_ONLY_TYPES` (comma-separated labels; `'*'` = all types
   → LLM node dedupe disabled, v4)
5. `feat: relaxed second-lookup key for exact-only resolution` (cdb_tqs#773 v4.1)
   — cosmetic variants (hyphen/space, trailing punctuation, DIN prefix)
6. `feat: EDGE_DEDUP_EXACT_ONLY — keep distinct facts between identical
   endpoints` (cdb_tqs#773 v5, fixes F9) — edge merge only via the verbatim
   fast-path; LLM fact-dedupe **and contradiction-invalidation** are disabled
   (consumers must clear stale edges themselves on document re-processing)

Validation status (2026-07-17, independently re-verified 2026-07-21/22 —
full adversarial scan of every edge + source-fidelity samples + inverse
coverage): **rollout AC met with the full stack v1–v5.**
gpt-5: 0.74% clearly-incoherent (4/542), recall canary 20/25.
gpt-5.4: 0.18% (4/2282) at 4x edge density, recall 16/25.
Zero number hallucinations in either run. Down from 9–10% unpatched and a
7.5% production baseline. Details: cdb_tqs `docs/GRAPHITI.md` §4/§5/§7.

## Upgrade runbook (new upstream release X.Y.Z)

1. `git checkout -b bik/X.Y.Z vX.Y.Z`
2. Cherry-pick the patch set; drop patches upstream has absorbed
3. `pytest tests/utils/` + run the cdb_tqs scaling bench as acceptance test
   (25-doc curve, target <3% incoherent edges)
4. Tag `vX.Y.Z-bik.1`, re-pin cdb_tqs, fast-forward `bik/main` + default branch
