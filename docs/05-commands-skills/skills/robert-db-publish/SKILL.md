---
name: robert-db-publish
version: "1.0"
description: "Package a Robert database as a publishable release — Data Package v2 + codebook + croissant.jsonld + WEB_MANIFEST.json + CITATION.cff + llms.txt — leak-scrub by severity, then mirror to the configured destination with grep-verify discipline."
when-to-use: '"User wants to publish, package, or mirror a Robert database release; build the Data Package / codebook / Croissant / CITATION.cff / llms.txt; or do the publish-gated leak scrub."'
search-hints: "robert db publish package data package v2 codebook croissant jsonld WEB_MANIFEST CITATION.cff llms.txt leak scrub mirror grep verify version"
argument-hint: "[project]"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit, Agent
requires: robert-db-audit
part-of: Robert Database Framework v1.0
---

# robert-db-publish — Stage 6 (PUBLISH)

Turn an audited database into a versioned, self-describing, publishable package and
mirror it to the configured destination. Publish is the only stage permitted to
write outside `Technical/RobertDB/` (to `publish.mirror_to`), and only after the
audit gate passes.

## Purpose

Generate the publication bundle, scrub leaks at the contract severity, and mirror
the bundle — verifying the mirror with grep, never trusting a registry alone.

## Preconditions

- `robert-db-audit --gate` exits `0` (or the user explicitly authorizes
  `--allow-warn` for unresolved `warn`-severity items).
- `config.publish.*` is set: `license`, `public`, `corpus_title`, `corpus_slug`,
  `mirror_to`.

## Package layout (Data Package v2)

```
publish/<corpus_slug>-<vX.Y>/
|-- datapackage.json        # Frictionless Data Package v2 descriptor (resources, schemas)
|-- data/                   # the published views (CSV [+ Parquet]) — derived from the DB
|-- codebook.md             # field-by-field codebook incl. the two-axis quality vocab
|-- croissant.jsonld        # ML Commons Croissant 1.0 dataset metadata
|-- WEB_MANIFEST.json       # website export contract (resource list, columns, types)
|-- CITATION.cff            # how to cite the corpus
|-- llms.txt                # machine-readable corpus summary for LLM consumers
`-- README.md              # human entry point
```

## Procedure

1. **Confirm the gate.** Re-run `robert-db-audit --gate`; do not proceed on a failing
   gate.
2. **Build + mirror.**
   ```bash
   PYTHONIOENCODING=utf-8 python rdb_publish.py \
       --config <P>/Technical/RobertDB/robertdb_config.json \
       --version vX.Y [--allow-warn] [--skip-mirror]
   ```
   This regenerates the views, assembles the package layout above, runs the leak
   scrub, and (unless `--skip-mirror`) backs up + copies the bundle to
   `publish.mirror_to`. Use `--skip-mirror` to build the package for inspection
   without touching the mirror destination.
3. **Leak scrub (severity by `publish.public`).** The scrub looks for workspace
   paths, absolute machine paths, API keys, and Arcanum-internal
   references in every published file.
   - `public: true` → any leak is **FAIL-severity**: the publish aborts. Fix the
     source view/codebook (never just delete the line from the package) and rebuild.
   - `public: false` (internal corpora) → leaks are WARN-severity and
     reported, but internal paths in an internal package are tolerated.
4. **Mirror grep-verify discipline (MANDATORY).** A registry or a script's
   "mirrored OK" message is **not** proof. After mirroring, verify the bytes landed:
   ```bash
   Test-Path "<mirror_to>/<corpus_slug>-vX.Y/datapackage.json"
   PYTHONIOENCODING=utf-8 python -c "import pathlib,sys; d=pathlib.Path(sys.argv[1]); \
print('files',sum(1 for _ in d.rglob('*') if _.is_file())); \
print('has_datapackage',(d/'datapackage.json').exists())" \
       "<mirror_to>/<corpus_slug>-vX.Y"
   ```
   Confirm the file count and key files match the built bundle. Never report
   "published" until grep/Test-Path confirms the destination. (This mirrors the
   Robert sync index-merge lesson: never trust a registry alone.)

## Outputs

- `publish/<corpus_slug>-<vX.Y>/` (full Data Package v2 bundle).
- A backup-first copy at `publish.mirror_to/<corpus_slug>-<vX.Y>/` (unless
  `--skip-mirror`).
- A `publish` run-ledger row.

## Failure handling

- **FAIL-severity leak on a public corpus** → publish aborts; fix the underlying
  view/codebook source and rebuild. Do not hand-edit files inside the package to
  pass the scrub.
- **Mirror not present after copy** → re-run with grep/Test-Path verification; treat
  a missing destination file as a failed publish even if the script exited 0.
- **Gate not green** → return to `robert-db-audit`; publish never bypasses the gate
  except via an explicit, user-justified `--allow-warn`.
