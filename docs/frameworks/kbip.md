# KBIP — The Knowledge-Base Integration Pipeline (v1.0)

**Version:** 1.0 · **Status:** in production · **Layer:** integration / storage / database on-ramp

KBIP is the **method-agnostic** layer that takes raw knowledge-base extractions of source
documents (PDFs, scans, books) and integrates them into a project's permanent storage and
its research database — **regardless of how each document was read**. Its central idea is a
*read-method spine*: every document records, in a self-declaring and machine-readable way,
which extraction engine produced it and how much each transcribed value can be trusted. Two
very different reading methods — a cloud language-model agent and an offline local vision
model — feed the **same** downstream pipeline because KBIP normalizes their output to one
shape and keys everything by method.

---

## Why method-agnosticism matters

A serious research corpus is rarely read one way. Some documents are born-digital and parse
cleanly; some are poor scans that need optical character recognition; some are dense tables
or charts that only a specialized model can recover. Over time the *tools* also change. If
the storage and database layers were welded to one extraction engine, then:

- **Provenance would be lost.** You could not later answer "how was this figure obtained,
  and how reliable is it?" — which is the difference between a usable research database and a
  pile of numbers.
- **Reproducibility would break.** Re-running or auditing a result requires knowing the exact
  engine, version, and models that produced each value.
- **Mixed corpora could not be compared.** A document read by one method and a document read
  by another could not sit side-by-side in the same database without ambiguity.

KBIP solves this by treating the **reading method as first-class data** that travels with the
document through every layer, rather than as an implementation detail that gets discarded once
the text is extracted.

---

## The read-method spine

KBIP records one key — `read_method` — in three places, with the **same value**, joined by the
MD5 hash of the source file. The three surfaces are deliberately redundant so the question
"how was this read?" can always be answered, whether you have the document folder, the project
ledger, or only the raw body text in front of you.

The frozen `read_method` vocabulary (v1.0) covers both families of engine:

| `read_method` | Engine family | What it is | Native metadata |
|---|---|---|---|
| `HDARP` (and variants) | Cloud agent reading | A language-model agent reads the document via a Read tool: one model extracts, a second validates. | Newer versions emit a per-table metadata sidecar; older ones do not. |
| `Hopper` | Offline local vision model | A pipeline of specialized vision/OCR models running on a local GPU (structural OCR, faithful-script OCR, escalation, chart-to-data, captioning). | Always emits a metadata sidecar plus a per-block/per-page confidence file. |
| `manual` | Human transcription | Direct human entry. | As recorded. |

> **A naming rule worth stating plainly:** these engines are genuinely distinct and are never
> conflated. They happen to produce an **identical four-artifact shape** — body text, tables,
> equations, figures — which is exactly what lets one downstream pipeline treat them uniformly.

### The three self-declaration surfaces (per document)

1. **A per-document tag file** (`READ_METHOD.json`) — a small schema-versioned record holding
   `read_method`, the engine name, the engine version, the extraction method, the models used,
   whether native enrichment is present, the source-file hash, and the read date. This answers
   the question **from the document folder alone**.
2. **The document manifest** carries the same `read_method` / `read_version` fields.
3. **A header comment** inside each body-text file (e.g. `<!-- read_method: ... -->`), so even a
   plain text search over the corpus reveals how a document was read.

### The `source_md5` join key

The MD5 of the source file threads through every layer, so the same question can also be answered
**from the ledger alone**, and so two engines can be joined side-by-side:

```
source file hash
  → processing log   (records the engine / process type)
  → KB catalog       (record of the stored extraction + counts)
  → provenance ledger(records the processing protocol + metadata sidecar)
  → KBIP document audit (read_method, schema version)
  → research database (the document's process_type)
```

The governing invariant: the engine named in the database equals the engine in the processing
log equals the engine in the provenance ledger equals the `read_method` in the per-document tag.
A mixed-engine knowledge base therefore sorts and filters cleanly by reading method.

---

## The two-axis honesty model

KBIP carries an explicit, conservative model of data quality that rides on the **tables**
artifact. Every extracted table gets one metadata line recording two independent axes:

- **`obs_status`** — the quality of the *source datum itself* (using the SDMX observation-status
  convention). This is left unset at integration time and assigned later, during enrichment, so
  it is never assumed.
- **`transcription_status`** — how faithfully the value was *transcribed* from the page, on a
  five-level scale (verified / high / low / review-needed / unreadable).

Each field also records a **`field_basis`**: was it taken mechanically, taken directly from the
source text, inferred by an agent, or **not captured at all**. Crucially, "not captured" is a
**first-class value** — an honest absence is recorded as such and never invented.

The offline-vision engine has a provenance advantage here: it derives `transcription_status`
**mechanically** from its own confidence sidecar, via a frozen, deliberately conservative map
(born-digital text scores highest; high-confidence reads are trusted; mid-confidence or escalated
reads are downgraded; low-confidence reads in the review queue are flagged; illegible blocks are
marked unreadable; chart-estimated values are downgraded). The map never assigns the top
"verified" grade automatically — that can only be earned later by audit. The cloud-agent engine's
source-datum quality is likewise set during a separate enrichment step, never defaulted.

---

## Where KBIP sits: upstream of the database build

KBIP is an **integration and on-ramp** layer. It does **not** read documents (the extraction
engines do that) and it does **not** build the final research database (a separate database
framework does that). It is the connective tissue in between:

```
  EXTRACTION ENGINES                KBIP (this layer)                 DATABASE BUILD
  ─────────────────                 ─────────────────                 ──────────────
  cloud-agent reading   ─┐                                         ┌─ harvest tables
  offline-vision reading ─┼──►  LAND   the extraction into the     │  (reads the metadata
  human transcription   ─┘      project knowledge base             │   sidecars natively,
                                TAG    each doc with read_method    │   pre-graded by the
                                CATALOG into method-tagged catalogs │   transcription axis)
                                LEDGER write provenance rows        ├─ enrich / organize
                                SAFE-PACKAGE a dated, gated backup  ├─ audit (honesty gate)
                                          │                         └─ publish
                                          └──────────────────────────►
```

### The integration stages

1. **LAND** — move a finished extraction from the working area into the project's permanent
   knowledge base in a consistent folder layout.
2. **TAG** — write the three self-declaration surfaces so each document declares its reading
   method, engine version, and models used.
3. **CATALOG** — record every document, table, equation, and figure in **method-tagged catalogs**
   (a single superset schema, so engine-specific columns are simply blank where they do not
   apply, and `read_method` is always present).
4. **LEDGER** — write provenance rows into the project's processing log, knowledge-base catalog,
   and provenance ledger, all keyed by the source-file hash and written idempotently (re-running
   produces no duplicates).
5. **SAFE-PACKAGE** — build a dated backup archive and run an integrity gate (all four artifacts
   present, body-text chunk markers present, metadata schema valid, self-declaration present,
   provenance complete, nothing left pending). The archive is only combined into the canonical
   store **on a passing gate**, and the prior state is archived rather than deleted. A
   deliberately broken document fails the gate and blocks the merge.

Only after KBIP finishes does the **database build** run. Because KBIP has already tagged every
document and pre-graded the transcription axis, the database harvester ingests the metadata
sidecars **natively**: a project read entirely by the offline-vision engine lands with its
documents marked as such and its transcription grades pre-populated, **with no change to the
database schema** versus a project read by the cloud agent. The two engines differ only in a
`read_method` column — everything downstream is identical.

---

## Worked proof: a mixed corpus

KBIP v1.0 was proven end-to-end on a 305-document knowledge base read entirely by the
offline-vision engine. Every document received all three self-declaration surfaces; thousands of
per-table metadata lines validated cleanly against the database's schema; provenance rows landed
in all three ledgers; and the database round-trip succeeded with the transcription grades
pre-populated from the confidence sidecars and the audit gate passing — **with no schema change**.
A unified scratch view then placed those 305 offline-vision documents and roughly 1,700
cloud-agent documents **side-by-side**, each correctly tagged and joinable by source-file hash,
sortable by reading method. That side-by-side join is the whole point: it is what
method-agnosticism buys you.

---

## Summary

KBIP is the layer that makes a knowledge base **honest about its own origins**. By recording the
reading method as first-class, self-declaring, hash-joined data — and by normalizing every
engine's output to one four-artifact shape with a two-axis quality model — it lets documents read
by completely different methods flow into the same per-project storage and the same research
database, while preserving the provenance needed for reproducibility and audit. It sits squarely
between the extraction engines and the database build, owning the LAND → TAG → CATALOG → LEDGER →
SAFE-PACKAGE on-ramp, and it is engineered so that adding a new reading engine later requires
nothing downstream to change.
