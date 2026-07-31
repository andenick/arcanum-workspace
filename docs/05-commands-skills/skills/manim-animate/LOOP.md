# LOOP.md — the executable RITL self-correction loop

The heart of `manim-animate`. This is the algorithm an agent runs to take a natural-language
description to a correct, good-looking animation. Two grounded principles drive it: **the loop
beats the model** (RITL/RITL-DOC, arXiv 2604.18364) and **rendering ≠ looking right** (so a
multimodal vision-review pass is mandatory).

```
PY = python.exe
R  = render.py
N_MAX = 5            # hard cap on total render attempts (TheoremExplainAgent / RITL convention)
attempt = 0
```

---

## Step 0 — PLAN (optional)
If the ask is non-trivial (multi-section, a derivation, a model with several moving parts):
emit a short Markdown storyboard (see SKILL.md §b). Skip for a single chart/shape.

## Step 1 — GENERATE
Write a **contract-compliant** `GenScene` to `scene.py`, **code-only** (strip all prose/backticks):
- `from manim import *` then `from arcanum_manim import *`; subclass `ArcanumScene`
  (or `ThreeDScene` for 3D). Class name is **exactly `GenScene`**.
- CE 0.19 idioms only; relative positioning; explicit `z_index`; `FadeOut` between sections;
  `self.wait()` beats. Prefer `Text()` over `MathTex` unless math is required (LaTeX is slow).

## Step 2 — RENDER (fast)
```bash
"$PY" "$R" --code-file scene.py --no-cache -ql        # increment attempt
```
Parse the returned JSON (`status`, `error_tail`, `mp4_path`, `animations`).

## Step 3 — BRANCH ON RESULT

### 3a. status == "error"  →  RITL-DOC repair
1. **Read `error_tail`** — it is already truncated to the **last ~10 lines** (do NOT ask for the
   full log; the tail is the signal — RITL last-10-lines rule).
2. **Identify the failing symbol / message** (e.g. `NameError: name 'ShowCreation'`,
   `error converting to dvi`, `AttributeError: ... 'get_graph'`).
3. **RITL-DOC injection:** grep that symbol in `KNOWLEDGE/manim_ce_0190_api.md`:
   - matches **§1 deprecation delta** → swap old→CE idiom.
   - matches **§3 failure-mode table** → apply the prescribed fix.
   - LaTeX error → raw strings, escape specials, or switch `MathTex`→`Text`.
4. **Regenerate** scene.py with the targeted fix injected. **Go to Step 2.**
5. If `attempt == N_MAX` and still failing → STOP, report the last `error_tail` + the scene to the
   user with the diagnosis. Do NOT fabricate a "success".

### 3b. status == "ok"  →  VISION REVIEW (mandatory)
1. Render the **last frame** as PNG:
   ```bash
   "$PY" "$R" --code-file scene.py --no-cache -ql --frame   # increment attempt
   ```
   (For multi-beat scenes also sample a mid frame: `-n A,B` then `--frame`.)
2. **`Read` the `last_frame_png`** from the JSON (Claude is multimodal — actually look at it).
3. **Critique** against this checklist:
   - **Overlap/collision** — labels/curves/fills crowding? → add `buff`, `arrange`, fix z-order.
   - **Off-screen/clipped** — anything past the frame edge (x≈±7, y≈±4)? → scale-to-fit, `to_edge`.
   - **Legibility** — text ≥30 (body) / ≥40 (title)? high contrast on black bg?
   - **Color semantics** — palette honored (focal TEAL, force RED/YELLOW, secondary GREEN,
     scaffolding GREY)?
   - **Pacing** — abrupt? add `self.wait()` / raise `run_time` / reveal incrementally.
   - **Black/empty frame** — nothing added or all FadeOut'd? → fix construction.
4. **If issues found** → fix scene.py → **go to Step 2** (counts toward N).
   **If clean** → proceed to Step 4.

## Step 4 — STEP UP QUALITY → FINAL
Frame reads clean. Render the deliverable at high quality, into the canonical output dir:
```bash
"$PY" "$R" --code-file scene.py -qh --media-dir "Outputs"
```
(Use `--gif` / `--transparent` per the request; transparent = Cairo only.) Copy/confirm the mp4
in `Outputs`, then report the final path to the user.

---

## Invariants (state these to yourself each iteration)
- **`N ≤ 5`** total render attempts (generate-fix retries + vision-fix retries combined). At the
  cap, stop and report — never silently give up, never fabricate a success.
- **Traceback-tail rule:** only the last ~10 lines feed the fix (render.py enforces this). Full
  build logs saturate context and are not needed.
- **Vision pass is mandatory** on the first clean render — a 0 exit code is necessary but not
  sufficient. You must *see* a frame before declaring done.
- **Non-destructive / never fabricate:** if you cannot make it render or look right within N, hand
  back the scene + diagnosis. Do not invent an output path or claim a render that did not happen.
- **Caching:** `--no-cache` during the loop so edits actually show; cache ON is fine only for the
  final if iterating is over.
