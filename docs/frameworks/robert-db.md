# The Robert Database Framework

**Version 1.0**

The Robert Database Framework turns a project's pile of document extractions into a
**research-grade, queryable database** — one database per project, with honest
per-field provenance and an explicit, machine-readable record of *how much you can
trust each value*.

---

## 1. The problem it solves

Reading a corpus of PDFs (historical statistical yearbooks, central-bank reports,
scholarly monographs) with an AI extraction pipeline produces, per document, a set of
artifacts: full body text, a folder of table CSVs, equations, and figure
descriptions. That output is *complete*, but it is not *data you can query*:

- the tables are loose CSVs written in many different transcription conventions;
- their titles, source page numbers, units, and footnotes are unevenly captured;
- there is no structured, honest record of **how confident the transcriber was that
  the digitized value matches the printed page**.

A researcher who wants to cite a specific historical table — and know whether its page
reference was read off the source or merely inferred — cannot do that from raw
extraction output.

The Robert Database Framework closes the gap. It promotes a project's extraction
output into a single canonical database with:

- a canonical **SQLite** store per project, plus regenerated views (CSV / Parquet);
- **honest per-field provenance** — every metadata field records *how* its value was
  obtained;
- a **two-axis quality model** that separates *source data quality* from *reading
  fidelity*;
- a ratified **taxonomy** and cross-document **concordances**;
- publishable, leak-scrubbed release packages in community data standards.

Two principles govern everything:

> **`NOT CAPTURED` is a first-class value** — an honest "we assessed this and could not
> recover it" is recorded explicitly, never silently left blank.
>
> **The source page is the authority** — a value read directly from the page always
> wins over anything inferred.

---

## 2. Architecture at a glance

There is **one database per project**, and no cross-project content ever lands in a
project's database — zero bleed by design.

```
<project>/RobertDB/
  robertdb_config.json     # the only project-specific input to the engine
  robertdb.sqlite          # CANONICAL store
  views/                   # regenerated CSV / Parquet + a manifest (derived, never hand-edited)
  state/                   # build state + run ledger + taxonomy proposals
  enrichment/patches/      # validated JSONL metadata patches
  audit/                   # the audit report
  publish/                 # release bundles
```

The engine itself is generic: a single shared copy of the processing scripts, with no
project name or project-specific logic baked in. The per-project **config file** is the
only project-specific input.

### The provenance spine

Every queryable cell traces back to a source PDF through a fixed chain of custody:

```
pdf_md5  →  doc_id  →  table_uid  →  (columns, field provenance, quality)
```

- **`pdf_md5`** identifies the exact source file by content hash. If it cannot be
  resolved, it is stored as the literal `NOT_CAPTURED` (never a silent blank) and
  flagged.
- **`doc_id`** is the document's folder name.
- **`table_uid`** is an immutable per-table identifier minted once and never
  renumbered, even when a project is re-harvested.

### SQLite is canonical; views are regenerated

`robertdb.sqlite` is the single source of truth. The `views/` directory (CSV, and
Parquet where configured) and its manifest are **regenerated from the database** after
every stage that changes data. Views are never hand-edited. To change a published
value you change the database — by re-harvesting or merging a reviewed patch — and
regenerate the views.

Core tables cover documents, the extracted tables, per-field provenance, columns, the
taxonomy, concordances, quality flags, named entities, and a ledger in which every
processing run records itself.

---

## 3. The two-axis quality model

The framework's central methodological commitment is that **the quality of the
underlying historical datum and the fidelity of our reading of it are two different
things** — and both must be machine-readable for every table. Established macrohistory
databases record source caveats; what they do *not* record, in a structured field, is
how confident the transcriber was that the digitized value matches the page. That
second axis is this framework's distinctive contribution.

### Axis 1 — source data quality (`obs_status`)

This axis reuses the **SDMX `CL_OBS_STATUS`** code list, the same vocabulary official
statistics agencies use, so the source-quality axis stays interoperable with existing
tooling. The dominant code for a table is stored on the table. Codes include:

| Code | Meaning (source-side) |
|------|-----------------------|
| `A`  | Normal value |
| `B`  | Time-series break |
| `D`  | Definition differs / dubious |
| `E`  | Estimated value |
| `P`  | Provisional value |
| `U`  | Low reliability |
| `I`  | Imputed value |
| `O`  | Missing value |
| `M`  | Not applicable |
| `L`  | Missing — data exist but were not collected |

### Axis 2 — reading fidelity (`transcription_status`)

A custom five-value vocabulary recording how confident we are that the *digitized*
table matches the *source page* — independent of whether the source datum was any
good:

| Code | Meaning (reading-side) |
|------|------------------------|
| `V`  | **Verified** against the source page (spot-checked, matches) |
| `H`  | **High-confidence** read (clear scan, unambiguous) |
| `L`  | **Low-confidence** read (poor scan, ambiguous glyphs or numerals) |
| `R`  | **Reconstructed** (structure or values inferred from a damaged/partial source) |
| `X`  | **Illegible** (could not be read; recorded as such, never guessed) |

The two axes are orthogonal. A table can have a clean source datum (`obs_status = A`)
that we are unsure we read correctly (`transcription_status = L`), or vice versa.
Keeping them separate is exactly what lets a downstream researcher filter on *either*
"the source flagged this as estimated" *or* "we are not confident we transcribed it
faithfully."

---

## 4. Per-field provenance and the no-downgrade rule

Beyond the two quality axes, the framework records **one provenance row per metadata
field per table** — capturing the *basis* on which each field (title, page, units,
footnotes, and so on) got its value:

| Basis | Meaning |
|-------|---------|
| `mechanical`    | Derived by the harvester from the CSV's own structure |
| `from_source`   | Visible verbatim in the extracted text |
| `agent_inferred`| Reasoned from context (translations, inferred units, etc.) |
| `not_captured`  | Assessed and **not recoverable** — a first-class honest negative |
| `unknown`       | Not yet assessed |

These bases are ranked, and a **no-downgrade rule** is enforced when metadata is
merged: a `from_source` value can never be overwritten by an inferred or mechanical
one; only `not_captured` / `unknown` fields may be upgraded. This makes metadata
recovery **monotonic** — an automated agent can only ever *improve* a field's
provenance, never silently replace a source-verified value with a guess.

---

## 5. Quality grade

Each table receives a deterministic `confidence_score` (0–100) and a letter
`quality_grade` (A / B / C / D), computed from recorded attributes — document quality,
extraction vintage and method, scan complexity, parse cleanliness, and any
verification adjustments. The calibration intent:

- an unverified clean read of a validated extraction lands around **C**;
- ragged or unvalidated reads land **D**;
- high-fidelity verbatim extractions land **B**;
- **A requires verification against the source** (a spot-check or human confirmation).

A grade of A is therefore a claim about *checked* fidelity — never about model
confidence alone. The score is **deterministic and reproducible** from the
configuration plus the row's attributes (not an opaque AI judgment), so recalibrating
the weights replays identically and is logged as a new run.

---

## 6. Identifiers

Per project, the framework mints typed, project-prefixed identifiers:

| Kind | Shape | What it names |
|------|-------|---------------|
| Table       | `<PROJ>-T-NNNNNN` | one extracted table (immutable) |
| Concordance | `<PROJ>-K-NNNN`   | a family of related tables across documents |
| Taxonomy term | `<PROJ>-X-NNNN` | a node in the project taxonomy |
| Row-set     | `<PROJ>-RS-NNNN`  | reserved for future row-level groupings |

Because every identifier is project-prefixed and carries a typed infix, these
namespaces are collision-proof against the constructed-series identifiers used by
sibling frameworks — so the same underlying PDF can be referenced from both layers
with no ambiguity.

---

## 7. The pipeline

The build runs in six stages, orchestrated by a `build` step that is resumable from a
saved build-state file:

| # | Stage | What it does |
|---|-------|--------------|
| 1 | **init**     | Scaffold the workspace and create the database from the config. The only stage that creates the database. |
| 2 | **harvest**  | Mechanically ingest table CSVs: sniff conventions, mint immutable table identifiers, parse rows and columns, record mechanical field provenance, then score quality and regenerate views. |
| 3 | **enrich**   | Recover honest table metadata (titles, pages, units, footnotes, two-axis quality) that the mechanical pass could not. |
| 4 | **organize** | Build the project taxonomy and the cross-document concordances. |
| 5 | **audit**    | Run the QA gate and a stratified spot-check; a passing audit is required before publishing. |
| 6 | **publish**  | Package a leak-scrubbed release and mirror it to its destination. |

Enrichment and organization both depend only on harvest and may interleave; the audit
strictly gates publication.

### Enrichment subagents never touch the database

A key safety property: the agents that recover metadata in the **enrich** stage **do
not write to the database**. Instead, each emits a **validated JSONL patch** — one
record per table, declaring every field's value *and its provenance basis*. A separate
merge step validates each patch against the patch contract and applies it under the
no-downgrade rule, rejecting anything malformed into a side file. The database is only
ever mutated by trusted engine code, never directly by a recovery agent. This is what
makes large, parallel metadata recovery safe: the worst a misbehaving agent can do is
have its patch rejected.

### Captured at read time where possible

When the upstream extraction pipeline supports it, the honest metadata (titles, pages,
units, two-axis quality) is captured **at the moment of reading** and emitted alongside
the extraction as a sidecar. The harvest stage ingests that sidecar directly, so the
database lands already carrying honest provenance — and the enrich stage collapses to a
lightweight verification pass. Extractions without a sidecar take the full recovery
pass unchanged. The framework is also extraction-engine-agnostic: it records which
reading method produced each document, so corpora built by different engines coexist in
one database with no schema change.

---

## 8. Publishing

The publish stage emits a **Frictionless Data Package v2** release directory — a
self-describing bundle of typed CSV resources with an accompanying set of standard
metadata files:

| Artifact | Why it's included |
|----------|-------------------|
| **Data Package v2** descriptor + resource schemas | De-facto standard for self-describing tabular data |
| **Croissant** dataset metadata | Makes the corpus discoverable to machine-learning tooling |
| **`CITATION.cff`** | Machine-readable citation metadata |
| **`llms.txt`** | A machine-readable corpus summary for LLM consumers |
| **Web manifest** | The website-export contract (resource list, columns, types) |
| **Codebook** | A field-by-field guide, including the two-axis quality vocabulary |

Some heavier standards (full SDMX messaging, catalog-of-catalogs vocabularies, and
per-table DOIs) are deliberately out of scope for v1.0 — the **corpus** is the citable
unit, and a single package describes one corpus.

**Leak scrubbing** is severity-graded. For a corpus intended for public release, any
workspace path, secret, or internal reference is a **publish-aborting failure**; for an
internal corpus it is a warning. The mirror step always verifies that the bytes
actually landed at the destination rather than trusting a success message.

---

## 9. Where it sits

The Robert Database Framework runs **downstream** of document extraction and
integration: extraction produces the per-document knowledge base, and Robert promotes
that knowledge base into a queryable, citable, publishable per-project database. It is a
distinct framework with its own contracts — a config spec, a schema, and a patch
contract — and it never alters the upstream extraction output it reads from.

---

*Robert Database Framework v1.0 — established 2026-06-11.*
