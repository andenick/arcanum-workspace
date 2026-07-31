---
name: hdarp-campaign
description: "Set up an HDARP (cloud Claude Read-tool) processing campaign from raw PDFs — inventory, dedup, classify, wave plan, batch creation, and documentation. For OFFLINE local-VLM extraction on RTX 5090, use /hopper instead."
when-to-use: '"User wants to set up a new HDARP campaign, process a new folder of PDFs via cloud Claude Read-tool, create waves/batches for sphdarp. If they want offline / local / quota-free / GPU-resident extraction, use /hopper (the Hopper Line v2 engine) instead — both produce 4-artifact KB-ready output but via different stacks."'
search-hints: "hdarp campaign setup wave batch plan inventory dedup classify initialize cloud sonnet opus"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[project_path]"
requires: none
part-of: HDARP Framework v6.3 (cloud Claude Read-tool pipeline — distinct from Hopper Line v2 local VLM engine)
---

# HDARP Campaign Setup Skill

> **Scope note**: this skill operates the **cloud Claude Read-tool HDARP pipeline**. The DISTINCT local-GPU `/hopper` skill (Hopper Line v2) produces the same KB shape on the RTX 5090 without any API calls. Pick HDARP for: high accuracy, API quota available, complex content. Pick `/hopper` for: offline / no-quota / bulk / in-copyright corpora. **Never call Hopper "HDARP" or vice versa** — they're separate engines.



## Description

Set up a complete HDARP processing campaign from a folder of PDFs. Handles inventory, deduplication, classification, wave assignment, batch creation, and BATCH_STATE registration. Produces a campaign plan ready for `/preparehdarp` and `/sphdarp`.

## Usage

```
/hdarp-campaign                    # Interactive — asks for input folder
/hdarp-campaign <input_folder>     # Direct — starts discovery on folder
/hdarp-campaign status             # Show existing campaign status
```

## HDARP Lifecycle Position

This skill is step **1** (CAMPAIGN) in the HDARP lifecycle:
1. **`/hdarp-campaign`** — inventory, dedup, classify, wave/batch plan
2. `/preparehdarp` — chunk and prepare PDFs
3. `/sphdarp` (or variants) — extract all 4 content types via agents
4. `/hdarp-wrapup` — validate, remediate, document, close out
5. `/hdarp-integrate-pipeline` — catalog, classify, crossref, Robert sync (Phase 7 calls `/robert-pdf-sync` per the v2.0 standard: `ROBERT_PDF_LIBRARY_V2_STANDARD.md`)

## Engine

All heavy lifting delegates to:
```
hdarp_campaign_engine.py
```

Requires: PyMuPDF (`pip install PyMuPDF`)

## Workflow (6 Phases)

### Phase 1: DISCOVERY (automated)

1. Identify the input folder. If not provided, ask the user.
2. Identify the project root and KB directory (for dedup cross-reference).
3. Create campaign directory: `<campaign_name>`
4. Run the engine:
   ```bash
   python "hdarp_campaign_engine.py" discover \
     "<input_dir>" --kb-dir "<kb_dir>" --output "<campaign_dir>"
   ```
5. Report to user: file count, pages, size, duplicates, OCR needs.

### Phase 2: DEDUP REVIEW (present to user)

Discovery also runs deduplication. Present results:
- Internal duplicates: N sets found, M files marked REMOVE
- KB overlaps: K HIGH-confidence (auto-excluded), J MEDIUM (flagged)
- Ask: "Any overrides before I proceed to classification?"

### Phase 3: CLASSIFY (automated)

The plan command handles classification:
- Generates `document_id` for each PDF (YYYY_Author_Title format, max 80 chars)
- Assigns `cf_risk_level` (HIGH/MEDIUM/LOW) based on content filter heuristics
- Flags special documents: MONSTER_DOC (>100 chunks), EXCLUSIVE_BATCH (>10 chunks), NEEDS_OCR
- Per-page pdf_type classification (Sraffa 4.0): digital / scanned / mixed — stored in campaign metadata for downstream routing

### Phase 4: WAVE PLAN (configurable)

Ask user for wave strategy (or use `balanced` default):
- **balanced** (default): Greedy bin-packing for equal total chunks per wave
- **folder**: One wave per source subfolder (preserves provenance)
- **priority**: User specifies tier 1/2/3; front-load high-priority docs
- **custom**: User provides wave assignments as JSON

Also ask for starting batch number (check BATCH_STATE.json for the next available ID).

Run the engine:
```bash
python "hdarp_campaign_engine.py" plan \
  "<campaign_dir>" --wave-strategy balanced --starting-batch <N>
```

### Phase 5: REVIEW GATE (human approval required)

Read `campaign_report.md` from the campaign directory and present to user:
```
═══ CAMPAIGN PLAN: <name> ═══
Input: <folder> (N PDFs, X pages, Y GB)
Excluded: M duplicates, K KB overlaps
Documents: J to process
Waves: W (strategy: balanced)
Batches: B total (BATCH_NNN – BATCH_MMM)
Estimated /sphdarp rounds: ~R

[Wave breakdown table]
[Risk flags summary]

Ready to register in BATCH_STATE.json?
═══════════════════════════════════════
```

**Wait for explicit user approval.** User may request:
- Different wave strategy
- Exclude specific documents
- Adjust batch size or wave count
- Change starting batch number

### Phase 6: REGISTER + PREPARE

After user approves:

1. Register batches in BATCH_STATE.json:
   ```bash
   python "hdarp_campaign_engine.py" register \
     "<campaign_dir>" --batch-state "<batch_state_path>"
   ```

2. Update HDARP_MASTER_CATALOG.csv with new document entries (status: PENDING).

3. Run `/preparehdarp --wave Wave_01` to chunk and prepare the first wave.

4. Report completion:
   ```
   Campaign registered. N batches across W waves.
   Wave_01 PREPARED (M batches, K chunks).
   Run `/sphdarp 5 --wave 1` to begin processing.
   ```

## Campaign Directory Structure

After setup, the campaign directory contains:
```
<project>/<name>/
  inventory.csv          # Every file scanned
  duplicates.csv         # Internal duplicate decisions
  kb_overlaps.csv        # KB cross-reference matches
  classified.csv         # Doc IDs, CF risk, special flags
  batch_plan.csv         # Batch assignments (batch_id, wave, doc_id, filename)
  campaign_config.json   # All parameters for reproducibility
  campaign_report.md     # Human-readable summary
```

## Integration with Existing Tools

| Tool | Integration |
|------|-------------|
| `/preparehdarp` | Reads BATCH_STATE.json PENDING entries, handles PENDING→PREPARED |
| `/sphdarp` | Processes PREPARED batches by wave (`--wave Wave_01`) |
| `/enrichhdarp` | Post-processing on completed waves |
| `/hdarp-cleanup` | Cleanup after verification |
| BATCH_STATE.json | Campaign writes initial PENDING entries |
| HDARP_MASTER_CATALOG.csv | Campaign creates initial rows |

## Determining Starting Batch

Before running Phase 4, check BATCH_STATE.json for the highest existing batch number:
```python
import json
with open(batch_state_path, 'r', encoding='utf-8') as f:
    state = json.load(f)
max_batch = max(int(bid.split('_')[1]) for bid in state['batches'].keys()) if state['batches'] else 0
starting_batch = max_batch + 1
```

## Error Handling

- **PyMuPDF not installed**: Engine falls back to size-only classification (no page counts)
- **Empty input folder**: Report and exit
- **BATCH_STATE.json doesn't exist**: Create a new one with empty batches/queues
- **Duplicate doc_ids**: Append numeric suffix (`_2`, `_3`)
- **Campaign already exists**: Ask user to confirm overwrite or create new name

## Skill Version

**Version**: 6.2
**Created**: 2026-04-28
**Updated**: 2026-04-30
**Engine**: `hdarp_campaign_engine.py`
**OCR Processor**: `sraffa40_processor.py` (Sraffa 4.0)
**Depends on**: PyMuPDF, BATCH_STATE_PROTOCOL.md, HDARP_FAILURE_TAXONOMY.md v2.0, SRAFFA_4_PROTOCOL.md

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
