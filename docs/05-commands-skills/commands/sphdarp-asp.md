---
description: "SPHDARP-ASP — HDARP Framework v6.4 (command v1.0): Smart Parallel HDARP with Handwriting (5 content types) for Anwar Shaikh Papers"
allowed-tools: Bash, Read, Write, Glob, Grep, Task
argument-hint: "[number]"
---

**HDARP Framework v6.4** — see `VERSION_REGISTRY.md`

> **Native RDB metadata capture (v6.4):** as each table CSV is written, append one
> `RDB_METADATA*.jsonl` line per the doctrine in `hdarp-processing.md`
> ("Native RDB Enrichment Capture") + spec `NATIVE_ENRICHMENT_CONTRACT.md`.
> **ASP specific:** a table transcribed from **handwriting** gets `transcription_status` from the
> handwriting confidence — low/ambiguous hand → `L`, heavily reconstructed → `R` (never `V`).

# SPHDARP-ASP: Smart Parallel HDARP with Handwriting v1.0

**Command**: /sphdarp-asp [N]
**Full Name**: Smart Parallel Hybrid Direct Agent Reading Protocol — ASP Handwriting Extension
**Version**: 6.3
**Based On**: SPHDARP v4.5
**Created**: 2026-03-23
**Project**: ASP (Anwar Shaikh Papers) ONLY

## What This Is

SPHDARP-ASP extends standard SPHDARP v4.5 with a **5th content type: Handwriting**.
Both DARP vision AND Sraffa-ASP OCR attempt handwriting capture.

This command is ASP-specific. The Shaikh Papers collection is 15.2% handwriting (93 docs, 8,044 pages) — flash cards, research notes, lecture annotations, mathematical derivations.

## Usage

```
/sphdarp-asp        # 5 processors + 1 validator, continues until all batches done
/sphdarp-asp 3      # 3 processors, continues until all batches done
/sphdarp-asp 10     # 10 processors (max), continues until all batches done
/sphdarp-asp 5 --single   # Process ONE batch only, then stop
```

## What Changes from Standard SPHDARP

| Feature | SPHDARP v4.5 | SPHDARP-ASP |
|---------|-------------|-------------|
| Content Types | 4 (A-D) | **5 (A-E)** |
| Handwriting | Not extracted | **Classified + transcribed** |
| OCR Fallback | Sraffa 4.0 | Sraffa 4.0 + **Sraffa-ASP** |
| Quality Score | 27 points | **32 points** |
| Min Pass | 22/27 | **26/32** |
| Output Dirs | 4 | **5** (adds handwriting/) |

Everything else is inherited from SPHDARP v4.5:
- Smart batching (one document per agent)
- Sonnet Mandatory model policy (Sonnet processors, Opus validator, Haiku BANNED)
- Mandatory catalog sync
- Error recovery with /phdarp retry
- Foreground-only execution

---

## 5 Content Types (HDARP-ASP Protocol)

### Per-Chunk Extraction (Agent Instructions)

Each processor agent extracts ALL 5 types from every chunk:

**A. Tables → CSV** (unchanged, 98%+ accuracy)
- Detect tabular structures, extract to CSV files
- One file per table: `CSV_Tables/chunk_NNN_table_001.csv`

**B. Equations → LaTeX** (unchanged, 100% target)
- Transcribe mathematical notation as LaTeX
- One file per equation block: `Equations/chunk_NNN_equation_001.md`

**C. Figures → Markdown** (unchanged, 200+ words)
- Describe charts, graphs, diagrams
- One file per figure: `Figures/chunk_NNN_figure_001.md`

**D. Body Text → Markdown** (agent-read; Sraffa 4.0 is the mandatory verbatim sibling, NOT a fallback)
- Extract printed/typed text by **agent reading** of the chunk PDF
- One file per chunk range: `FULL_TEXT_chunks_NNN_NNN.md` — the current convention, **with a
  `<!-- chunk_NNN -->` boundary marker around each chunk's text.** The markers are what prove agent
  reading. The old `body_text/chunk_NNN_body.txt` pattern is **auto-FAILed** by the extraction-integrity
  check in `sphdarp.md` Phase 3.25 as "a PyMuPDF dump, not HDARP" — never emit it.
- The Sraffa 4.0 OCR pass is the **mandatory end-of-run verbatim sibling** for every document, landing
  in `Knowledge_Base/_OCR_Only/<short_id>/`; it augments this type and never replaces it. Canonical
  rule: `hdarp-processing.md`, "Hybrid body text = two layers". (Sraffa-**ASP** below stays a genuine fallback — for Type E handwriting only.)

**E. Handwriting → Markdown** (NEW)
1. **Detect**: Scan each page for handwritten content
2. **Classify**: Assign one of:
   - `HW_NONE` — No handwriting detected (most pages)
   - `HW_MARGINAL` — Annotations in margins of typed text
   - `HW_FULL_PAGE` — Entirely handwritten page
   - `HW_MATHEMATICAL` — Handwritten equations/derivations
   - `HW_FLASH_CARD` — Flash card format (front/back)
   - `HW_DIAGRAM` — Hand-drawn diagrams with labels
3. **Transcribe**: Convert handwritten content to markdown
   - Text → markdown with paragraph structure
   - Math → LaTeX notation
   - Diagrams → descriptive text + label transcription
4. **Confidence**: Rate 0.0-1.0 per element
5. **Uncertainty**: Mark `[illegible]` for unreadable, `[uncertain: word?]` for guesses
6. **Output**: `handwriting/chunk_NNN_handwriting.md` + `chunk_NNN_handwriting_meta.json`

**If no handwriting detected**: Write "No handwriting detected in this chunk." to the handwriting file. This scores full points (correct detection of absence).

---

## Sraffa-ASP OCR Fallback for Handwriting

When DARP vision is content-filter blocked OR handwriting confidence is low:

```bash
# Run Sraffa-ASP on a document's chunks
python "sraffa_asp_chunk_ocr.py" \
  --chunks "{doc_id}"
```

**Sraffa-ASP Pipeline**:
1. Surya DetectionPredictor → line bounding boxes
2. Surya RecognitionPredictor + TrOCR → dual recognition per line
3. 5-rule consensus adjudication:
   - Perfect Agreement (10.7% of lines)
   - English Word Voting (18.4%)
   - Anti-Hallucination (0.2% — catches Surya repetition loops)
   - Character Similarity (45.3% — word-level merge)
   - Default to Surya (23.1%)
4. Output: JSON with text + confidence + consensus stats

**Calibration Results** (112 pages, 23 docs):
- Consensus English ratio: **0.282** (vs Surya 0.271, TrOCR 0.223)
- Consensus beats Surya on 41% of pages
- Consensus beats TrOCR on 89% of pages

**When to use Sraffa-ASP vs DARP**:
- DARP (Claude Vision): Primary for all content. Best context understanding, handles mixed content.
- Sraffa-ASP: Fallback when DARP is CF-blocked on handwriting pages, or when DARP handwriting confidence < 0.5.

---

## Output Structure

```
Knowledge_Base/{document_id}/
├── CSV_Tables/                     (Type A)
│   └── chunk_NNN_table_001.csv
├── Equations/                      (Type B)
│   └── chunk_NNN_equation_001.md
├── Figures/                        (Type C)
│   └── chunk_NNN_figure_001.md
├── FULL_TEXT_chunks_NNN_NNN.md      (Type D — agent-read, `<!-- chunk_NNN -->` markers required)
├── handwriting/                    (Type E — NEW)
│   ├── chunk_NNN_handwriting.md        (best transcription)
│   ├── chunk_NNN_handwriting_meta.json (classification, confidence, source)
│   └── chunk_NNN_handwriting_ocr.md    (Sraffa-ASP output, if CF-blocked)
└── SPHDARP_PROCESSING_SUMMARY.md
```

---

## Quality Scoring: 32 Points

| Component | Points | Criteria |
|-----------|--------|----------|
| Tables | 12 | Structure, accuracy, completeness |
| Equations | 3 | LaTeX correctness |
| Figures | 3 | Description quality (200+ words) |
| Body Text | 5 | Completeness, accuracy |
| OCR Confidence | 2 | Sraffa 4.0 / Sraffa-ASP confidence |
| Formatting | 2 | Consistent structure |
| **Handwriting Detection** | **2** | Correct HW_NONE/HW_* classification |
| **Handwriting Accuracy** | **3** | Transcription quality (English ratio, legibility) |
| **Total** | **32** | **Minimum pass: 26/32 (81%)** |

**Handwriting scoring for HW_NONE documents**:
- Detection: 2/2 (correctly identified no handwriting)
- Accuracy: 3/3 (N/A = full marks)
- Total: 5/5 handwriting points

---

## Document Processor Agent Template (ASP Extension)

**Model**: sonnet (Sonnet Mandatory — Haiku BANNED)
**Agent Role**: Process ALL assigned chunks from ONE ASP document using HDARP-ASP protocol

**Agent Assignment**:
- YOUR DOCUMENT: {document_name}
- YOUR CHUNKS: {chunk_list} ({chunk_count} chunks total)
- OUTPUT DIRECTORY: Knowledge_Base/{document_name}/
- HANDWRITING CLASSIFICATION: {hw_classification from handwriting_survey.csv}

**Smart Batching**: Same as SPHDARP v4.5 (one document per agent, 2-5 chunks max)

**Sequential Processing**:
1. Read Chunk N → Extract all **5** types → Quality Check
2. Repeat for each assigned chunk in order
3. Create Document Completion Report

**HDARP-ASP Protocol (5 Content Types per Chunk)**:
- A. Tables → CSV (98%+ accuracy)
- B. Equations → LaTeX (100% target)
- C. Figures → Markdown (200+ words per figure)
- D. Body Text → agent-read `FULL_TEXT_chunks_NNN_NNN.md` with `<!-- chunk_NNN -->` markers
  (Sraffa 4.0 = mandatory end-of-run verbatim sibling, not a fallback — see `hdarp-processing.md`, "Hybrid body text = two layers")
- **E. Handwriting → Markdown (classify + transcribe)**
  - Classify: HW_NONE / HW_MARGINAL / HW_FULL_PAGE / HW_MATHEMATICAL / HW_FLASH_CARD / HW_DIAGRAM
  - Transcribe handwritten content to markdown
  - Math notation → LaTeX
  - Mark [illegible] for unreadable, [uncertain: word?] for guesses
  - Rate confidence 0.0-1.0
  - If CF-blocked: Sraffa-ASP handles handwriting via OCR

**Handwriting Hints** (from calibration):
- Flash cards (Boxes 06, 07): Clear block writing, 4-5/5 OCR quality
- Research notes (Boxes 10, 14): Mixed quality, 2-4/5
- Dense lecture notes (Boxes 11, 12): Often mixed typed + handwritten
- Math-heavy (Boxes 01, 15): Equations need LaTeX, variable names critical
- Many "handwriting" docs are mixed content — classify per PAGE, not per document

---

## Handwriting Survey Reference

Classification of all 613 ASP PDFs:
- `handwriting_survey.csv`
- PURE_HANDWRITING: 47 docs (5,153 pages)
- LIKELY_HANDWRITING: 46 docs (2,891 pages)
- SCANNED_TYPED: 520 docs (30,795 pages)

Use this to prioritize: PURE_HANDWRITING and LIKELY_HANDWRITING docs get special attention for Type E extraction. SCANNED_TYPED docs should still check for handwriting (some have marginal annotations) but expect HW_NONE for most pages.

---

## Sraffa-ASP Engine Location

```
sraffa_asp/
├── __init__.py
├── sraffa_asp_engines.py       # Surya + TrOCR wrappers
├── sraffa_asp_consensus.py     # 5-rule handwriting consensus
├── sraffa_asp_processor.py     # Main orchestrator
└── sraffa_asp_chunk_ocr.py     # HDARP chunk wrapper (CLI)
```

---

## Inherited from SPHDARP v5.1

All other features are identical to `/sphdarp`:
- Smart batch assignment algorithm
- Quality-first model policy (Sonnet minimum)
- BATCH_STATE integration
- Agent execution mode (foreground-only)
- Phase 1 error handling
- Phase 2 error recovery (/phdarp retry)
- Phase 3 mandatory catalog sync
- **Phase 4 batch continuation** (v5.1 — automatic multi-batch processing, `--single` flag, inter-batch summaries, BLOCKED detection). See SPHDARP v5.1 for full Phase 4 specification.
- Quality validator agent template
- Performance benchmarks

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added for ASP table and handwriting-table outputs. -->
