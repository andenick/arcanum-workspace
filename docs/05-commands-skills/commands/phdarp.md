---
description: "Parallel HDARP v6.2: CF Progressive Decomposition + Task Hygiene + Batch Continuation + Sonnet Mandatory + Opus Validator + Mandatory Catalog Sync + Error Recovery"
allowed-tools: Bash, Read, Write, Glob, Grep, Task
argument-hint: "[number]"
---

**HDARP Framework v6.3** — see `VERSION_REGISTRY.md`

> **Native RDB metadata capture (v6.3):** processors emit one `RDB_METADATA*.jsonl` line per table CSV per `hdarp-processing.md` ("Native RDB Enrichment Capture") + spec `NATIVE_ENRICHMENT_CONTRACT.md`. Same chunk-range/whole-doc + WARN-only validator rules as `sphdarp.md`.

# Parallel HDARP Command (PHDARP) v6.2

**Command**: `/phdarp [N]`
**Full Name**: Parallel Hybrid Direct Agent Reading Protocol
**Version**: 6.2
**Updated**: 2026-04-28

## 🛑 CRITICAL RULE: NEVER FABRICATE (v6.0)

If you encounter a content filter error (API 400 `Output blocked by content filtering policy`): the **orchestrator** handles recovery via the CF Progressive Decomposition Protocol (v6.0) — bisect the chunk, re-dispatch Sonnet subagents, recursively decompose down to single pages, then OCR only as last resort. See the Content Filter Recovery Protocol section under Phase 2: Error Recovery. For timeouts or other API errors: RECORD the affected chunk in Failures and move on. **DO NOT** generate substitute content, paraphrase from memory, or write block quotes / attributions you haven't verified verbatim from the source PDF. Mark uncertain passages `[approximate]` — a gap marker is always preferable to a fabrication. See `HDARP_CONTENT_FILTER_PATTERNS.md` for the Wave 7 fabrication incident that motivated this rule.

**DO NOT silently substitute extraction methods.** If agent extraction fails, STOP and report. Do not switch to PyMuPDF, bulk scripts, or any non-HDARP method without explicit user approval. See the 2026-05-06 Wave_07 silent degradation incident in sphdarp.md.

## Scrounger Standard

PHDARP is a scrounger — it does not give up. Only three outcomes are acceptable reasons for a chunk to have no full extraction:

1. **DUPLICATE** — confirmed identical content already in KB (MD5 hash match or manual KB verification — not "looks similar")
2. **CF_EXHAUSTED** — content filter hit on every single page after the full L1/L2 retry ladder (see Content Filter Recovery Protocol below)
3. **QUARANTINED** — file is physically unreadable by every tool (0 bytes, crashes PyMuPDF)

Everything else gets extracted. Confusion, scan quality, unknown content type, timeout, billing errors — these are reasons to try harder, not reasons to defer.

**The Content Filter Recovery Protocol (Phase 2 below) is the scrounger L1/L2/L3 retry ladder.** CF_EXHAUSTED is only declared after that full ladder has been run on every page — not on first CF hit. L3 (OCR queue) fires only after both L1 bisect and L2 scholarly framing have failed on a specific page.

See `sphdarp-scrounger.md` for the full acceptable-outcomes taxonomy, mandatory retry behavior tree, and validator scrounger check.

## BATCH_STATE Integration

All DARP commands now integrate with BATCH_STATE.json:

**Location**: {Project}/BATCH_STATE.json

**Key Changes**:
- Command reads batch_to_process from state
- Validator validates PREVIOUS batch (not current)
- State updated after processing completes

See BATCH_STATE_PROTOCOL.md for full details.

---

## Usage

```bash
/phdarp                       # 5 processors + 1 validator, continues until all batches done
/phdarp 3                     # 3 processors, continues until all batches done
/phdarp 10                    # 10 processors (max), continues until all batches done
/phdarp 5 --single            # Process ONE batch only, then stop
/phdarp 5 --wave Wave_02      # Process ONLY Wave_02 batches, lowest-numbered first
/phdarp 5 --wave 2            # Same — short form accepted (normalized to Wave_02)
```

## DARP Command Family

| Command | Smart Batching | Hybrid (OCR) | Use Case |
|---------|----------------|--------------|----------|
| /pdarp | No | No | Quick DARP extraction, one chunk per agent |
| **/phdarp** | No | Yes | Full extraction, one chunk per agent |
| /spdarp | Yes | No | DARP extraction, all chunks from one document |
| /sphdarp | Yes | Yes | Full extraction, all chunks from one document |

**Key Distinctions**:
- **P** = Parallel (all commands have this)
- **S** = Smart (document-aware batching)
- **H** = Hybrid (includes OCR body text)

## What's New in v4.5

- **Mandatory Catalog Synchronization**: All processing MUST update HDARP_MASTER_CATALOG.csv
- **Sonnet Mandatory Model Policy**: ALL processor agents use Sonnet, validator uses Opus — Haiku BANNED
- **Error Recovery Phase**: Failed chunks automatically retried
- **Quality Score Propagation**: Scores flow from processing to catalog automatically
- **Unified v5.0 Standard**: Consistent with all DARP commands

### From v5.0 (Sonnet Mandatory Model Policy)

- **Sonnet Mandatory**: ALL processor agents use Sonnet — Haiku BANNED from entire pipeline
- **Opus Validator**: Validator agent uses Opus for quality judgment
- **Maintained Quality**: 98%+ accuracy for structured extraction

### From v4.1 (Foreground)

- **Foreground Execution**: All agents MUST run in foreground (blocking mode)
- **Catalog Updates**: Validator now updates HDARP_MASTER_CATALOG.csv
- **Chunk Management**: Validator verifies chunk counts and integrity
- **Enhanced Validator**: Quality checks + catalog + cleanup in single agent

### From v4.0 (Sraffa 4.0)

- **Version Detection**: Auto-detect v4.0 vs v3.3 from manifest
- **Quality Scoring**: 27 points total (v4.0) or 25 points (v3.3)
- **Sraffa 4.0 OCR**: Document-adaptive routing (PyMuPDF digital / EasyOCR+Agent QA scanned / Chandra 2 escalation)
- **OCR Confidence**: New 2-point quality component
- **Post-Validation Cleanup**: Automatic cleanup of validated chunk artifacts

## What This Does

Spawns **N+1 agents** in parallel (single message, multiple Task calls):
- **1 Quality Validator Agent**: Validates chunks + updates catalog + cleanup
- **N Processing Agents**: Each processes ONE unprocessed chunk

All agents spawn simultaneously following HDARP v4.0 protocol.

---

## Scope Specification (NEW — fixes silent cross-wave processing)

By default `/phdarp` processes any PREPARED batch in numeric order, regardless of which wave or campaign segment it belongs to. This is dangerous when a campaign has multiple waves and the user only wants one of them processed.

The `--wave` flag restricts ALL batch selection (Phase 1 initial pick **and** Phase 4 auto-continuation) to a single wave.

### Accepted Forms

- `--wave Wave_02` — canonical form
- `--wave 02` — zero-padded short form
- `--wave 2` — bare integer

The assistant MUST normalize all of these to the canonical `Wave_NN` form (e.g., `2` → `Wave_02`) before filtering, by reading the `wave` field on `BATCH_STATE.json` batch entries and matching exactly.

### The Invariant

> **Within scope, pick the numerically-lowest unfinished PREPARED batch. Outside scope, NEVER touch a batch — even if PREPARED batches with lower numbers exist outside the scope.**

This invariant applies to BOTH the initial batch pick and every Phase 4 continuation step.

### CRITICAL: Scope Inference from Natural Language

The user will frequently specify scope conversationally rather than with the flag. The assistant MUST infer scope from natural language and apply it as if `--wave` had been passed. Trigger phrases include (but are not limited to):

- "do wave 2", "process wave 2", "run phdarp on wave 2"
- "until wave 2 is complete", "finish wave 2"
- "wave_02 batches", "the wave 2 stuff"
- Any phrase that names a single wave in the context of a DARP command

When the user mentions a wave in their message and then invokes (or has already invoked) a DARP command, treat that wave as the scope. **Never silently process batches outside the user-stated scope.** If the user's scope intent is ambiguous, ask before processing — do not guess.

If the user explicitly says "all waves" or "everything" or invokes the command with no wave mention and no prior wave context in the conversation, then no scope is set (default behavior).

### Scope Filter Algorithm

When picking the next batch (initial Phase 1 pick AND every Phase 4 continuation step):

```python
# scope_wave is set if user passed --wave or implied a wave in natural language
# scope_wave is None if no scope was specified

def in_scope(batch_id):
    if scope_wave is None:
        return True
    return state["batches"][batch_id].get("wave") == scope_wave

prepared_in_scope = sorted(
    [bid for bid, b in state["batches"].items()
     if b.get("status") == "PREPARED" and in_scope(bid)],
    key=lambda x: int(x.split("_")[1])
)
batch_to_process = prepared_in_scope[0] if prepared_in_scope else None
# IMPORTANT: when scope_wave is set, IGNORE stale current_batch / next_to_process
# fields and always re-derive from the batches map.
```

If `batch_to_process` is None, no PREPARED batch matches the scope — print a clear message ("No PREPARED batches in scope {WAVE_ID}") and exit without spawning agents.

### Scope and the Validator

The validator is **NOT** scope-filtered. It always picks the oldest batch from `validation_queue` regardless of wave. This is intentional: validation is bookkeeping that should clear out the queue independent of which wave is currently processing.

---

## Document Type Detection (NEW — adaptive routing foundation)

Not all documents are created equal. A 7-page repo brief and a 936-page scanned banking history both look like "one batch" in the queue, but they need fundamentally different treatment.

```python
def classify_document(doc_id, manifest, chunks_dir):
    """Return one of: SMALL_TEXT, LARGE_TEXT, DENSE_TECHNICAL, SCANNED_BOOK, MEGA_DOC."""
    n_chunks = len(list(chunks_dir.glob('*.pdf')))
    pages = manifest.get('total_pages', 0)
    density = manifest.get('density_category', 'MEDIUM')  # LOW/MEDIUM/HIGH
    pages_per_chunk = pages / max(n_chunks, 1)

    if n_chunks > 100:
        return 'MEGA_DOC'
    if n_chunks > 50 and pages_per_chunk < 1.5 and density == 'HIGH':
        return 'SCANNED_BOOK'
    if density == 'HIGH' and n_chunks <= 20:
        return 'DENSE_TECHNICAL'
    if 11 <= n_chunks <= 50:
        return 'LARGE_TEXT'
    return 'SMALL_TEXT'
```

| Category | Recommended Tool | Why |
|----------|------------------|-----|
| SMALL_TEXT | phdarp (agent processing) | Standard agent processing |
| LARGE_TEXT | phdarp (agent processing) | Multiple agents on different chunk ranges |
| DENSE_TECHNICAL | phdarp (agent processing) | Visual extraction is the point |
| SCANNED_BOOK | phdarp (agent processing) | All categories use agent extraction |
| MEGA_DOC | phdarp (agent processing) | All categories use agent extraction |

### phdarp Behavior on SCANNED_BOOK / MEGA_DOC

Classification is **informational only**. phdarp processes all document categories via agent extraction regardless of the detected category. There is no adaptive routing to sraffa-ocr. On encountering a SCANNED_BOOK or MEGA_DOC document, phdarp logs the classification and continues with standard agent processing:

```
INFO: BATCH_XXX contains a SCANNED_BOOK doc (XXXX_some_book, 280 chunks).
Classification: SCANNED_BOOK (informational only)
Proceeding with agent extraction for all chunks...
```

No user action is required. All categories are processed identically via agent extraction.

---

## Sonnet Mandatory Model Policy (v5.0)

**Haiku is BANNED from the entire HDARP pipeline. No task uses Haiku.**

| Agent Role | Model | Rationale |
|------------|-------|-----------|
| Processors (N) | Sonnet | Extraction accuracy is paramount |
| Validator (1) | Opus | Quality judgment + catalog sync |
| Haiku | BANNED | Never used in HDARP pipeline |

Opus for processors: ONLY when user explicitly specifies (e.g., `--opus` flag).

### Task Tool Configuration
When spawning agents:
```yaml
# Processor agents
model: sonnet  # MANDATORY for all processors

# Validator agent
model: opus    # MANDATORY for validator
```

---

## Agent Execution Mode (CRITICAL)

**All agents MUST run in foreground (blocking mode)**:

```yaml
model: sonnet             # MINIMUM - Quality-first (never use haiku for extraction)
run_in_background: false  # MANDATORY - Never set to true
```

**Foreground Execution Rules**:
- NEVER use `run_in_background: true` for any spawned agent
- All N+1 agents spawn in ONE message and block until ALL complete
- Main agent waits for results before proceeding
- Results are immediately visible and actionable

**Why Foreground-Only**:
- Prevents orphaned agents consuming resources
- Enables immediate error detection and handling
- Maintains conversation coherence
- Allows proper catalog/state updates after completion
- Validator can update catalog only after processors complete
- **User can see all agents running in real-time**

**Parallel ≠ Background**:
- **Parallel**: Multiple agents in one message, all blocking together
- **Background**: Agent runs asynchronously, main continues (FORBIDDEN)

**Best Practices Reference**: See `SUBAGENT_MANAGEMENT_GUIDE.md`

---

## Execution Workflow

### Phase 1: Context Discovery (Main Agent)

1. **Locate Current Batch**
   - Search for `BATCH_*_CAMPAIGN_MANIFEST.md`
   - Read manifest to identify chunks and output directories

2. **Identify Available Chunks**
   - Scan `HDARP_Chunks/` or `HDARP_Processing/` for chunks
   - Pattern: `{document_name}_chunk_{NN}.pdf` or `chunk_{NN}_pages_*.pdf`

3. **Detect HDARP Version**
   - Read manifest.json for each document
   - Check for `density_mb_per_page` field
   - If present: v4.0 (27-point scoring)
   - If absent: v3.3 (25-point scoring)

4. **Assign Chunks**
   - **Validator**: Last 3-5 completed chunks + catalog duties
   - **Processors**: Next N unprocessed chunks

### Phase 2: Parallel Agent Spawning

Spawn all agents in parallel using **ONE message** with N+1 Task calls.

**CRITICAL**: All agents must use `run_in_background: false` (foreground blocking).

---

## Quality Validator Agent Prompt Template

```markdown
# HDARP Quality Validation & Catalog Agent (v4.1)

**TASK**: Validate chunks, update catalog, verify chunk management, cleanup artifacts.

**ASSIGNED CHUNKS**: {chunk_list}
**CATALOG PATH**: HDARP_MASTER_CATALOG.csv

## EXECUTION MODE

**FOREGROUND ONLY** - This agent runs in blocking mode. Do NOT use background execution.

## Phase 1: HDARP Validation

For EACH chunk, verify ALL 4 content types:
- Tables: CSV_Tables
- Equations: Equations
- Figures: Figures
- Body Text: Text

Quality Scoring:
- v4.0: 27 points (22/27 minimum)
- v3.3: 25 points (20/25 minimum)

## Phase 2: Catalog Update (NEW in v4.1)

After validation, update HDARP_MASTER_CATALOG.csv:
1. Create backup before updating
2. Update status: PENDING → COMPLETE
3. Update chunks_complete, quality_score
4. Update processing_agent: PHDARP_v4.1
5. Update completion_date, last_updated
6. Update extraction counts (tables, figures, equations)

## Phase 3: Chunk Management Verification

Verify chunk integrity:
1. Count chunk PDFs in HDARP_Processing/{doc}/chunks/
2. Count processing summaries in {doc}
3. Compare with manifest.json chunk_count
4. Flag discrepancies in validation report

## Phase 4: Post-Validation Cleanup

After PASSING validation, cleanup chunk artifacts:
- Delete chunk PDFs (after verification)
- Log in validation report

**START IMMEDIATELY**
```

---

## Chunk Processor Agent Prompt Template

```markdown
# HDARP Chunk Processing Agent {agent_number} (v4.0)

**ROLE**: Process ONE chunk using HDARP v4.0 protocol (4 content types)

**YOUR ASSIGNMENT**: Chunk {chunk_number}
**CHUNK PATH**: {chunk_path}
**OUTPUT DIRECTORY**: {output_directory}

## EXECUTION MODE

**FOREGROUND ONLY** - This agent runs in blocking mode.

## HDARP Protocol (4 Content Types)

A. Tables -> CSV Files (DARP Vision)
B. Equations -> LaTeX
C. Figures -> Markdown Descriptions
D. Body Text -> Text Storage

## Critical Requirements

- **ONE CHUNK ONLY**: Process assigned chunk only
- **ALL 4 TYPES**: Extract tables, equations, figures, AND body text
- **DOCUMENT WORK**: Create processing summary (20-40 KB)
- **VERSION AWARE**: Check manifest for v4.0 vs v3.3
- **NO BACKGROUND**: Synchronous execution only

**START NOW**: Read chunk {chunk_number} and begin extraction
```

---

## Catalog Schema Reference

**HDARP_MASTER_CATALOG.csv Fields** (for validator updates):

| Field | Type | Description |
|-------|------|-------------|
| document_id | string | Unique document identifier |
| status | string | PENDING/IN_PROGRESS/COMPLETE |
| chunks_complete | int | Number of processed chunks |
| quality_score | float | Average quality (0-25 or 0-27) |
| processing_agent | string | Agent used (PHDARP_v4.1) |
| completion_date | date | Date completed (YYYY-MM-DD) |
| tables_extracted | int | Total tables extracted |
| figures_extracted | int | Total figures described |
| equations_extracted | int | Total equations transcribed |

---

## Phase 2: Error Recovery (v4.5)

After all Phase 1 agents complete, check for failures and automatically retry:

### Error Detection

After Phase 1 completes, collect failure information:
- Chunks that failed to process (timeout, crash)
- Chunks with quality scores below threshold (< 22/27)
- Chunks with missing output files

### Retry Strategy

1. **Wait for ALL Phase 1 agents** to complete
2. **Collect failed chunk list** from agent results
3. **Spawn /phdarp retry agents** (max 5 at a time)
4. **One chunk per retry agent** (simpler, more reliable)
5. **Wait for ALL retries** to complete
6. **Generate recovery report**

### Retry Agent Spawning

```
Failed Chunks: 1-5   -> Spawn N /phdarp agents (one per chunk)
Failed Chunks: 6-10  -> Spawn 5 /phdarp agents, wait, then spawn remaining
Failed Chunks: >10   -> Flag for manual intervention
```

### Billing/Entitlement Errors (API Tier Limits)

If a processor agent fails with `API Error: Extra usage is required for 1M context`, this is a **transient billing check failure**, NOT a content or processing error (GitHub #44117). The subagent doesn't need 1M context — it starts clean with Sonnet's 200k default. The error occurs because the parent runs on Opus[1m] and the entitlement check incorrectly propagates.

**Recovery**: Wait 30 seconds, retry same spawn. If still failing, ask user to run `/extra-usage`. Do NOT treat as a content error or stop the run — agent retry WILL work once billing resolves.

### Content Filter Recovery Protocol (v6.0 — orchestrator-driven progressive decomposition)

When a processor agent reports CF (`API Error 400: Output blocked by content filtering policy`), recovery is **orchestrator-driven** — identical logic to SPHDARP v6.0.

The orchestrator calls `bisect_chunk()` from `pdf_splitter_orchestrator.py` on the failed chunk, spawns new Sonnet subagents on the halves, recursively bisects halves that still fail (down to 1-page minimum), then falls back to single-page scholarly-framing Sonnet agents (L2), and finally records exhausted pages in `ocr_pending_queue` (L3).

Sonnet subagents remain primary processors at all recovery levels. Opus does not take over unless the user explicitly requests it.

See SPHDARP v6.0 "Content Filter Recovery Protocol" for the full procedure, `cf_retry_log` schema, and proactive scholarly framing for `cf_risk_level=HIGH` documents.
See `HDARP_FAILURE_TAXONOMY.md` v2.0 for the canonical ladder definition.

Do NOT defer to OCR on first CF hit. Do NOT set DEFERRED_TO_OCR on the batch.

---

## Phase 3: Mandatory Catalog Synchronization (v4.5)

After processing and error recovery complete, MUST synchronize with catalog:

### Catalog Update Requirements

For EACH processed chunk, update HDARP_MASTER_CATALOG.csv:

| Field | Value | Source |
|-------|-------|--------|
| hdarp_status | COMPLETE/IN_PROGRESS | Processing result |
| quality_score | XX.X/27 | Quality assessment |
| completion_date | ISO date | Processing timestamp |
| chunk_count | Integer | Chunks processed |

### Sync Verification

After catalog update, verify:
1. All processed documents have catalog entries
2. Status values match BATCH_STATE.json
3. Quality scores are within valid range (0-27)

---

## Phase 3.5: Task Hygiene (v5.2)

After catalog sync and before batch continuation, clean up completed tasks to prevent token bloat during multi-batch sessions.

### Why This Matters

Each batch round creates N+1 tasks (processors + validator) plus error recovery tasks. Without cleanup, a 50-batch session accumulates 300+ stale tasks, injecting ~15,000 tokens of overhead per turn and degrading shell responsiveness.

### Cleanup Algorithm

1. **Enumerate** all tasks via `TaskList`
2. **Delete** every task with status `completed`, `failed`, or `cancelled` via `TaskUpdate` with `status: "deleted"`
3. **Verify** remaining count — only active tasks from the CURRENT round should remain

### Task Budget

**Maximum 25 active (non-deleted) tasks.** If exceeded after cleanup, log:

```
⚠ TASK BUDGET: {count} tasks remain after cleanup (budget: 25)
```

This is informational — do NOT halt processing.

### When to Run

- **Batch continuation (default)**: After EVERY Phase 3, before Phase 4 loops to Phase 1
- **Single-batch mode (`--single`)**: Once after Phase 3, before command exit

---

## Phase 4: Batch Continuation (v5.1)

After Phase 3 and Phase 3.5 (task hygiene) complete, automatically continue to the next PREPARED batch. This is the **default behavior** — processing continues until all batches are done.

### Continuation Algorithm

1. **Read BATCH_STATE.json** after Phase 3 completes
2. **Find next batch (scope-aware)**:
   - Build candidate list: every batch in `batches` map where `status == PREPARED`
     **AND** (`scope_wave is None` OR `batch.wave == scope_wave`)
   - Sort candidates by numeric batch ID (BATCH_001 < BATCH_002 < ...)
   - If candidate list is empty → no next batch (trigger "Scope exhausted" stopping condition)
   - Otherwise: next batch = first candidate
   - **Do NOT trust stale `next_to_process` field when a scope is set** — re-derive every iteration from the filtered candidate list. The stored `next_to_process` may point to an out-of-scope batch or an already-processed batch.
3. **Check stopping conditions** (see below)
4. **If continuing — advance state**:
   - Set `current_batch` = next batch ID
   - Set `next_to_process` = batch after that (or null if none remain)
   - Update `last_updated` timestamp
   - Write updated BATCH_STATE.json
5. **Print inter-batch summary** (see template below)
6. **Loop back to Phase 1** with new batch — spawn N+1 agents again

### Status Vocabulary (extended)

| Status | Meaning | Auto-continuation behavior |
|--------|---------|----------------------------|
| PENDING | Not yet prepared by /preparehdarp | Skip |
| PREPARED | Chunks ready, awaiting processor | **Pickable** |
| IN_PROGRESS | Some chunks done, more remain | Pickable |
| COMPLETE | All chunks done, awaiting validator | Skip |
| VERIFIED | Validator passed | Skip |
| BLOCKED | Unrecoverable error | **Skip + STOP** |
| **DEFERRED_TO_OCR** (LEGACY) | Was set by adaptive routing (now removed). Existing instances should be reset to PREPARED. New runs will never auto-set this status. | **Skip + ADVANCE** (does NOT stop) |
| **PARTIAL_DEFERRED** (LEGACY) | Was set by adaptive routing (now removed). Existing instances should be reset to PREPARED. New runs will never auto-set this status. | Pickable |

**Critical**: BLOCKED stops auto-continuation; DEFERRED_TO_OCR does NOT.

---

### Stopping Conditions

#### Context Budget is NOT a Stopping Condition (CRITICAL)

**Auto-compression handles context limits transparently.** From the system prompt: "The system will automatically compress prior messages in your conversation as it approaches context limits. This means your conversation with the user is not limited by the context window."

For HDARP processing specifically, auto-compression is **zero-cost**: persistent state lives in `BATCH_STATE.json` and `HDARP_MASTER_CATALOG.csv` on disk, NOT in conversation memory.

**The assistant MUST NOT** write a handoff document as a "context management" strategy, stop processing because "context is filling up", or stop preemptively based on round-count estimates.

**The assistant MAY stop ONLY when** genuine model degradation is observable, a real Stopping Condition below is met, or the user explicitly says stop.

If the assistant catches itself thinking "I should write a handoff and stop because context is getting tight" — that thought is wrong. Continue processing.

#### Real Stopping Conditions

Processing stops when ANY of these are true:
- **Scope exhausted** — when a scope is set, all PREPARED batches matching the scope have been processed → print final summary and exit. Do NOT cross over to other waves.
- **No more PREPARED batches** remain in BATCH_STATE.json (no scope set) → print final summary and exit
- **Next batch is BLOCKED** → stop, report which batch is blocked and why. (DEFERRED_TO_OCR is NOT BLOCKED — see Status Vocabulary. DEFERRED batches are skipped without stopping.)
- **User passed `--single` flag** → skip Phase 4 entirely (process exactly one batch)
- **User explicitly requested stop** during the run

Context budget does not appear in this list. It is not a stopping condition.

### Inter-Batch Summary Template

Print between batches (keep brief — do NOT ask for confirmation). Include the `Scope:` line ONLY when a scope is set; omit it for unscoped runs:

```
─── BATCH CONTINUATION ───
Scope: Wave_02 (28 of 165 in scope done)
Completed: BATCH_390 (10 chunks, 1 doc, 24.2/27 avg quality)
Next: BATCH_391 (8 chunks, 1 doc)
Batches remaining in scope: 137 PREPARED
Validation queue: [BATCH_388, BATCH_389, BATCH_390]
Continuing with N processors + 1 validator...
───────────────────────────
```

### Final Summary Template

Print when processing stops. Include the `Scope:` line ONLY when a scope is set:

```
═══ BATCH PROCESSING COMPLETE ═══
Scope: Wave_02
Batches processed this session: 165 (BATCH_363 through BATCH_527)
Total chunks: 3248
Total documents: 316
Average quality: 24.1/27
Validation queue: [BATCH_525, BATCH_526, BATCH_527]
Remaining PREPARED in scope: 0
Scope status: COMPLETE
═════════════════════════════════
```

### Validator Across Batch Boundaries

The previous-batch validation architecture handles transitions naturally:
- Batch N completes → added to `validation_queue`
- Batch N+1 starts → validator picks up batch N from queue
- **No special handling needed** — the architecture already supports seamless transitions

### Error Handling During Continuation

- **Batch ends COMPLETE** (some chunks may need manual review) → **CONTINUE** to next batch
- **Batch ends BLOCKED** (>10 failures, unrecoverable errors) → **STOP**, report: "BATCH_XXX BLOCKED — manual intervention required. Processed N batches before stopping."
- **Batch ends IN_PROGRESS** (not all chunks done) → **CONTINUE** — remaining chunks will be picked up in a future run

### Single-Batch Mode

When user passes `--single`:
```
/phdarp 5 --single    # Process ONE batch only (pre-v5.1 behavior)
```
Skip Phase 4 entirely. Command ends after Phase 3.

### CRITICAL: Do NOT Prompt Between Batches

**The continuation loop must NOT ask for user confirmation between batches.** The entire point is autonomous multi-batch processing. The only user interaction points are:
- Start: User invokes `/phdarp N`
- End: Processing stops (no more batches, BLOCKED, or `--single`)

---

## Error Handling

- **No batch manifest**: "No active HDARP batch found"
- **Insufficient chunks**: Adjust N to match available
- **N > 10**: Cap at 10 processors maximum
- **No completed chunks**: Skip validator, spawn processors only
- **Catalog update failure**: Log error, do NOT cleanup chunks

---

## Examples

```bash
/phdarp        # 5 processors + 1 validator, continues until done
/phdarp 3      # 3 processors + 1 validator, continues until done
/phdarp 10     # 10 processors + 1 validator (max), continues until done
/phdarp 5 --single   # ONE batch only, then stop
```

---

## Best Practices Reference

See `SUBAGENT_MANAGEMENT_GUIDE.md` for:
- Context management strategies
- Foreground execution rationale
- Parallel agent patterns
- HDARP-specific orchestration

---

**Command Version**: 6.1 Batch Continuation + Sonnet Mandatory + Opus Validator + Mandatory Catalog Sync + Error Recovery
**Status**: PRODUCTION READY
**Created**: 2025-12-22
**Updated**: 2026-04-06
**HDARP Protocol**: v6.3 (Batch Continuation, Sonnet Mandatory, Opus Validator, 10-chunk batching)
**New in v5.1**: Automatic batch continuation (Phase 4), `--single` flag, inter-batch summaries, BLOCKED detection
**v5.0**: Sonnet Mandatory model policy, Opus validator, 10-chunk batch sizing
**v4.5**: Mandatory catalog synchronization, Phase 3 catalog sync, unified features
**v4.1**: Foreground execution, catalog updates, chunk management
**Key**: P=Parallel (all commands), no S=no Smart batching, H=Hybrid (includes OCR)

---

## HDARP Lifecycle Position

This command is step **3** (PROCESS) in the HDARP lifecycle:
1. `/hdarp-campaign` — campaign setup
2. `/preparehdarp` — chunk and prepare PDFs
3. **`/phdarp`** — extract all 4 content types (tables, equations, figures, body text)
4. `/hdarp-wrapup` — validate, remediate, document, close out
5. `/hdarp-integrate-pipeline` — catalog, classify, crossref, Robert sync

**ALL 4 content types are MANDATORY per chunk.** If a chunk has no tables/equations/figures, the agent must explicitly confirm "none found" — silent omission is not acceptable.

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added. -->
