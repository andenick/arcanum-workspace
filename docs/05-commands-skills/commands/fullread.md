---
name: fullread
description: Extract text from all PDFs in a folder using Sraffa 4.0 (no DARP) — HDARP Framework v6.3 command
---

**HDARP Framework v6.3** — see `VERSION_REGISTRY.md`

> **Native RDB metadata capture (v6.3):** `/fullread` extracts **body text only** and produces no
> table CSVs, so it emits **no** `RDB_METADATA*.jsonl` sidecar. Its output is harvested mechanically
> (metadata `not_captured`); honest table metadata for such docs is recovered later by
> `robert-db-enrich`. (Doctrine: `hdarp-processing.md`.)

# /fullread — Standalone PDF Text Extraction

## Usage

```
/fullread <input_path> [options]
```

**Arguments**:
- `input_path`: Folder of PDFs or single PDF path

**Options**:
- `--output DIR`: Output location (default: `Technical/Knowledge_Base/_OCR_Only/`)
- `--batch-size N`: PDFs per batch (default: 5)
- `--wave WAVE_ID`: Process only a specific wave
- `--single`: Process one batch only, then stop
- `--skip-qa`: Skip agent QA (use programmatic confidence threshold only — NOT recommended)
- `--languages LANGS`: EasyOCR language codes (default: `en,ru`)

**Examples**:
```bash
/fullread Inputs/PDFs/                          # Process all PDFs in folder
/fullread Inputs/PDFs/ --batch-size 3           # Smaller batches
/fullread Inputs/PDFs/ --single                 # One batch only
/fullread Inputs/PDFs/document.pdf              # Single PDF
/fullread Inputs/PDFs/ --output Technical/Knowledge_Base/_OCR_Only/  # Default location (explicit)
```

## What This Does

Processes all PDFs in a folder using **Sraffa 4.0** protocol without DARP. Produces searchable `FULL_TEXT.md` + `page_manifest.json` for each document. Supports batching, waving, and auto-continuation.

**This is NOT HDARP.** There is no table CSV extraction, no equation LaTeX, no figure descriptions. Use `/sphdarp` for full HDARP treatment.

## Anti-Fabrication / No-Excuses Rule

**CRITICAL**: Agents executing /fullread MUST:
- Extract **every page** of every assigned PDF
- Extract **all text** on each page — no truncation
- Perform **Agent QA** on every scanned page
- **Never skip pages** citing context limits, time pressure, or token budgets
- If context fills up, **compact prior output** per standard context management, then continue
- **Never fabricate text** — record gaps honestly in `page_manifest.json`

---

## Phase 0: Discovery

1. Glob `input_path/**/*.pdf` (recursive)
2. For each PDF: compute page count, file size, density estimate
3. Sort by size (largest first for load balancing)
4. Write `FULLREAD_STATE.json` to `Technical/` with document inventory

---

## Phase 1: Batch Planning

1. Group PDFs into batches of `--batch-size` (default 5)
2. Large PDFs (>100 pages): exclusive batch
3. Never split a PDF across batches
4. Update `FULLREAD_STATE.json` with batch assignments

---

## Phase 2: Processing (Parallel Agents)

Spawn **N processor agents** (Sonnet) in one message, all foreground (blocking).

Each agent gets 1-2 PDFs to process sequentially.

### Processor Agent Template

```
Model: Sonnet (mandatory)

YOUR ASSIGNMENT:
- PDFs: {list of pdf paths}
- Output: Technical/Knowledge_Base/_OCR_Only/{document_id}/

SRAFFA 4.0 PROTOCOL (for each PDF):
1. Import: from sraffa40_processor import Sraffa40Processor
2. processor = Sraffa40Processor()
3. classification = processor.classify_document(pdf_path)
4. For each page:
   - Digital (pdf_type == "digital"): processor.extract_digital_page(doc, page_num)
   - Scanned/Mixed: processor.ocr_scanned_page(doc, page_num)
5. AGENT QA on every scanned page:
   - Read the OCR output text
   - View the page image using Read tool
   - Judge quality: qa_pass / qa_pass_with_notes / qa_fail_escalate
   - Record qa_status and qa_notes in page result
6. For qa_fail_escalate pages: processor.chandra2_escalate(doc, page_num)
7. Assemble: processor.assemble_full_text(page_results, mode="fullread", document_meta=...)
8. Write: processor.write_output(output_dir, full_text, manifest, summary)

EXTRACT EVERYTHING. NO EXCUSES. NO TRUNCATION. NO SKIPPING.
If context is filling up, compact your notes and continue processing.
```

### Output per Document

```
Technical/Knowledge_Base/_OCR_Only/{document_id}/
    FULL_TEXT.md              # Complete text in reading order
    page_manifest.json        # Per-page classification, engine, confidence, QA
    processing_summary.md     # Extraction statistics
```

---

## Phase 3: Report

After all agents complete:
1. Per-batch summary: documents processed, pages extracted, escalations, failures
2. Update `FULLREAD_STATE.json`: mark batch COMPLETE
3. Log total characters extracted, average confidence, escalation rate

---

## Phase 4: Continuation

1. Re-read `FULLREAD_STATE.json`
2. Find next PENDING batch
3. Auto-advance without prompting
4. **Stop when**: all batches COMPLETE, `--single` flag, no PENDING batches

---

## FULLREAD_STATE.json Schema

```json
{
  "schema_version": "1.0",
  "last_updated": "2026-04-30T12:00:00Z",
  "input_path": "Inputs/PDFs/",
  "output_path": "Technical/Knowledge_Base/_OCR_Only/",
  "total_documents": 25,
  "languages": ["en", "ru"],
  "batches": {
    "BATCH_001": {
      "status": "COMPLETE",
      "documents": [
        {
          "document_id": "doc_name",
          "source_pdf": "doc_name.pdf",
          "pages": 50,
          "pdf_type_summary": {"digital": 48, "scanned": 2},
          "status": "COMPLETE"
        }
      ]
    },
    "BATCH_002": {
      "status": "PENDING",
      "documents": [...]
    }
  }
}
```

---

## Relationship to HDARP

`/fullread` and HDARP are **complementary**:

- `/fullread` gives you searchable text fast. No table CSV extraction, no equation LaTeX, no figure descriptions.
- `/sphdarp` gives you the full HDARP treatment: DARP + Sraffa 4.0 text + integration.
- **Output separation**: `/fullread` writes to `Technical/Knowledge_Base/_OCR_Only/` while HDARP writes to `Technical/Knowledge_Base/`. This prevents OCR-only output from conflicting with full HDARP extractions.
- A project can run `/fullread` first for quick access, then `/sphdarp` later. The HDARP output lives in a separate folder so both coexist. The `_OCR_Only/` version serves as a reference/fallback.
- Both use the same `sraffa40_processor.py` core and the same Sraffa 4.0 protocol.

---

## Canonical Tools

| Tool | Path | Purpose |
|------|------|---------|
| Sraffa 4.0 Processor | `sraffa40_processor.py` | Core OCR processor |
| OCR Engines | `sraffa30_ocr_engines.py` | Engine wrappers |
| Protocol | `SRAFFA_4_PROTOCOL.md` | Protocol reference |

## Related Commands

- `/sphdarp` — Smart parallel HDARP (full extraction with DARP)
- `/preparehdarp` — Prepare PDFs for HDARP processing
- `/sraffa-ocr` — Direct OCR on single files
- `/enrichhdarp` — Remediate incomplete HDARP extractions

---

**Command Version**: 6.3  
**Protocol**: Sraffa 4.0  
**Updated**: 2026-06-13

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native-enrichment applicability clarified; fullread remains body-only and Sraffa Protocol remains 4.0. -->
