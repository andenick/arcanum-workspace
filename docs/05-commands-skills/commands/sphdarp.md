---
description: "Parallel HDARP Smart v6.4: Native RDB Enrichment + CF Progressive Decomposition + Task Hygiene + Batch Continuation + Sonnet Mandatory + Opus Validator + Mandatory Catalog Sync"
allowed-tools: Bash, Read, Write, Glob, Grep, Task
argument-hint: "[number]"
---

**HDARP Framework v6.4** — see `VERSION_REGISTRY.md`

## v6.2 MANDATE — Per-Document Processing Detail + Full-Mirror Robert Sync

Every SPHDARP run MUST emit a per-document processing-detail record into `HDARP_MASTER_CATALOG.csv` + `HDARP_Integration/DOCUMENT_AUDIT.csv`: `source_md5`, `chunks_split`, `chunks_processed`, `chunks_complete`, `tables_count`, `equations_count`, `figures_count`, `full_text_bytes`, `full_text_chunks`, `handwriting_count` (if any), `quality_score`, `status`, `pdf_type`, `preparation/processing/completion dates`, `processing_agent`, `validator`. **A document is not COMPLETE until this row exists** (absence of a content type is recorded as `0`, never omitted). Integration then performs **full-mirror robert-sync**: mirror PDFs + KB + catalogs into Robert AND merge the project's rows into the 5 `_UNIFIED/*.csv` indexes (never registry-only — verify with `grep -c <Project>`). Full spec: `hdarp-processing.md` (per-document-detail + full-mirror robert-sync sections) and `VERSION_REGISTRY.md`.

# Parallel HDARP Smart Batching Command (SPHDARP) v6.4

**Command**: /sphdarp [N]
**Full Name**: Smart Parallel Hybrid Direct Agent Reading Protocol
**Version**: 6.4
**Updated**: 2026-08-27

## 🛑 CRITICAL RULE: NEVER FABRICATE (v6.0)

**If you encounter a content filter error (API 400 `Output blocked by content filtering policy`), a timeout, or any other API error that prevents extracting a chunk:**

1. **RECORD** the affected chunks in the Failures list
2. **CONTINUE** with remaining chunks in your assigned range
3. **DO NOT** generate substitute content from memory or inference
4. **DO NOT** paraphrase or reconstruct quotations you can't verify verbatim from the source PDF
5. **DO NOT** write block quotes, letters, poems, or attributed quotations unless the exact text is visible in the PDF page you're reading
6. When uncertain whether a passage is verbatim, mark it `[approximate]` or `[paraphrased]` — **a gap marker is always preferable to a fabrication**
7. **DO NOT silently substitute extraction methods.** If SPHDARP agent extraction fails (content filter, timeout, path error), you MUST report the failure and stop. You may NOT switch to PyMuPDF `page.get_text()`, OCR scripts, or any other bulk extraction method **in place of** agent reading without explicit user approval. A text dump is NOT SPHDARP output — it is missing tables, equations, figures, and structured markup. **This forbids substitution, not the Hybrid sibling:** **Hybrid Stage 5** (this file's content type **D2**; executed as **Phase 5** below) runs *after* a complete agent extraction, into a separate tree, and adds to it — see `hdarp-processing.md`, "Hybrid body text = two layers". Running D2 early never cures a failed agent extraction, and D2 output is never validated as SPHDARP output.

**Incident 1 — Fabrication (2026-04-13):** BATCH_1194 (Kynaston *City of London*) contained hallucinated block quotes — a fake Marianne Thornton letter, an invented Peacock poem, a fabricated Cardinal Manning recollection, and a Bagehot quote that doesn't exist in the source. The processor had hit a content filter error mid-chunk and tried to "save the round" by generating plausible-sounding substitutes. Undetected fabrications in academic source material corrupt downstream citations and are strictly worse than honest gap markers. See `HDARP_CONTENT_FILTER_PATTERNS.md`.

**Incident 2 — Silent Method Degradation (2026-05-06):** An orchestrator attempted SPHDARP on 31 scanned books. When agents failed (content filter, path errors, timeouts), the orchestrator silently switched to PyMuPDF `page.get_text()` batch extraction, produced body-text-only output missing all tables/equations/figures, validated at 24-25/27, marked 35 batches VERIFIED, and deleted 5.79 GB of chunk PDFs. The user discovered the degradation only when asking about the processing method. This is as dangerous as fabrication: it produces output that LOOKS complete but is missing 75% of the structured content that HDARP exists to extract. See `PLAN_ANTI_SILENT_DEGRADATION.md`.

## SCROUNGER PRINCIPLE (v6.1 — added 2026-05-06)

See `sphdarp-scrounger.md` for the full standard.

**Summary**: Every chunk gets full 4-type extraction. The only acceptable non-extraction outcomes are DUPLICATE (confirmed identical), CF_EXHAUSTED (after full L1 bisect / L2 scholarly / L3 OCR ladder on every page), or QUARANTINED (physically unreadable file). Everything else gets extracted — confusion, scan quality, and partial failure are reasons to try harder, not reasons to defer. The protocol is a scrounger: it retries, bisects, reframes, and adapts until the content is extracted or provably impossible.

## Usage

/sphdarp                       # 5 processors + 1 validator, continues until all batches done
/sphdarp 3                     # 3 processors, continues until all batches done
/sphdarp 10                    # 10 processors (max), continues until all batches done
/sphdarp 5 --single            # Process ONE batch only, then stop
/sphdarp 5 --wave Wave_02      # Process ONLY Wave_02 batches, lowest-numbered first
/sphdarp 5 --wave 2            # Same — short form accepted (normalized to Wave_02)
/sphdarp 5 --section A         # Process ONLY drain_section=="A" batches (multi-session sharding, v6.2)

## CONTINUOUS MODE (v6.2 — continuousness-first)

> Full doctrine: `orchestration-cadence.md`. Continuousness is the first priority; looping is only a resume aid.

**Default behavior is CONTINUOUS, not one-round-and-yield.** When `/sphdarp N` runs (with or without `/loop`), the orchestrator processes **as many rounds inline within a single turn** as it can — select → spawn N processors + 1 validator → commit → immediately select the next round — until a real **yield trigger** fires. Returning control to the user (or scheduling a `/loop` wake-up) after a single round is an **anti-pattern**: it idles the session and pays a cold prompt-cache cost (~5-min TTL) for zero extraction.

**Yield triggers (the ONLY reasons to stop a turn):**
1. Disk guard trips (C: < 20 GB) — pause and report.
2. A non-transient blocker needs the user (BLOCKED batch, ambiguous scope, repeated unrecoverable error).
3. The scope/campaign/section is fully drained (PREPARED == 0) **and Phase 5 (Hybrid Stage 5) has
   run with its completion test passing.** Draining is not finishing — see Phase 5.
4. Context is about to overflow and checkpointing now loses less work than after a forced compaction.
5. A hard rate/billing wall that 30s→5min retry did not clear.

**`/loop` is the resume hook, not the pacing mechanism.** Use `/loop /sphdarp N [--section X]` ONLY to re-enter the drain *after* a session/turn boundary is forced (overnight, forced compaction). When you must schedule, keep the cache window in mind: stay <270s (warm) or commit to ≥1200s (amortized) — never schedule 300s, never loop once per round. For an objective-bounded autonomous run ("drain section A to zero"), keep the plan→act→observe loop running **inline, in-session, until the objective is met** — see `orchestration-cadence.md` §4.

## DARP Command Family

| Command | Smart Batching | Hybrid (OCR) | Use Case |
|---------|----------------|--------------|----------|
| /pdarp | No | No | Quick DARP extraction, one chunk per agent |
| /phdarp | No | Yes | Full extraction, one chunk per agent |
| /spdarp | Yes | No | DARP extraction, all chunks from one document |
| **/sphdarp** | Yes | Yes | Full extraction, all chunks from one document |

**Key Distinctions**:
- **P** = Parallel (all commands have this)
- **S** = Smart (document-aware batching)
- **H** = Hybrid — **both readings of the same pages, woven**: agent-read body text (the layer
  validated as HDARP) **plus** a Sraffa 4.0 verbatim OCR sibling in
  `Knowledge_Base/_OCR_Only/<short_id>/`, run for **every** document as **Hybrid Stage 5 (D2)** —
  a mandatory stage of this command, executed as **Phase 5** below.
  Hybrid is **augmentation, never substitution** — the OCR layer is added to a completed agent
  extraction, never put in its place. Canonical rule: `hdarp-processing.md`, "Hybrid body text = two layers"
  (which also carries the alias list — Stage 5 / `--augment` / Mode 4 / D2 are one stage).

## What is New in v6.0 (CF PROGRESSIVE DECOMPOSITION)

### From v6.0 (CF Progressive Decomposition)

- **Orchestrator-driven CF recovery**: Content filter recovery is now handled by the orchestrator, not individual subagents. When a subagent reports CF, the orchestrator splits the chunk and re-dispatches to new Sonnet subagents.
- **`bisect_chunk()` integration**: Orchestrator calls `bisect_chunk()` from `pdf_splitter_orchestrator.py` to physically split failed chunk PDFs. Supports recursive bisection (chunk_005_a.pdf → chunk_005_aa.pdf / chunk_005_ab.pdf).
- **Three-level recovery ladder**: L1 bisect+re-dispatch (recursive) → L2 single-page Sonnet with scholarly framing → L3 OCR fallback (only for truly exhausted pages)
- **Proactive scholarly framing**: Documents with `cf_risk_level=HIGH` in manifest get analytical framing on ALL processor agents from first dispatch
- **Subagents remain primary**: Opus never takes over CF handling. All recovery uses Sonnet subagents.
- **Replaces**: The v5.3 subagent-driven 3-strategy CF Retry Protocol (Strategy 1/2/3)

## What is New in v5.2 (TASK HYGIENE)

### From v5.2 (Task Hygiene)

- **Phase 3.5 Task Cleanup**: Completed tasks deleted after each batch round to prevent token bloat
- **Task Budget**: Maximum 25 active tasks monitored (informational, not blocking)
- **Long-Session Stability**: Eliminates ~15,000 token overhead that previously made 50+ batch sessions unusable

## What is New in v5.1 (BATCH CONTINUATION)

### From v5.1 (Batch Continuation)

- **Automatic Continuation**: After completing a batch, automatically advances to next PREPARED batch and loops back to Phase 1
- **Inter-Batch Summary**: Brief progress report printed between batches
- **`--single` Flag**: Process exactly one batch then stop (pre-v5.1 behavior)
- **BLOCKED Detection**: Stops automatically when a batch requires manual intervention
- **Default is continuous**: `/sphdarp N` processes ALL available PREPARED batches unless `--single` is passed

## What is New in v4.5 (MANDATORY CATALOG SYNC)

- **Mandatory Catalog Synchronization**: All processing commands MUST update HDARP_MASTER_CATALOG.csv
- **Phase 3 Added**: Post-processing catalog sync phase (after error recovery)
- **Quality Score Propagation**: Scores flow from processing to catalog automatically
- **Unified v4.5 Standard**: All DARP commands now use consistent features

### From v5.0 (Sonnet Mandatory Model Policy)

- **Sonnet Mandatory**: ALL processor agents use Sonnet — Haiku BANNED from entire pipeline
- **Opus Validator**: Validator agent uses Opus for quality judgment
- **Maintained Quality**: 98%+ accuracy for structured extraction

### From v4.2 (Error Recovery)

- **Reduced Max Chunks**: Agents receive 2-5 chunks max (down from 6) to prevent timeouts
- **Phase 2 Error Recovery**: Failed chunks automatically retried with /phdarp agents
- **Recovery Reporting**: Detailed reports on retry outcomes
- **Proven Architecture**: Builds on v4.1 smart batching success

### From v4.1 (Smart Batching)

- **Document-Aware Smart Batching**: Each processor gets ALL remaining chunks from ONE document
- **Dynamic Allocation**: Agents receive 2-5 chunks based on document size
- **Context Efficiency**: Agent builds document understanding ONCE, applies to all chunks
- **Proven Performance**: 59 chunks/hour achieved in BATCH_36 processing

### Why Smart Batching is Better

| Approach | Context Switches | Efficiency | Quality |
|----------|------------------|------------|---------|
| Random 3 chunks | High (3 docs) | Low | Variable |
| Smart Batching | None | High | Consistent |

Key Insight: An agent processing chunks 01-04 of Document_A builds deep context. Processing chunks from 3 different documents wastes that context.

## What This Does

Spawns N+1 agents in parallel (single message, multiple Task calls):
- 1 Quality Validator Agent: Validates completed chunks + catalog + cleanup
- N Processing Agents: Each processes ALL chunks from ONE assigned document

Key Difference from v4.0: Chunks are grouped BY DOCUMENT, not distributed randomly.

---

## Scope Specification (NEW — fixes silent cross-wave processing)

By default `/sphdarp` processes any PREPARED batch in numeric order, regardless of which wave or campaign segment it belongs to. This is dangerous when a campaign has multiple waves and the user only wants one of them processed.

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

- "do wave 2", "process wave 2", "run sphdarp on wave 2"
- "until wave 2 is complete", "finish wave 2"
- "wave_02 batches", "the wave 2 stuff"
- Any phrase that names a single wave in the context of a DARP command

When the user mentions a wave in their message and then invokes (or has already invoked) a DARP command, treat that wave as the scope. **Never silently process batches outside the user-stated scope.** If the user's scope intent is ambiguous, ask before processing — do not guess.

If the user explicitly says "all waves" or "everything" or invokes the command with no wave mention and no prior wave context in the conversation, then no scope is set (default behavior).

### Scope and the Validator

The validator is **NOT** scope-filtered. It always picks the oldest batch from `validation_queue` regardless of wave. This is intentional: validation is bookkeeping that should clear out the queue independent of which wave is currently processing.

### Section Sharding (`--section`, v6.2 — multi-session parallel drains)

`--section <X>` restricts batch selection to `drain_section == "X"`, exactly like `--wave` restricts by wave (same invariant, same Phase-1 + Phase-4 application). It exists so a large campaign can be drained by **multiple independent sessions at once**, each owning a disjoint slice with **zero collision**.

- **Setup:** partition the remaining PREPARED batches by **whole document** (greedy-balanced) and tag each batch with `drain_section` in `BATCH_STATE.json`. Whole-document partitioning is mandatory so chunk-range consolidation has a single owner per book. Full procedure + reference scripts: `HDARP_SECTION_SHARDING.md`.
- **Isolation (recommended for concurrent sessions):** give each session its own state slice (`BATCH_STATE.section_X.json`) + per-section selector/committer so there are **no shared-file writes**; reconcile to the master with a merge script. See the sharding doc and `HDARP_RESUMABLE_DRAIN.md`.
- **Invariant:** within the section, pick the numerically-lowest PREPARED batch; outside the section, NEVER touch a batch. The validator remains unscoped (clears its own slice's queue).
- **Natural language:** "do section A", "drain the A third", "section_A batches" imply `--section A`, exactly as wave phrases imply `--wave`.

---

## Document Type Detection (informational classification)

Not all documents are created equal. A 7-page repo brief and a 936-page scanned banking history both look like "one batch" in the queue, but they have different cost profiles. The classifier below runs at batch-load time for **informational logging only** — all categories are processed by agents via full HDARP.

### Classification Function

Read the prep manifest at `Technical/HDARP_Processing/{doc_id}/manifest.json` and inspect the chunks dir to compute the document category:

```python
def classify_document(doc_id, manifest, chunks_dir):
    """Return one of: SMALL_TEXT, LARGE_TEXT, DENSE_TECHNICAL, SCANNED_BOOK, MEGA_DOC."""
    n_chunks = len(list(chunks_dir.glob('*.pdf')))
    pages = manifest.get('total_pages', 0)
    size_mb = manifest.get('total_size_mb', 0)
    density = manifest.get('density_category', 'MEDIUM')  # LOW/MEDIUM/HIGH from prep
    pages_per_chunk = pages / max(n_chunks, 1)

    # MEGA_DOC: any document with >100 chunks — INFORMATIONAL ONLY, not a routing decision.
    # All documents are processed by agents regardless of size (Scrounger Principle).
    # The HDARP_FAILURE_TAXONOMY.md entry routing MEGA_DOC to Sraffa 4.0 is OUTDATED
    # and superseded by this command file. Chunk-range parallelism handles large documents.
    if n_chunks > 100:
        return 'MEGA_DOC'

    # SCANNED_BOOK: >50 chunks AND ~1 page per chunk AND HIGH density (image-heavy)
    # These are scanned books that need OCR, not visual extraction
    if n_chunks > 50 and pages_per_chunk < 1.5 and density == 'HIGH':
        return 'SCANNED_BOOK'

    # DENSE_TECHNICAL: small chunk count, high density (formulas/tables/figures)
    # — perfect for agent extraction, do NOT route away
    if density == 'HIGH' and n_chunks <= 20:
        return 'DENSE_TECHNICAL'

    # LARGE_TEXT: 11-50 chunks, moderate density (academic papers, long reports)
    if 11 <= n_chunks <= 50:
        return 'LARGE_TEXT'

    # SMALL_TEXT: 1-10 chunks (typical journal article)
    return 'SMALL_TEXT'
```

### Category Reference Table (informational — no routing action)

| Category | Processing Approach | Notes |
|----------|---------------------|-------|
| SMALL_TEXT | Standard agent processing | 1-2 rounds per batch |
| LARGE_TEXT | Chunk-range parallelism (see Smart Batch Assignment) | Multiple agents on different chunk ranges of same doc |
| DENSE_TECHNICAL | Standard agent processing | Visual extraction is the point — tables, equations, figures matter |
| SCANNED_BOOK | Agent processing with chunk-range parallelism | Large scanned books may need many rounds; use chunk-range parallelism |
| MEGA_DOC | Agent processing with chunk-range parallelism | >100 chunks — use chunk-range parallelism to maximize throughput per round |

**All categories are processed by agents via full HDARP.** Classification is used for cost estimation and logging only — never for routing documents away from agent processing. **OCR is never a substitute for agent processing**, and there are two distinct OCR roles: the scrounger ladder's **L3 per-page rescue** remains a last resort for pages that exhausted the CF ladder (see Content Filter Retry Protocol in Phase 2), while **Hybrid Stage 5 (D2)** runs for *every* document at end of run (Phase 5) regardless of category. Being unconditional, D2 is not routing at all — nothing is chosen and nothing is diverted.

The classifier informs pre-flight cost estimation and helps agents anticipate session length for large documents.

---

## Smart Batch Assignment Algorithm

Goal: Assign each processor ALL chunks from ONE document (up to 5 chunks max)

Assignment Rules:
1. One Document Per Agent: Each processor works on exactly ONE document
2. Largest First: Documents with most chunks assigned first
3. Max 5 Chunks: Cap at 5 chunks per agent to prevent timeouts (reduced from 6 in v4.2)
4. Min 2 Chunks: If document has less than 2 chunks, use /phdarp instead

Example Assignment (5 processors):
- Processor 1: Document_A chunks 01-05 (5 chunks)
- Processor 2: Document_B chunks 03-05 (3 chunks)
- Processor 3: Document_C chunks 01-03 (3 chunks)
- Processor 4: Document_D chunks 01-02 (2 chunks)
- Processor 5: Document_E chunks 01-03 (3 chunks)
- Validator: Quality check + catalog update

### Chunk-Range Parallelism (NEW — when docs < processors)

The strict "One Document Per Agent" rule starves smart batching when a batch has fewer remaining unprocessed docs than N processors. Empirically (a 59-chunk single-document batch), this caused round 1 to spawn 1 active processor and 4 idle slots — wasting 80% of throughput.

The fix: when docs < processors AND any remaining doc has >5 unprocessed chunks, fall through to chunk-range parallelism.

**v6.2 mega-doc default — scale UP, don't idle.** When the lowest unfinished document has many unprocessed chunks (a book of 80–1,000+ chunks is the common case in book campaigns) and `N < 8`, prefer spawning up to **8–10 chunk-range processors** on that single document per round (each a non-overlapping ≤5-chunk slice) rather than running 5 and leaving the document to drag across twice as many rounds. Empirically (a 51-round mega-doc drain at N=5), holding to 5 processors roughly doubled the round count versus what 10 would have achieved. The 1 Opus validator is unchanged. Respect the user's explicit `N` if they set one; this default applies when `N` is unspecified or the doc dwarfs the pool.

```python
def assign_chunk_range_processors(batch, n_processors):
    """Spawn N processors on the same large doc, each with a non-overlapping chunk range."""
    unprocessed_docs = [d for d in batch.docs if d.has_unprocessed_chunks()]

    if len(unprocessed_docs) >= n_processors:
        # Standard smart batching: one doc per processor
        return standard_smart_batch_assignment(unprocessed_docs, n_processors)

    # Fewer docs than processors — find the largest doc and split its chunks
    largest = max(unprocessed_docs, key=lambda d: len(d.unprocessed_chunks))

    if len(largest.unprocessed_chunks) <= 5:
        # Not enough chunks to split — fall back to standard with idle slots
        return standard_smart_batch_assignment(unprocessed_docs, n_processors)

    # Split the largest doc's unprocessed chunks into contiguous slices of up to 5
    slices = [largest.unprocessed_chunks[i:i+5]
              for i in range(0, len(largest.unprocessed_chunks), 5)][:n_processors]

    assignments = []
    for slice_chunks in slices:
        first, last = slice_chunks[0], slice_chunks[-1]
        assignments.append({
            'doc': largest,
            'chunks': slice_chunks,
            'output_suffix': f'_chunks_{first:03d}_{last:03d}',
            'instructions': 'Write ONLY to chunk-range-suffixed files (no race with other processors)'
        })
    return assignments
```

### Output File Naming for Chunk-Range Processors

Each chunk-range processor MUST write to chunk-range-suffixed files to avoid race conditions:
- Body text → `FULL_TEXT_chunks_NNN_NNN.md` (NOT `FULL_TEXT.md`)
- Tables → `CSV_Tables/table_NNN_NNN_NN.csv`
- Equations → `equations/equations_chunks_NNN_NNN.tex`
- Figures → `figures/figures_chunks_NNN_NNN.md`
- **Native RDB metadata (v6.3) → `RDB_METADATA_chunks_NNN_NNN.jsonl`** (NOT `RDB_METADATA.jsonl`)

Where `NNN_NNN` is the first and last chunk number of the processor's slice (e.g., `_chunks_006_010`).

### Native RDB Metadata Capture (v6.3 — applies to ALL processors, whole-doc and chunk-range)

As each table CSV is written, append ONE JSON line for it to the doc's native-enrichment sidecar
(`RDB_METADATA.jsonl` for a whole-doc processor; `RDB_METADATA_chunks_NNN_NNN.jsonl` for a chunk-range
processor), capturing what you can honestly assess **from the chunk you are already reading**: title
(`title_raw` verbatim / `title_translit` / `title_en`), `page` + `page_basis`, `units`, `footnotes`,
`period_coverage`, `geography`, a `field_basis{}` for every field present (`from_source` only if visible
verbatim; translations/transliterations are `agent_inferred`; otherwise `null` + `not_captured`),
`obs_status` (source datum) and `transcription_status` (your read fidelity — emit `H`/`L`/`R`/`X`,
assessed per table, never `V`). Keys `source_relpath` (CSV path relative to project root) + `doc_id` are
REQUIRED. **Full spec: `NATIVE_ENRICHMENT_CONTRACT.md`; doctrine + honesty
rules: `hdarp-processing.md` "Native RDB Enrichment Capture".** This annotates the existing
Tables type — it does NOT change the 4-type rule. Write incrementally; one line per table; never fabricate.

### Consolidation After Chunk-Range Round

After all chunk-range processors complete (and before marking the batch as COMPLETE), the main agent runs a consolidation step:

1. **FULL_TEXT.md**: Concatenate all `FULL_TEXT_chunks_*.md` files in chunk order, with HTML chunk-boundary markers (`<!-- chunks 006-010 -->`) preserved between sections
2. **equations/equations.tex**: Concatenate all `equations/equations_chunks_*.tex` files, with `% --- chunks NNN-NNN ---` LaTeX comment markers
3. **figures/figures.md**: Concatenate all `figures/figures_chunks_*.md` files with `## chunks NNN-NNN` section headers
4. **CSV_Tables/*.csv**: Already unique by suffix; no consolidation needed (they remain as separate files)
5. **RDB_METADATA*.jsonl (v6.3)**: dedup the `RDB_METADATA_chunks_*.jsonl` shards by `source_relpath` (last writer wins on a re-extracted range; identical lines collapse). Either concatenate the deduped lines into a single `RDB_METADATA.jsonl` or leave the shards in place — `robert-db-harvest` reads both. Do NOT fabricate lines for tables a shard did not cover.

A reference consolidation script exists at `consolidate_marx_capital.py` (from the BATCH_363 run). This should be generalized into `consolidate_chunk_ranges.py` as a reusable helper. Until that helper exists, the main agent runs an inline consolidation script at the end of each chunk-range round.

### Invariant

The rule "one agent works on one doc" still holds — each agent has exactly one doc assigned. The new rule is "multiple agents may work on the same doc as long as their chunk ranges don't overlap and they write to range-specific files." This preserves the smart batching context-efficiency benefit (each agent builds doc context on its assigned slice) while scaling gracefully to single-doc large batches.

---

## Critical Requirements

- ONE DOCUMENT PER AGENT: Each processor works on exactly ONE document
- ALL CHUNKS SEQUENTIALLY: Process in numerical order within document
- ALL 4 TYPES PER CHUNK: Tables, equations, figures, body text
- QUALITY TARGET: 22/27 per chunk (v5.0 — 27-point scoring only)
- NO BACKGROUND: Synchronous execution only

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

## BATCH_STATE Integration (v4.3)

### Batch Sizing (v5.0)

Batches are pre-created by `/preparehdarp` using the **10-chunk / finish-the-document** rule (~10 chunks per batch, always finish the document, large docs get their own batch). See BATCH_STATE_PROTOCOL.md for full rules.

### Reading Batch State (scope-aware)

On command start, read {Project}/Technical/HDARP_Processing/BATCH_STATE.json. Pick the next batch using the scope filter (see Scope Specification section):

```python
# scope_wave is set if user passed --wave or implied a wave in natural language
# scope_wave is None if no scope was specified

def in_scope(batch_id):
    if scope_wave is None:
        return True
    return state["batches"][batch_id].get("wave") == scope_wave

# Pick the numerically-lowest PREPARED batch within scope.
# IMPORTANT: when scope_wave is set, IGNORE stale current_batch / next_to_process
# fields and always re-derive from the batches map.
prepared_in_scope = sorted(
    [bid for bid, b in state["batches"].items()
     if b.get("status") == "PREPARED" and in_scope(bid)],
    key=lambda x: int(x.split("_")[1])
)
batch_to_process = prepared_in_scope[0] if prepared_in_scope else None

# Validator queue is NOT scope-filtered (it processes whatever is queued)
batch_to_validate = state["validation_queue"][0] if state["validation_queue"] else None
```

If `batch_to_process` is None, no PREPARED batch matches the scope — print a clear message ("No PREPARED batches in scope {WAVE_ID}") and exit without spawning agents.

### Pre-Flight Cost Estimation (NEW)

Before spawning processors for the chosen batch, estimate how many sphdarp rounds the batch will need at the chosen parallelism level. This surfaces expensive batches before they consume the session.

```python
def estimate_rounds(batch_id, n_processors=5):
    """Estimate how many sphdarp rounds this batch will need."""
    docs = get_docs_in_batch(batch_id)

    # Sum total unprocessed chunks across all docs in the batch
    total_chunks = sum(count_unprocessed_chunks(d) for d in docs)

    # Smart batching: each round can do up to n_processors * 5 chunks if we
    # have enough docs OR can use chunk-range parallelism on a single doc.
    chunks_per_round = n_processors * 5

    # Round up
    return -(-total_chunks // chunks_per_round)
```

**Decision rules**:

1. Run `classify_document()` for each doc in the batch (informational logging only)
2. Compute `estimate_rounds(batch_id)` for all docs in the batch
3. If estimate > 3 rounds: print a pre-flight warning so the user and assistant both know what they're committing to
4. Proceed with processing — the estimate is informational, not a stop condition
5. **No document is ever deferred or routed away from agent processing based on classification**

**Pre-flight warning template**:

```
─── PRE-FLIGHT ───
Batch: BATCH_XXX
Docs: N (categories: 3 SMALL_TEXT, 1 LARGE_TEXT, 1 SCANNED_BOOK [informational])
All docs: agent processing (full HDARP)
Total: 5 docs, 48 chunks
Cost estimate: ~4 rounds
Proceeding...
───────────────────
```

If the estimate is >= 5 rounds, print the warning in bold and note the expected session length. The default is still to proceed — large documents are processed by agents like all others.

### Previous-Batch Validation (CRITICAL)

**The validator validates the PREVIOUS completed batch, not the current batch.**

When running /sphdarp 5 on BATCH_004:
- Processors (5): Work on BATCH_004 chunks
- Validator (1): Validate BATCH_003 (from validation_queue)

**Benefits**:
- Eliminates race conditions between processor and validator
- Enables true parallel execution
- Validator works on already-complete data

### Updating Batch State

After processing completes, update BATCH_STATE.json:

```python
if all_chunks_complete:
    state["batches"][batch_id]["status"] = "COMPLETE"
    state["validation_queue"].append(batch_id)
else:
    state["batches"][batch_id]["status"] = "IN_PROGRESS"
    state["batches"][batch_id]["docs_complete"] = completed_count
```

### Status Vocabulary (extended)

| Status | Meaning | Auto-continuation behavior |
|--------|---------|----------------------------|
| PENDING | Not yet prepared by /preparehdarp | Skip (not in DARP processing scope) |
| PREPARED | Chunks ready, awaiting processor | **Pickable** by Phase 1/4 |
| IN_PROGRESS | Some chunks done, more remain | Pickable (resumes from where it stopped) |
| COMPLETE | All chunks done, awaiting validator | Skip (not pickable for processing) |
| VERIFIED | Validator passed | Skip |
| BLOCKED | Unrecoverable error (e.g. corrupt source PDF) | **Skip + STOP** (Phase 4 stops if next batch is BLOCKED) |
| DEFERRED_TO_OCR (LEGACY) | Was set by adaptive routing (now removed). Existing instances should be reset to PREPARED via migration script. New runs will never auto-set this status. | Skip + ADVANCE |
| PARTIAL_DEFERRED (LEGACY) | Was set by adaptive routing for partial deferrals. Same migration treatment as DEFERRED_TO_OCR. | Pickable (treat as IN_PROGRESS) |

**Note on legacy statuses**: The DEFERRED_TO_OCR and PARTIAL_DEFERRED statuses were introduced by an adaptive routing system that has been removed. All documents are now processed by agents regardless of size or density. Existing batches in these states should be reset to PREPARED for normal agent processing.

### Catalog Updates

After processing, update HDARP_MASTER_CATALOG.csv for each processed document:
- Set hdarp_status to COMPLETE or IN_PROGRESS
- Update processing_date, quality_score, chunk_count

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

## Comparison: v4.0 through v4.5

| Feature | v4.0 | v4.1 | v4.2 | v4.4 | v4.5 (Current) |
|---------|------|------|------|------|----------------|
| Chunk Assignment | Random 3 | Smart 1-doc | Smart 1-doc | Smart 1-doc | Smart 1-doc |
| Chunks per Agent | Fixed: 3 | Dynamic: 2-6 | Dynamic: 2-5 | Dynamic: 2-5 | Dynamic: 2-5 |
| Context Efficiency | Low | High | High | High | High |
| Error Recovery | None | None | /phdarp retry | /phdarp retry | /phdarp retry |
| Model | Inherited | Inherited | Inherited | **Sonnet** | **Sonnet** |
| Catalog Sync | Optional | Optional | Optional | Optional | **MANDATORY** |

---

## Performance Benchmarks

From BATCH_36 processing (2025-12-28):

| Metric | Value |
|--------|-------|
| Chunks Processed | 147 |
| Processing Time | ~2.5 hours |
| Speed | 59 chunks/hour |
| Quality Score | 9.2/10 |
| Completion Rate | 100 percent |



## Document Processor Agent Template (v4.1)

When spawning processor agents, use this structure:

**Model**: sonnet (Sonnet Mandatory — Haiku BANNED)
**Agent Role**: Process ALL assigned chunks from ONE document using HDARP protocol

**Agent Assignment**:
- YOUR DOCUMENT: {document_name}
- YOUR CHUNKS: {chunk_list} ({chunk_count} chunks total)
- OUTPUT DIRECTORY: Knowledge_Base/{document_name}/

**Smart Batching Advantage**:
- Build document context ONCE from first chunk
- Apply consistent understanding across all chunks
- Maintain formatting/terminology consistency

**Sequential Processing**:
1. Read Chunk N -> Extract all 4 types -> Quality Check
2. Repeat for each assigned chunk in order
3. Create Document Completion Report

**HDARP Protocol (4 Content Types per Chunk)**:
- A. Tables -> CSV (98%+ accuracy)
- B. Equations -> LaTeX (100% target)
- C. Figures -> Markdown (200+ words per figure)
- D. Body Text -> **two layers, both required** (canonical rule: `hdarp-processing.md`, "Hybrid body text = two layers"):
  - **D1 — agent-read body text (the HDARP layer, YOUR job as a processor).** Read the chunk PDF
    yourself and write `FULL_TEXT_chunks_NNN_NNN.md` with chunk-boundary markers, page references and
    a quality assessment. Nothing may stand in for D1; only D1 is validated as HDARP.
  - **D2 — Hybrid Stage 5, the Sraffa 4.0 verbatim OCR sibling.** A **mandatory stage over EVERY
    document** — not a content-filter fallback, not conditional on scan quality, not "if needed".
    It is **not a processor's job and not a per-chunk job**: it is executed once per run by the
    SPHDARP orchestrator as **Phase 5** — see "Phase 5: Hybrid Stage 5" below for the single owner,
    what "end of run" means, the on-disk completion test, and the closed three-reason exception
    list. It lands in `Knowledge_Base/_OCR_Only/<short_id>/`, **augments** D1, and never substitutes
    for it.
  - Page-level routing inside D2 (digital vs scanned, QA, escalation) is Sraffa's business, not a
    processor's — see the Sraffa 4.0 Routing section of the rule above. Processor agents produce D1.

---

## Quality Validator Agent Template (v4.3)

**Model**: opus (Opus Mandatory for validator)
**CRITICAL**: Validator validates the PREVIOUS completed batch, not the current batch.

**Validator Assignment**:
- YOUR BATCH: {previous_batch_id} (from validation_queue)
- NOT the current batch being processed

**Validator Tasks**:
1. Read BATCH_STATE.json to get validation_queue
2. Validate oldest batch in queue (BATCH_{N-1})
3. Verify Document Completeness (all chunks processed)
4. Quality Assessment (score each document)
5. Update HDARP_MASTER_CATALOG.csv (status -> VERIFIED)
6. Update BATCH_STATE.json (batch status -> VERIFIED)

**ANTI-AUTO-VALIDATION RULE (CRITICAL — added 2026-04-28)**:
NEVER auto-validate or rubber-stamp a batch. Every validation MUST include:
1. Reading BATCH_STATE for the batch's total_chunks count
2. Verifying KB output files (body text files) match chunks_processed count
3. Spot-checking at least 2 body text files for substantive content (>200 chars)
4. Writing substantive validation_notes (minimum 50 characters describing what was checked)
5. If chunks_processed < total_chunks: set status = IN_PROGRESS, NOT VERIFIED
6. If validation_notes would be empty: the validation is incomplete — do not mark VERIFIED
**Background**: The 2026-04-28 abdication audit found 20 batches auto-validated with "Phase 7 closeout" stamps, perfect 27/27 scores, and only 1-4 of 5-19 chunks actually processed. This is unacceptable. An additional 693 batches had empty validation_notes from bulk legacy conversion.

**4-TYPE COMPLETENESS CHECK (CRITICAL — added 2026-05-06)**:
Before scoring, verify the KB directory structure for EACH document in the batch:
7. Does `CSV_Tables/` exist with CSV files or `_no_tables.txt` marker?
8. Does `equations/` exist with `.tex` files or `_no_equations.txt` marker?
9. Does `figures/` exist with `.md` files or `_no_figures.txt` marker?
10. Do body text files have HDARP chunk markers (`<!-- chunk_NNN -->`) or are they raw text dumps?
**If ANY content type has no directory AND no marker file → FAIL validation.** Score 0/27, revert to PREPARED, do NOT delete chunk PDFs. A KB directory with ONLY `Text/` and `FULL_TEXT.md` (no CSV_Tables/, equations/, figures/) is a PyMuPDF dump, NOT valid HDARP output.

**Scope note — the `_OCR_Only` sibling is deliberately NOT checked here.** This check scores the
**agent-read layer (D1)** only. Do not look for **Hybrid Stage 5 (D2)** and do not deduct for its
absence: Stage 5 is a per-**run** stage (Phase 5), not a per-chunk or per-batch one, so requiring it
here would fail every batch of every drain. It is enforced by Phase 5's completion test and by
`/hdarp-wrapup`'s Stage 5 gate. Its absence is never grounds to accept a text dump in the document's
own KB directory.

11. **Post-Validation Cleanup (MANDATORY)**: After marking batch as VERIFIED:
   - Delete chunk PDFs: `Technical/HDARP_Processing/{doc_id}/chunks/chunk_*.pdf`
   - Verify Knowledge_Base content is intact before deleting
   - Update HDARP_MASTER_CATALOG.csv status to ARCHIVED
   - Log freed space: "Cleaned batch {batch_id}: freed {X} MB"
   - This prevents the 100+ GB artifact accumulation that degrades system performance

**Quality Scoring** (v5.0):
- 27 points (minimum 22/27)

---

## Phase 1 Error Handling

- No remaining work: All chunks processed - batch complete
- Fewer docs than N: Reduce N to match available documents
- N > 10: Cap at 10 processors maximum
- Document with >5 chunks: Assign first 5, remaining for next round
- Document with 1 chunk: Use /phdarp instead
- Agent spawn fails with "Extra usage required": Billing/entitlement error, NOT a processing failure. Auto-retry after 30 seconds. See "Billing/Entitlement Errors" in Phase 2.

---

## Phase 2: Error Recovery (NEW in v4.2)

After all Phase 1 agents complete, check for failures and automatically retry:

### Error Detection

After Phase 1 completes, collect failure information:
- Chunks that failed to process (timeout, crash)
- Chunks with quality scores below threshold (< 22/27)
- Chunks with missing output files

### Retry Strategy

1. **Wait for ALL Phase 1 agents** to complete (never retry during Phase 1)
2. **Collect failed chunk list** from agent results
3. **Spawn /phdarp retry agents** (max 5 agents at a time)
4. **One chunk per retry agent** (simpler, more reliable)
5. **Wait for ALL retries** to complete
6. **Generate recovery report**

### Retry Agent Spawning

If failed chunks > 0:

```
Failed Chunks: 1-5   -> Spawn N /phdarp agents (one per chunk)
Failed Chunks: 6-10  -> Spawn 5 /phdarp agents, wait, then spawn remaining
Failed Chunks: >10   -> Flag for manual intervention
```

### Failure History Recording (MANDATORY — added 2026-04-28)

After EVERY error recovery round, append to the batch's `failure_history` array:
```json
{
  "round": 1,
  "timestamp": "ISO_DATE",
  "chunks_failed": ["chunk_03", "chunk_07"],
  "error_type": "timeout|content_filter|quality|crash|billing",
  "resolution": "retried|cf_retry_strategy_1|cf_retry_strategy_2|ocr_fallback|manual"
}
```
**Background**: The 2026-04-28 audit found failure_history was `[]` on EVERY batch in the campaign despite many failures occurring. The field was initialized but never populated. All failure evidence was scattered across validation_notes, notes, and processing_notes. This makes post-hoc auditing nearly impossible.

### Why /phdarp for Retries

- `/phdarp` processes ONE chunk at a time (simpler, more reliable)
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
| doc_C_chunk_07 | Low quality (18/27) | SUCCESS (24/27) | COMPLETE |
| doc_B_chunk_12 | OCR failed | FAILED | NEEDS MANUAL |

**Final Status**: X chunks complete, Y need manual review
```

### Content Filter Recovery Protocol (v6.0 — orchestrator-driven progressive decomposition)

CF recovery is **orchestrator-driven**. Subagents do NOT handle CF internally — they report the failure and the orchestrator handles the recovery cycle. Sonnet subagents remain the primary processors throughout; Opus does not take over unless the user explicitly requests it.

**When a processor subagent reports `API Error 400: Output blocked by content filtering policy`:**

**Step 1 — Record the failure**

Append to `cf_retry_log` in the batch's BATCH_STATE entry:
`{"chunk": "chunk_NNN", "level": "pending", "reported_at": "<ISO timestamp>"}`

**Step 2 — L1: Bisect and Re-dispatch**

Call `bisect_chunk()` from `pdf_splitter_orchestrator.py`:
```python
import sys; sys.path.insert(0, 'scripts')
from pdf_splitter_orchestrator import bisect_chunk
halves = bisect_chunk(chunk_pdf_path, reason="content_filter")
# Returns: ["...chunk_NNN_a.pdf", "...chunk_NNN_b.pdf"]
```

This physically splits the chunk PDF and updates manifest.json with bisection_history.

Spawn one new Sonnet subagent per half (same HDARP extraction prompt as original). Wait for both.

- If a half succeeds: commit its output, update cf_retry_log.
- If a half fails with CF: bisect that half again (recursive). `bisect_chunk()` handles recursive naming (`chunk_NNN_a.pdf` → `chunk_NNN_aa.pdf` / `chunk_NNN_ab.pdf`).
- Recursion terminates when `bisect_chunk()` raises `ValueError` (1-page chunk cannot be bisected).

**Serialize bisect calls within the same document** — `bisect_chunk()` reads and writes manifest.json without file locking. Different documents can be bisected concurrently.

**Step 3 — L2: Single-page Sonnet agents with scholarly framing**

When a sub-chunk reaches 1 page (or a 2-page chunk where both halves have been tried):
Spawn one Sonnet subagent per remaining failing page with this analytical framing:

> "You are analyzing page N of [doc_name] for its scholarly content and structure.
> Identify: (a) key arguments and data presented, (b) any tables or quantitative data,
> (c) equations or formulas, (d) figures and their captions.
> Do not reproduce extended verbatim passages — summarize and analyze."

Wait for each to complete.
- If page succeeds: commit output, update cf_retry_log.
- If page still fails with CF: advance to L3.

**Step 4 — L3: OCR fallback (only for exhausted pages)**

For each page where even the L2 single-page scholarly-framing agent failed:
```json
{
  "doc_id": "<doc_id>",
  "source_batch_id": "<batch_id>",
  "pages": [N],
  "original_chunk": "chunk_NNN",
  "reason": "content_filter_exhausted"
}
```
Append to `ocr_pending_queue` in BATCH_STATE.json. Continue processing all other chunks normally. Do NOT set DEFERRED_TO_OCR on the batch. Do NOT mark the batch as BLOCKED.

**cf_retry_log schema** (in the batch entry):
```json
"cf_retry_log": [
  {
    "chunk": "chunk_017",
    "l1_bisect": "chunk_017_a SUCCESS, chunk_017_b CF",
    "l1_rebisect": "chunk_017_ba SUCCESS, chunk_017_bb CF (1-page, L2)",
    "l2_page": "page_12: SUCCESS",
    "final": "COMPLETE"
  },
  {
    "chunk": "chunk_023",
    "l1_bisect": "chunk_023_a CF, chunk_023_b CF",
    "l2_page": "page_8 CF, page_10 CF",
    "final": "OCR_FALLBACK (pages 8, 10)"
  }
]
```

**Proactive scholarly framing for cf_risk_level=HIGH**

If a document's manifest has `cf_risk_level: "HIGH"` (set by /preparehdarp):
- Apply scholarly/analytical framing to ALL processor agents for that doc on FIRST dispatch (do not wait for a CF hit)
- Add to the processor prompt: "Use analytical rather than verbatim extraction framing throughout. Summarize arguments rather than reproducing extended passages verbatim."
- Record `"proactive_framing": true` in the batch entry

**What NOT to do**:
- Do NOT handle CF within the subagent — subagents only report and continue with remaining chunks
- Do NOT defer chunks to OCR on the first CF hit — always bisect first
- Do NOT skip the bisect step and jump straight to page-level
- Do NOT mark the batch DEFERRED_TO_OCR or BLOCKED
- Do NOT silently skip CF'd chunks without recording them in cf_retry_log
- Do NOT have Opus take over processing — Sonnet subagents handle all recovery levels

### Billing/Entitlement Errors (API Tier Limits)

If a processor agent fails with `API Error: Extra usage is required for 1M context`, this is a **transient billing check failure**, NOT a content or processing error (GitHub #44117). The subagent doesn't actually need 1M context — it starts with a clean slate and only uses Sonnet's default 200k. The error occurs because the parent session runs on Opus[1m] and the entitlement check incorrectly propagates.

**Recovery action (auto-retry)**:
1. Wait 30 seconds (billing check flakiness is often timing-related)
2. Retry the exact same agent spawn — same prompt, same model, same chunks
3. If retry also fails: ask the user to run `/extra-usage` to enable billing, then retry
4. If still failing after user intervention: mark affected chunks for next round (NOT DEFERRED_TO_OCR — agent retry WILL eventually work once billing is resolved)

**What NOT to do**:
- Do NOT treat this as a content filter error (the content was never sent)
- Do NOT mark chunks as DEFERRED_TO_OCR (this is a billing issue, not a content issue)
- Do NOT stop the entire run — other agents in the same round may succeed
- Do NOT switch to a different model — the error is about the parent's billing tier

**Diagnostic**: Print: `BILLING ERROR on agent {N}: Known entitlement check bug (GitHub #44117). Retrying in 30s. If persistent, user should run /extra-usage.`

### Unrecoverable Errors

These require manual intervention:
- Document not found -> Skip, report
- Retry also failed (and not a content filter error or billing error) -> Flag for manual review
- >10 chunks failed in Phase 1 -> Flag for manual intervention

---

## Phase 3: Mandatory Catalog Synchronization (NEW in v4.5)

After processing and error recovery complete, MUST synchronize with catalog:

### Catalog Update Requirements

For EACH processed document, update HDARP_MASTER_CATALOG.csv:

```csv
document_id,hdarp_status,quality_score,completion_date,chunk_count,tables_extracted,equations_extracted
DOC_001,COMPLETE,24.5,2026-02-09,5,12,3
```

### Required Fields

| Field | Value | Source |
|-------|-------|--------|
| hdarp_status | COMPLETE/IN_PROGRESS | Processing result |
| quality_score | XX.X/27 | Quality assessment |
| completion_date | ISO date | Processing timestamp |
| chunk_count | Integer | Chunks processed |
| tables_extracted | Integer | Table count |
| equations_extracted | Integer | Equation count |
| figures_extracted | Integer | Figure count |

### Sync Verification

After catalog update, verify:
1. All processed documents have catalog entries
2. Status values match BATCH_STATE.json
3. Quality scores are within valid range (0-27)
4. Dates are properly formatted

### Catalog Sync Report Template

```markdown
## Catalog Synchronization Report

**Documents Synced**: X of Y
**New Entries**: N
**Updated Entries**: M

| Document | Status | Quality | Sync Result |
|----------|--------|---------|-------------|
| 2020_IMF_WP_COVID | COMPLETE | 24.5/27 | SUCCESS |
| 2019_Fed_DFAST | COMPLETE | 23.0/27 | SUCCESS |

**Catalog Integrity**: VERIFIED
```

---

## Phase 3.25: Extraction Integrity Check (CRITICAL — added 2026-05-06)

**Background:** The 2026-05-06 incident proved that an agent can silently replace SPHDARP with PyMuPDF and pass all existing validation checks. This phase prevents that.

Before marking ANY batch COMPLETE, verify extraction integrity:

### Mandatory Checks

For EACH document processed in this round:

1. **4-Type Evidence Check**: The KB directory MUST contain evidence that all 4 content types were attempted:
   - `CSV_Tables/` directory exists (with CSV files or `_no_tables.txt` marker)
   - `equations/` directory exists (with `.tex` files or `_no_equations.txt` marker)
   - `figures/` directory exists (with `.md` files or `_no_figures.txt` marker)
   - Body text files with HDARP chunk markers (`<!-- chunk_NNN -->`)

2. **Agent Provenance Check**: Each body text file MUST have been written by an SPHDARP agent, not by a bulk script. Check for:
   - Chunk-range suffixed files: `FULL_TEXT_chunks_NNN_NNN.md`
   - Processing summaries or chunk markers written by the agent
   - If body text files are named `chunk_NNN_body.txt` with raw `page.get_text()` output and NO structured markers → **FAIL: this is a PyMuPDF dump, not HDARP**

3. **Method Declaration**: After each round, the orchestrator MUST declare:
   ```
   Round N extraction method: SPHDARP agent (Sonnet) — {X} chunks by {Y} agents
   ```
   If the method was anything other than SPHDARP agent extraction, the round is INVALID.

4. **Digest-mode anti-fabrication check (v6.2)**: when a document is processed in analytical-digest mode (in-copyright/trade books), the validator spot-checks that body text is genuine paraphrase/digest with a neutral encyclopedic tone — NOT fabricated verbatim quotes, letters, or anecdotes. Invented attributed text → FAIL (ties to the NEVER FABRICATE rule above).

5. **ROLLING_OVERLAP legitimacy (v6.2)**: a chunk whose pages are wholly contained in an adjacent already-extracted chunk may carry an explicit overlap marker naming the canonical chunk (a `/preparehdarp` splitter artifact). This is LEGITIMATE per `sphdarp-scrounger.md` §1a — count it as extracted (≤1-pt deduction), do NOT treat it as silent degradation.

6. **Native RDB metadata check (v6.3 — WARN-only)**: confirm a `RDB_METADATA.jsonl` (or `_chunks_*.jsonl` shards) exists for the doc with ≥1 line per non-marker table CSV; each line parses, its `source_relpath` resolves to a real CSV, `field_basis` covers every present field (valid vocab), and `transcription_status` ∈ {H,L,R,X} (NEVER V). **Honesty spot-check:** sample ≥2 lines — any `from_source` field must be visible verbatim in the CSV/body, else correct it. **Severity: WARN deduction (≈ −2/27) recorded in `validation_notes`, NOT a hard fail** — the 4-type checks above are the hard gate. Missing metadata is backfillable by `enrichhdarp` Type E and must NOT block COMPLETE or trigger PREPARED revert.

**Not checked here — the `_OCR_Only` verbatim sibling (Hybrid Stage 5 / D2).** Stage 5 is a per-**run**
stage (**Phase 5**), not a per-chunk or per-batch one: it runs once, after the whole scope drains, so
requiring it in this per-batch check would fail every batch of every drain. It is enforced by Phase
5's completion test and by `/hdarp-wrapup`'s Stage 5 gate. Its absence here is never grounds to accept
a text dump in the document's own KB directory.

### Failure Response

If any check fails:
- Do NOT mark the batch COMPLETE
- Do NOT add to validation_queue
- Do NOT delete chunk PDFs
- Report: "EXTRACTION INTEGRITY FAILURE: {which check failed}. Batch remains PREPARED."
- STOP processing and inform the user

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

### Stopping Conditions

#### Context Budget is NOT a Stopping Condition (CRITICAL — read this first)

**Auto-compression handles context limits transparently.** From the system prompt:

> The system will automatically compress prior messages in your conversation as it approaches context limits. This means your conversation with the user is not limited by the context window.

For HDARP processing specifically, auto-compression is essentially **zero-cost**: the persistent state lives in `BATCH_STATE.json` and `HDARP_MASTER_CATALOG.csv` on disk, NOT in conversation memory. If old agent summaries get compressed away, nothing is lost — the actual extracted content is already in the Knowledge_Base, the state updates have been committed, and the next round can re-derive everything it needs from BATCH_STATE.json.

**The assistant MUST NOT**:
- Write a handoff document as a "context management" strategy
- Stop processing because "context is filling up"
- Stop processing because "rounds 1-N have accumulated a lot of summaries"
- Estimate that "I can probably do N more rounds before context becomes tight" and stop preemptively at N
- Frame stopping as "let me write a handoff for the next session" when the only reason for stopping is anticipated context pressure

**The assistant MAY stop ONLY when**:
- Genuine model degradation is observable (responses become incoherent, tool calls fail consistently, the assistant itself reports inability to reason)
- A real Stopping Condition from the list below is met (scope exhausted, no more PREPARED, BLOCKED, --single)
- The user explicitly says "stop", "pause", "handoff", or similar

**If the assistant catches itself thinking "I should write a handoff and stop because context is getting tight" — that thought is wrong. Continue processing.** This was an empirically-observed failure mode during a 2026-04-09 marathon extraction session: three premature handoffs were written when no real stopping condition existed.

#### Real Stopping Conditions

Processing stops when ANY of these are true:
- **Scope exhausted** — when a scope is set (e.g., `--wave Wave_02`), all PREPARED batches matching the scope have been processed → **go to Phase 5 (Hybrid Stage 5)**, then print final summary stating `Scope {WAVE_ID} complete: N batches processed` and exit. **Do NOT cross over to other waves even if PREPARED batches exist outside the scope.** This is the most important stopping condition — it is the entire reason scope filtering exists.
- **No more PREPARED batches** remain in BATCH_STATE.json (no scope set) → **go to Phase 5 (Hybrid Stage 5)**, then print final summary and exit
- **Next batch is BLOCKED** → stop, report which batch is blocked and why. (Note: DEFERRED_TO_OCR is NOT BLOCKED — see Status Vocabulary. DEFERRED batches are skipped and Phase 4 advances past them without stopping.)
- **User passed `--single` flag** → skip Phase 4 entirely (process exactly one batch)
- **User explicitly requested stop** during the run

Note that "context budget" does not appear in this list. It is not a stopping condition.

**Which stops trigger Phase 5.** Only the two *drain-completed* conditions above (scope exhausted /
no more PREPARED) end the run and therefore require **Phase 5**. The other three (next batch
BLOCKED, `--single`, user stop) — and any yield trigger 1/2/4/5 — **pause** the run rather than end
it: do NOT run Phase 5, and state in the final summary that Stage 5 is still owed for the scope.

### Inter-Batch Summary Template

Print between batches (keep brief — do NOT ask for confirmation). Include the `Scope:` line ONLY when a scope is set; omit it for unscoped runs:

```
─── BATCH CONTINUATION ───
Scope: Wave_02 (28 of 165 in scope done)
Completed: BATCH_390 (10 chunks, 1 doc, 24.2/27 avg quality)
Next: BATCH_391 (8 chunks, 1 doc)
Batches remaining in scope: 137 PREPARED
Validation queue: [BATCH_388, BATCH_389, BATCH_390]
Tasks cleaned: {deleted_count} deleted, {remaining_count} active
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
Hybrid Stage 5 (Phase 5): 316 in scope = 314 siblings + 2 exception rows — COMPLETE
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
/sphdarp 5 --single    # Process ONE batch only (pre-v5.1 behavior)
```
Skip Phase 4 entirely. Command ends after Phase 3.

### CRITICAL: Do NOT Prompt Between Batches

**The continuation loop must NOT ask for user confirmation between batches.** The entire point is autonomous multi-batch processing. The only user interaction points are:
- Start: User invokes `/sphdarp N`
- End: Processing stops (no more batches, BLOCKED, or `--single`)

If you need to stop for any reason not listed in Stopping Conditions, print a clear explanation and stop — do not ask "should I continue?"

---

## Phase 5: Hybrid Stage 5 — the verbatim OCR sibling (MANDATORY, once per run)

> **This phase is the reason v6.4 exists.** While the mandate lived only in prose, an orchestrator
> that executed Phases 1 → 2 → 3 → 3.25 → 3.5 → 4 mechanically finished a whole drain having OCR'd
> nothing — which is how the stage ran on **5 of 143 documents** in one wave while 353 of 384
> documents were extracted in `analytical_digest` mode, leaving the verbatim text of ~92% of that
> corpus in existence nowhere. **A drain that skips Phase 5 is not complete, whatever Phase 4's final
> summary says.** A mandatory stage that is not a numbered step is decorative.

### What "end of run" means (previously undefined — this is the executable definition)

**The run = the drain scope of one `/sphdarp` invocation**: the `--wave` / `--section` slice if one
was set, otherwise every PREPARED batch that invocation was allowed to touch.

**End of run = the moment a drain-completed Phase 4 stopping condition fires** ("Scope exhausted" or
"No more PREPARED batches") — after the Final Summary numbers are computed, before the command exits.

It is **not** the end of a document, **not** the end of a chunk-range round, and **not** the end of a
batch. A 165-batch drain runs Phase 5 **once**, over the whole scope — not 165 times. A run that
paused instead of draining (BLOCKED, `--single`, disk guard, user stop, context yield) has not reached
end of run: Stage 5 stays owed, and is recorded as owed.

**The document set** = every document with a Knowledge_Base directory touched by any batch in the
scope — derive it from the `HDARP_MASTER_CATALOG.csv` rows for those batches, not from the last
round's agent reports.

### Owner (one owner — this sentence appears identically in `sraffa-ocr.md` and `hdarp-wrapup.md`)

**The `/sphdarp` orchestrator owns Hybrid Stage 5 and runs it here, as Phase 5, once per drain
scope**, by invoking `/sraffa-ocr --augment` after `/hdarp-wrapup` has gated the scope — in lifecycle
order: `/preparehdarp → /sphdarp → /enrichhdarp → /hdarp-wrapup → /sraffa-ocr --augment →
/hdarp-cleanup → /kb-integrate-pipeline`.

Stated as negatives, so no reader has to infer them:

- **`/sraffa-ocr` does NOT schedule itself and is NOT invoked inline mid-round** — not by a processor,
  not by the validator, not by any Phase 1–4 step. It runs when this phase calls it.
- **`/hdarp-wrapup` GATES Stage 5 and never RUNS it.** Its Stage 5 gate counts siblings and exception
  rows and FAILs on a document that has neither; it does not perform the pass.
- **Processor agents never produce D2.** They produce D1 only (see the processor template above).

### Execution

1. Finish `/enrichhdarp` then `/hdarp-wrapup` for the scope (lifecycle steps 3 and 4).
2. Enumerate the scope's documents. For each, resolve its **`short_id`** — the filesystem-safe key
   that names its `_OCR_Only/` directory. **`short_id` is NOT `doc_id`:** `doc_id` must equal the KB
   folder name verbatim (`NATIVE_ENRICHMENT_CONTRACT.md`), while
   `short_id` is a shortened stem (e.g. `01_Foley_1986` for KB folder `[1986] Foley - …`). Confusing
   the two mis-keyed 114 documents on 2026-08-27. Record **both** keys in every manifest and every
   exception row.
3. Run `/sraffa-ocr --augment --batch <the project's Knowledge_Base dir>` (Mode 4). **Never
   `--chunks`** — its >500-char skip would skip exactly the pages that *have* agent text, i.e. nearly
   the whole corpus, satisfying the mandate on paper while running nothing.
4. The pass is splittable: the PyMuPDF text-layer half needs no GPU; only pages with no usable text
   layer need the EasyOCR half. Running the text-layer half while the GPU is committed elsewhere is
   expected, and leaves a bounded named queue — not a skipped document (see `GPU_DEFERRED`).
5. Re-run the completion test below until it passes.

### Completion test — what must exist on disk for this phase to be DONE

For **every** document in the scope, exactly one of the following must hold on disk:

**(a) The sibling exists** — all three, under the project's `Knowledge_Base/_OCR_Only/<short_id>/`.
(A project's KB root is `<Project>/Knowledge_Base/` in some trees and
`<Project>/Technical/Knowledge_Base/` in others — both are real; `_OCR_Only/` sits directly under
whichever root that project uses.)
- `FULL_TEXT.md`, non-empty
- `textlayer_manifest.json` **or** `page_manifest.json`
- one manifest entry per source page, each naming the engine that produced it — no page absent, no
  page silently dropped

**(b) The document has a bounded exception row** in the run's named artifact,
`Knowledge_Base/_OCR_Only/STAGE5_EXCEPTIONS.csv`, columns
`short_id,doc_id,source_md5,reason,detail,recorded_by,recorded_date` — one row per document, `reason`
drawn from the closed list below. **A reason asserted in prose, in a wrap-up report, in a commit
message or in a chat summary does not count.** No CSV means no exceptions.

The gate is a countable identity:

```
documents_in_scope == documents_with_sibling + exception_rows_for_this_scope
```

with every exception row resolving to a real document in scope and **no document in both halves**.
Anything else means Phase 5 is NOT done. **Report all three numbers explicitly** — "Stage 5 complete"
without them is precisely the claim that passed while 138 documents had nothing.

### The closed exception list — exactly three reasons

`sphdarp-scrounger.md` bounds non-extraction to three named outcomes; Stage 5 is bounded
the same way. **This list is closed: a reason not on it is not acceptable, however carefully
recorded.**

| `reason` | The bar | Required `detail` |
|---|---|---|
| `DUPLICATE` | The identical source (same `source_md5`) already carries a complete sibling under another `short_id`. Confirmed by hash — never by "looks like the same book" (scrounger §1). | the canonical `short_id` holding the sibling |
| `QUARANTINED` | The source PDF is physically unreadable: 0 bytes, or no tool can open it (scrounger §3). Poor scan quality, Gothic type, size, and "it's born-digital anyway" are **NOT** this. | what was tried, and how each attempt failed |
| `GPU_DEFERRED` | The text-layer half completed; named pages still need the EasyOCR half and the GPU is committed. **NON-TERMINAL — a queue entry, not a disposition.** | the specific pages + the queue file (`_OCR_Only/_GPU_OCR_QUEUE.md`) |

**`GPU_DEFERRED` does not close the gate.** A scope holding any open `GPU_DEFERRED` row is **Stage 5
INCOMPLETE** and must be reported as such; it closes only when the GPU half runs and the row is
replaced by a sibling. This is what stops 138 deferrals from passing as 138 recorded reasons.

**There is no `CF_EXHAUSTED` here.** Sraffa 4.0 is a local OCR tool, not a model behind a content
filter — the scrounger's CF rung has no analogue in Stage 5.

**Explicitly NOT acceptable reasons**, each an excuse that produced or would reproduce the incident:
the document is born-digital / already has a text layer; its agent extraction scored 27/27; it was
extracted in `analytical_digest` mode; the document is large; the corpus is large; OCR "adds nothing
here"; "the L3 rescue already covered it".

### What Phase 5 must never do

- **Never write into a document's own KB directory.** The sibling lives in `_OCR_Only/` and nowhere
  else; that separation is what makes this augmentation instead of substitution.
- **Never let D2 stand in for a missing or failed D1.** A document whose agent extraction failed is
  not repaired by running Stage 5 on it — it goes back through Phases 1–2. The
  **ANTI-SILENT-DEGRADATION RULE** (`hdarp-processing.md`) is untouched by this phase
  and forbids exactly that substitution.
- **Never validate `_OCR_Only/` output as SPHDARP output** or score it on the 27-point rubric.

---

## Canonical Tools

| Tool | Path |
|------|------|
| PDF Splitter | pdf_splitter_orchestrator.py |
| OCR Processor | sraffa40_processor.py |
| Protocol Reference | SRAFFA_4_PROTOCOL.md |

---

**Command Version**: 6.4 Hybrid Coherence + Native RDB Enrichment + CF Progressive Decomposition + Batch Continuation + Sonnet Mandatory + Opus Validator + Mandatory Catalog Sync
**Status**: PRODUCTION READY
**Created**: 2025-12-22
**Updated**: 2026-08-27
**HDARP Protocol**: v6.4 (Hybrid Coherence: Phase 5 verbatim OCR sibling; Native RDB Enrichment, CF Progressive Decomposition, Batch Continuation, Sonnet Mandatory, Opus Validator)
**New in v6.0**: Orchestrator-driven CF recovery via bisect_chunk() + single-page scholarly framing + OCR-only-last-resort. Replaces v5.3 subagent-driven 3-strategy approach. See HDARP_FAILURE_TAXONOMY.md v2.0.
**⚠ Correction of record (2026-08-27, v6.4)** — the dated v6.0 row above stands as written and is not rewritten. Its "OCR-only-last-resort" describes the scrounger ladder's **L3 per-page CF rescue** only, and remains true of that. It never governed **Hybrid Stage 5 (D2)**, which is mandatory over **every** document and runs whether or not anything failed — see **Phase 5**.
**v5.2**: Task hygiene (Phase 3.5), 25-task budget
**v5.1**: Automatic batch continuation (Phase 4), `--single` flag, inter-batch summaries, BLOCKED detection
**v5.0**: Sonnet Mandatory model policy, Opus validator, 10-chunk batch sizing
**v4.5**: Mandatory catalog synchronization
**Key**: S=Smart batching, P=Parallel (all commands), H=Hybrid (agent-read D1 **plus** the mandatory Stage 5 verbatim OCR sibling D2 — Phase 5)

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added; Sraffa engine remains 4.0. -->
