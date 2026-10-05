---
name: hopper
description: Extract a PDF (or folder of PDFs) OFFLINE on a local consumer-GPU machine into structured 4-artifact + Anu-ready output via the Hopper Line v2 (HL2) engine. Use when the user wants to OCR/parse/extract a document locally with zero API calls — body text + tables + equations + figures, with per-region model routing (GLM-OCR structural, dots.ocr Cyrillic-faithful, Qwen3.6-VL-REAP charts). Distinct from HDARP (cloud Claude Read-tool). NEVER call this engine "HDARP" / "Local-HDARP" / "Sraffa N".
version: "2.0"
part-of: Hopper Line v2 (HL2)
argument-hint: "<pdf|folder> [--profile general|ussr|kalendern|math|charts] [--out DIR] [--escalate] [--postcorrect]"
requires: a dedicated local-GPU venv (torch cu128-class, sm_120), the hopperline package, llama.cpp + GGUF roster in Models/
---

**Family**: hopper — engine skill; knowledge map: `/hopper-index` (INDEX.md).
**Downstream**: a finished extraction is landed by /hopper-integrate, then integrated by /kb-integrate-pipeline --engine hopper, then robert-db-build.

# /hopper — Hopper Line v2 (HL2) offline document extraction

Run an **offline, zero-API** PDF→structured-data engine on a **local consumer-GPU machine (32 GB VRAM
class, sm_120)**. One command: PDF/folder in → HDARP-style **4-artifact** output (body text + tables +
equations + figures) + `content_list.json` + chart_data + an Anu-ready KB. Definitive spec:
`HOPPER_LINE_V2_PROTOCOL.md` *(workspace-internal, not shipped in this export; readable
overview: [`docs/frameworks/hopper.md`](../../../frameworks/hopper.md))*.

> **Naming rule (hard):** this is **Hopper / the Hopper Line**, a *distinct engine from HDARP* (which is the
> cloud Claude Read-tool agent). **Never** call any local-model build "HDARP", "Local-HDARP", or "Sraffa N".

## What HL2 does (per-region routing — the R9 rule, real-gold validated)
- **born-digital text** → text-layer bypass (no GPU).
- **table / scanned-English-or-Latin / equation regions** → **GLM-OCR** (MIT) — real-gold Tables TEDS 0.946 ·
  Scanned CER 0.291 · Equations edit-dist 0.266.
- **Cyrillic / non-English-Latin prose** → **dots.ocr** (faithful orthography; no real gold to override).
- **charts** → **Qwen3.6-VL-REAP** (ChartQA 0.85) + chart→data CSV (flagged `chart_estimated`).
- **figure prose** → **gemma-4-26B-A4B**; **gate-fail escalation** → **olmOCR-2-7B**; **reading order** → xy_cut.
Mode C (cheap-default + confidence-gated escalation); low-confidence pages land in `_REVIEW_QUEUE.csv`.

## Runbook

### 1. Preflight (always — single-launch discipline is a HARD rule)
```powershell
$py = "python.exe"
# (a) byte-verify the pinned Blackwell torch build is intact (must print the pinned version, e.g.
#     2.12.0.dev20260408+cu128):
& $py -c "import torch; print(torch.__version__, torch.cuda.is_available())"
# (b) floor exists + GPU FREE (< ~3 GB VRAM, no foreign llama-server):
Test-Path <Hopper> ; nvidia-smi --query-gpu=memory.used,memory.total --format=csv
Get-Process llama-server -ErrorAction SilentlyContinue   # must be empty
# (c) ZERO ZOMBIES from prior Hopper runs — verify NO hopperline / run_rscd / gpu_compose / gpu_namer
#     python is alive. Do NOT trust an empty CIM result if the query took >5s under load.
(Get-CimInstance Win32_Process -Filter "Name='python.exe'" |
   Where-Object { $_.CommandLine -match 'hopperline|run_rscd|gpu_compose|gpu_namer' }).Count   # MUST be 0
```
If torch differs from the pinned build → STOP and report (do not run). If GPU is busy or any of
my run-family pythons are alive → wait or kill PID-scoped first (NEVER `taskkill /F /IM llama-server.exe` —
blanket kill breaks other GPU agents AND your own children, see the 12-zombie incident 2026-05-26).
One resident VLM at a time on 32 GB.

### 2. Run the line
```powershell
cd Hopper
$env:PYTHONUTF8=1 ; $env:PYTHONIOENCODING="utf-8"
& $py -m hopperline run <pdf|folder> --profile <general|ussr|kalendern|math|charts>
#   --cpu          GPU-free born-digital-only pass
#   --no-describe  skip figure description (faster)
#   --pipeline-mode C|A
```
Pick the **profile** by corpus: `general` (any PDF) · `ussr` (Cyrillic scanned yearbooks) · `kalendern`
(Swedish archival tables) · `math` (equation-dense) · `charts` (chart/figure-heavy). Output lands under
`<Outputs>/<doc_id>/` (`content_list.json` = truth; `hdarp/` = 4-artifact view; `tables/`;
`confidence.json` + `_REVIEW_QUEUE.csv`).

### 3. Validate
```powershell
& $py validate.py <Outputs>        # offline KB self-validation (chunk markers, 4 artifacts)
& $py harness/race_e2e.py                           # A.6 guard (engine script, not shipped in this export)
```
A failing A.6 means a born-digital page lost content — investigate before trusting the run.

### 4. (Optional) Hand off to Anu
```powershell
& $py hopper_to_anu.py <Outputs> --kb <ANU_KB>   # KB + _catalogs (incl chart_extracted, flagged)
```

### 5. Report back to the user
Summarize: # docs/pages, born-digital %, which models loaded (swap count), the **confidence histogram** +
`_REVIEW_QUEUE.csv` count (pages needing human review), table/equation/figure counts, and the A.6 result.
Flag any QUARANTINED (corrupt) or low-confidence docs explicitly.

## Rules
- **Foreground only** (per workspace CLAUDE.md); 1–2 GPU models per invocation.
- Estimated chart data is **flagged, never primary**. Low-confidence pages are surfaced, not hidden.
- Real-gold regression gate before trusting champion changes: `harness/hl2_regress.py` *(engine script, not shipped)*.
- This is a **billed-nothing, fully-local** path — ideal for quota-blocked / sensitive / bulk work that
  HDARP (cloud) can't do.
