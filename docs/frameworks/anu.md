# The Anu Framework

*A pipeline for constructing research-grade economic data series with full provenance.*

**Framework version:** v12.2 (the "web-readiness" release)

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
current release comprises **19 active skills** (plus 2 superseded, still shipped in full)
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
workbooks, and the interactive visualizations are all generated *from* the
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
decision; nothing downstream is allowed to contradict it.

### Stage 2 — Adequacy (a gate)

Before any construction begins, the framework asks: *is what we have actually
sufficient to build this series faithfully?* The adequacy stage scores the
research and sources and produces a readiness report. A project advances only
when the score clears a threshold. This gate prevents the common failure of
starting to "build" a series that the evidence cannot support.

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
dataset ships a generated data dictionary.

### Floating and infrastructure skills

Three skills run *at any stage* rather than at a fixed point in the pipeline:

- **Review** — the quality audit (see below);
- **Docs** — per-series documentation, including the web-facing **Anu Explainer**
  (a fixed five-section template: what the series is, where the data comes from,
  how it was constructed, why it matters, and a quote from the source);
- **Variant** — tracking of alternative construction methodologies, each with its
  own provenance record.

Two further skills are pure infrastructure: a **ledger** that inventories every
artifact and is regenerated after each stage, and a **doctor** that audits both
the framework's internal consistency and an individual project's consistency.

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

In practice a project is initialized once and then either run to completion or
advanced one stage at a time, with a status command available at any point.

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
gaps and action items rather than a single opaque grade.

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
content types, faithful replication — backed by a fourteen-dimension review.
The result is economic data series that an outside researcher can trust, audit,
and reproduce.
