# AUKA — The Arcanum Unified Knowledge Architecture

**Version 1.1** · A unified architecture for documents, extractions, and provenance

AUKA is the way a research program keeps *all* of its source documents and the
knowledge it reads out of them in **one honest, canonical place** — instead of
scattering them across many overlapping project folders. This document explains
the architecture conceptually, for an external researcher who wants to understand
how the pieces fit together. It is not an operations manual.

---

## The problem AUKA solves

A long-running research program accumulates documents the hard way. Over years it
spawns many sub-projects, each with its own collection of PDFs and its own folder
of extracted text, tables, and figures. Left unmanaged, this produces three chronic
failures:

1. **Duplication.** The same book or paper is downloaded, stored, and re-read under
   several projects. Disk fills with near-identical copies, and there is no single
   answer to "do we already have this?"
2. **Cross-project bleed.** When the same document lives in two places, edits,
   re-extractions, and corrections diverge. Two projects can hold two different
   "truths" about the same source.
3. **Broken provenance.** Once copies multiply, it becomes impossible to say with
   confidence *which* file a given extraction came from, when it was read, or how it
   was processed. The chain of custody from original document to derived claim is lost.

AUKA's premise is that these are not three problems but one: **there is no single
canonical store and no single provenance spine.** Fix that, and the rest follows.

---

## The core idea, in one line

> **One store. Two axes. Two query layers. One ledger.**

Everything in AUKA is an elaboration of that sentence.

### One store

Every source document — every PDF — and every extraction read out of it lives in
**one canonical library**, called the **Robert** library. There is exactly one
authoritative copy of each document, deduplicated by content hash (MD5). No other
project keeps its own copy of a source PDF; they all point at Robert.

The physical layout is the **Robert PDF Library v2.0** standard:

- **Project-flat folders.** Each project has a single flat folder holding the
  canonical copies of its documents — no nested subdirectories, one file per
  document, deduplicated by hash.
- **A content-type "view" by symbolic link.** A parallel index groups the same
  files by *what they are* — classical texts, modern theory, empirical replications,
  methodology handbooks, regulatory filings, dissertations, lectures, and so on.
  This grouping costs no extra disk: it is made of symbolic links that point back
  into the project-flat folders. A document appears in exactly one canonical
  location and is *referenced* from wherever else it logically belongs.

This is what makes "one store" practical: a single deduplicated copy on disk, but
many ways to look at it.

### Two axes

The store is organized along two independent axes simultaneously:

- **Axis 1 — provenance.** *Which project does this document belong to?* This is the
  flat project folder. It answers "where did this come from."
- **Axis 2 — taxonomy.** *What kind of thing is this, in a single comprehensive,
  non-overlapping classification of the whole library?* This is the content-type
  index — the "Unified Library" view. It answers "what is this, library-wide."

Axis 2 is strictly an **index over** Axis 1. The taxonomy never holds its own
copies; it only re-points at the canonical files. A reader can therefore traverse
the entire collection as one coherent whole, by subject, without any document ever
being stored twice.

### Two query layers

On top of the single store sit **two thin query/index layers**. Crucially, neither
of them holds any documents or raw extractions of its own — they are *views* that
point into the store:

- **A theory layer** ("what we think") — a catalog/registry of the program's
  theoretical commitments and the texts that ground them.
- **A methods layer** ("how we know") — a catalog/registry of the program's
  methodological sources and techniques.

Before AUKA, these were separate knowledge bases with their own PDFs and their own
extractions — a guaranteed source of duplication and drift. Under AUKA they become
**lenses**: each is a registry that resolves to documents and reads already sitting
in the one canonical store. They curate and cross-reference; they never copy.

### One ledger

Binding everything together is a single **provenance ledger** — one append-only
record that ties each document to its content hash, to the extraction campaign and
batch that read it, to the resulting knowledge-base artifacts, to its documentation,
to its safe backup copy, and to its identifier in the library-wide taxonomy.

The ledger is the **single source of truth that nothing is orphaned.** Every PDF can
be traced forward to everything derived from it, and every derived artifact can be
traced back to exactly one source document. This is the "one honest provenance
spine" the whole architecture exists to provide.

---

## How a document is named

To keep a single library legible across very different acquisition routes, AUKA
reconciles **two naming conventions**, both anchored to the document's content hash
so they can always be tied back together:

- **Tracked acquisitions** — documents pulled deliberately against a wishlist carry
  a structured name encoding their wishlist identifier, an abbreviated title, and the
  acquisition channel. These are the documents the program went looking for on purpose.
- **Bulk library** — documents acquired in bulk are given human-readable names of the
  form `[Year] Author - Title`, produced by an automated naming pass. Collisions are
  disambiguated with a short hash suffix.

Both forms coexist inside the same project folder. The content hash, recorded in the
unified catalog, is the authoritative cross-reference between them — so the *name* can
vary while *identity* never does.

---

## Why centralize instead of federate?

A reasonable alternative is to leave documents where they are and merely *index* them
in place. AUKA rejects this for one reason: **provenance integrity is only as strong
as the guarantee of a single canonical copy.** If a document can exist in two places,
then sooner or later the two copies — and the extractions taken from them — disagree,
and no index can adjudicate which is correct. By collapsing to one physical copy per
document, deduplicated by hash, AUKA makes the provenance ledger *enforceable* rather
than aspirational. The theory and methods layers lose nothing: they still see the
whole collection, through views, with full freedom to organize and annotate.

---

## Migration: by cohort, gated on completion

A program does not move years of accumulated material into a new architecture in one
step. AUKA migrates in **cohorts**, and the cohorts are **gated**:

- **Per-project eligibility.** A project becomes migration-eligible only when *its own*
  extraction work is complete and verified. A project still actively being read is left
  in place until its reads settle — moving it mid-flight would fork the very provenance
  AUKA is trying to protect.
- **First cohort = the cleanest, largest, already-finished projects.** These act as the
  proving run for the migration mechanics and establish the canonical layout.
- **Later cohorts wait their turn.** Projects with in-flight extraction campaigns are
  deferred to a subsequent cohort and synced once their reads reach the verified state.
- **New work goes straight into the new structure.** Projects that have not yet started
  acquiring or reading do not need migrating at all — once AUKA is in place, they are
  built natively inside it.

This sequencing means the architecture is adopted **without ever destabilizing
in-progress research.** At each step, only settled material moves; everything still in
motion keeps its existing home until it, too, is finished.

In the v1.1 realization of this plan, the first cohort consolidated the bulk of the
program's projects into the canonical store, while a handful of projects with active
extraction campaigns remained explicitly deferred to a later cohort — each tagged in the
project registry with whether the canonical PDF sync applies to it yet.

---

## What changed in v1.1

Version 1.0 established the architecture: one store, two axes, two query layers, one
ledger. **Version 1.1** made the canonical store concrete by adopting the **Robert PDF
Library v2.0** layout (project-flat folders plus the zero-cost content-type symlink
view) as *the* canonical store, standardizing the document lifecycle — acquire →
canonicalize-into-store → (optionally) normalize names — and extending the project
registry so every project records which naming convention it follows and whether it has
been synced into the canonical store yet.

---

## The mental model to keep

If you remember nothing else: **AUKA is a single, deduplicated, hash-anchored library
of source documents and their extractions, organized two ways at once (by origin and by
subject), with thin theory and methods catalogs layered on top as views, and one ledger
that can trace any artifact back to exactly one source.** The architecture's whole
purpose is to make that traceability *true* — not merely claimed — for an entire,
long-lived body of research.
