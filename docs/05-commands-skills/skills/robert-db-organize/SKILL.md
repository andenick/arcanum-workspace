---
name: robert-db-organize
version: "1.0"
description: "Build the per-project taxonomy (seed from catalogs, agent-proposed 2-level hierarchy, USER-RATIFIED before classification) and the concordances (propose candidate families, agent-review, commit) for a Robert database."
when-to-use: '"User wants to classify a project''s tables under a topic hierarchy, ratify a taxonomy, or group related tables across documents into concordance families."'
search-hints: "robert db organize taxonomy classify hierarchy ratify concordance candidates propose commit cluster year-series panel family"
argument-hint: "[project]"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit, Agent
requires: robert-db-harvest
part-of: Robert Database Framework v1.0
---

# robert-db-organize — Stage 4 (ORGANIZE)

Give the harvested tables a researcher-facing structure: a **ratified 2-level
taxonomy** and **concordances** (families of related tables across documents).
Organize depends only on harvest and may run alongside enrich.

## Purpose

1. Seed a per-project taxonomy from source catalogs, have an agent propose a clean
   2-level hierarchy, get the **user to ratify it**, then classify tables under it.
2. Propose concordance candidates mechanically, have an agent review them, and
   commit the curated families.

## Preconditions

- `robert-db-harvest` has run; `xtables` is populated.
- `CLASSIFICATION_MASTER.csv` / catalog category fields are available where they
  exist (used as taxonomy seed and a classification signal).

## Procedure — Taxonomy

1. **Seed.** Collect distinct `documents.category_raw` and any catalog topic columns
   into candidate `taxonomy_terms` with `source='seed_catalog'`, `ratified=0`.
2. **Propose hierarchy (agent).** Spawn a Sonnet 5.5 subagent (foreground; Opus only
   per `model-efficiency.md`) to fold the
   seeds into a **2-level** hierarchy (top-level domains → child terms), writing a
   proposal file `Technical/RobertDB/state/TAXONOMY_PROPOSAL.md` with each term's
   `label`, `definition`, parent, and `source='agent_proposed'`. The agent proposes;
   it does not classify yet.
3. **USER RATIFICATION (REQUIRED).** Present the proposal to the user. **Do not
   classify any table until the user ratifies the term list.** On ratification, set
   `ratified=1` on the accepted terms (mark `source='curated'` for user edits).
   Unratified terms must never drive classification.
4. **Classify.** Once ratified, run classification passes writing `table_taxonomy`
   rows with honest `basis`: `catalog_inherited` (from the doc's catalog category),
   `rule` (deterministic match), or `agent` (subagent topical judgment, with a
   `confidence`). Multi-label is allowed.

## Procedure — Concordances

1. **Propose candidates.**
   ```bash
   PYTHONIOENCODING=utf-8 python rdb_concordance_candidates.py \
       --config <P>/Technical/RobertDB/robertdb_config.json propose \
       [--min-size 2] [--kind-hint year_series_family panel_family]
   ```
   This writes a candidates JSONL (proposed clusters with member `table_uid`s and a
   `kind` guess: `year_series_family` | `panel_family` | `scenario_family` |
   `topical_group`).
2. **Agent review.** Spawn a Sonnet 5.5 subagent (foreground; Opus only per the Model
   Efficiency Policy) to read the candidates
   JSONL plus the member tables' context, accept/reject/relabel each cluster, set a
   human-readable `label` + `description`, fix `kind`, and order members
   (`ordering_key` = year / cycle / volume). Write the reviewed JSONL.
3. **Commit.**
   ```bash
   PYTHONIOENCODING=utf-8 python rdb_concordance_candidates.py \
       --config <P>/Technical/RobertDB/robertdb_config.json commit \
       --reviewed <reviewed.jsonl>
   ```
   Accepted clusters become `concordances` rows (id `<PJ>-K-NNNN`, `method` =
   `agent`/`hybrid`/`curated`) with `concordance_members`.

## Outputs

- `taxonomy_terms` (ratified) + `table_taxonomy` assignments.
- `concordances` + `concordance_members`.
- `TAXONOMY_PROPOSAL.md` and the reviewed concordance JSONL retained under `state/`.
- Views regenerated (run `rdb_views.py` if a step did not chain it).

## Failure handling

- **No ratification yet** → STOP before classifying; classification on an
  unratified taxonomy is a contract violation.
- **Candidate cluster the agent cannot justify** → reject it; an unjustified
  concordance is worse than none. Record the rejection reason in the reviewed JSONL.
- **Term collision / re-seed** → re-running propose-hierarchy must not silently drop
  ratified terms; preserve `ratified=1` curated terms across re-seeds.
