---
description: "Smart Parallel DARP v6.4: Native RDB Enrichment + Task Hygiene + Batch Continuation + Sonnet Mandatory + Opus Validator + Mandatory Catalog Sync + Document-aware batching (NO OCR)"
allowed-tools: Bash, Read, Write, Glob, Grep, Task
argument-hint: "[number]"
---

**HDARP Framework v6.4** — see `VERSION_REGISTRY.md`

> **Native RDB metadata capture (v6.4):** SPDARP extracts tables (no OCR) — its tables are the whole point, so native capture is high-value. Processors emit one `RDB_METADATA*.jsonl` line per table CSV per `hdarp-processing.md` ("Native RDB Enrichment Capture") + spec `NATIVE_ENRICHMENT_CONTRACT.md`. Same chunk-range/whole-doc + WARN-only validator rules as `sphdarp.md`.

# Smart Parallel Direct Agent Reading Protocol (SPDARP) v6.4

**Command**: /spdarp [N]
**Full Name**: Smart Parallel Direct Agent Reading Protocol
**Version**: 6.3
**Created**: 2026-01-01
**Updated**: 2026-04-14

## 🛑 CRITICAL RULE: NEVER FABRICATE (v5.3)

If you encounter a content filter error, timeout, or any API error: RECORD the affected chunks in Failures and CONTINUE. **DO NOT** generate substitute content, paraphrase from memory, or write block quotes / attributions you haven't verified verbatim from the source PDF. Mark uncertain passages `[approximate]` — gap markers are always preferable to fabrications. See `HDARP_CONTENT_FILTER_PATTERNS.md`.

**DO NOT silently substitute extraction methods.** If agent extraction fails, STOP and report. Do not switch to PyMuPDF, bulk scripts, or any non-DARP method without explicit user approval. See the 2026-05-06 silent degradation incident in sphdarp.md.

## Scrounger Standard

SPDARP is a scrounger for its 3 content types (tables, equations, figures). Only three outcomes are acceptable reasons for a chunk to have no extraction:

1. **DUPLICATE** — confirmed identical content already in KB (MD5 hash match, not assumption)
2. **CF_EXHAUSTED** — content filter on every page after full bisect (L1) + scholarly framing (L2) retry on every page
3. **QUARANTINED** — file physically unreadable by every tool (0 bytes, crashes PyMuPDF)

**On CF**: Bisect the chunk, retry halves (L1). If still CF, retry each page with scholarly/analytical framing (L2). Only after both levels fail on a specific page is that page declared CF_EXHAUSTED. Do NOT defer the whole batch on first CF hit.

**On timeout**: Bisect the chunk, retry smaller pieces. Single-page agents if needed.

**SPDARP-specific note**: SPDARP has no OCR body text — L3 OCR queue deferral does not apply to the 3 DARP types. Smart batching (document-aware assignment) does not change the scrounger obligation: each document's tables/equations/figures must be extracted by agents.

See `sphdarp-scrounger.md` for the full acceptable-outcomes taxonomy and retry behavior tree.

## BATCH_STATE Integration

All DARP commands now integrate with BATCH_STATE.json:

**Location**: {Project}/Technical/HDARP_Processing/BATCH_STATE.json

**Key Changes**:
- Command reads batch_to_process from state
- Validator validates PREVIOUS batch (not current)
- State updated after processing completes

See BATCH_STATE_PROTOCOL.md for full details.

---

## Usage

/spdarp                       # 5 processors + 1 validator, continues until all batches done
/spdarp 3                     # 3 processors, continues until all batches done
/spdarp 10                    # 10 processors (max), continues until all batches done
/spdarp 5 --single            # Process ONE batch only, then stop
/spdarp 5 --wave Wave_02      # Process ONLY Wave_02 batches, lowest-numbered first
/spdarp 5 --wave 2            # Same — short form accepted (normalized to Wave_02)

## Purpose

DARP-only extraction when OCR is already complete. Extracts:
- **Tables** -> CSV files (DARP vision, 98%+ accuracy)
- **Equations** -> LaTeX format (100% target)
- **Figures** -> Markdown descriptions (200+ words per figure)

**NO OCR/Body Text extraction** - Use /sphdarp if OCR is also needed.

## DARP Command Family

| Command | Smart Batching | Hybrid (OCR) | Use Case |
|---------|----------------|--------------|----------|
| /pdarp | No | No | Quick DARP extraction, one chunk per agent |
| /phdarp | No | Yes | Full extraction, one chunk per agent |
| **/spdarp** | Yes | No | DARP extraction, all chunks from one document |
| /sphdarp | Yes | Yes | Full extraction, all chunks from one document |

**Key Distinctions**:
- **P** = Parallel (all commands have this)
- **S** = Smart (document-aware batching)
- **H** = Hybrid (includes OCR body text)

## What is New in v4.5

- **Mandatory Catalog Synchronization**: All processing MUST update HDARP_MASTER_CATALOG.csv
- **Sonnet Mandatory Model Policy**: ALL processor agents use Sonnet, validator uses Opus — Haiku BANNED
- **Error Recovery Phase**: Failed chunks automatically retried
- **Quality Score Propagation**: Scores flow from processing to catalog automatically
- **Unified v5.0 Standard**: Consistent with all DARP commands

### From v1.1 (Error Recovery)

- **Error Recovery**: Failed chunks automatically retried with /pdarp agents
- **Max 5 Chunks**: Reduced from 6 to prevent timeouts

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

## When to Use /spdarp vs /sphdarp

| Scenario | Command |
|----------|---------|
| OCR already complete (FULL_TEXT.md exists) | **/spdarp** |
| Need full extraction (OCR + DARP) | /sphdarp |
| Tables, equations, figures only | **/spdarp** |
| Complete document processing | /sphdarp |

## What This Does

Spawns N+1 agents in parallel (single message, multiple Task calls):
- 1 Quality Validator Agent: Validates completed chunks + catalog + cleanup
- N Processing Agents: Each processes ALL chunks from ONE assigned document

Key Feature: Document-aware smart batching - each processor handles ALL chunks from ONE document.

---

## Scope Specification (NEW — fixes silent cross-wave processing)

By default `/spdarp` processes any PREPARED batch in numeric order, regardless of which wave or campaign segment it belongs to. This is dangerous when a campaign has multiple waves and the user only wants one of them processed.

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

- "do wave 2", "process wave 2", "run spdarp on wave 2"
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
| SMALL_TEXT | spdarp / sphdarp (current) | Standard agent processing |
| LARGE_TEXT | sphdarp with chunk-range parallelism | Multiple agents on different chunk ranges |
| DENSE_TECHNICAL | spdarp / sphdarp (current) | Visual extraction is the point |
| SCANNED_BOOK | spdarp / sphdarp (agent processing) | All categories processed by agents |
| MEGA_DOC | spdarp / sphdarp (agent processing) | All categories processed by agents |

### spdarp Behavior on SCANNED_BOOK / MEGA_DOC

Document classification is **informational only**. All document categories — including SCANNED_BOOK and MEGA_DOC — are processed by agents using standard smart-batched DARP extraction. No adaptive routing or deferral to external tools occurs. The classification output may be logged for reference but does not change the processing path.

---

## Smart Batch Assignment Algorithm

Goal: Assign each processor ALL chunks from ONE document (up to 5 chunks max)

Assignment Rules:
1. One Document Per Agent: Each processor works on exactly ONE document
2. Largest First: Documents with most chunks assigned first
3. Max 5 Chunks: Cap at 5 chunks per agent to prevent timeouts
4. Min 2 Chunks: If document has less than 2 chunks, combine with similar

Example Assignment (5 processors):
- Processor 1: Document_A chunks 01-05 (5 chunks)
- Processor 2: Document_B chunks 03-05 (3 chunks)
- Processor 3: Document_C chunks 01-03 (3 chunks)
- Processor 4: Document_D chunks 01-02 (2 chunks)
- Processor 5: Document_E chunks 01-03 (3 chunks)
- Validator: Quality check + catalog update

---

## Critical Requirements

- ONE DOCUMENT PER AGENT: Each processor works on exactly ONE document
- ALL CHUNKS SEQUENTIALLY: Process in numerical order within document
- **3 CONTENT TYPES PER CHUNK**: Tables, equations, figures (NO body text)
- QUALITY TARGET: 18/22 (80%) per chunk
- NO BACKGROUND: Synchronous execution only

---

## Agent Execution Mode (CRITICAL)

**All agents MUST run in foreground (blocking mode)**:

```yaml
model: sonnet             # Processors: Sonnet mandatory (Haiku BANNED)
run_in_background: false  # MANDATORY - Never set to true
# Validator uses: model: opus
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
- **User can see all agents running in real-time**

**Parallel ≠ Background**:
- **Parallel**: Multiple agents in one message, all blocking together
- **Background**: Agent runs asynchronously, main continues (FORBIDDEN)

**Best Practices Reference**: See `SUBAGENT_MANAGEMENT_GUIDE.md`

---

## Quality Scoring (SPDARP - 22 Points Max)

| Component | Max Points | Description |
|-----------|------------|-------------|
| **A. Tables** | 10 points | Complete CSV extraction |
| - Complete extraction | 4 points | All tables captured |
| - Accurate headers | 3 points | Headers match source |
| - Numerical precision | 3 points | Numbers accurate |
| **B. Equations** | 5 points | LaTeX transcription |
| - All equations captured | 3 points | None missed |
| - LaTeX accuracy | 2 points | Compiles correctly |
| **C. Figures** | 5 points | Markdown descriptions |
| - All figures described | 3 points | None missed |
| - Comprehensive analysis | 2 points | 200+ words each |
| **D. Completeness** | 2 points | Processing quality |
| - Summary complete | 1 point | Summary file present |
| - No errors | 1 point | Clean processing |

**Thresholds**:
- Minimum (REJECT): <18/22 points (<80%)
- Target (IDEAL): >=20/22 points (>=90%)
- Acceptable: 18-19/22 points (80-86%)

---

## Document Processor Agent Template (SPDARP v1.0)

When spawning processor agents, use this structure:

**Agent Role**: Process ALL assigned chunks from ONE document using DARP protocol (3 content types)

**Agent Assignment**:
- YOUR DOCUMENT: {document_name}
- YOUR CHUNKS: {chunk_list} ({chunk_count} chunks total)
- OUTPUT DIRECTORY: Knowledge_Base/{document_name}/

> **DARP-only — no body OCR, and the Hybrid mandate does not apply.** HDARP's mandatory Stage-5 verbatim OCR sibling (`/sraffa-ocr --augment`, see `CLAUDE.md` → HDARP lifecycle) belongs to the **H** commands. ``/spdarp`` extracts no body text, so it produces no sibling and is never gated on one. Use ``/sphdarp`` when body text is needed.

**SPDARP Extraction (3 Content Types - NO OCR)**:
- A. Tables -> CSV (98%+ accuracy, DARP vision)
- B. Equations -> LaTeX (100% target)
- C. Figures -> Markdown (200+ words per figure)

**DO NOT EXTRACT**: Body text (OCR already complete or not needed)

**Sequential Processing**:
1. Read Chunk N -> Extract all 3 types -> Quality Check
2. Repeat for each assigned chunk in order
3. Create Document Completion Report

**Output Structure**:
```
Knowledge_Base/{document_name}/
├── CSV_Tables/
│   ├── table_001.csv
│   └── table_002.csv
├── Equations/
│   ├── equation_001.md
│   └── equation_002.md
├── Figures/
│   ├── figure_001.md
│   └── figure_002.md
└── SPDARP_PROCESSING_SUMMARY.md
```

---

## Quality Validator Agent Template (SPDARP v1.0)

**Validator Tasks (validates PREVIOUS batch, not current)**:
1. Verify Document Completeness (all chunks processed)
2. Quality Assessment (score each document on 22-point scale)
3. Update tracking files (catalog, remaining_work.json)
4. **Post-Validation Cleanup (MANDATORY)**: After marking batch as VERIFIED:
   - Delete chunk PDFs: `Technical/HDARP_Processing/{doc_id}/chunks/chunk_*.pdf`
   - Verify Knowledge_Base content is intact before deleting
   - Update HDARP_MASTER_CATALOG.csv status to ARCHIVED
   - Log freed space: "Cleaned batch {batch_id}: freed {X} MB"

**Quality Check (3 Content Types)**:
- Tables: CSV_Tables/ directory present with files
- Equations: Equations/ directory present with files
- Figures: Figures/ directory present with files

**NOT CHECKED**: Body text (not extracted by SPDARP)

---

## Comparison: SPDARP vs SPHDARP

| Feature | SPHDARP (Full) | SPDARP (DARP Only) |
|---------|----------------|-------------------|
| Full Name | Smart Parallel Hybrid DARP | Smart Parallel Direct Agent Reading |
| Tables | Yes (DARP) | Yes (DARP) |
| Equations | Yes (LaTeX) | Yes (LaTeX) |
| Figures | Yes (Markdown) | Yes (Markdown) |
| Body Text OCR | Yes (Sraffa 4.0) | **NO** |
| Hybrid Stage-5 verbatim sibling | **Mandatory, every document** | **N/A** — no body text, so no sibling |
| Quality Points | 27 max | 22 max |
| Minimum Score | 22/27 (80%) | 18/22 (80%) |
| Use Case | Full extraction | Tables/equations/figures only |
| When to Use | New documents | OCR already done |

---

## Phase 1 Error Handling

- No remaining work: All chunks processed - batch complete
- Fewer docs than N: Reduce N to match available documents
- N > 10: Cap at 10 processors maximum
- Document with >5 chunks: Assign first 5, remaining for next round
- Document with 1 chunk: Combine with another small document

---

## Phase 2: Error Recovery (NEW in v1.1)

After all Phase 1 agents complete, check for failures and automatically retry:

### Error Detection

After Phase 1 completes, collect failure information:
- Chunks that failed to process (timeout, crash)
- Chunks with quality scores below threshold (< 18/22)
- Chunks with missing output files

### Retry Strategy

1. **Wait for ALL Phase 1 agents** to complete (never retry during Phase 1)
2. **Collect failed chunk list** from agent results
3. **Spawn /pdarp retry agents** (max 5 agents at a time)
4. **One chunk per retry agent** (simpler, more reliable)
5. **Wait for ALL retries** to complete
6. **Generate recovery report**

### Retry Agent Spawning

If failed chunks > 0:

```
Failed Chunks: 1-5   -> Spawn N /pdarp agents (one per chunk)
Failed Chunks: 6-10  -> Spawn 5 /pdarp agents, wait, then spawn remaining
Failed Chunks: >10   -> Flag for manual intervention
```

### Why /pdarp for Retries

- `/pdarp` processes ONE chunk at a time (simpler, more reliable)
- Reduces complexity when dealing with problematic chunks
- Easier to diagnose issues with isolated processing
- Better error messages for individual failures

### Recovery Report Template

After retries complete, generate:

```markdown
## Error Recovery Report

**Phase 1 Results**: X of Y chunks successful
**Failed Chunks**: Z chunks required retry

| Chunk | Original Failure | Retry Result | Final Status |
|-------|------------------|--------------|--------------|
| doc_A_chunk_03 | Timeout | SUCCESS | COMPLETE |
| doc_C_chunk_07 | Low quality (14/22) | SUCCESS (20/22) | COMPLETE |
| doc_B_chunk_12 | Table extraction failed | FAILED | NEEDS MANUAL |

**Final Status**: X chunks complete, Y need manual review
```

### Billing/Entitlement Errors (API Tier Limits)

If a processor agent fails with `API Error: Extra usage is required for 1M context`, this is a **transient billing check failure**, NOT a content or processing error (GitHub #44117). The subagent doesn't need 1M context — it starts clean with Sonnet's 200k default. The error occurs because the parent runs on Opus[1m] and the entitlement check incorrectly propagates.

**Recovery**: Wait 30 seconds, retry same spawn. If still failing, ask user to run `/extra-usage`. Do NOT mark as DEFERRED_TO_OCR or content filter error — agent retry WILL work once billing resolves. Do NOT stop the run.

### Unrecoverable Errors

These require manual intervention:
- Document not found -> Skip, report
- Retry also failed -> Flag for manual review
- >10 chunks failed in Phase 1 -> Flag for manual intervention

---

## Phase 3: Mandatory Catalog Synchronization (v4.5)

After processing and error recovery complete, MUST synchronize with catalog:

### Catalog Update Requirements

For EACH processed document, update HDARP_MASTER_CATALOG.csv:

| Field | Value | Source |
|-------|-------|--------|
| hdarp_status | COMPLETE/IN_PROGRESS | Processing result |
| quality_score | XX.X/22 | Quality assessment |
| completion_date | ISO date | Processing timestamp |
| chunk_count | Integer | Chunks processed |
| tables_extracted | Integer | Table count |
| equations_extracted | Integer | Equation count |
| figures_extracted | Integer | Figure count |

### Sync Verification

After catalog update, verify:
1. All processed documents have catalog entries
2. Status values match BATCH_STATE.json
3. Quality scores are within valid range (0-22)

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
| **DEFERRED_TO_OCR** (LEGACY) | Was set by adaptive routing (now removed). Existing instances should be reset to PREPARED. New runs will never auto-set this status. | **Skip + ADVANCE** |
| **PARTIAL_DEFERRED** (LEGACY) | Was set by adaptive routing (now removed). Existing instances should be reset to PREPARED. New runs will never auto-set this status. | Pickable |

**Critical**: BLOCKED stops auto-continuation. DEFERRED_TO_OCR and PARTIAL_DEFERRED are legacy statuses from adaptive routing which has been removed — reset any existing instances to PREPARED.

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
Completed: BATCH_390 (10 chunks, 1 doc, 20.1/22 avg quality)
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
Average quality: 20.0/22
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
/spdarp 5 --single    # Process ONE batch only (pre-v5.1 behavior)
```
Skip Phase 4 entirely. Command ends after Phase 3.

### CRITICAL: Do NOT Prompt Between Batches

**The continuation loop must NOT ask for user confirmation between batches.** The entire point is autonomous multi-batch processing. The only user interaction points are:
- Start: User invokes `/spdarp N`
- End: Processing stops (no more batches, BLOCKED, or `--single`)

---

## Canonical Tools

| Tool | Path | Use |
|------|------|-----|
| PDF Splitter | pdf_splitter_orchestrator.py | Chunking |

**NOT USED by SPDARP**:
- sraffa30_processor.py (OCR - not needed)
- sraffa30_ocr_engines.py (OCR - not needed)
- sraffa30_consensus.py (OCR - not needed)

---

## Best Practices

1. **Use SPDARP when**: FULL_TEXT.md already exists in Knowledge Base
2. **Use SPHDARP when**: Need complete extraction including body text
3. **Check first**: Verify KB directory has FULL_TEXT.md before using SPDARP
4. **Smart batching**: Let algorithm assign documents, don't override
5. **Quality focus**: 18/22 minimum, aim for 20/22

---

**Command Version**: 6.3 Native RDB Enrichment + Batch Continuation + Sonnet Mandatory + Opus Validator + Mandatory Catalog Sync
**Status**: PRODUCTION READY
**Created**: 2026-01-01
**Updated**: 2026-06-13
**Protocol**: DARP-only (3 content types, no OCR)
**Quality Scoring**: 22 points (18/22 minimum)
**New in v5.1**: Automatic batch continuation (Phase 4), `--single` flag, inter-batch summaries, BLOCKED detection
**v5.0**: Sonnet Mandatory model policy, Opus validator, 10-chunk batch sizing
**v4.5**: Mandatory catalog synchronization, Phase 3 catalog sync, unified features
**Key**: S=Smart batching, P=Parallel (all commands), no H=no Hybrid (no OCR)

---

## HDARP Lifecycle Position

This command is step **3** (PROCESS) in the HDARP lifecycle:
1. `/hdarp-campaign` — campaign setup
2. `/preparehdarp` — chunk and prepare PDFs
3. **`/spdarp`** — extract structured content with smart batching (tables, equations, figures — NO body text OCR)
4. `/hdarp-wrapup` — validate, remediate, document, close out
5. `/hdarp-integrate-pipeline` — catalog, classify, crossref, Robert sync

**ALL 3 DARP content types are MANDATORY per chunk** (tables, equations, figures). If a chunk has none, the agent must explicitly confirm "none found."

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added. -->
