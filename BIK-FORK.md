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
7. `feat: GRAPHITI_FORCE_REASONING_MODEL — treat tier-alias deployments as
   reasoning models` (cdb_tqs#843, F12) — upstream 0.29 gates the reasoning
   effort/verbosity behind `model.startswith('gpt-5'|'o1'|'o3')`
   (`openai_client.py`, both `_create_structured_completion` and
   `_create_completion`). Azure/gateway deployments are addressed by a tier
   alias (`"high"`/`"low"`), so the gate is always False → the configured
   effort silently never ships and the API falls back to its default
   (`medium`) — a regression vs. 0.22, which forwarded it unconditionally.
   Env flag (read at call time, no import-timing trap) forces the gate open.
   **Known limitation:** effort resolution keys on the alias string, so it
   cannot detect a gpt-5.5 deployment (which needs `'none'`, rejects
   `'minimal'`); safe for the current gpt-5 / gpt-5-nano deployments (both
   accept `'minimal'`). A real alias→model-family lookup is a follow-up.
8. `feat: keep node summaries in the source language` (cdb_tqs#864) — the
   summary prompts never receive `custom_extraction_instructions`, so on
   German documents the LLM summary path drifted to English (~16–26 % of
   summaries measured on the LJ chlorine KA, while facts — which do carry the
   cdb_tqs extraction directive — were at 0 %). Adds a source-language rule to
   `summary_instructions` (snippets.py; also marks the English example as
   format-only) and to `_entity_episode_summary_system_prompt`
   (extract_nodes.py). No new env knob — the rule is universal (keep the
   source language); fact/name language stays governed by the env-configurable
   extraction instructions in cdb_tqs.

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
