---
name: pdf-naming-protocol
version: "1.2"
description: "Normalize bulk-library PDF filenames in `_PDF_LIBRARY/<Project>/` to the unified `[YYYY] Author - Title.<ext>` form via a multi-model VLM+LLM consensus pipeline on a local GPU (Hopper engine). V2 panel: GLM-OCR title-page reader + Qwen3-32B + gemma-4-31B + Qwen3-Coder consensus composer. Operates only on bulk-library entries; wishlist-tracked `<wl_id>__...` files are left alone. Writes to UNIFIED_RENAME_HISTORY.csv + RENAME_LEDGER.csv + PDF_REGISTRY.csv atomically."
when-to-use: '"User wants to normalize messy bulk-library filenames (e.g., `nd__anon__INVE-POST-0024__768d62b10f.pdf`, `652.pdf`) to human-readable `[YYYY] Author - Title.<ext>`. Run after `/robert-pdf-sync` has placed PDFs into Robert; before HDARP campaigns or `/hopper` extraction if filename readability matters."'
search-hints: "pdf naming rename vlm gpu hopper [YYYY] author title bulk library normalize human-readable filename protocol glm-ocr qwen3 gemma multi-model consensus v2"
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
requires: "local Hopper venv (CUDA-enabled torch), Hopper engine + gpu_namer_*/gpu_compose scripts, llama.cpp + GGUF roster (Qwen3-32B + gemma-4-31B + Qwen3-Coder + GLM-OCR)"
part-of: "PDF Canonicalization Pipeline (acquisition → robert-pdf-sync → pdf-naming-protocol → /hopper OR /hdarp-campaign). The acquisition step is not part of this export."
---

## V2 Multi-Model Consensus Pipeline (shipped 2026-05-26)

The renamer now runs as a **3-model consensus** for empirical-confidence calibration:

| Stage | Tool | Model | Port | Role |
|---|---|---|---:|---|
| 1 | `gpu_namer_extract.py` | PyMuPDF (CPU) | — | Probe text vs vlm vs error; first-page excerpt |
| 2 | `gpu_namer_vlm.py` | GLM-OCR via `engine.server.ModelServer` | 8090 | Title-page read for scanned PDFs |
| 3a | `gpu_compose.py --model Qwen3-32B` | Qwen3-32B-Q4_K_M | **8088** | Lead composer |
| 3b | `gpu_compose.py --model gemma-4-31B` | gemma-4-31B-it-Q4_K_M | **8088** | Cross-check (different lineage) |
| 3c | `gpu_compose.py --model Qwen3-Coder` | Qwen3-Coder-30B-A3B-Q4_K_M | **8088** | Tertiary (different architecture) |
| 4 | `consensus.py` | (none, CPU) | — | Vote on year (majority), author (token-set), title (Jaccard); empirical_confidence = 0.30·year_agree + 0.30·author_agree + 0.40·mean_title_jac |
| 5 | `apply_naming_v2.py --apply` | (none) | — | Atomic on-disk rename + ledger append (user-gated only) |

**Port discipline (HARD)**: gpu_compose uses port **8088** and manages only its own proc; `engine.server.ModelServer` (used by VLM) uses port **8090** and currently does blanket-kill of llama-server.exe (pending P1 patch). Both can run sequentially on the single GPU; never concurrently.

**Quality benchmarks** (2026-05-26):
- 30-PDF calibration: 28/30 = **93% production-acceptable** (vs 80% single-model v1)
- RSCD-METHLIB (508 books): **86% strong consensus** (emp_conf ≥ 0.7)
- Q1 Robert LOWCONF (235): 48 production-ready (real title + emp_conf ≥ 0.7)
- Multi-model is **2-3× better than any simpler baseline** (filename / PyMuPDF metadata / regex)

**Audit trail**: every run writes proposals to **`UNIFIED_RENAME_HISTORY.csv`** (29,144 rows as of 2026-05-26 = 28,371 APPLIED historic + 773 PROPOSED). Refresh via `build_unified_rename_history.py`. See `UNIFIED_RENAME_HISTORY_README.md`.

---

# pdf-naming-protocol — VLM-driven PDF filename normalization

## Invocation

```
/pdf-naming-protocol <project> [--engine vlm|llm|both] [--dry-run] [--scope bulk_only|all]
```

Examples:
```
/pdf-naming-protocol RSCD --dry-run                     # preview proposed renames
/pdf-naming-protocol Volcker --engine vlm --scope bulk_only
/pdf-naming-protocol ALL                                 # iterate every v2 project folder
```

## What it does

For each PDF in `_PDF_LIBRARY/<Project>/` matching scope:
1. **Scope filter**: by default (`bulk_only`), skip files matching `^WL-...__` (wishlist-tracked). With `--scope all`, attempt to normalize everything except files already in `[YYYY] Author - Title.<ext>` form.
2. **VLM read**: render first N pages (default 3), pass through a local Hopper VLM (Llama-3.2-Vision or similar, local GPU) to extract `{title, author, year, language}`.
3. **LLM compose**: a small LLM agent composes the final `[YYYY] Author - Title.<ext>` per the strict format (Title ≤ 10 words; sanitize illegal filename chars; collapse whitespace).
4. **Confidence gate**: if VLM confidence below threshold OR LLM cannot compose, mark `needs_review` and skip the rename.
5. **Atomic rename**: move file → new name; update PDF_REGISTRY `canonical_name` + `current_flat_name`; update RENAME_LEDGER with `operation: pdf_naming_protocol_v1`; rebuild content-type symlinks pointing at new name.

## Engine spec (current, as deployed)

- **GPU**: a 32 GB-class consumer GPU. Hopper engine venv with CUDA-enabled torch.
- **VLM (title-page read for scanned/opaque)**: **GLM-OCR** via Hopper `ModelServer` on port 8088 (not 8090 — separate from main Hopper stack). The `gpu_namer_vlm.py` script handles render-on-demand + resumable extraction.
- **LLM (compose)**: **Qwen3-32B** via local llama.cpp. Frozen config (hard-won over 14 iters — DO NOT regress):
  - `enable_thinking:false` + parser strips `<think>` tags + ```json fences
  - Restart server every ~200 requests (slot/KV degradation under sustained load)
  - `--ctx-size 16384` (parallel-4 → 4096 tokens/slot; long academic excerpts overflow at 4096)
  - Manage only own proc/port — **NEVER blanket-kill** `llama-server.exe` (coexistence rule)
- **Implementation files**:
  - `gpu_namer_extract.py` (text + VLM router; resumable; outputs `name_extractions.jsonl`)
  - `gpu_namer_vlm.py` (GLM-OCR title-page read for scanned; outputs `name_vlm_results.jsonl`)
  - `gpu_compose.py` (Qwen3-32B composer; outputs `name_proposal_local.csv`)
  - `apply_naming_v2.py` (atomic rename + symlink repoint + ledger; dry-run by default, `--apply` to execute; reversible by md5)
  - `_n1_verify.py` (tie-out check after rename — MUST stay clean: 0 dangling, 0 reg-missing, 0 mismatch)
- **Reference implementation backbone** *(workspace-internal, not shipped in this export)*:
  `GPU_NAMING_PIPELINE_PLAN.md` (spec) + `GPU_NAMING_PROGRESS.md` (Iter 1–14 execution log with all the lessons)

## Filename rules (output form)

```
[YYYY] Author - Title.<ext>
```

- `YYYY` — 4-digit year, or `[nd]` if unknown
- `Author` — primary author surname (or first two surnames joined with `-` for 2-author works; `et al` for ≥3)
- `Title` — sentence-cased, ≤10 words, illegal chars removed, single hyphen-spaces
- `<ext>` — preserves original (.pdf, .html, .xls)

Examples:
- `[1995] Kurz - Theory of Production.pdf`
- `[2011] BLS - PPI Handbook Chapter.pdf`
- `[nd] anon - Annual Report Fragment.pdf`

## Scope: what gets renamed vs. left alone

| Filename pattern | Default scope (`bulk_only`) | `--scope all` |
|---|---|---|
| `WL-X-...__title__source.pdf` (wishlist) | SKIP | SKIP |
| `[YYYY] Author - Title.pdf` (already normalized) | SKIP | SKIP |
| `[YYYY]__Author__Title__hash.pdf` (legacy bulk) | RENAME | RENAME |
| `nd__anon__<stem>__<hash>.pdf` (poor-metadata bulk) | RENAME | RENAME |
| `<hash>.pdf`, `<digits>.pdf`, anything else | LEAVE | RENAME |

## Output artifacts

- `pdf_naming_run_<date>_<project>.csv` — every file processed: old name, new name, vlm_confidence, llm_decision, action
- `_UNIFIED/RENAME_LEDGER.csv` — append rows for every APPLIED rename (production transaction log; written by `apply_naming_v2.py`)
- **`_UNIFIED/UNIFIED_RENAME_HISTORY.csv`** — unified ledger of ALL naming actions (proposed + applied) across all corpora. Refresh via `build_unified_rename_history.py`. See `_UNIFIED/UNIFIED_RENAME_HISTORY_README.md` for schema.
- `_UNIFIED/PDF_REGISTRY.csv` — update `canonical_name` + `current_flat_name`
- Updated symlinks under `_PDF_LIBRARY/_BY_CONTENT_TYPE/<type>/`

## Acceptance criteria

- [ ] All in-scope files have a `[YYYY] Author - Title.<ext>` filename OR are flagged `needs_review`
- [ ] Wishlist-tracked files are unchanged (idempotency on wl_id files)
- [ ] Every rename appears in RENAME_LEDGER (md5-anchored)
- [ ] Every content-type symlink points at the new filename
- [ ] Re-running with `--dry-run` after execute reports 0 changes (idempotent)

## Idempotency

Re-running on a project that has already been normalized:
- Files in `[YYYY] Author - Title.<ext>` form → SKIP
- Files renamed in a prior run + still matching → SKIP
- New unprocessed bulk files arriving (e.g., post-Wave-2 sync) → RENAME

## Related

- acquisition (upstream): builds the target list and acquires the PDFs. Acquisition tooling is
  not part of this export; this skill takes the resulting files as its input.
- `robert-pdf-sync` (immediately upstream): places PDFs into `_PDF_LIBRARY/<Project>/`
- `pdf-naming-protocol` (this skill): normalizes filenames after placement
- `hdarp-campaign` / `preparehdarp` (downstream): operate on the canonicalized library

## Scripts (`_UNIFIED/_integration_audit/`)

This skill orchestrates (does not duplicate) the GPU VLM scripts under the Hopper engine and the rename infrastructure under `_UNIFIED/_integration_audit/`. **As deployed 2026-05-26**:

- **GPU side** (`Hopper`): `gpu_namer_extract.py` + `gpu_namer_vlm.py` + `gpu_compose.py`
- **Apply side** (`_integration_audit/`): `apply_naming_v2.py` (the orchestrator that does the actual file move + symlink repoint + ledger append + catalog reconcile, dry-run by default)
- **Verify side** (`_integration_audit/`): `_n1_verify.py` — MUST be run after any bulk rename; tie-out CLEAN means 0 dangling symlinks of N, 0 reg-missing, 0 mismatch. **Do not trust a run's own self-check — re-run `_n1_verify.py` independently.** (Iter 14 lesson: the run's "0 dangling" was stale; actual state had 37 dangling + 40 stale registry rows that the apply step missed.)

## Standard reference

`GPU_NAMING_PIPELINE_PLAN.md` — technical spec.
`GPU_NAMING_PROGRESS.md` — execution ledger (Iter 1–14, with all the lessons including the duplicate-run / coexistence / KV-degradation / ctx-overflow fixes).
