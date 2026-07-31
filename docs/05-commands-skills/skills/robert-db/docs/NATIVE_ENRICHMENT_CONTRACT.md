# Native Enrichment Contract (Robert Database Framework v1.0 · HDARP v6.3)

> **Purpose.** Let HDARP extraction capture honest table metadata **at read time** so that
> `robert-db-harvest` ingests it directly and `robert-db-enrich` collapses from a full recovery
> pass to a thin verification/spot-check. This is the on-disk artifact the HDARP suite (v6.3
> "Native Enrichment") emits; this document is its **authoritative spec**.
>
> One database per project; this contract changes **no** existing extraction output — it adds a
> sidecar file. Legacy Knowledge_Base output that predates v6.3 simply has no sidecar and harvests
> exactly as before (all metadata `mechanical`/`not_captured`, then the legacy enrich pass).

This artifact is a near-twin of `docs/PATCH_CONTRACT.md`. The only structural difference: an
enrichment patch is keyed by **`table_uid`** (which the harvester mints); HDARP does not know the
`table_uid`, so the sidecar is keyed by **`source_relpath`** instead. Harvest resolves
`source_relpath → table_uid` and feeds the lines through the unchanged `rdb_merge_patches.py`, so
every validation, auto-repair, no-downgrade, and rejection rule in the patch contract applies
identically here.

---

## 1. Where it lives

Emitted **inside the per-document KB folder**, alongside the existing 4-type output. Additive only —
never overwrite `CSV_Tables/`, `Tables/*.md`, `FULL_TEXT*.md`, `equations/`, `figures/`.

```
<doc>/RDB_METADATA.jsonl                  # whole-doc processors: one file, one line per table
<doc>/RDB_METADATA_chunks_NNN_NNN.jsonl   # chunk-range processors: one shard per chunk-range
```

Chunk-range shards mirror the existing `FULL_TEXT_chunks_NNN_NNN.md` naming so a dense yearbook
processed by many parallel agents produces many shards that `hdarp-wrapup` consolidates (§5).

## 2. One line per table

One JSON object per line, one line per **table CSV** the extraction wrote that round. Write the line
**as each CSV is written** (never buffer the whole batch — respect the 32K agent-output cap; a dense
chunk-range is split into shards, never trimmed of fields).

```jsonc
{
  // --- identity (REQUIRED) ---
  "source_relpath": "<doc>/CSV_Tables/table_006_010_02.csv",  // natural key: CSV path
                                                                              // relative to PROJECT ROOT
  "doc_id": "<doc folder name>",            // REQUIRED, must equal the KB folder name verbatim

  // --- recovered metadata (include ONLY fields actually assessed) ---
  // null   = assessed and NOT recoverable  (=> field_basis must say not_captured)
  // absent = not assessed                  (field left untouched at harvest)
  "title_raw": "Вывозъ изъ Россіи за 1913 г.",   // verbatim as printed (preserve pre-1918 orthography)
  "title_translit": "Vyvoz iz Rossii za 1913 g.",
  "title_en": "Exports from Russia, 1913",
  "page": "142",
  "page_basis": "exact",                    // exact | approx | chunk_derived | not_captured
  "units": "thousand rubles",
  "footnotes": "Source note: customs returns; gold rubles.",
  "period_coverage": "1913",
  "geography": "Russia (imperial borders)",

  // --- REQUIRED for every metadata field present above ---
  "field_basis": {
    "title_raw": "from_source",             // visible verbatim in this table's CSV / Tables md / the
    "title_translit": "agent_inferred",     //   chunk's FULL_TEXT/Text context
    "title_en": "agent_inferred",
    "page": "from_source",
    "units": "agent_inferred",
    "footnotes": "from_source"
  },

  // --- column-level recoveries (optional) ---
  "columns": [
    { "col_index": 2, "header_en": "Provisions", "units": "thousand rubles",
      "role": "measure", "basis": "agent_inferred" }
  ],

  // --- two-axis quality (optional but strongly encouraged) ---
  "obs_status": "A",                        // SDMX CL_OBS_STATUS dominant code for the SOURCE table
  "transcription_status": "H",              // our READ fidelity: H | L | R | X  (NEVER V — see §4)

  // --- organization hints (consumed later by robert-db-organize; not merged into columns) ---
  "taxonomy_suggestions": ["Foreign_Trade/Exports"],
  "concordance_hint": "annual exports-by-commodity family, customs yearbooks",
  "quality_observations": ["row totals consistent with components"],
  "enrichment_confidence": "high"           // high | medium | low
}
```

Everything except `source_relpath`/`doc_id` is identical in name and meaning to a patch record. A
marker / "NO TABLES" CSV needs **no** line (harvest flags it `is_marker_file`).

## 3. Vocabularies (authoritative — copied verbatim from `rdb_lib.py`)

| Slot | Allowed values |
|---|---|
| `field_basis.*` (`BASIS_VOCAB`) | `mechanical`, `from_source`, `agent_inferred`, `not_captured`, `unknown` |
| basis rank (`BASIS_RANK`, no-downgrade) | `from_source`=3 · `mechanical`=`agent_inferred`=2 · `unknown`=`not_captured`=1 |
| `page_basis` (`PAGE_BASIS_VOCAB`) | `exact`, `approx`, `chunk_derived`, `not_captured` |
| `transcription_status` (`TRANSCRIPTION_VOCAB`) | `V`, `H`, `L`, `R`, `X` — **agents emit only `H`/`L`/`R`/`X`** |
| `obs_status` (`OBS_STATUS_VOCAB`) | `A`,`B`,`D`,`E`,`P`,`U`,`I`,`O`,`M`,`L` (SDMX CL_OBS_STATUS) |
| `columns[].role` | `dimension`, `measure`, `metadata_embedded`, `unknown` |
| `enrichment_confidence` | `high`, `medium`, `low` |

`page_basis` is a **qualifier** — it rides on `page` and does NOT take a `field_basis` entry.

## 4. The honesty contract (non-negotiable — same text as `robert-db-enrich`)

- **NOT CAPTURED is a first-class value.** A title, page, unit, or footnote that is not visibly
  present is either `agent_inferred` (defensibly derivable from the chunk you are reading) or sent as
  `null` with basis `not_captured`. Never fabricate titles, footnotes, units, quotes, or page numbers.
- **The source page is the authority.** A field is `from_source` **only** if it is visible verbatim in
  this table's CSV, its `Tables/*.md` note, or the `FULL_TEXT`/`Text` context for that chunk.
  Transliterations and translations are **always** `agent_inferred`, never `from_source`.
- **No downgrade.** Harvest applies every line through `record_basis()`, so a `from_source` value can
  never be clobbered by an `agent_inferred`/`mechanical` one; `not_captured`/`unknown` may be upgraded.
- **`obs_status`** describes the SOURCE datum's quality (a printed «—»/estimate/break), NOT your read.
- **`transcription_status`** describes YOUR read fidelity vs the source page, orthogonal to `obs_status`.
  Assess it per table — **do not default everything to `L`**: `H` = confident the digitized cells match
  the page; `L` = uncertain (faint scan, ambiguous digits); `R` = reconstructed from a damaged region;
  `X` = illegible. **`V` is RESERVED** for the formal audit spot-check (`robert-db-audit`); extraction
  agents must never emit `V`.

## 5. Sharding & consolidation (chunk-range processors)

- A chunk-range processor (e.g. `sphdarp` splitting a 386-table yearbook across agents) writes its own
  `RDB_METADATA_chunks_NNN_NNN.jsonl` shard — never appends to a shared file (avoids write races).
- `hdarp-wrapup` Phase 4 consolidates shards into the doc's canonical record by **dedup on
  `source_relpath`** (last-writer-wins on a re-extracted range; identical lines collapse). It may either
  merge shards into a single `RDB_METADATA.jsonl` or leave the shards in place — harvest reads both.

## 6. How harvest ingests it (`rdb_harvest.py`)

Per document, after each table's `table_uid` is minted and its row written:

1. Glob `RDB_METADATA.jsonl` + `RDB_METADATA_chunks_*.jsonl` in the doc folder; index lines by
   **normalized `source_relpath`** (posix + casefold; same `norm_name` discipline used for catalog joins).
2. Resolve each line's `source_relpath` to the CSV being harvested. Unmatched lines are a **notice**
   (logged, counted) — never silently applied to the wrong table, never a hard error.
3. Convert each matched line into an enrichment patch line **keyed by the minted `table_uid`** and write
   it to `enrichment/patches/NATIVE_<ts>/<doc_id>.jsonl`.
4. After the harvest loop, if any native patches were written, chain
   `rdb_merge_patches.py --run-id NATIVE_<ts>` **before** `rdb_quality.py`/`rdb_views.py`. The merger does
   all validation, vocab auto-repair, no-downgrade, `obs_status`/`transcription_status` set-when-NULL, and
   `_rejected.jsonl` handling — **zero new merge logic**. Tables with ≥1 merged field become
   `enrichment_status='enriched'`; the rest stay `pending`/`mechanical_only` for the legacy enrich pass.
5. Record `native_sidecars_ingested=N` in the harvest run note. Absent sidecar ⇒ no flag, no patch dir.

Config gate (`robertdb_config.json`, `kb.native_enrichment`):
`auto` (default — ingest when present), `off` (force the legacy enrich path), `require` (flag any
non-marker doc lacking a sidecar via `RDB_META_MISSING`).

## 7. Backward compatibility

The hook is purely presence-gated. A pre-v6.3 KB has no sidecar → the glob finds nothing → harvest writes
metadata `mechanical`/`not_captured` exactly as today → `robert-db-enrich` runs its full doc-batch
recovery. A later native ingest can only **upgrade** (no-downgrade), never clobber, a value an old enrich
run already set `from_source`. The two in-flight builds (USSR, Volcker) finish the old way; `auto` is a
no-op for them.

## 8. Size discipline

One line per table; `footnotes` ≤ ~2,000 chars (truncate with `…[truncated]` — the merger also enforces
this). Write incrementally as CSVs are produced; split dense chunk-ranges into shards, never by trimming
fields. A whole-doc processor emits one `RDB_METADATA.jsonl`; a chunk-range processor emits one shard.

---
*Companion: `docs/PATCH_CONTRACT.md` (the `table_uid`-keyed twin merged by `rdb_merge_patches.py`).
Vocabularies are owned by `scripts/rdb_lib.py`; schema by `schema/robertdb_schema.sql`.*
