# robertdb_config.json — Specification (Robert Database Framework v1.0)

The config is the **only** project-specific input to the engine. Every script takes
`--config <path>`. No engine script may contain a project name, project path, or
project-specific logic. Location: `robertdb_config.json`.

All relative paths are relative to **project_root**.

```jsonc
{
  "framework_version": "Robert Database Framework v1.0",
  "config_version": "1.0",
  "project": "Volcker",                  // human name
  "project_code": "VLK",                 // 3-letter UPPERCASE; prefixes all IDs
  "project_root": "Volcker",

  "kb": {
    "kb_root": "Knowledge_Base",         // per-doc folders live here
    // Per-doc artifact layout hints; harvest still sniffs per file.
    // 'volcker' layout: <doc>/{FULL_TEXT*.md, CSV_Tables/, equations/, figures/}
    // 'ussr'    layout: <doc>/{Text/, CSV_Tables/, Tables/, Equations/, Figures/}
    "layout": "volcker",                 // volcker | ussr | auto
    // HDARP v6.3 native enrichment sidecar (docs/NATIVE_ENRICHMENT_CONTRACT.md):
    //   auto    (default) ingest RDB_METADATA*.jsonl when present, else legacy enrich path
    //   off     ignore sidecars entirely (force the full enrich recovery pass)
    //   require flag any non-marker doc lacking a sidecar via RDB_META_MISSING
    "native_enrichment": "auto",         // auto | off | require   (optional; default auto)

    // P4.1 LOCAL-MODEL enrichment triage (docs/A4_TRIAGE_HANDOFF.md). DEFAULT OFF:
    // the Opus-subagent enrich path stays the default. When mode != "off",
    // robert-db-enrich runs scripts/triage/rdb_triage_route.py FIRST — the ALGRP A4
    // champion (gemma-4-31B, GBNF-constrained) drafts each table's metadata locally;
    // a confidence gate auto-commits the easy majority as JSONL patches (merged by
    // rdb_merge_patches.py unchanged) and routes only low-confidence tables to Opus.
    //   off   (default) no local triage; full Opus-subagent enrichment as today
    //   draft local model drafts; gate auto-patches high-conf, queues the rest for Opus
    //   (the harness NEVER launches a GPU/server; the user runs llama-server first)
    "enrich_triage": {                    // optional; entire block defaults to off
      "mode": "off",                      // off | draft
      "endpoint": "http://localhost:8080",// running llama-server (gemma-4-31B); user-launched
      "gate": {                           // optional overrides of the default gate policy
        "min_confidence": "high",         // auto-accept floor (high|medium|low)
        "require_from_source_grounding": true,
        "review_if_transcription_in": ["R", "X"]
      }
    }
  },

  "catalogs": {
    // Whatever exists; absent keys are skipped with a logged notice.
    "hdarp_master_catalog": "HDARP_MASTER_CATALOG.csv",
    "document_audit": "DOCUMENT_AUDIT.csv",
    "table_catalog": "TABLE_CATALOG.csv",
    "entity_catalog": "ENTITY_CATALOG.csv",
    "classification_master": "CLASSIFICATION_MASTER.csv",
    "batch_state": "BATCH_STATE.json"
  },

  "unified": {
    // READ-ONLY cross-project provenance spine. Absolute paths allowed here only.
    "pdf_registry": "PDF_REGISTRY.csv",
    "kb_catalog": "KB_CATALOG.csv"
  },

  "adapters": {
    // Convention codes the harvester may apply, in sniff order.
    // Defined in adapters.py; detection is per-file signature, never assumed.
    "enabled": [
      "marker_no_tables",        // 'NO TABLES' marker CSVs
      "meta_block_then_data",    // USSR: metadata header block, blank line, data
      "meta_cols_embedded",      // Volcker c2: table_id/source_doc/chunk/page/title columns
      "meta_stub_only",          // metadata-only stub rows (no data)
      "multirow_header",         // bilingual/trilingual header rows (ru/translit/en, ru/fr)
      "source_page_col",         // data CSV with a source_page/notes attribution column
      "plain_data"               // pure data CSV; metadata not_captured
    ]
  },

  "tiers": {
    // Tier policy: matched against documents.category_raw (exact) or "campaign:<X>".
    // First match wins; default applies otherwise.
    "A": ["Foreign_Trade", "Agriculture", "BUDGET", "Finance", "NK_SSSR", "Yearbooks"],
    "B": [],
    "default": "C"
  },

  "quality": {
    // Deterministic confidence_score inputs (0-100; grade A>=85, B>=70, C>=50, else D)
    "campaign_weights": { "First": 0.80, "Second": 0.88, "Third": 0.97, "Fourth": 1.00 },
    "method_weights":   { "verbatim": 1.00, "analytical_digest": 0.85, "unknown": 0.80 },
    "language_penalty": { "ru": 5, "ru_fr": 6, "mixed": 4 }   // points subtracted (scan complexity)
  },

  "flag_vocabulary": [
    "RAGGED_ROWS", "EMPTY_TABLE", "METADATA_EMBEDDED_IN_DATA", "NO_TITLE_CAPTURED",
    "PAGE_APPROXIMATE", "ANALYTICAL_DIGEST_NOT_VERBATIM", "CAMPAIGN_VINTAGE_EARLY",
    "MD5_UNRESOLVED", "COUNT_MISMATCH_VS_AUDIT", "TRANSLATION_UNCERTAIN",
    "OCR_SUSPECT_NUMERALS", "ENRICHMENT_PENDING", "UNPARSEABLE_CONVENTION",
    "DUPLICATE_CONTENT_HASH", "NONSTANDARD_ENCODING", "RDB_META_MISSING"
  ],
  // RDB_META_MISSING (HDARP v6.3): raised by harvest when kb.native_enrichment="require"
  // and a non-marker doc has no RDB_METADATA*.jsonl sidecar. Add it to a project's
  // flag_vocabulary only if that project opts into "require".

  "publish": {
    "license": "CC-BY-4.0",              // or "internal" (Volcker)
    "public": false,                     // true => leak scrubbing is FAIL-severity
    "corpus_title": "Volcker Research Database",
    "corpus_slug": "volcker-db",
    "mirror_to": "Database"
  }
}
```

## Invariants

1. **Anti-bleed:** the only cross-project paths permitted are `unified.*` (read-only)
   and `publish.mirror_to` (write-once at publish, backup-first).
2. `project_code` is immutable once any `table_uid` has been minted.
3. Tier lists may change between runs (promotion); `enrichment_tier` on existing rows
   is updated by re-running the tier assigner, but `enrichment_status=enriched` rows
   are never demoted.
4. `quality.campaign_weights` may be recalibrated from spot-check issue rates; every
   recalibration is a new run-ledger entry and triggers a quality recompute.
