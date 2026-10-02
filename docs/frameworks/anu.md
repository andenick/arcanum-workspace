# The Anu Framework

*A pipeline for constructing research-grade economic data series with full provenance.*

**Framework version:** v12.4 (the "rebuild-programme" release, 2026-09-23; v12.3
"enforcement", 2026-09-05; v12.2 "web-readiness", June 2026)

---

## What problem does Anu solve?

Empirical economics runs on data series — a rate of profit measured back to the
nineteenth century, a wage-to-productivity ratio, a measure of the wealth of
nations. When such a series appears in a book or a study, reproducing it is
usually hard: the original construction logic is buried in footnotes, the
sources are described informally, units are implicit, and any extension to the
present day silently introduces substitutions that nobody documents.

The **Anu Framework** is a structured, multi-stage process for turning source
materials (books, papers, statistical-agency releases) into **research-grade
data series that anyone can reproduce and audit**. Its defining commitment is
*honest provenance*: every number in every output traces back to a published
source or a documented analytical method, and the framework refuses to fabricate
a value when the real one cannot be found.

Anu is organized as a family of cooperating **skills** — small, versioned units
of process, each responsible for one job — driven by a single orchestrator. The
current release is a **19-active-skill framework (plus 2 deprecated redirect stubs = 21 skill folders)**
spanning the full lifecycle from initial research to public distribution.

> **Note on scope:** this document describes the Anu Framework; the individual `anu-*` skill
> definition files are **not part of this export**. What is published here is the framework's
> design, its conventions and its output formats — enough to re-implement the approach, not a
> drop-in copy of our skill tree.

---

## The single source of truth: `series_registry.json`

At the centre of every Anu project sits one file: **`series_registry.json`**.

This registry is the canonical description of every data series in the project —
its identity, its units, its content type, its sources, its construction method,
and its publication status. The cardinal rule of the framework is that **every
output format reads from the registry**. The machine-readable CSVs, the Excel
workbooks, and the interactive visualizations are all generated *from the*
registry; nothing is allowed to bypass it. If a value is not described in the
registry, it does not appear in any output.

Each series record carries, among other fields:

- a stable **series ID** and a public **`display_name`** (identical across the
  website, the downloads, and any API, so a series is named the same everywhere);
- **`units`** (e.g. `billions_usd`, `index_2017=100`, `percent`, `ratio`) — a
  mandatory field, with per-subseries units where a series has components;
- a **`content_type`** classification (see below);
- a **`construction`** descriptor (`direct`, `formula`, or `composite`);
- a **`publish`** flag and a **`triage`** verdict governing whether the series is
  released publicly.

Since schema version 2.4.0 the registry also supports **multi-arm series**. A
series that has been rebuilt — carried forward to modern data, reconstructed
from a better source, or superseded — may carry several parallel *arms* (book,
current, extension, combined, reconstruction, legacy, variant), each addressed
by a hyphen-token subseries identifier and classified by a `role` and a
`role_class`. **Seams** record where two arms join; the series' `period` is a
finite span measured in both directions; and a **purge ledger** reserves the
identifiers of withdrawn arms forever, so a retired ID can never be silently
reused.

Because the registry is the one authoritative artifact, a project's internal
consistency can be checked mechanically, and multiple agents (or people) can hand
work off to one another without losing track of state.

---

## The pipeline

Anu organizes construction as a numbered pipeline. Each stage has a dedicated
skill (or skills) and produces well-defined artifacts that the next stage
consumes.

### Stage 1 — Research

Mine the source material for everything relevant to a given series: quotations,
references, footnotes, methodology descriptions, benchmark figures. The output is
a per-series research record. This is the evidentiary base for every later
decision; nothing downstream is allowed to contradict it. Since v12.4 the
quotation contract is strict: a recorded quote is **verbatim only**, and carries
its verifier, its locator and the time it was captured.

### Stage 2 — Adequacy (a gate)

Before any construction begins, the framework asks: *is what we have actually
sufficient to build this series faithfully?* The adequacy stage scores the
research and sources and produces a readiness report. A project advances only
when the score clears a threshold. This gate prevents the common failure of
starting to "build" a series that the evidence cannot support. Since v12.4 only
**independent anchors** count toward adequacy — a benchmark the project computed
itself can never validate the project — and anchor sets are reconciled by two
separate readers.

### Stage 3 — Ingestion

Construct the `series_registry.json`: assign identities, decompose composite
series into their parts, classify each series by content type, record units, and
author the per-series **Data Provenance Records (DPRs)** that document source,
methodology, and every transformation. This is where the source material becomes
a formal, queryable model of the project.

### Stage 4 — Extension

Many series are most useful when carried forward to the present using live data
from statistical agencies and public APIs. The extension stage does this under
strict faithfulness rules (below) and records an **Extension Provenance Record
(EPR)** for each extended series, along with a divergence register that captures
any point where the extension departs from the original.

When a series is *rebuilt* rather than merely extended, a strict **adoption
doctrine** applies (v12.4): a new primary construction never overwrites the old
one. The previous arm is preserved byte-for-byte under a reconstruction or
legacy role, the seam between old and new is re-measured before and after the
adoption, and every consumer of the old arm is listed before the switch is
allowed.

### Stage 5 — Replication

Build a **self-contained, versioned replication package** — code that reproduces
every series *without any agent or human intervention*. The package follows a
disciplined four-phase script layout:

- **L## (Loading)** — fetch or read the raw source data;
- **P## (Processing)** — construction and transformation, and nothing else;
- **V## (Validation)** — check the constructed outputs against the book's
  published benchmark values;
- **M## (Manual adjustment)** — any documented hand corrections.

A single orchestrator script runs the whole package end-to-end with a hash audit
trail. This is what makes an Anu project *reproducible without agents*: a third
party can clone the package and regenerate the data.

Since v12.4 the replicator also ships a **clean-room replay** harness. The
package is rebuilt inside a room that can read only its declared, hash-pinned
sources; six negative controls must be denied (an undeclared input fails, it
does not warn); two offline rounds compare values and bytes separately; and the
replay emits a **receipt** recording the cryptographic identity of every
artifact it certified. Two patch releases (4.3.1 and 4.3.2) tightened the
closure audit — measuring the files a replay actually *produces*, and failing
any read from outside the room that is not a pinned external input.

### Stage 6 — Output formats

The validated data is rendered into two complementary formats:

- **6a — Machine-readable CSV** ("chopped"): a structured CSV with a metadata row,
  a column-ID row, and then the data, designed for programmatic consumption.
- **6b — Human-readable Excel** ("extenbook"): a self-contained four-sheet
  workbook — **Data**, **Provenance**, **Research**, and **Construction** — that
  lets a reader see a single series' entire construction story in one file.

### Stage 7 — Visualization

Build an interactive application (Plotly Dash or R Shiny) that presents each
series as a multi-source chart with methodology panels, source quotes, and the
extension data, all driven from the same registry and chopped CSVs.

### Stage 8 — Distribution

The finished project is published through three sibling channels aimed at three
audiences:

- **Replication repository** — for developers who want to clone the code and
  re-run the construction;
- **Consumer package** — for scholars who want the finished data files without
  touching a command line;
- **Audit-grade archive** — a comprehensive transparency bundle (code, data,
  per-series provenance, validation logs, methodology, checksums) for reviewers
  and journal data editors.

Publication runs through a strict gate that scrubs internal references and
verifies units, a data dictionary, and that no unpublished series leaks out.
Public downloads are offered as **CSV and Parquet** (not JSON), and every public
dataset ships a generated data dictionary. Public websites consume a dedicated
**`web` publish profile** generated from the registry — never the internal
project tree — and assert their build against a manifest. Since v12.4 the
publication gate runs checks **P01–P20**: a rights roster decides the
redistributable subset, a `publish:false` subseries can never ship, the builder
must be present in the project tree, no internal or interpreter paths may leak,
and publication requires a certifying replay receipt. The audit-grade archive
channel checks that every archived copy carries the certified bytes.

### Floating and infrastructure skills

Three skills run *at any stage* rather than at a fixed point in the pipeline:

- **Review** — the quality audit (see below). Since v12.4 a review is not closed
  until a **cold-read stage** passes: an independent reader who authored nothing
  reads the rendered pages as an outsider would, checks a sample of claims
  against the original sources, and returns a fix-first verdict before delivery.
  An **offset-paste detector** additionally hunts for validation values pasted
  one row or one year off.
- **Docs** — per-series documentation, including the web-facing **Anu Explainer**
  (a fixed five-section template: what the series is, where the data comes from,
  how it was constructed, why it matters, and a quote from the source). Since
  v12.4 a report layer also gates generated reports: every quoted span must
  match a verified quote-ledger row, and decision rows and citation keys are
  counted and checked.
- **Variant** — tracking of alternative construction methodologies, each with its
  own provenance record.

Two further skills are pure infrastructure: a **ledger** that inventories every
artifact and is regenerated after each stage, and a **doctor** that audits both
the framework's internal consistency (framework checks D01–D22) and an
individual project's consistency (project checks P01–P59). The doctor is
read-only by default: structured JSON output, printing of every item, and
stamping a certification block are all opt-in (`--json`, `--full`, `--stamp`).
A certification claim is valid only for the doctor version and date that issued
it — a claim stamped by an older doctor must be re-run before it is quoted.

---

## The orchestrator: `anu-build`

Co-ordinating nineteen skills by hand would be error-prone, so the framework
provides a master orchestrator, **`anu-build`**. It drives the whole pipeline
through nine stages (an initial inventory stage through to distribution),
computes the correct construction order automatically (some series depend on
others), enforces mandatory acceptance gates between stages, and maintains a
**documentation cascade** so that work can be handed off reliably:

- an append-only **event log** (one structured line per action);
- a regenerated **ledger** of per-series artifact state;
- a chronological **build narrative** readable by humans and agents alike;
- top-level **orchestration state** updated at each stage boundary.

Since v1.4 the orchestrator also provides a **rebuild-family mode**. When an
entire family of related series must be rebuilt together, it runs a fixed
21-stage cycle (C1–C21) with gate discipline at every stage: a frozen build
specification, declared sources, a fault catalogue, preservation checks, and
generated numbers and decision tables. The event log itself has a **single
writer** — step identifiers are allocated centrally, and a write whose
pre-recorded hash does not match is refused — so concurrent agents can never
interleave a corrupted log.

In practice a project is initialized once and then either run to completion or
advanced one stage at a time, with a status command available at any point.

---

## The v12.3 and v12.4 releases

Two releases since v12.2 moved the framework from documented discipline to
enforced discipline, and from enforced discipline to rebuild readiness.

### v12.3 — "enforcement" (2026-09-05)

A skeptical review found the mechanical gate green while a series of recurring
defect classes, observed across many projects, had no enforcing check at all.
The enforcement release closed that gap:

- **anu-doctor 2.7 → 2.12.** Twelve new project checks, each shipped with an
  expected-red test proving it can fail: path-literal resolution (archived roots
  fail), no silent exceptions, UTF-8 writers, no secrets, a status taxonomy
  enforced on every registry dialect, coverage checks on every dialect,
  units-versus-source and magnitude guards, data authenticity (a random-number
  producer hiding under an agency source fails), certification-claim semantics,
  rights carried into exports, export parity, and anchor independence (a
  validation anchor must be independent of the thing it validates). A
  registry-shape gate normalises list-form registries instead of crashing on
  them.
- **Read-only by default.** The doctor stopped writing anything unless
  explicitly asked; matrix output and certification stamps became opt-in flags.
- **Framework mode** grew a replication-package contract check (D20) and
  wrapper-integrity checks (D22): thin wrappers around canonical skill code may
  never diverge from it.
- **anu-publish 2.3** extended the publication gate; **anu-ingestion 5.5** made
  the object-form registry canonical, with the older array dialect tolerated on
  read.
- Consequence: no project may call itself "certified" against a doctor older
  than 2.12 without re-running the certification stamp.

### v12.4 — "rebuild programme" (2026-09-23)

v12.4 turned the practices a multi-family series-rebuild programme had to invent
into generic framework features:

- **anu-doctor 2.12 → 2.13 — project checks P01–P59.** New hard failures:
  purge-ledger identifier reuse (P52), a validation PASS recorded without its
  arm and anchor plus unbounded waivers (P53), and conflicting duplicate
  series-year keys (P54). New warnings: bundled-copy parity (P55),
  divergence-register schema (P56), code tolerances looser than the registry
  declares (P57), report-of-record drift against the live JSON (P58), and a
  research-quote contract (P59 — quotes must be verbatim, with a verifier, a
  locator and a time). Existing checks were made arm-aware: authenticity and
  coverage are scored per arm, the subseries grammar follows schema 2.4.0, and
  anchor classes distinguish printed-independent, author-workbook and
  tautological anchors — a tautological anchor never counts as independent.
- **Multi-arm series** — registry schema 2.4.0 (see the registry section):
  anu-ingestion 5.6, anu-extension 4.4, anu-chopped 3.1.
- **Family-cycle engine** — anu-build 1.4 rebuild-family mode, stages C1–C21,
  with the single-writer event log.
- **Clean-room replay** — anu-replicator 4.3 plus patches 4.3.1 and 4.3.2,
  emitting replay receipts that the publication and archive gates consume.
- **Publication gate P01–P20** — anu-publish 2.4 (see Stage 8); the audit-grade
  archive channel follows at anu-archive 2.1.
- **Report, quote and review integrity** — anu-docs 3.1 (quote gate and report
  gate), anu-review 5.2 (cold-read stage, offset-paste detector), anu-research
  3.1 (verbatim-only quote contract, name-keyed workbook anchors), anu-adequacy
  2.1 (independent anchors only, two-reader reconciliation).

The certification consequence repeats at every doctor bump: a "certified, zero
failures" claim issued under 2.12 is not a 2.13 claim — re-run the stamp before
quoting it.

### Skill versions at v12.4

| Skill | Version |
|---|---|
| anu-build (orchestrator) | 1.4 |
| anu-research | 3.1 |
| anu-adequacy | 2.1 |
| anu-ingestion | 5.6 |
| anu-extension | 4.4 |
| anu-scaffold | 2.2 |
| anu-replicator | 4.3 (patches 4.3.1 / 4.3.2) |
| anu-chopped | 3.1 |
| anu-extenbook | 4.0 |
| anu-visualize | 6.1 |
| anu-publish | 2.4 |
| anu-drive | 2.1 |
| anu-archive | 2.1 |
| anu-review | 5.2 |
| anu-docs | 3.1 |
| anu-variant | 2.0 |
| anu-ledger | 3.0 |
| anu-architecture | 3.0 |
| anu-doctor | 2.13 |

The two deprecated redirect stubs (`anu-rebuild`, `anu-pipeline`) still point at
`anu-build` and are never invoked directly.

---

## The non-negotiable principles

Anu's credibility rests on a handful of hard rules. They are enforced
mechanically, not left to good intentions.

**No synthetic data.** No skill may generate synthetic, estimated, placeholder,
approximated, or "frozen" values to fill a gap. If a value cannot be obtained,
the series is marked `data_unavailable` and its output is left empty. Every value
must trace either to a faithful replication of a published source or to a
documented analytical method with full provenance. (A random-number call in a
construction script is treated as a defect to be investigated and removed.)

**No proxies.** An extension must use the *exact* source the original author
used. CPI is not PPI; earnings are not compensation; a yield is not a total
return. Where the original series is genuinely discontinued, any substitution
must be flagged as a proxy and justified with a written explanation of why the
substitute measures the same concept.

**No lazy splices on derived quantities.** If the original computed a *formula*,
the extension must compute that same formula with extended component data — not
splice a growth rate onto the result. Growth-rate splicing is valid only for
series that were themselves directly observed.

**Unit documentation is mandatory.** Every series declares its units, every
loader validates that fetched data matches the expected units, and any script
combining series of different units must show its dimensional reasoning. (This
rule exists because real mismatches — millions divided by billions — once
produced ratios off by a factor of a thousand.)

**Content-type classification is mandatory.** Every series is classified as
`time_series`, `cross_sectional`, `theoretical`, or `derived`. Extension to the
present applies *only* to genuine time series; the pipeline refuses to "extend" a
point-in-time cross-section or a theoretical construct.

**Faithful replication over convenience.** Reviewers must read the original
source material and check constructed values against the author's published
figures and tables. The source is the ground truth; the code must conform to it,
not the other way around.

---

## The review: fourteen dimensions

Quality is assessed by a dedicated review skill that scores a data chapter or
module across **fourteen dimensions**. Twelve of these are weighted quality
dimensions (summing to 100%) covering completeness, methodological fidelity,
documentation, validation, and so on. The remaining two are **gates** — pass/fail
checks that no amount of strong scoring elsewhere can override:

- a **Data Authenticity** gate, which verifies that every value is genuinely
  sourced and that provenance has not drifted; and
- an **Outward-Facing Intelligibility** gate, which verifies that the published
  artifacts are actually understandable to an outside reader.

The review can be run at any stage, and it produces a scored report with concrete
gaps and action items rather than a single opaque grade. Since v12.4 it closes
with the cold-read stage described above.

---

## Output formats at a glance

| Format | Audience | What it is |
|---|---|---|
| Machine-readable CSV | Programmatic use | Structured CSV: metadata row + column IDs + data |
| Four-sheet Excel workbook | Human readers | Data / Provenance / Research / Construction in one file |
| Interactive visualization | Web / exploration | Multi-source charts with methodology panels and source quotes |
| Public download bundle | General public | CSV + Parquet, with a generated data dictionary |

---

## In short

The Anu Framework treats data construction as a first-class scientific activity
with its own discipline: a single registry as the source of truth, a gated
pipeline from research to distribution, an orchestrator that makes multi-step
work reproducible and handoff-safe, and a small set of non-negotiable rules —
no synthetic data, no proxies, no lazy splices, documented units, classified
content types, faithful replication — backed by a fourteen-dimension review, a
fifty-nine-check project doctor, and clean-room replay receipts on everything
that ships. The result is economic data series that an outside researcher can
trust, audit, and reproduce.
