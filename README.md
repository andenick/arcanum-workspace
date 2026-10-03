# Arcanum — AI-First Research Infrastructure

> *"Those of us who are concerned with the social sciences … are engaged in an uncertain
> enterprise; perhaps we shall win no great treasures for mankind. But certainly it is our task to
> work out this lead with all the intelligence and the energy we possess until its richness or
> sterility be demonstrated."*
>
> — **Wesley C. Mitchell**, Presidential Address to the American Statistical Association,
> *Statistics and Government* (1919)

An open-source framework for AI-assisted scholarly research — built to put the tools of rigorous
empirical work in the hands of independent researchers, not only those with institutional access to
proprietary databases and large research teams.

It is a **documentation export**, deliberately curated: it shares the *methods and standards*, not
private data, credentials, or internal operational notes.

## Why this exists

Arcanum is an attempt to build a shared framework for collaboration between social scientists in the
age of large language models and agentic coding assistants — collaboration in two senses at once:

- **Between researchers** — a common set of methods, standards, and provenance conventions so that
  social scientists can build on one another's work: extractions, datasets, and replication packages
  that are interoperable, auditable, and reusable, not one-off and locked inside a single lab.
- **Between researcher and machine** — patterns for working *with* an AI agent as an intellectual
  partner rather than a black box: the agent does the labor, but every step stays transparent,
  validated, and reproducible, and the researcher keeps command of the method.

The wager is that the tools of rigorous empirical research — long gatekept behind licenses,
institutional access, and large teams — can be opened up, so that a single researcher working
alongside capable agents can meet professional standards and share the result.

The intellectual tradition is specific, and it is why **replication is treated here as the engine of
progress, not its afterthought**: Sraffa spent four decades reconstructing the works and
correspondence of Ricardo from manuscripts; Leontief built his input–output tables from primary
census data; Shaikh rebuilt the empirics of classical political economy series by series. These
scholars did not merely theorize — they built knowledge from sources, one table at a time. The
faithful reproduction *was* the discovery.

Arcanum is the infrastructure that makes that kind of work possible at scale with AI agents. Every
methodology here has been tested on real research: extracting tables from Soviet statistical
yearbooks, digitizing pre-war tax directories, reconstructing decades of macroeconomic data from
scattered government reports.

## What's inside

| Layer | Read |
|---|---|
| **Document extraction (cloud agent)** | [`docs/frameworks/hdarp.md`](docs/frameworks/hdarp.md) — HDARP v6.4 |
| **Document extraction (offline GPU)** | [`docs/frameworks/hopper.md`](docs/frameworks/hopper.md) — Hopper Line v2 |
| **Knowledge-base integration** | [`docs/frameworks/kbip.md`](docs/frameworks/kbip.md) — KBIP v1.0 |
| **Per-project databases** | [`docs/frameworks/robert-db.md`](docs/frameworks/robert-db.md) — Robert DB v1.0 |
| **Data-series construction** | [`docs/frameworks/anu.md`](docs/frameworks/anu.md) — Anu Framework v12.4 |
| **Unified knowledge architecture** | [`docs/frameworks/auka.md`](docs/frameworks/auka.md) — AUKA v1.1 |
| **Skill + command templates** | [`docs/05-commands-skills/`](docs/05-commands-skills/) |

> **Note on HDARP:** the public [`hdarp`](https://github.com/andenick/hdarp) repo currently ships the **v5.1** OCR-consensus layer, while the protocol documented here is **v6.4** (full v6.4 publication planned).

```
   PDFs / scans
        │
        ▼
  ┌─────────────┐   cloud-agent reading (HDARP)  ── or ──  offline local-GPU (Hopper)
  │  EXTRACTION │   → body text + tables + equations + figures, with honest provenance
  └─────────────┘
        │
        ▼
  ┌─────────────┐   KBIP — method-agnostic integration (records HOW each document was read)
  │ INTEGRATION │
  └─────────────┘
        │
        ▼
  ┌─────────────┐   Robert DB — one queryable database per project; two-axis quality
  │  DATABASE   │   (source quality × reading fidelity); honest per-field provenance
  └─────────────┘
        │
        ▼
  ┌─────────────┐   Anu — construct reproducible data series (faithful replication + extension,
  │ CONSTRUCTION│   no synthetic data, no proxies, full provenance)
  └─────────────┘
        │
        ▼
   publishable, citable datasets
```

## The ethic

The lineage above is not decoration; it is the method. A few commitments run through everything:

- **Replication is progress.** Reproducing a result from its sources is not drudgery to be automated
  away — it is how cumulative knowledge is actually built. The pipeline is engineered to make faithful
  reconstruction *and* honest extension routine.
- **Honest uncertainty.** "Not captured," "illegible," and "low-confidence read" are first-class
  values — never silently blanked or guessed. Quality is two-axis: whether the source datum was sound
  is tracked separately from whether we read it correctly.
- **No synthetic data, no proxies.** If a value cannot be obtained, it is marked missing — not
  fabricated, not quietly swapped for a different concept.
- **Bashers, not sweepers.** (After Vonnegut's distinction.) Work proceeds step by step — each one
  verified before the next — on real data only.
- **The agent as partner, not oracle.** AI agents do the heavy lifting, but every analytical step is
  transparent, validated, and reversible — quality comes from meticulous verification, not from
  trusting a black box. The researcher investigates *with* the agent, and stays accountable for the method.
- **Method in the open.** Rigorous empirical tools have long been gatekept behind licenses,
  institutional access, and teams. AI changes that equation: one researcher with well-designed
  infrastructure can now meet professional standards. Sharing that infrastructure — making method
  transparent — is itself the point.

This is why Mitchell sits at the top. Quantitative social science is an *uncertain enterprise*; no one
can promise it will bear fruit. But there is a responsibility to build the tools honestly, apply them
rigorously, and find out — to work the lead until its richness or sterility is demonstrated.

## Core principles (portable to any workspace)

- **One mandatory rule:** every project keeps untouched original sources in an `Inputs` folder
  (read-only); processing happens elsewhere. Everything else is flexible.
- **Provenance everywhere** — every series, table, and figure records where it came from and how it
  was derived.
- **Standards a machine can check** — versioned protocols + automated audits, so quality is enforced,
  not merely hoped for.

## Getting started

```bash
git clone https://github.com/andenick/arcanum-workspace.git
```

- **For AI agents:** read the framework docs in [`docs/frameworks/`](docs/frameworks/), then the
  skill templates in [`docs/05-commands-skills/`](docs/05-commands-skills/).
- **For researchers:** start with the methodology in [`docs/frameworks/`](docs/frameworks/), then
  browse [`projects/`](projects/) to see it applied.
- **Multi-LLM:** the protocols are platform-agnostic — Claude, Gemini, or any capable coding/vision agent.

## License

MIT (see [`LICENSE`](LICENSE)). Citation metadata in [`CITATION.cff`](CITATION.cff). The **published
tree** contains no personal data, credentials, or private research data — the studies that apply
these methods live elsewhere. Earlier commits in this repository's history predate that standard and
are being reviewed separately; treat the current tree, not the history, as the curated export.

---

**Frameworks:** HDARP v6.4 · Anu v12.4 · Sraffa 4.0 OCR · Robert DB v1.0 · KBIP v1.0 · Hopper Line v2 (HL2 v1.0) · AUKA v1.1
