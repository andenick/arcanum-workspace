---
name: manim-animate
description: Generate/create a Manim animation or video of an economic or mathematical concept from a natural-language description, using an agentic render→read→fix self-correction loop. Use when the user asks to animate, visualize, or make a video/animation/explainer of a model, equation, chart, derivation, or economic idea (supply-demand, IS-LM, Solow, Phillips curve, a FRED time series, an extracted equation, etc.).
when-to-use: '"animate <concept>", "make a manim video of <X>", "create an animation of the <model>", "visualize this equation/derivation", "turn this chart into an animated explainer".'
search-hints: "manim animation video render economic math visualization RITL generative-manim GenScene scene mp4 gif 3blue1brown axes equation derivation chart explainer"
version: "1.0"
part-of: Manim Framework v1.0
requires-library: arcanum_manim (sibling house-style package at arcanum_manim)
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
argument-hint: "<natural-language description of the animation>"
---

# manim-animate — natural-language → economics animation (RITL)

Turn a plain-English description into a rendered Manim animation by **writing a scene,
rendering it, reading the result back, and fixing it** — the agentic Renderer-In-The-Loop
(RITL) self-correction loop. This is "generative-manim, done natively for Claude Code": we
borrow its *ideas* (a strict code contract, a renderer-as-verifier loop, API-doc grounding),
not its Flask/Vercel stack. **The loop beats the model**, and **rendering ≠ looking right** —
so Claude renders frames and *reads them back* (native multimodal) before declaring done.

## Hard environment facts (obey)
- **Interpreter for ALL renders:** `python.exe`
  (Manim **CE 0.19.0**). NEVER the shared `.venv`. `scripts/render.py` already
  resolves this.
- **Manim CE v0.19 idioms ONLY** — `Create` not `ShowCreation`, `axes.plot` not `get_graph`,
  `always_redraw`, `TransformMatchingTex`. The deprecation deltas live in
  `KNOWLEDGE/manim_ce_0190_api.md` §1.
- **LaTeX first-compile is SLOW (minutes).** Iterate with caching ON, **prefer `Text()` over
  `MathTex`** where reasonable, use `-ql` (854×480) and `-s` (single last frame) for layout
  checks; only step up to `-qh` for the final.
- **House-style library:** generated scenes should `from arcanum_manim import *` and subclass
  **`ArcanumScene`** (a sibling agent owns this package — do NOT author it here; assume its
  public API: theme/palette, self-updating econ-model VGroups, live readouts, chart presets).
  If the import is unavailable, degrade gracefully to `from manim import *` + the built-in
  semantic colors noted in the KNOWLEDGE sheet.

---

## (a) THE CODE CONTRACT (lifted from generative-manim's GenScene)

Every scene the agent emits MUST satisfy this contract so the render command is **mechanical**:

1. **Exactly one Scene subclass, named `GenScene`.** Fixed name → the render command is always
   `manim <file> GenScene`. (Use `class GenScene(ArcanumScene)`, or `GenScene(ThreeDScene)` for 3D.)
2. **`construct(self)` is the only method body.** No top-level execution, no `if __name__`.
3. **Code-only output.** Strip ALL prose/markdown/backticks before writing the `.py`. The file
   must be a runnable Python module and nothing else.
4. **Imports:** `from manim import *`, then `from arcanum_manim import *` (preferred), plus
   `numpy as np` if needed. **No invented helper modules**; only built-in mobjects +
   `arcanum_manim` primitives.
5. **Relative positioning only** — `next_to` / `arrange` / `to_edge` / `move_to(ORIGIN)`.
   Never hard-code pixel coordinates.
6. **Explicit `z_index`** whenever mobjects overlap (labels over fills, curves over grids).
7. **Scene cleanup between sections** — `FadeOut` the previous group before the next so the
   frame doesn't accumulate clutter.
8. **Pacing beats** — `self.wait(...)` between logical steps (generous, 3b1b-style).

Prefer `arcanum_manim` primitives (ArcanumScene base, semantic palette, econ-model VGroups,
chart presets) over re-deriving layout by hand. Skeleton:

```python
from manim import *
from arcanum_manim import *   # ArcanumScene, theme palette, models, readouts, charts

class GenScene(ArcanumScene):
    def construct(self):
        title = Text("Supply & Demand", color=TEAL).to_edge(UP)
        self.play(Write(title))
        # ... build, reveal incrementally, self.wait() beats ...
        self.play(FadeOut(VGroup(*[m for m in self.mobjects if m is not title])))
        self.wait(0.5)
```

---

## (b) OPTIONAL PLAN→CODE (two-stage, for non-trivial asks)

For anything beyond a single chart (multi-section explainers, derivations, models with several
moving parts), first emit a **short Markdown storyboard**, then translate it to code
(TheoremExplainAgent pattern). Single-shot is fine for a simple chart/shape.

Storyboard template (keep it short — it's scaffolding, not a deliverable):
```
TOPIC: <one line>
PALETTE: focal=TEAL, emphasis/force=RED/YELLOW, secondary=GREEN, scaffolding=GREY
BEATS:
  1. <on-screen elements> | <LaTeX if any> | <animation> | <wait>
  2. ...
CLEANUP: FadeOut between beats N→N+1
```

---

## (c) THE RITL LOOP

The executable self-correction loop. **Full spec in `LOOP.md`** — summary:

```
plan (optional)  →  generate GenScene code (contract-constrained)
  → render  scripts/render.py --code-file scene.py --no-cache -ql
      ├─ status=error → read error_tail (last ~10 lines)
      │                 → grep the failing symbol in KNOWLEDGE/manim_ce_0190_api.md (RITL-DOC)
      │                 → regenerate with the delta/fix injected → retry   [cap N = 5]
      └─ status=ok → VISION REVIEW: render last frame  --frame  (PNG)
                     → Read the PNG back (Claude is multimodal)
                     → critique: overlap? off-screen? legibility? wrong color?
                       ├─ issues → fix → re-render   [counts toward N]
                       └─ clean  → step up quality (-qh) → final mp4 to Outputs
```

**Non-negotiables (from the literature):** `N ≤ 5` total attempts; **truncate tracebacks to the
tail** (`render.py` already returns only the last ~10 lines — full logs saturate context); the
vision pass is **mandatory** (code that compiles ≠ code that looks right).

## (d) THE VISION-REVIEW PASS

After a successful render, call `render.py --frame` to emit the **last frame as PNG**, then
**`Read` that PNG** and critique it as you would a designer's proof:
- **Overlap/collision** — do labels, curves, fills crowd each other? → add `buff`, `arrange`, z-order.
- **Off-screen/clipping** — is anything past the frame edge? → scale-to-fit, `to_edge` with buff.
- **Legibility** — is text big enough and high-contrast on the black background?
- **Color semantics** — do colors follow the palette (focal/force/secondary/scaffolding)?
- **Pacing** — for multi-beat scenes, sample a mid frame too (`-n A,B` then `--frame`).

If issues are found, fix and re-render (counts toward N). Only when the frame reads clean do you
step up to final quality.

## (e) SANDBOX POSTURE

Rendering executes LLM-written Python. **Default: local render in a disposable per-run workdir**
on this trusted single-user machine — `render.py` already writes the scene to a fresh
`tempfile.mkdtemp()` and cleans it up. For **untrusted/batch input**, opt into the official Manim
**Docker** image instead (mount the workdir, run `manimcommunity/manim` — `--docker` is a documented
escalation, not wired by default here). Never render attacker-controlled code on the host.

## (f) OUTPUT CONVENTIONS

- **Final mp4** → `Outputs` (canonical plural `Outputs`). Dev renders live
  in the disposable workdir / temp; copy the approved artifact to `Outputs` at the end.
- **Quality:** `-ql` during the loop, `-qh` for the final deliverable (`-qk` 4k only on request).
- **GIF:** `--gif` (`--format gif`). **Transparent:** `--transparent` — **Cairo renderer only**
  (OpenGL drops alpha → use Cairo, the stable default).
- Report the final artifact path back to the user.

---

## Quick start (the mechanical commands)

```bash
PY="python.exe"
R="render.py"

# 1. write GenScene to scene.py (code-only), then loop-render (fast):
"$PY" "$R" --code-file scene.py --no-cache -ql
# 2. vision-review last frame:
"$PY" "$R" --code-file scene.py --no-cache -ql --frame   # → last_frame_png in JSON; Read it
# 3. final:
"$PY" "$R" --code-file scene.py -qh --media-dir "Outputs"
```

`render.py` returns structured JSON: `{status, mp4_path|null, last_frame_png|null,
error_tail|null, animations, returncode, scene, workdir, cmd}`. Drive the loop off `status` and
`error_tail`. Self-test the wrapper anytime with `--self-test`.

## Files in this skill
- `SKILL.md` — this file (workflow, contract, vision pass, conventions).
- `LOOP.md` — the executable RITL loop spec (the heart of the skill).
- `KNOWLEDGE/manim_ce_0190_api.md` — CE 0.19 cheat-sheet + deprecation deltas + failure→fix table.
- `scripts/render.py` — stdlib-only render wrapper (temp-file → subprocess → progress parse →
  error-tail / last-frame PNG → structured JSON). `--self-test` proves both paths.

## Provenance
Ideas borrowed (not the stacks): generative-manim's GenScene contract + `get_preview` loop
(Apache-2.0), arXiv 2604.18364 (RITL / RITL-DOC, last-10-lines feedback), TheoremExplainAgent
(plan→code, N=5), makefinks/manim-generator (writer/reviewer + vision). House style = 3Blue1Brown
pedagogy via the sibling `arcanum_manim` library.
