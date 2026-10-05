# robertdb_config.json — Specification (Robert Database Framework v1.0)

The config is the **only** project-specific input to the engine. Every script takes
`--config <path>`. No engine script may contain a project name, project path, or
project-specific logic. Location: `<P>/Technical/RobertDB/robertdb_config.json`.

All relative paths are relative to **project_root**.

```jsonc
{
  "framework_version": "Robert Database Framework v1.0",
  "config_version": "1.0",
  "project": "DemoProject",               // human name
  "project_code": "DEMO",                 // 3-letter UPPERCASE; prefixes all IDs
  "project_root": "DemoProject",

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

    // P4.1 LOCAL-MODEL enrichment triage (docs/A4_TRIAGE_HANDOFF.md — not shipped). DEFAULT OFF:
    // the subagent enrich path (Sonnet 5.5 default) stays the default. When mode != "off",
    // robert-db-enrich runs scripts/triage/rdb_triage_route.py FIRST — the ALGRP A4
    // champion (gemma-4-31B, GBNF-constrained) drafts each table's metadata locally;
    // a confidence gate auto-commits the easy majority as JSONL patches (merged by
    // rdb_merge_patches.py unchanged) and routes only low-confidence tables to Opus.
    //   off   (default) no local triage; full subagent enrichment as today (Sonnet 5.5
    //                     default; Opus only as a recorded escalation)
    //   draft local model drafts; gate auto-patches high-conf, queues the rest for Opus escalation
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
      "meta_block_then_data",    // metadata header block, blank line, then data (the 'ussr' layout family)
      "meta_cols_embedded",      // c2 dialect: table_id/source_doc/chunk/page/title columns (the 'volcker' layout family)
      "meta_stub_only",          // metadata-only stub rows (no data)
      "multirow_header",         // bilingual/trilingual header rows (ru/translit/en, ru/fr)
      "source_page_col",         // data CSV with a source_page/notes attribution column
      "plain_data"               // pure data CSV; metadata not_captured
    ]
  },

  "tiers": {
    // Tier policy: matched against documents.category_raw (exact) or "campaign:<X>".
    // First match wins; default applies otherwise.
    "A": ["Foreign_Trade", "Agriculture", "BUDGET", "Finance", "Statistical_Yearbooks"],
    "B": [],
    "default": "C"
  },

  "quality": {
    // Deterministic confidence_score inputs (0-100; grade A>=85, B>=70, C>=50, else D)
    "campaign_weights": { "First": 0.80, "Second": 0.88, "Third": 0.97, "Fourth": 1.00 },
    "method_weights":   { "verbatim": 1.00, "analytical_digest": 0.85, "unknown": 0.80,
                          "analytical_digest+verbatim_sibling": 0.85 },
    // "analytical_digest+verbatim_sibling" (HDARP v6.4 "Hybrid Coherence") — an agent layer read in
    // analytical_digest mode that ALSO has a complete verbatim OCR sibling at
    // Knowledge_Base/_OCR_Only/<short_id>/ (the mandatory Stage 5 pass; see the
    // "Hybrid body text = two layers" rule).
    //
    // The weight is DELIBERATELY IDENTICAL to plain analytical_digest (0.85). The sibling makes the
    // PAGE quotable; it does not make the DIGEST verbatim. The two layers are not interchangeable,
    // and the sibling must never upgrade the agent layer's score — a document whose tables and
    // metadata were read against a paraphrase is exactly as reliable as it was before the sibling
    // existed. Recording the case explicitly is the point: 0.85 becomes a stated decision instead
    // of a silent default, so no document is quietly scored 0.85 while holding complete verbatim
    // text with nothing anywhere saying why the two facts do not interact.
    //
    // The sibling is recorded on its own axis, not in this multiplier: DOCUMENT_AUDIT.csv
    // (KBIP schema v1.1) ocr_layer_path / ocr_layer_complete, and READ_METHOD.json verbatim_layer.
    // ANALYTICAL_DIGEST_NOT_VERBATIM (flag_vocabulary below) still fires for these documents —
    // having a sibling does not clear it, because the flag describes the agent layer.
    //
    // WHEN THIS KEY FIRES: rdb_quality.py looks up documents.extraction_method. Per weave-1.0
    // (HDARP_V64_WEAVE_FORMAT.md sec 6) the canonical extraction_method stays
    // "analytical_digest" precisely so no consumer mistakes a paraphrase for the author's words,
    // so this key only fires where a project deliberately records the compound value. Either way
    // the resulting weight is 0.85 — which is the intended, and the safe, outcome.
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
    "license": "CC-BY-4.0",              // or "internal"
    "public": false,                     // true => leak scrubbing is FAIL-severity
    "corpus_title": "Demo Research Database",
    "corpus_slug": "demo-db",
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
