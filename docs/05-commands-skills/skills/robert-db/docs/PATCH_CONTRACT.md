# Enrichment Patch Contract (Robert Database Framework v1.0)

Enrichment subagents NEVER write to robertdb.sqlite. They emit **JSONL patch files**
(one JSON object per line, one line per table) to
`<doc_id>.jsonl`.
`rdb_merge_patches.py` validates and merges them transactionally.

## Patch record

```jsonc
{
  "table_uid": "VLK-T-004217",          // REQUIRED, must exist in xtables
  "doc_id": "017_FDIC_1997_...",        // REQUIRED, must match the table's doc
  "run_id": "ENR_20260615_A03",         // REQUIRED, the batch run id

  // Recovered metadata — include ONLY fields the agent actually assessed.
  // null  = assessed and NOT recoverable (=> basis must say not_captured)
  // absent = not assessed (field untouched)
  "title_raw": "Вывоз из России за 1913–17 г.г.",
  "title_translit": "Vyvoz iz Rossii za 1913-17 g.g.",
  "title_en": "Exports from Russia, 1913-1917",
  "page": "5",
  "page_basis": "exact",                // exact|approx|chunk_derived|not_captured
  "units": "thousand rubles",
  "footnotes": "Source note: Customs returns ...",
  "period_coverage": "1913-1917",
  "geography": "Russia (imperial borders)",

  // REQUIRED for every metadata field present above:
  "field_basis": {
    "title_raw": "from_source",         // visible verbatim in CSV/Tables md/FULL_TEXT
    "title_en": "agent_inferred",
    "page": "from_source",
    "units": "agent_inferred",
    "footnotes": "not_captured"
  },

  // Column-level recoveries (optional):
  "columns": [
    { "col_index": 2, "header_en": "Provisions", "units": "thousand rubles",
      "role": "measure", "basis": "agent_inferred" }
  ],

  // Two-axis quality assessment (optional but encouraged):
  "obs_status": "A",                    // SDMX CL_OBS_STATUS dominant code for table
  "transcription_status": "H",          // V|H|L|R|X

  // Organization hints (consumed by robert-db-organize, not merged directly):
  "taxonomy_suggestions": ["Foreign_Trade/Exports"],
  "concordance_hint": "annual exports-by-commodity-class family, customs yearbooks",

  "quality_observations": ["row totals consistent with components"],
  "enrichment_confidence": "high"       // high|medium|low
}
```

## Merge rules (enforced by rdb_merge_patches.py — violations reject the LINE, not the file)

1. `table_uid` must exist; `doc_id` must match.
2. `field_basis` must cover every metadata field present; basis values must be in
   the vocabulary {mechanical, from_source, agent_inferred, not_captured, unknown}.
3. **No downgrade:** an existing `from_source` basis is never overwritten by
   `agent_inferred`/`mechanical`. Existing `not_captured`/`unknown` may be upgraded.
4. A field sent as `null` with basis `not_captured` records the honest negative
   (field_provenance row written; xtables column left NULL).
5. On successful merge of >=1 field: `enrichment_status='enriched'`, `updated_at` set.
6. Rejected lines are written to `patches/<run_id>/_rejected.jsonl` with a reason —
   never silently dropped.
7. Merges are idempotent: replaying a patch file yields the same DB state.

## Size discipline

One patch line per table; keep `footnotes` <= ~2,000 chars (truncate with `…[truncated]`).
A doc-batch agent emits ONE .jsonl per document. Respect the 32K output cap: for
table-dense yearbooks, split the assignment by chunk-range, not by trimming fields.

## Native capture twin (HDARP v6.3)

`docs/NATIVE_ENRICHMENT_CONTRACT.md` defines the **same record** emitted by HDARP
extraction at read time, keyed by `source_relpath` instead of `table_uid`. `rdb_harvest.py`
resolves `source_relpath → table_uid` and feeds those lines through THIS merger unchanged,
so every rule above applies identically whether metadata arrives natively (at harvest) or
via a dedicated enrichment pass.
