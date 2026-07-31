# Hopper Line v2 — Offline Local-VLM Document Extraction

**Version:** HL2 v1.0
**What it is:** A fully offline, zero-API engine that turns PDFs into structured, research-grade data on a single local GPU.

---

## What Hopper Line v2 is

Hopper Line v2 ("HL2") is a document-extraction engine that reads a PDF — or a whole
folder of PDFs — and produces a clean, structured, four-artifact representation of its
contents:

1. **Body text** — the prose, in reading order, with page/section boundary markers.
2. **Tables** — reconstructed as machine-readable CSV (and Parquet), one file per table.
3. **Equations** — transcribed to LaTeX.
4. **Figures and charts** — described in prose, with embedded text transcribed and, for
   charts, an attempt to recover the underlying data points.

Everything runs **locally on a single consumer- or prosumer-class GPU**. There are **no
API calls, no cloud round-trips, and no data ever leaves the machine.** A PDF goes in;
structured artifacts come out, entirely on hardware you control.

The output is deliberately uniform across documents, so a downstream pipeline can ingest
a thousand heterogeneous PDFs the same way it ingests one.

---

## Hopper is a distinct engine — it is **not** a cloud-agent protocol

This is the single most important thing to understand about Hopper.

There are two completely separate ways to extract a document in this ecosystem:

| | **Hopper Line v2 (HL2)** | **A cloud-agent reading protocol** |
|---|---|---|
| Where it runs | Local GPU, fully offline | Cloud, via a hosted model API |
| How it reads | Vision/OCR models look at rendered page images | An agent reads the document through a tool |
| Cost model | Electricity only; no per-token billing | Per-token API billing |
| Data exposure | Nothing leaves the machine | Document is sent to a hosted service |

The two engines emit the **same four-artifact output shape** so that downstream tooling
can consume either one interchangeably. But they share **no code, no models, and no
runtime.** They are different engines that happen to agree on an output contract.

> **Naming rule.** Hopper is its own thing. It is never referred to as, or conflated
> with, the cloud-agent reading protocol. When you see a Hopper artifact, it was produced
> locally by Hopper — not by a hosted model.

---

## Why and when offline local extraction is preferable

Offline local extraction is not merely a cheaper substitute for a cloud reader — for
several classes of work it is the *correct* choice:

- **Privacy and data control.** Sensitive, embargoed, licensed, or unpublished documents
  never have to leave the machine. There is no third party to trust with the content,
  and no terms-of-service or retention policy to reason about. For confidential corpora
  this is often a hard requirement rather than a preference.

- **Cost at scale.** Per-token API pricing makes large bulk runs expensive. A local
  engine has a fixed hardware cost and effectively zero marginal cost per page, so
  extracting tens of thousands of pages is a question of wall-clock time, not budget.

- **Bulk throughput.** When the task is "process this entire archive," a local engine
  can grind continuously, page-parallel, for as long as needed without rate limits,
  quotas, or service-availability concerns.

- **Non-Latin and degraded scripts.** Hopper routes non-Latin text (for example Cyrillic
  scans) to a model chosen specifically for script-faithful transcription, rather than a
  general model that may silently "normalize" or mistranscribe unfamiliar characters.
  Historical, archival, and multilingual corpora benefit directly from this.

- **Reproducibility.** Models run with deterministic settings and pinned versions, so the
  same PDF yields the same extraction. There is no hidden model drift behind an API.

Conversely, a cloud reader can be the better tool when a document needs genuine
*reasoning* about its contents, when the volume is tiny, or when no capable local GPU is
available. Hopper is built for the offline, bulk, privacy-sensitive, structural-extraction
end of that spectrum.

---

## How it works — the pipeline

Hopper processes a document as an ordered line of stations. The line is **page-parallel**:
there is no fixed chunking, and throughput scales with the number of pages.

1. **Intake.** Deduplicate by content hash, render each page to an image at a configurable
   resolution, and build a manifest. Unopenable or corrupt files are quarantined rather
   than allowed to poison a batch.

2. **Assess.** For every page, run four sub-assessments that drive routing:
   - **Born-digital probe** — if a page already carries a reliable embedded text layer,
     its text is taken directly and **never sent to OCR**.
   - **Layout detection** — find regions and label each one (text, title, table, figure,
     chart, equation, caption, header/footer, list) with a bounding box.
   - **Script/language identification** — per region, to pick the right model and prompt.
   - **Reading order** — recover the linear order of regions and estimate column count.

3. **Dispatch (per-region routing).** This is the heart of the engine. Each *region* — not
   just each page — is routed to the model best suited to its content type (see below).

4. **Extract.** Run the chosen model per region under a **cheap-default-then-escalate**
   policy: a fast default model runs first; only regions that fail a confidence or
   format check are re-run on a stronger recovery model. Low-confidence regions are
   **flagged and queued for review, never silently dropped.**

5. **Describe.** Crop each figure/chart/image and produce a prose description plus a
   transcription of any embedded text. For charts, additionally attempt to recover the
   plotted data points (always flagged as *estimated*, never treated as primary data).

6. **Assemble.** Flatten everything back into reading order and emit the output contract:
   a canonical structured record, a human-readable markdown rendering, the four-artifact
   view, per-table CSV/Parquet, and recovered chart data with provenance sidecars.

7. **Validate.** Run offline self-checks — all four artifact types accounted for,
   boundary markers present, born-digital recall above threshold, confidence distribution
   sane, review queue surfaced. A document that fails is marked *needs review* and is
   never auto-promoted.

---

## Per-region model routing

Hopper's guiding principle is **route each region to the right specialist**, rather than
running one model over everything. The division of labor:

| Region / content type | Handled by | Role |
|---|---|---|
| Born-digital text (good text layer) | Direct text-layer read | Bypass OCR entirely |
| Scanned/degraded text, tables, equations | A structural OCR model | Faithful structural extraction |
| Non-Latin / Cyrillic text | A script-faithful OCR model | Preserve characters exactly |
| Clean printed text | A fast default OCR model | High-throughput default |
| Gate-failed regions | A stronger recovery model | Escalation only when flagged |
| Charts | A chart/figure vision-language model | Recover data + reason about the chart |
| Figure prose | A figure-describing vision-language model | Rich descriptions |
| Layout & reading order | Lightweight detectors (CPU) | Structure before extraction |

The underlying rule: **specialist document models do the structural extraction**
(tables, scanned OCR, equations, layout), while **strong general vision-language models do
the reasoning** (interpreting and digitizing charts). Champions were selected by racing
candidate models against public benchmarks for tables, equations, post-OCR text, and
chart understanding, and the chosen models are configuration, not hard-coded — any station
can be swapped for a better model without rewriting the engine.

---

## Profiles for different corpora

Rather than tune flags by hand per document, Hopper ships **profiles** — named bundles of
resolution, model, and routing overrides tailored to a corpus type:

- **General** — sensible defaults for an arbitrary PDF.
- **Cyrillic scans** — force scanned-text handling, eager escalation, script-faithful routing.
- **Dense tabular directories** — table-first routing at high resolution, page-aware batching.
- **Equation-dense papers** — aggressive routing to the equation specialist, larger output budgets.
- **Chart/figure-heavy reports** — always emit recovered chart data.

A profile is selected at run time; everything else about the pipeline stays the same.

---

## Quality discipline

- **Born-digital bypass first** — a page with a usable text layer is never needlessly
  re-OCR'd, which preserves the most reliable text available.
- **Everything is flagged and provenanced** — every artifact records which model and method
  produced it and a confidence score. Estimated data (such as recovered chart points) is
  marked estimated and is never promoted to primary.
- **Nothing is silently dropped** — regions that can't be confidently extracted go to a
  review queue rather than disappearing.
- **Real-benchmark gating** — model and threshold choices are validated against public
  document benchmarks (table structure, equation transcription, post-OCR text, chart
  understanding) before they ship, and a regression suite re-checks them after any change.
- **Deterministic runs** — models run at fixed settings so extractions are reproducible.

---

## Output, in brief

For each document Hopper produces:

- A canonical, reading-ordered structured record (the source of truth).
- A human- and machine-readable markdown rendering.
- The four-artifact view: body text with boundary markers, per-table CSV, equation LaTeX,
  and figure/chart descriptions.
- Recovered chart data as CSV with provenance sidecars (flagged estimated).
- A confidence report and a review queue listing anything below threshold.
- A manifest recording status, models used, statistics, and validation results.

The result is a uniform, provenance-rich, research-grade dataset extracted entirely on
local hardware — suitable for downstream cataloguing, database construction, and analysis
without ever sending a single page to an external service.
