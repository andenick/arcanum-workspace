---
name: robert-db-init
version: "1.0"
description: "Scaffold a project's RobertDB workspace and create the canonical robertdb.sqlite from robertdb_config.json — the first stage of the Robert Database Framework build."
when-to-use: '"User wants to start a Robert database for a project, scaffold Technical/RobertDB/, or (re)create the empty robertdb.sqlite from a config."'
search-hints: "robert db init scaffold create database robertdb sqlite config project_code schema apply meta"
argument-hint: "[project]"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit, Agent
requires: none
part-of: Robert Database Framework v1.0
---

# robert-db-init — Stage 1 (INIT)

Scaffold the per-project RobertDB workspace and create the canonical
`robertdb.sqlite` with the v1.0 schema applied and the `meta` identity stamp
written. This is the only stage that may create the database file.

## Purpose

Take a validated `robertdb_config.json` and produce an empty, schema-stamped,
project-identified database plus the on-disk folder skeleton every later stage
reads and writes. After init the DB has tables but zero documents/tables.

## Preconditions

- The project has been through `/kb-integrate-pipeline` (KB + catalogs exist).
- A `robertdb_config.json` exists at `<P>/Technical/RobertDB/` per
  `CONFIG_SPEC.md`. In particular:
  - `framework_version` == `"Robert Database Framework v1.0"` (the lib enforces this).
  - `project_code` is **3 uppercase letters** and is **immutable** once any
    `table_uid` is minted — choose it correctly now.
  - `project_root` points at an existing directory.
- If you are writing the config, validate it against CONFIG_SPEC.md first; do not
  guess catalog paths — absent catalog keys are skipped with a logged notice, but
  a wrong path silently resolves to "absent".

## Procedure

1. **Locate or author the config.** Confirm
   `<P>/Technical/RobertDB/robertdb_config.json` exists. If authoring it,
   copy the CONFIG_SPEC.md skeleton, set `project`, `project_code`, `project_root`,
   `kb.layout` (`volcker` | `ussr` | `auto`), the `catalogs.*` that actually exist,
   the read-only `unified.*` spine paths, `tiers`, `quality`, `flag_vocabulary`,
   and `publish`.

2. **Run init.**
   ```bash
   PYTHONIOENCODING=utf-8 python rdb_init.py \
       --config <P>/Technical/RobertDB/robertdb_config.json
   ```
   Use `--force` **only** to re-apply schema to an existing DB (idempotent; does
   not drop data). Never `--force` to "start over" on a DB that already has minted
   `table_uid`s — that risks the immutable-id invariant.

3. **Verify.** Confirm the scaffold and stamp:
   ```bash
   PYTHONIOENCODING=utf-8 python -c "import sqlite3,sys; c=sqlite3.connect(sys.argv[1]); print(dict(c.execute('select key,value from meta').fetchall()))" \
       <P>/Technical/RobertDB/robertdb.sqlite
   ```
   Expect `schema_version=1.0.0`, `framework_version=Robert Database Framework v1.0`,
   `project`, `project_code`, `created_at`. An `init` row appears in the `runs`
   ledger and `state/runs/RUN_INIT_*.json` on disk.

## Outputs

```
<P>/Technical/RobertDB/
|-- robertdb_config.json          # input (validated)
|-- robertdb.sqlite               # canonical DB (schema applied, meta stamped)
|-- views/                        # (empty until harvest/views)
|-- state/                        # BUILD_STATE.json, runs/RUN_*.json
|-- enrichment/                   # patches/ (created by enrich)
|-- audit/                        # AUDIT_REPORT.* (created by audit)
`-- publish/                      # release packages (created by publish)
```

## Failure handling

- **`framework_version` mismatch / missing keys** → `rdb_lib.Config` raises; fix
  the config, do not edit the library.
- **`project_code` not 3 uppercase letters** → fix the config; the code prefixes
  every id and cannot change later.
- **`project_root not found`** → correct the path; relative paths in the config are
  resolved against `project_root`.
- **DB already exists** → init is safe to re-run; it applies schema idempotently.
  Do not delete an existing DB to "reset" — escalate if a true reset is intended.
