---
description: "Sraffa 4.0 Document-Adaptive OCR — HDARP Framework v6.4 command (Sraffa engine 4.0): Extract text from PDFs using PyMuPDF (digital), EasyOCR GPU (scans), Agent QA, Chandra 2 escalation"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
argument-hint: "<pdf_path> [output_path] | --batch <dir> | --chunks <doc_id>"
---

**HDARP Framework v6.4** — see `VERSION_REGISTRY.md`

> **Native RDB metadata capture (v6.4):** Sraffa is body-text OCR and normally emits **no** sidecar.
> **Only if** an OCR pass reconstructs a table into a CSV, emit one `RDB_METADATA*.jsonl` line for it
> with `transcription_status="R"` (reconstructed) and honest `field_basis` (OCR'd values are
> `agent_inferred` unless verbatim-legible). Spec:
> `NATIVE_ENRICHMENT_CONTRACT.md`.

# Sraffa 4.0 OCR Command

**Command**: /sraffa-ocr
**Version**: 6.4
**Created**: 2026-03-18
**Updated**: 2026-08-27

## What This Does

Invokes the **Sraffa 4.0** document-adaptive OCR system. Classifies each page as digital/scanned/mixed and routes to the appropriate engine. Digital pages get instant PyMuPDF extraction. Scanned pages get EasyOCR GPU with mandatory Agent QA. Only pages that fail QA get escalated to Chandra 2.

Replaces Sraffa 3.0 multi-engine consensus (retired after three rounds of benchmarks).

## Usage

```
/sraffa-ocr document.pdf                    # OCR single PDF, save to _OCR_Only/
/sraffa-ocr document.pdf output/            # OCR single PDF, save to custom dir
/sraffa-ocr --batch dir/                    # OCR all PDFs in a directory (to _OCR_Only/)
/sraffa-ocr --chunks DOC_ID                 # OCR hollow HDARP chunks (to Knowledge_Base/)  [CF fallback]
/sraffa-ocr --augment DOC_ID                # Hybrid Stage 5: EVERY page, no skips (to _OCR_Only/)
/sraffa-ocr --augment --batch <KB_dir>      # Hybrid Stage 5 across a whole campaign
```

> **`--chunks` and `--augment` are different jobs and must not be substituted for each other.**
> `--chunks` rescues pages the agent could not read (skips pages that already have content).
> `--augment` produces the mandatory verbatim sibling for pages the agent read *successfully*
> (skips nothing). See Mode 4 and the two-OCR distinction in `sphdarp-scrounger.md`.

## Modes

### Mode 1: Single PDF

Process one PDF file with Sraffa 4.0. Default output: `Knowledge_Base/_OCR_Only/<short_id>/` — see
**Key and path convention** below; the key is `short_id`, never `doc_id`.

**Implementation**:
```bash
python "sraffa40_processor.py" "INPUT.pdf" --output "OUTPUT_DIR" --mode fullread
```

Display results: page classification summary, confidence, method per page, total chars extracted.

### Mode 2: Batch Directory (`--batch dir/`)

Process all PDFs in a directory. Default output: `Knowledge_Base/_OCR_Only/` — each PDF gets its own
`<short_id>/` subdirectory (see **Key and path convention** below).

**Implementation**:
```bash
python "sraffa40_processor.py" "INPUT_DIR/" --output "OUTPUT_DIR" --mode fullread
```

Each PDF gets its own output subdirectory with FULL_TEXT.md + page_manifest.json.

### Mode 3: HDARP Chunks (`--chunks DOC_ID`)

Process hollow chunk PDFs for an HDARP document using Sraffa 4.0 — the scrounger ladder's **L3
content-filter rescue** for pages that failed agent reading after bisect + scholarly framing.

> 🔴 **Output goes to `_OCR_Only/`, NOT into the document's KB directory** (corrected
> 2026-08-27). This mode previously wrote `Knowledge_Base/{DOC_ID}/body_text/chunk_NNN_body.txt` and
> assembled a `FULL_TEXT.md` beside the agent extraction. That filename is the literal signature the
> extraction-integrity check **auto-FAILs** (`sphdarp.md` Phase 3.25, check 2) — it is what the
> 2026-05-06 silent-degradation incident produced. **Never emit it.** Nothing this command
> writes may land inside a document's own KB directory, in any mode.
>
> The recovered text still reaches the body — but by an **agent splicing the named page** into the
> chunk-range body file with an explicit L3 marker and `transcription_status: R`, never by this tool
> dropping a raw dump next to the real extraction. Substitution vs augmentation is defined once, in
> `hdarp-processing.md` ("Hybrid body text = two layers"); the L3-vs-Stage-5 distinction
> is in `sphdarp-scrounger.md`.

1. Find all chunk PDFs in `HDARP_Processing/{DOC_ID}/chunks/`
2. Skip chunks that already have real content (>500 chars body text) — **correct here, fatal in a
   Hybrid pass.** The heuristic exists because this mode's job is filling holes; under the mandatory
   Stage-5 pass it would skip exactly the pages that *have* agent text, i.e. nearly every page in the
   corpus, satisfying the mandate on paper and running nothing. **Never reach for `--chunks` when the
   job is Stage 5 — use `--augment` (Mode 4), which has no skip.**
3. Run Sraffa 4.0 on each hollow chunk (classify → extract → Agent QA → escalate if needed)
4. Write output to `Knowledge_Base/_OCR_Only/<short_id>/l3_rescue/chunk_NNN.txt`
5. Write `page_manifest_chunk_NNN.json` beside it, naming the pages rescued and why
6. After all chunks: assemble `_OCR_Only/<short_id>/L3_RESCUE.md` + a unified page manifest. **Do not
   write a `FULL_TEXT.md` or any `body_text/` file into `Knowledge_Base/{DOC_ID}/`**
7. Hand the rescued pages to an agent (or `/enrichhdarp` Type D) to splice into the chunk-range body
   file with an L3 marker — that splice, not this write, is what lands text in the KB

### Mode 4: Hybrid Augment Pass (`--augment`)

**This is HDARP Stage 5** — the mandatory verbatim OCR sibling for **every** document, including
documents whose agent extraction is complete and validated at 27/27. It is not a fallback and is not
conditional on scan quality, extraction mode, or whether anything failed.

> **Owner (one owner — this statement appears identically in `sphdarp.md` Phase 5 and
> `hdarp-wrapup.md`):** the **`/sphdarp` orchestrator** owns Hybrid Stage 5 and runs it as its
> **Phase 5**, once per drain scope, by invoking `/sraffa-ocr --augment` after `/hdarp-wrapup` has
> gated the scope. **This command never schedules itself and is never invoked inline mid-round** —
> not by a processor, not by the validator, not by any `/sphdarp` Phase 1–4 step. **`/hdarp-wrapup`
> GATES Stage 5 and never RUNS it.** "Once per drain scope" is defined precisely (what counts as the
> end of a run, and over which documents) in `sphdarp.md` Phase 5 — do not re-derive it here.

**What makes it different from Mode 3:** `--augment` processes **every page regardless of existing
text-layer length**. There is no >500-char skip, no hollow-chunk filter, no content threshold of any
kind. A page with 40,000 characters of text layer is OCR'd exactly like a blank one.

**Where it writes:** the separate tree `Knowledge_Base/_OCR_Only/<short_id>/` — **never** into the
document's own KB directory. That separation is what keeps this augmentation rather than
substitution; the definition and the substitution/augmentation test table live in
`hdarp-processing.md` ("Hybrid body text = two layers"). Do not re-derive them here.

1. Enumerate the document's source pages (source PDF, or all chunk PDFs under
   `HDARP_Processing/{DOC_ID}/chunks/`)
2. Route **every** page per the Sraffa 4.0 classifier — no skip test
3. Write `_OCR_Only/<short_id>/FULL_TEXT.md` + `textlayer_manifest.json` / `page_manifest.json`,
   each recording **both** keys: `short_id` (this directory) and `doc_id` (the KB folder it is a
   sibling of)
4. Record per page which engine produced the text and whether it needs the GPU half
5. Idempotent: a document with a complete manifest is skipped **as a document**, never page-by-page
   on a content threshold

**Split execution (the GPU half is separable).** The pass has two halves and they can run
independently: the PyMuPDF text-layer half needs no GPU, and only pages with no usable text layer
need EasyOCR. Reference implementations, both writing to `_OCR_Only/` and both idempotent:

| half | script | GPU |
|---|---|---|
| text layer | `textlayer_sweep.py` | none |
| OCR remainder | `sraffa_ocr_sweep.py` | required |

**Exit code 0** with a written manifest for every document in scope is the success condition; a
document with pages deferred to the GPU half exits 0 with those pages recorded, not silently dropped.

## Engine facts (measured 2026-08-27 — read before scheduling a run)

**(a) The processor hard-requires CUDA at startup and cannot share a busy GPU.**
`sraffa40_processor.py.__init__` (≈`:100-111`) probes `torch.cuda.is_available()` and **raises
`RuntimeError` unless a CUDA device is present**, unless `--no-gpu` is passed (CPU fallback, not
recommended). EasyOCR itself is **lazy-loaded** — `_ensure_easyocr()` (`:139-149`) is called only
from the scanned/mixed page path (`:345`), so a fully born-digital PDF never loads it. *Correction of
record:* the startup banner prints `GPU: True` from that probe, and `EasyOCR ready` is logged later,
on the first scanned page — the two lines have been read together as "EasyOCR loads at startup
unconditionally", which the code does not do. **The operational conclusion is unchanged:** the
process will not start while the GPU is unavailable, and will claim GPU memory the moment a scanned
page appears, so it must never be scheduled against the user's live GPU work. Split the pass and run
the text-layer half (pure PyMuPDF, no CUDA gate) when the GPU is committed.

**(b) OCR-derived values are `R`, never `V`.** Text this command produces is reconstructed, not
audited: any `RDB_METADATA*.jsonl` line emitted from an OCR pass takes `transcription_status="R"`
and `field_basis: agent_inferred` unless the value is verbatim-legible in the render. **`V` is
reserved for the formal audit spot-check and must never be emitted by an OCR pass.** Full vocabulary:
`NATIVE_ENRICHMENT_CONTRACT.md`.

## Key and path convention

**Every mode of this command writes under `Knowledge_Base/_OCR_Only/`, and no mode writes into a
document's own KB directory** (uniform since 2026-08-27). Modes 1–2 land at `_OCR_Only/<short_id>/`;
Mode 3 at `_OCR_Only/<short_id>/l3_rescue/`; Mode 4 at `_OCR_Only/<short_id>/`. Text reaches a
document's KB only when an **agent** splices it there with an explicit marker — never as a bulk write
from this tool.

**Where the KB root is.** `_OCR_Only/` sits directly under the project's Knowledge_Base root, and
**that root differs by tree**: `<Project>/Knowledge_Base/` in some trees,
`<Project>/Technical/Knowledge_Base/` in others. Both are real — resolve the
root for the project you are running against rather than assuming either shape.

🔴 **`short_id` is NOT `doc_id`.** `_OCR_Only/` subdirectories are keyed by **`short_id`**: a
shortened, filesystem-safe stem (`01_Foley_1986`). **`doc_id` must equal the KB folder name
verbatim** (`NATIVE_ENRICHMENT_CONTRACT.md`) — e.g.
`[1986] Foley - Understanding Capital`. They are different keys and are not interchangeable;
confusing them mis-keyed 114 documents on 2026-08-27. **Every manifest and exception row this
command produces records both.**

---

## Architecture: Sraffa 4.0 Pipeline

```
PDF Input
    │
    ▼
[Phase 1: Per-Page Classification]
    │ For each page: text_density + word_quality_check
    │
    ├── Digital (text_density > 0.80) ──▶ PyMuPDF get_text() → confidence: 1.0
    │
    ├── Scanned (text_density < 0.10) ──▶ EasyOCR GPU (300 DPI)
    │                                         │
    │                                         ▼
    │                                    [Agent QA]
    │                                    ├── qa_pass → Accept
    │                                    ├── qa_pass_with_notes → Accept + notes
    │                                    └── qa_fail_escalate → Chandra 2 (page-only)
    │
    └── Mixed (0.10-0.80) ──▶ Same as Scanned
    │
    ▼
[Phase 5: Assembly]
    FULL_TEXT.md + page_manifest.json + processing_summary.md
```

## Expected Accuracy

| Document Type | Engine | Accuracy | Confidence |
|--------------|--------|----------|------------|
| Digital PDF (text layer) | PyMuPDF | 100% | 1.0 |
| Clean scanned document | EasyOCR GPU | 90-95% | 0.85-0.95 |
| Degraded/aged scan | EasyOCR GPU → Chandra 2 | 95-98% | 0.90-0.95 |
| Heavily damaged scan | Chandra 2 (escalated) | 85-95% | 0.80-0.95 |

## Engine Requirements

| Engine | Package | Status | Notes |
|--------|---------|--------|-------|
| PyMuPDF | `PyMuPDF` (fitz) | Required | Page classification + digital extraction |
| EasyOCR | `easyocr` | Required | Primary OCR for scans (GPU mandatory) |
| Chandra 2 | `chandra-ocr[hf]` + `bitsandbytes` | Optional | Escalation engine (lazy-loaded) |
| PIL/Pillow | `Pillow` | Required | Image processing |
| NumPy | `numpy` | Required | Array operations |
| PyTorch | `torch` | Required | GPU/CUDA support |

Check availability:
```bash
python "sraffa40_processor.py" --help
```

## Output Formats

### FULL_TEXT.md
YAML frontmatter with provenance + per-page text with boundary comments.

### page_manifest.json
Per-page: pdf_type, extraction_method, confidence metrics, qa_status, escalation info.

### processing_summary.md
Extraction statistics, timing, page type distribution.

## Canonical Paths

| Resource | Path |
|----------|------|
| Sraffa 4.0 Processor | `sraffa40_processor.py` |
| OCR Engines | `sraffa30_ocr_engines.py` |
| Protocol Reference | `SRAFFA_4_PROTOCOL.md` |

## Related Commands

| Command | Relationship |
|---------|-------------|
| `/enrichhdarp` | Uses Sraffa 4.0 for Type D (OCR recovery) remediation |
| `/sphdarp` | **Owns Hybrid Stage 5 and runs it, as its Phase 5, once per drain scope** — by invoking `--augment` after `/hdarp-wrapup` has gated the scope. `/sphdarp` produces the agent-read layer (D1); this command's Mode 4 produces its mandatory verbatim sibling (D2). **This command is never called inline mid-round and never schedules itself.** Two-layer definition + alias list: `hdarp-processing.md` |
| `/hdarp-wrapup` | Step 4. Its Phase 2 calls `--chunks` for L3 CF gap pages. It **GATES Stage 5 and never RUNS it**: `--augment` runs after wrap-up, over every document, called by `/sphdarp` Phase 5 |
| `/preparehdarp` | Classifies pages during preparation, pre-extracts digital text |
| `/fullread` | Standalone text extraction using Sraffa 4.0 for entire folders |

---

**Command Version**: 6.4
**Status**: PRODUCTION READY
**Created**: 2026-03-18
**Updated**: 2026-08-27
**Sraffa Protocol**: 4.0 (Document-adaptive: PyMuPDF + EasyOCR GPU + Agent QA + Chandra 2)
**Replaces**: Sraffa 3.0 multi-engine consensus (retired)

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native metadata rule added for reconstructed table CSVs; Sraffa Protocol remains 4.0. -->
