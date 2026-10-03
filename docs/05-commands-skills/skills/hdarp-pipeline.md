---
name: hdarp-pipeline
version: "6.2"
description: "HDARP Framework lifecycle orchestrator. Documents the 7-stage pipeline from PDF acquisition through Knowledge Base delivery. Tracks state, manages handoffs between stages, connects to Anu Framework."
when-to-use: '"User wants to understand the full HDARP lifecycle, check campaign status, or plan a multi-document extraction campaign"'
search-hints: "hdarp pipeline lifecycle stages campaign status orchestrate knowledge base"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[status|plan] [project]"
requires: none
part-of: HDARP Framework v6.3 (cloud Claude Read-tool — distinct from Hopper Line v2 local VLM engine)
---

> **Two parallel extraction frameworks coexist in the workspace**:
> - **HDARP** (this skill family): cloud Claude Read-tool pipeline; Sraffa 4.0 OCR; lifecycle in `HDARP_HUB.md`.
> - **Hopper Line v2** (`/hopper`): offline local VLM engine on a local consumer GPU; spec in `HOPPER_LINE_V2_PROTOCOL.md`.
>
> Both produce 4-artifact KB-ready output (FULL_TEXT.md + CSV_Tables + equations + figures). Pick HDARP for: API available, high accuracy on hard content. Pick `/hopper` for: offline / no-quota / in-copyright / bulk. Never conflate the names — they are distinct engines.

# HDARP Pipeline — Lifecycle Orchestrator v1.0

The full HDARP lifecycle from PDF acquisition through Knowledge Base delivery. This meta-skill documents how the 9 HDARP Framework skills and 10+ commands work together as an integrated pipeline.

---

## Seven Stages

```
Stage 1: ACQUISITION     -> Wishlist + Robert PDF management
Stage 2: PREPARATION     -> /preparehdarp (chunking + manifests)
Stage 3: EXTRACTION       -> /sphdarp N (parallel agent extraction)
Stage 4: ENRICHMENT       -> /enrichhdarp (audit + remediate quality)
Stage 5: WRAPUP           -> /hdarp-wrapup (validation + batch closeout)
Stage 6: INTEGRATION      -> /hdarp-integrate-pipeline (7-phase KB organization)
Stage 7: CLEANUP          -> /hdarp-cleanup (remove chunk artifacts)

OUTPUT: Knowledge Base ready for Anu Framework consumption
        (KB feeds anu-research at Anu Framework Stage 1)
```

---

## Skill-Stage Mapping

| Stage | Skill | Command | Creates |
|-------|-------|---------|---------|
| 1 | (standalone: acquisition planning) | — | Target-list CSV/JSON, acquisition plan |
| 2 | hdarp-chunker | `/preparehdarp` | Chunk PDFs, manifest.json, BATCH_STATE.json |
| 3 | hdarp-extract | `/sphdarp N` | CSV_Tables/, equations/, figures/, FULL_TEXT.md |
| 4 | hdarp-extract | `/enrichhdarp` | Quality audit, remediated extractions |
| 5 | hdarp-extract | `/hdarp-wrapup` | Validated batches, wave closeout report |
| 6 | hdarp-integrate / hdarp-integrate-pipeline | `/hdarp-integrate-pipeline` | Organized Knowledge Base with catalogs |
| 7 | (cleanup utility) | `/hdarp-cleanup` | Chunk artifacts removed, disk space recovered |

### Supporting Skills (any stage)

| Skill | Purpose | When |
|-------|---------|------|
| hdarp-campaign | Campaign planning (waves, batches, dedup) | Before Stage 2 |
| hdarp-manifest-validator | Verify manifest compliance | After Stage 2 |
| hdarp-sraffa-ocr | Standalone OCR for specific pages | During Stage 3-4 |
| hdarp-auto | Automated VLM extraction | Alternative to Stage 3 |

---

## State Management

### BATCH_STATE.json

Central state file tracking all batches across waves:

```json
{
  "project": "ProjectName",
  "current_wave": 7,
  "batches": {
    "BATCH_001": {"status": "VERIFIED", "wave": 1},
    "BATCH_042": {"status": "IN_PROGRESS", "wave": 7},
    "BATCH_043": {"status": "PREPARED", "wave": 7}
  }
}
```

Batch statuses: `PENDING` → `PREPARED` → `IN_PROGRESS` → `COMPLETE` → `VERIFIED`

### HDARP_MASTER_CATALOG.csv

Document-level catalog tracking extraction status, quality scores, and Knowledge Base paths for every document in the campaign.

---

## Connection to Anu Framework

The HDARP pipeline produces Knowledge Bases that the Anu Framework consumes:

```
HDARP Pipeline                          Anu Framework Pipeline
==============                          ======================
Stage 7: CLEANUP                        
    |                                   
    v                                   
{document}              
    +-- FULL_TEXT.md                    Stage 1: anu-research
    +-- CSV_Tables/            ----->       mines KB for quotes,
    +-- equations/                          methodology, data sources
    +-- figures/                        
                                        Stage 2: anu-adequacy
                                            checks KB sufficiency
                                        
                                        Stage 3+: construction...
```

A project can run both pipelines: HDARP to build the KB, then Anu Framework to construct data from it.

---

## Campaign Planning

For multi-document projects (hundreds of PDFs), organize into waves and batches:

1. **Wave**: A group of batches processed together (e.g., Wave 1 = 50 most important documents)
2. **Batch**: 10-chunk group processed by one DARP command invocation
3. **Campaign**: All waves needed to process the entire document collection

Use `/hdarp-campaign` to plan wave structure before starting preparation. Acquisition planning is upstream of this export.

---

## Quality Gates

### Between Stage 3 and Stage 4
- All chunks in batch must have 4-type extraction attempted
- No chunks silently skipped or downgraded to PyMuPDF text dump
- Quality scores recorded for every chunk

### Between Stage 5 and Stage 6
- All batches in wave must be VERIFIED (not just COMPLETE)
- Validator (Opus model) has confirmed extraction quality
- Anti-silent-degradation rule enforced: chunk markers present in body text

### Between Stage 6 and Stage 7
- KB directories organized with catalogs (ENTITY_CATALOG, TABLE_CATALOG, etc.)
- Cross-references built between documents
- Robert sync complete

---

## Commands Quick Reference

| Command | Stage | Purpose |
|---------|-------|---------|
| `/preparehdarp BATCH_NNN` | 2 | Chunk PDFs with density-aware splitting |
| `/sphdarp 5` | 3 | Process all PREPARED batches with 5 agents |
| `/sphdarp 5 --single` | 3 | Process ONE batch only |
| `/enrichhdarp audit BATCH_NNN` | 4 | Audit extraction quality |
| `/enrichhdarp remediate BATCH_NNN` | 4 | Fix quality failures |
| `/sraffa-ocr --chunks DOC_ID` | 4 | OCR specific pages |
| `/hdarp-wrapup` | 5 | Validate + close out wave |
| `/hdarp-integrate-pipeline [project]` | 6 | 7-phase KB organization |
| `/hdarp-cleanup [doc]` | 7 | Remove processed chunks |

---

## Version History

- **v1.0** (May 2026) — Initial release. Documented 7-stage lifecycle, skill-stage mapping, connection to Anu Framework.

---

*Part of the HDARP Framework v6.3 — Lifecycle Orchestrator*

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
