# HDARP — Hybrid Direct Agent Reading Protocol (v6.4)

**HDARP** is a methodology for turning PDFs into high-accuracy, structured, research-grade
data. It is built for AI coding agents that have a vision-capable "read this file" tool, and it
pairs that agent vision with a document-adaptive OCR engine so that *every* kind of content on a
page — not just the prose — comes out clean and machine-usable.

This document explains the protocol for an external researcher who wants to understand or adopt
it. Current release: **HDARP Framework v6.4 ("Hybrid Coherence")**, released **2026-08-27** — a
restoration release that inherits every v6.3 mandate. The OCR engine it depends on is versioned
separately as **Sraffa 4.0**, and it stays 4.0 in this release: the engine did not change, its
place in the lifecycle did.

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

Since v6.4, this two-component split is also mirrored *inside* body text itself: the prose is read
**twice** — once by the agent and once, verbatim, by the OCR engine — as two distinct layers with
distinct jobs (§4). That is the "hybrid coherence" the release is named for.

> An important non-negotiable: a raw embedded-text dump is **not** an HDARP extraction. The
> digital-text pre-check is a *triage* step for routing, never a substitute for the full protocol —
> because a text dump throws away the tables, equations, and figures that are the whole point. (See
> §8, Anti-Silent-Degradation.)

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
| **Body text** | The **agent-read layer (D1)**: a structured reading of the chunk — verbatim transcription by default, or a neutral paraphrased digest for in-copyright sources (§6) — with mandatory chunk-boundary markers, page references, and a quality assessment. | Always required. |

This "4-type completeness" rule is the **hard gate** of the entire protocol. Validation can deduct
points for many things, but a chunk that hasn't accounted for all four types does not pass.

One thing v6.4 makes explicit: the body-text type ("D") is itself **two layers**, D1 and D2 — the
agent read plus a mandatory verbatim OCR sibling. The four-type gate is unchanged; the body-text
type simply has internal structure now. See §4.

---

## 4. Hybrid body text: two layers (D1 / D2)

The **H in HDARP has always meant Hybrid**, and v6.4 restores that meaning to the letter: body
text is **two readings of the same pages**, not one.

1. **D1 — the agent-read layer.** This *is* the HDARP body text. An agent reads the chunk PDF and
   writes a structured reading of it — verbatim transcription, or a digest where the extraction
   mode calls for one (§6) — into the document's own knowledge-base directory, with
   chunk-boundary markers, page references, and a quality assessment. D1 is the **only layer that
   can be validated as HDARP**: the four-type completeness rule, the scrounger ladder, and the
   27-point rubric all score this layer. **Nothing may stand in for D1.**
2. **D2 — the verbatim OCR sibling.** A Sraffa 4.0 verbatim transcription of the same pages,
   routed per the engine's page routing (§2), and landing in a **separate `_OCR_Only/<doc>/`
   tree** — never inside the document's own extraction directory — so it can never be mistaken
   for, or validated as, the agent layer. **A document is unfinished without D2.**

### Augmentation, never substitution

Hybrid body text is **augmentation, never substitution**, and the clause does not weaken the
anti-silent-degradation rule (§8) by one word. That rule forbids **substitution**: putting a
machine text dump *in place of* agent extraction, labelling it HDARP, and validating it. That
prohibition stays absolute. The Hybrid clause permits only **augmentation**: running OCR
*alongside* an extraction that is already complete, into a separate tree, under its own labels —
adding to what the agent produced and replacing nothing. Three checkable facts tell them apart:

| | substitution (FORBIDDEN) | augmentation (MANDATORY) |
|---|---|---|
| **when** | instead of agent reading — typically after it failed | after agent extraction is complete |
| **where** | inside the document's own extraction directory, labelled as the HDARP body text | the separate `_OCR_Only/` tree, under its own labels |
| **what it claims** | to *be* the extraction | to be a second, verbatim reading of the same pages |

So: an OCR layer standing where the agent layer is missing or failed is a substitution and a
violation. An OCR layer standing beside a complete agent layer is the protocol working as named.
And a missing OCR sibling makes a document incomplete — it is never a reason to accept a dump
instead.

### Hybrid Stage 5

D2 is produced by a **mandatory end-of-run pass over every document** — not a content-filter
fallback, and not conditional on scan quality. Executed by `/sraffa-ocr --augment` ("Mode 4"),
it processes **every page with no skips**. Earlier revisions described this stage as conditional
("if needed"); v6.4 removes the qualifier, and the closeout step gates it: a document that
carries neither a sibling nor a recorded reason for lacking one fails closeout.

This one stage goes by several synonymous names across the command family — **Hybrid Stage 5**
(canonical), **`/sraffa-ocr --augment`** / **Mode 4** (the executable form), **content type D2**,
and **the verbatim sibling** — but it is a single pass.

It must not be confused with a different job the same OCR command performs: the **per-page
rescue** for individual pages that exhausted the content-filter retry ladder (§9). The rescue is
*spliced into* the agent-layer body text and explicitly marked as a recovered page — a degraded
outcome for pages the agent could not read. The rescue and the augment pass are different jobs
and are never substituted for each other.

### The weave format (weave-1.0)

The two layers are **joined without being merged** by the **weave-1.0** format. Records are
anchored on page sequence and folio — never on character offsets — and the join emits a weave
record (`WEAVE.jsonl`), a disagreement log (`DISAGREEMENTS.jsonl`), and a reconstruction record
(`RECONSTRUCTION.json`). Where the two layers disagree, the disagreement is **recorded, never
auto-resolved**: both readings stay available for a human to adjudicate.

---

## 5. Mandatory chunking

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

## 6. Extraction modes and output discipline

Body-text extraction — the D1 layer — has two modes. Tables, equations, and figures are extracted
identically in both; only the body-text treatment differs. Both modes still require all four
content types and chunk-boundary markers.

- **Verbatim mode** (default for public-domain and technical sources): faithful, structured
  full-text transcription. Original line breaks and footnotes are preserved.
- **Analytical-digest mode** (default for in-copyright trade books, or any document where a chunk
  range would exceed the output budget): a neutral, encyclopedic, **paraphrased** digest of each
  chunk — arguments, events, data, and themes summarized in the reader's own words. Short genuine
  quotes only (under ~15 words, and only if visible verbatim on the page); never fabricate quotes,
  letters, or anecdotes; paraphrase inflammatory rhetoric in neutral terms. Digest mode
  simultaneously (a) avoids verbatim reproduction of copyrighted text and (b) keeps output within
  the token budget.

The mode used is recorded per document as the `extraction_method`.

**Modes govern the agent-read layer only.** The D2 sibling is **always verbatim and never carries
an extraction mode** — it is a transcription of the page, so there is nothing for a mode to select.
A document extracted in digest mode still gets a verbatim sibling; that is the point of the
Hybrid, and it is exactly why the sibling matters: for a digest-mode corpus, the sibling is the
only place the authors' actual words are preserved.

**Output-token discipline.** Agent turns have a hard per-turn output-token cap (observed at
roughly 32,000 tokens on the reference stack), and a verbose multi-chunk extraction can exceed it
and fail outright. Processors therefore keep total output per turn **under ~18,000 tokens** and
**write incrementally to disk, chunk by chunk**, never holding a whole batch in memory. If a range
is too dense to fit, the range is narrowed — output is never silently truncated.

---

## 7. The command family

HDARP is driven by a family of four parallel commands. The naming is compositional: **P** = run
processors in parallel, **S** = smart (document-aware) batching, **H** = hybrid — body text is
read **twice**, the agent-read layer plus the mandatory Sraffa 4.0 verbatim sibling. Read by
function:

| Command | Smart batching | Hybrid (body text) | Extracts |
|---------|:---:|:---:|----------|
| **pdarp**  | no  | no  | Tables, Equations, Figures |
| **spdarp** | yes | no  | Tables, Equations, Figures |
| **phdarp** | no  | yes | Tables, Equations, Figures, **Body text (D1 + D2)** |
| **sphdarp**| yes | yes | Tables, Equations, Figures, **Body text (D1 + D2)** |

The non-hybrid variants (`pdarp` / `spdarp`) deliberately skip body text entirely — both layers —
which is useful when you only need the structured layer (tables/equations/figures). The hybrid
variants (`phdarp` / `sphdarp`) do the full four-type extraction. `sphdarp` is the everyday
workhorse.

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
Sraffa engine on its own, in two distinct roles — a per-page rescue mode for pages that exhausted
the content-filter ladder, and the whole-document **augment** pass that produces the Stage-5
sibling; and a **cleanup** step that removes chunk intermediates once a document is verified and
frees disk.

A typical lifecycle:

```
prepare → extraction (sphdarp or family) → enrich (if issues) → wrap-up → sraffa-ocr --augment (Stage 5) → cleanup → integrate
```

The final integration stage records each document's extraction mode and sibling completeness, and
excludes underscore-prefixed sibling trees from document enumeration so `_OCR_Only/` is never
miscounted as a document.

---

## 8. Integrity rules

HDARP's value depends on never lying about what it read. Two hard rules enforce that, both written
after real production incidents:

**Anti-Silent-Degradation.** If agent extraction fails (content filter, timeout, error), the
orchestrator must **stop and report** — it must never quietly fall back to a raw embedded-text dump
and pass that off as a complete extraction. A text dump is missing ~75% of what HDARP exists to
capture. The validator specifically checks for chunk-boundary markers in body text precisely because
those markers are absent from a raw text dump, so silent downgrades get caught. If a text dump is
ever used for triage, it is labelled as such, the batch stays "prepared," and no chunk files are
deleted. The v6.4 augmentation clause (§4) does not weaken this rule by one word: it permits OCR
only *alongside* an extraction that is already complete — never *in place of* one.

**Never Fabricate.** If extraction of a passage fails, the agent records the gap and continues with
the rest. It must **never** generate substitute content, paraphrase from memory, or invent quotes,
letters, or citations to "save" a chunk. A visible gap marker is always preferable to an undetected
fabrication — fabricated source material corrupts every downstream citation built on it. Uncertain
passages are flagged (`[approximate]` / `[paraphrased]`); illegible ones get an `[unreadable]`
marker.

---

## 9. The Scrounger Standard

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

A missing Stage-5 OCR sibling is not a scrounger outcome either. The sibling is *expected* on every
document, so its absence is a completeness defect, not a deferral reason — and it can never justify
accepting a substitute for the agent layer. Conversely, a present sibling never excuses an
incomplete agent extraction. Stage 5 runs after the retry ladder has finished, over the whole
document, and cannot change any extract-or-defer decision the ladder makes.

---

## 10. What's new in v6.4 — Hybrid Coherence

v6.4 ("Hybrid Coherence", released 2026-08-27) is a **restoration of what the protocol already
said**, not a new feature. The H in HDARP has always meant agent reading **plus** a verbatim OCR
pass; what drifted was the OCR stage's *status*, not its existence. The release inherits every
v6.3 mandate (2026-06-13) and, through it, v6.2's (2026-05-29). The **Sraffa engine stays 4.0** —
the engine did not change; its place in the lifecycle did.

**Why a restoration was needed.** Measured on a production corpus in August 2026: the OCR stage
had come to be treated as the content-filter fallback, so in one wave it ran on only **5 of 143
documents**. And because roughly nine in ten documents of that corpus had been extracted in
analytical-digest mode — a deliberate, legitimate paraphrase, not a defect — **the verbatim body
text of most of the corpus existed nowhere**. A reader could not quote the author, check a claim
against the page, or reconstruct the document. A line-by-line review also found seven places where
the command and rule files contradicted each other about the OCR stage's role; v6.4 reconciles
them.

**What v6.4 changes:**

1. **Content type D splits into D1 / D2** (§4) — nothing may stand in for the agent-read layer,
   and a document is unfinished without its verbatim sibling.
2. **The augmentation clause** — Hybrid is augmentation, never substitution, and it does not
   weaken the anti-silent-degradation rule by one word.
3. **`/sraffa-ocr --augment` (Mode 4)** — a mode that processes **every page with no skips**. The
   per-page rescue mode and the augment pass are different jobs and are never substituted for each
   other.
4. **Hybrid Stage 5 is mandatory** — the "(if needed)" qualifier is gone from the lifecycle, and
   closeout fails a document that carries neither a sibling nor a recorded reason for lacking one.
5. **Integration guards** — the integration stage excludes underscore-prefixed sibling trees from
   document counts, and records each document's extraction mode and sibling completeness, which
   re-arms the check that flags digest-mode documents lacking a verbatim layer.
6. **The weave format (`weave-1.0`)** — the join format for the two layers (§4): anchored on page
   identity, never character offsets; disagreements recorded, never auto-resolved.

---

## 11. What v6.3 added — Native Enrichment

v6.3 (2026-06-13) was an **additive** release: every prior rule stayed in force, and the four-type
completeness gate was unchanged. What it added was **capturing structured metadata about each
table at read time** — a mandate that v6.4 inherits unchanged.

The motivation: a downstream database-building stage used to re-open every extracted table just to
recover metadata — its title, page, units, footnotes, period and geographic coverage, and a
quality assessment — that the extracting agent *already had in front of it* while reading the page.
That was wasted compute. v6.3 captured it once, during the initial read.

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
incomplete sidecar line is a **warning-level** deduction (roughly −2 of 27 points) recorded in the
validation notes, never a hard failure. This means in-flight and legacy campaigns are never
retroactively failed — they can simply be backfilled later. The metadata "rides on" the Tables type
and never changes whether a chunk is considered extracted.

---

## 12. Quality assurance

Each batch is scored by the validator on a **27-point rubric**: tables (10), equations (5), figures
(5), body text (5), and OCR confidence (2). Thresholds:

- **Reject** below 22/27 (under ~80%)
- **Acceptable** at 22–25/27
- **Target** at 26/27 or above (~96%)

The validator reads all four output types for every chunk, confirms the chunk-boundary markers that
catch silent text dumps, cross-checks OCR routing statistics, verifies that the extraction state and
the catalog agree, and writes a per-batch verdict with remediation hints. As of v6.3 it also runs
the warning-level native-metadata check described above. The rubric scores the **agent-read layer
(D1)**; the verbatim sibling is checked for *presence* at closeout (the missing-sibling gate)
rather than scored against the agent extraction.

---

## 13. Roles and boundaries

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
figures (described in Markdown), and body text — by combining **agent vision** for structure with
**document-adaptive OCR** for prose, targeting 95–98% accuracy across all four. Body text is
**two layers**: the agent-read layer (D1) plus a mandatory verbatim OCR sibling (D2), produced by a
no-skips end-of-run pass (`/sraffa-ocr --augment`, Hybrid Stage 5) into a separate `_OCR_Only/`
tree — **augmentation, never substitution** — and joined by the **weave-1.0** format without being
merged. Large files are **mandatorily chunked** (>10 pages or >1 MB) and processed one chunk at a
time. A family of four commands (`pdarp` / `spdarp` / `phdarp` / `sphdarp`) covers structured-only
vs. full-hybrid extraction, with parallel processors, a strongest-model validator, and a 27-point
quality gate. The **Scrounger Standard** permits only three honest reasons to skip content —
*duplicate*, *content-filter-exhausted*, or *physically unreadable* — and hard rules forbid silent
degradation and fabrication. **v6.4 ("Hybrid Coherence", 2026-08-27)** is a restoration release: it
restores the two-layer body text, makes Stage 5 mandatory, and adds weave-1.0, while inheriting
every prior mandate — **v6.3** had added **native enrichment** (per-table metadata captured at read
time into a sidecar), which remains in force unchanged.
