# HDARP — Hybrid Direct Agent Reading Protocol (v6.3)

**HDARP** is a methodology for turning PDFs into high-accuracy, structured, research-grade
data. It is built for AI coding agents that have a vision-capable "read this file" tool, and it
pairs that agent vision with a document-adaptive OCR engine so that *every* kind of content on a
page — not just the prose — comes out clean and machine-usable.

This document explains the protocol for an external researcher who wants to understand or adopt
it. Current release: **HDARP Framework v6.3 ("Native Enrichment")**. The OCR engine it depends on
is versioned separately as **Sraffa 4.0**.

---

## 1. The problem it solves

Most PDF-extraction tooling forces a bad trade-off:

- **Pure OCR pipelines** read body text reasonably well but mangle tables, drop equations, and
  ignore figures entirely. You get a wall of text with the *structured* content — the part a
  researcher actually needs — destroyed.
- **Pure "let the LLM read it" approaches** are fluent but unreliable at scale: they hallucinate
  table cells, silently summarize instead of transcribing, and have no discipline about what they
  could and couldn't actually read.

HDARP's premise is that these two methods are good at *different things*, and the right answer is
a **hybrid**: use agent vision where it is strongest (the two-dimensional structure of tables,
the notation of equations, the semantics of figures) and use a dedicated, document-adaptive OCR
engine where *it* is strongest (faithful character-level transcription of long body text,
including non-Latin scripts). The result targets **95–98% accuracy across all content types**, not
just the easy one.

It is the right tool when content needs structured downstream consumption: academic papers,
statistical yearbooks, government and agency reports, bilingual or non-Latin-script documents, and
anything where the tables and equations matter as much as the prose. For a casual one-off glance at
a page, you don't need HDARP — you just read the PDF. HDARP is the *structured-extraction*
protocol.

---

## 2. The hybrid approach

> **On the accuracy figures below.** The percentages quoted here and elsewhere in this repository
> are **internal working estimates** from routine spot-checks during production runs. They are not
> the output of a published benchmark: no fixed evaluation corpus, ground-truth transcription set,
> sample size or CER/WER methodology has been published alongside them, and none ships in this
> repository. Treat them as an indication of the range the protocol operates in, not as a measured
> result — and do not cite them as one.

A full HDARP extraction is the union of two specialized components:

- **DARP (Direct Agent Reading Protocol)** — the agent uses its native vision to read **table
  structures** directly off the page, achieving ~98% structural accuracy. It handles complex
  multi-level headers, preserves layout, and keeps numerical precision exact.
- **Sraffa 4.0** — a **document-adaptive OCR engine** for **body text**, achieving 95–98%
  character-level accuracy. Rather than running a single OCR model over everything, it routes each
  page to the right method:
  1. **Digital-text pre-check** — pages with an embedded text layer are extracted instantly and
     losslessly (no OCR needed).
  2. **GPU OCR** — genuinely scanned pages go through a GPU OCR pass.
  3. **Agent QA** — every scanned page's OCR output is reviewed and graded (pass / pass-with-notes
     / fail-and-escalate).
  4. **VLM escalation** — only the pages that fail QA are escalated to a heavier
     vision-language-model pass. This is lazy-loaded, so the expensive path only runs where it's
     needed.

This routing is why HDARP handles mixed-quality documents gracefully: a born-digital report, a
clean modern scan, and a degraded historical scan can all live in the same file and each page gets
the treatment it deserves.

> An important non-negotiable: a raw embedded-text dump is **not** an HDARP extraction. The
> digital-text pre-check is a *triage* step for routing, never a substitute for the full protocol —
> because a text dump throws away the tables, equations, and figures that are the whole point. (See
> §6, Anti-Silent-Degradation.)

---

## 3. The four content types

Every chunk of a document must produce — or explicitly confirm the absence of — **all four**
content types. A chunk that has body text but no statement about the other three is considered
**incomplete**, not done.

| Type | Output | "None present" handling |
|------|--------|-------------------------|
| **Tables** | One CSV per logical table. Headers preserved exactly; empty cells stay empty (not null); full numerical precision. | A note recording "no tables present." |
| **Equations** | LaTeX transcription, with variable definitions when the source provides them. | A note recording "no equations present." |
| **Figures** | A comprehensive Markdown *description* — figure type, axis labels, trends, key observations, caption. This is interpretation, not OCR of the image. | A note recording "no figures present." |
| **Body text** | Full, faithful transcription (never a summary) with mandatory chunk-boundary markers, original line breaks, and footnotes preserved. | Always required. |

This "4-type completeness" rule is the **hard gate** of the entire protocol. Validation can deduct
points for many things, but a chunk that hasn't accounted for all four types does not pass.

---

## 4. Mandatory chunking

PDFs are visually dense, and reading too much at once causes context overflow and unreliable
extraction. So HDARP enforces a strict rule:

> **Any PDF that is more than 10 pages OR more than 1 MB MUST be chunked before processing. No
> exceptions.**

Chunking is a flat **10 pages per chunk** maximum, with a **1 MB** soft size cap. A single page
that is itself larger than 1 MB (e.g. a high-resolution cover) is accepted with a recorded warning;
processing continues normally. Each split produces a `manifest.json` recording, chunk by chunk, the
page range, size, status, and any warnings.

Processing is then strictly **one chunk at a time**: read a chunk, extract its four content types,
write the outputs, mark it complete, compact context if needed, and only then move to the next
chunk. Chunks are never read in parallel — that is the single most common cause of context blowups.

---

## 5. The command family

HDARP is driven by a family of four parallel commands. The naming is compositional: **P** = run
processors in parallel, **S** = smart (document-aware) batching, **H** = hybrid (include the
Sraffa body-text OCR pass). Read by function:

| Command | Smart batching | Hybrid (body-text OCR) | Extracts |
|---------|:---:|:---:|----------|
| **pdarp**  | no  | no  | Tables, Equations, Figures |
| **spdarp** | yes | no  | Tables, Equations, Figures |
| **phdarp** | no  | yes | Tables, Equations, Figures, **Body text** |
| **sphdarp**| yes | yes | Tables, Equations, Figures, **Body text** |

The non-hybrid variants (`pdarp` / `spdarp`) deliberately skip the body-text OCR pass — useful when
you only need the structured layer (tables/equations/figures). The hybrid variants (`phdarp` /
`sphdarp`) do the full four-type extraction. `sphdarp` is the everyday workhorse.

All four share the same operational backbone:

- **Model policy.** Extraction processors run on a fast, accurate mid-tier model; a single
  strongest-tier model acts as the **validator**. The weakest tier is banned outright — its
  accuracy on degraded scans, table structure, and equation transcription is insufficient.
- **Parallel processors + one validator.** A run dispatches *N* processors over the prepared chunks
  and one validator that scores the batch and reconciles the catalog. (`sphdarp 5` = five
  processors plus the validator.)
- **Automatic batch continuation.** Runs auto-advance through all prepared work, stopping only when
  the queue is empty, a batch is blocked, or a `--single` flag was passed.
- **Mandatory catalog sync.** A batch is never marked verified without matching catalog entries.

Supporting commands round out the lifecycle: a **prepare** step (discovers PDFs, diagnoses chunking
needs, populates extraction state — idempotent and re-runnable); a **wrap-up** validation/closeout
step; an **enrich** remediation step for flagged issues; a standalone **OCR** command that runs the
Sraffa engine on its own; and a **cleanup** step that removes chunk intermediates once a document is
verified and frees disk.

A typical lifecycle:

```
prepare → sphdarp (or family) → wrap-up → enrich (if issues) → integrate → cleanup
```

---

## 6. Integrity rules

HDARP's value depends on never lying about what it read. Two hard rules enforce that, both written
after real production incidents:

**Anti-Silent-Degradation.** If agent extraction fails (content filter, timeout, error), the
orchestrator must **stop and report** — it must never quietly fall back to a raw embedded-text dump
and pass that off as a complete extraction. A text dump is missing ~75% of what HDARP exists to
capture. The validator specifically checks for chunk-boundary markers in body text precisely because
those markers are absent from a raw text dump, so silent downgrades get caught. If a text dump is
ever used for triage, it is labelled as such, the batch stays "prepared," and no chunk files are
deleted.

**Never Fabricate.** If extraction of a passage fails, the agent records the gap and continues with
the rest. It must **never** generate substitute content, paraphrase from memory, or invent quotes,
letters, or citations to "save" a chunk. A visible gap marker is always preferable to an undetected
fabrication — fabricated source material corrupts every downstream citation built on it. Uncertain
passages are flagged (`[approximate]` / `[paraphrased]`); illegible ones get an `[unreadable]`
marker.

---

## 7. The Scrounger Standard

The protocol's working attitude is that of a **scrounger**: it does not give up. Confusion,
unfamiliarity, poor scan quality, an unusual typeface, a too-large file, a content filter on the
first try, a timeout, a transient billing error — none of these are reasons to skip content. They
are reasons to **try harder**: chunk smaller, re-frame, route to OCR, retry the transient.

There are **exactly three** acceptable reasons for a chunk *not* to receive full four-type
extraction:

1. **DUPLICATE** — the exact same content (same source, same pages) has already been fully
   extracted elsewhere. This must be *confirmed* (content-hash match or direct confirmation), not
   assumed from a "looks similar" hunch.
2. **CF_EXHAUSTED** — a content filter blocked *every* page, and only after the full recovery
   ladder was run: recursively bisect the chunk down to single pages → re-attempt each failing page
   with neutral scholarly/analytical framing → only then queue the specific still-failing pages for
   OCR. Everything else in the batch must still be extracted; only the genuinely stuck pages are
   set aside.
3. **QUARANTINED** — the file is *physically* unreadable: zero bytes, crashes the PDF library, no
   recoverable pixels or text. "Poor scan" does not qualify — poor scans go through OCR.

Anything that is not provably one of these three gets extracted. This is what keeps coverage honest
and prevents the slow, invisible erosion of a corpus through a thousand small "I'll skip this one"
decisions.

---

## 8. What's new in v6.3 — Native Enrichment

v6.3 is an **additive** release: every prior rule stays in force, and the four-type completeness
gate is unchanged. What it adds is **capturing structured metadata about each table at read time**.

The motivation: a downstream database-building stage used to re-open every extracted table just to
recover metadata — its title, page, units, footnotes, period and geographic coverage, and a
quality assessment — that the extracting agent *already had in front of it* while reading the page.
That was wasted compute. v6.3 captures it once, during the initial read.

Concretely, as each table CSV is written, the processor appends **one JSON line** describing that
table to a per-document **sidecar file** (an `RDB_METADATA` JSON-lines file, one per document or per
chunk-range). Each line records what the agent can *honestly* assess from the chunk it is already
reading:

- table title (verbatim, plus transliteration/translation where applicable), page and how the page
  was determined, units, footnotes, period coverage, geography;
- a per-field **basis** flag distinguishing what was *visibly present in the source* from what was
  *inferred* (e.g. a translation) — with **"not captured" treated as a first-class, legitimate
  value** that is never fabricated to look more complete than it is;
- a two-axis quality signal: the status of the *source datum* and the agent's *transcription
  fidelity*.

This metadata **annotates the existing Tables content type — it does not add a fifth type or change
the completeness rule.** Body-text-only extractions emit no sidecar; "no tables" markers need no
line. Downstream, the database-build stage ingests these sidecars directly, so the heavy
re-enrichment pass collapses into a thin verification.

Crucially, native metadata is held to a **soft** standard, not the hard gate: a missing or
incomplete sidecar line is a **warning-level** deduction recorded in the validation notes, never a
hard failure. This means in-flight and legacy campaigns are never retroactively failed — they can
simply be backfilled later. The metadata "rides on" the Tables type and never changes whether a
chunk is considered extracted.

---

## 9. Quality assurance

Each batch is scored by the validator on a **27-point rubric**: tables (10), equations (5), figures
(5), body text (5), and OCR confidence (2). Thresholds:

- **Reject** below 22/27 (under ~80%)
- **Acceptable** at 22–25/27
- **Target** at 26/27 or above (~96%)

The validator reads all four output types for every chunk, confirms the chunk-boundary markers that
catch silent text dumps, cross-checks OCR routing statistics, verifies that the extraction state and
the catalog agree, and writes a per-batch verdict with remediation hints. As of v6.3 it also runs
the warning-level native-metadata check described above.

---

## 10. Roles and boundaries

A few framing notes for anyone mapping HDARP onto their own setup:

- HDARP is the **cloud agent-reading** protocol (an agent with a vision-capable Read tool). A
  separate, offline local-VLM extraction engine exists for fully air-gapped GPU processing; it is a
  *different* engine with its own failure taxonomy and should never be conflated with HDARP.
- The **database-building** framework that consumes HDARP output (and the native-enrichment
  sidecars) is a **distinct, downstream** framework. It runs *after* HDARP integration; it is not
  part of HDARP and does not change HDARP's version.
- Within the originating workspace, responsibilities are split by role — a standards/protocol
  steward owns the spec and the rule files; a library/tooling role maintains the canonical chunking
  and OCR scripts. These are organizational conventions; the protocol itself is portable.

---

## Summary

HDARP extracts PDFs into four faithful, structured content types — tables (CSV), equations (LaTeX),
figures (described in Markdown), and body text (full transcription) — by combining **agent vision**
for structure with **document-adaptive OCR** for prose, targeting 95–98% accuracy across all four.
Large files are **mandatorily chunked** (>10 pages or >1 MB) and processed one chunk at a time. A
family of four commands (`pdarp` / `spdarp` / `phdarp` / `sphdarp`) covers structured-only vs.
full-hybrid extraction, with parallel processors, a strongest-model validator, and a 27-point
quality gate. The **Scrounger Standard** permits only three honest reasons to skip content —
*duplicate*, *content-filter-exhausted*, or *physically unreadable* — and hard rules forbid silent
degradation and fabrication. **v6.3** adds **native enrichment**: per-table metadata captured at
read time into a sidecar, eliminating a redundant downstream pass without touching the core
completeness guarantee.
