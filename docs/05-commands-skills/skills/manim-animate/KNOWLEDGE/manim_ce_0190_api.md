# Manim CE 0.19.0 — API Cheat-Sheet (RITL-DOC grounding)

> **Pinned target: Manim Community Edition v0.19.0.** This file is the doc-grounding
> corpus for the RITL-DOC step: on a render failure, the agent greps the class/idiom that
> failed, re-reads the relevant rows below, and regenerates. **Version drift (ManimGL ↔ old-CE
> ↔ current CE) is the #1 failure source** — section 1 exists to kill it.
>
> Render interpreter is always `python.exe`.

---

## 1. DEPRECATION DELTA — old/ManimGL → CE 0.19 (grep here first on NameError)

| Old / ManimGL / pre-0.19 | CE 0.19.0 (use this) | Note |
|---|---|---|
| `ShowCreation(m)` | `Create(m)` | The single most common LLM error. |
| `axes.get_graph(f)` | `axes.plot(f)` | Returns a `ParametricFunction`. |
| `axes.get_graph_label` | `axes.get_graph_label` (still exists) but pass the *plot* mobject | unchanged name, check signature |
| `axes.get_area(graph, ...)` | `axes.get_area(graph, x_range=(a,b))` | `x_range` not `dx`/`t_min`. |
| `TexMobject(...)` | `MathTex(...)` | math mode. |
| `TextMobject(...)` | `Tex(...)` | text mode (LaTeX) / use `Text()` for non-LaTeX. |
| `CONFIG = {...}` class dict | pass args to `__init__` / set attrs | CE removed the `CONFIG` pattern entirely. |
| `self.play(ApplyMethod(m.shift, RIGHT))` | `self.play(m.animate.shift(RIGHT))` | `.animate` syntax. |
| `m.scale_in_place(2)` | `m.scale(2)` | scales about center by default. |
| `GraphScene` | `Scene` + `Axes` mobject | `GraphScene` removed in CE. |
| `MovingCameraScene` self.camera_frame | `MovingCameraScene` + `self.camera.frame` | attribute path changed. |
| `ContinualAnimation` | `add_updater` / `always_redraw` | continual-anim class removed. |
| `ShowCreationThenDestruction` | `ShowPassingFlash` | renamed family. |
| `DrawBorderThenFill` | `DrawBorderThenFill` | still valid in CE. |
| `get_center()`, `get_top()` etc. | unchanged | positional getters OK. |
| `VMobject.points` direct manip | prefer `set_points_*` helpers | low-level API churns. |
| `manimlib` import | `from manim import *` | never `import manimlib`. |
| `--high_quality` / `-i` (gif) | `-qh` / `--format gif` | CLI flags changed. |
| `numberline.number_to_point` | `axes.c2p(x, y)` / `axes.coords_to_point` | coordinate mapping. |

**Rule:** if a `NameError`/`AttributeError` names a symbol in the left column, swap to the right
column and regenerate. If a symbol is unknown to you, do NOT invent it — use a built-in from §2.

---

## 2. Core class / idiom reference (~30 most-used)

### Scenes & camera
- **`Scene`** — base. Override `construct(self)`. Drive with `self.play(...)`, `self.add(...)`,
  `self.wait(t)`, `self.remove(...)`. **The skill's contract class is always `class GenScene(Scene)`**
  (or `GenScene(ArcanumScene)` / `GenScene(ThreeDScene)`).
- **`ThreeDScene`** — `self.set_camera_orientation(phi=70*DEGREES, theta=-45*DEGREES)`;
  `self.begin_ambient_camera_rotation(rate=0.1)`; `self.move_camera(phi=..., theta=..., run_time=3)`.
  Use `ThreeDAxes` not `Axes`. Add labels with `self.add_fixed_in_frame_mobjects(label)`.
- **`MovingCameraScene`** — `self.camera.frame.animate.scale(0.5).move_to(target)` to zoom/pan.

### Coordinate systems & plotting
- **`Axes(x_range=[xmin,xmax,step], y_range=[...], axis_config={"include_numbers":True})`**.
  Plot with `graph = axes.plot(lambda x: x**2, color=YELLOW, x_range=[0,3])`.
  Map data→screen with `axes.c2p(x, y)` (coords-to-point); screen→data `axes.p2c(point)`.
  Label axes: `axes.get_axis_labels(x_label="t", y_label="Y")`.
  Label a curve: `axes.get_graph_label(graph, label="f(x)", x_val=2)`.
- **`axes.plot_line_graph(x_values=[...], y_values=[...], line_color=TEAL, add_vertex_dots=False)`**
  — for data series (returns a `VDict` with `"line_graph"`, `"vertex_dots"`). The canonical way to
  draw a time series from arrays. (No pandas→manim plugin exists; pass python lists/np arrays.)
- **`axes.get_area(graph, x_range=(a, b), color=BLUE, opacity=0.5)`** — shaded region under a curve
  (integrals, consumer/producer surplus). For area between two curves:
  `axes.get_area(graph1, bounded_graph=graph2, x_range=(a,b))`.
- **`NumberPlane(...)`** — gridded background plane (vector fields, transforms).
- **`BarChart(values=[...], bar_names=[...], y_range=[0,10,2], bar_colors=[...])`** — bar charts.
  Animate updates with `chart.animate.change_bar_values([...])`. The **bar-chart-race** idiom:
  loop over time rows, `self.play(chart.animate.change_bar_values(row), rate_func=linear)`.
  Get labels via `chart.get_bar_labels()`.

### Text & math
- **`Text("plain", color=WHITE, font_size=36)`** — **prefer this over MathTex when no math is needed**
  (no LaTeX compile → much faster loop). Supports `t2c={"word": RED}` for per-substring color.
- **`MathTex(r"x^2 + y^2 = r^2")`** — math mode. **ALWAYS raw strings** (`r"..."`) so backslashes
  survive. Multiple args become indexable submobjects: `MathTex(r"a", r"+", r"b")[0]`.
  Use `.set_color_by_tex("x", RED)` to color a token.
- **`Tex(r"Some \textbf{LaTeX} text")`** — text-mode LaTeX (full prose with math). Raw strings.
- **LaTeX is SLOW on first compile (minutes).** Keep caching ON between iterations; prefer `Text()`;
  batch math into fewer `MathTex` calls; avoid exotic packages.
- **`DecimalNumber(3.14, num_decimal_places=2)`** — live-updatable number. Pair with a
  `ValueTracker` + `add_updater(lambda m: m.set_value(tracker.get_value()))` for counting readouts.
- **`MarkupText`** — pango markup (`<b>`, color spans) without LaTeX.

### Mobjects & grouping
- **Built-in shapes:** `Circle`, `Square`, `Rectangle`, `Line`, `Arrow`, `Dot`, `Polygon`,
  `RegularPolygon`, `Arc`, `Triangle`, `DoubleArrow`, `Vector`, `SurroundingRectangle(m, buff=0.1)`,
  `Brace(m, direction=DOWN)` (+ `brace.get_text("label")`).
- **`VGroup(a, b, c)`** — group vector mobjects; transform/position as one. Self-updating economic
  models subclass `VGroup` (see arcanum_manim). `Group(...)` for mixed (incl. images).
- **`m.arrange(DOWN, buff=0.4)`** — lay out a VGroup's children along a direction.
  **`m.arrange_in_grid(rows=2, cols=3)`** — grid layout.

### Positioning (prefer RELATIVE; never hard-code pixel coords)
- `m.next_to(other, RIGHT, buff=0.5)` · `m.to_edge(UP)` · `m.to_corner(UL)` · `m.move_to(ORIGIN)`
  · `m.shift(2*RIGHT + UP)` · `m.align_to(other, LEFT)` · `m.center()`.
- Directions: `UP DOWN LEFT RIGHT ORIGIN UL UR DL DR`. Frame edges via `to_edge`/`to_corner`.
- **`z_index`** — set `m.set_z_index(2)` or `m.z_index = 2` so later/overlapping mobjects draw on top.
  Always set explicit z-order when mobjects overlap (labels over fills, curves over grids).

### Animations
- **Creation:** `Create(m)` (draw), `Write(text_or_tex)` (write-on), `FadeIn(m, shift=UP)`,
  `GrowFromCenter(m)`, `DrawBorderThenFill(m)`, `SpiralIn(m)`.
- **Removal / cleanup:** `FadeOut(m)`, `Uncreate(m)`. **Between sections, FadeOut the previous
  group** so the frame doesn't accumulate clutter: `self.play(FadeOut(VGroup(*self.mobjects)))`.
- **Transform family:**
  - `Transform(a, b)` — morph `a` into `b`'s shape; `a` remains the on-screen mobject.
  - `ReplacementTransform(a, b)` — morph and **replace** `a` with `b` (b is now on screen). Prefer
    this when you keep referencing the target afterward.
  - `TransformMatchingTex(eq1, eq2)` — morph matching LaTeX tokens between two `MathTex`; ideal for
    **step-by-step equation derivations** (Robert→Manim use case). Use consistent token splitting.
  - `TransformMatchingShapes(a, b)` — shape-based matching for non-Tex.
- **Movement / emphasis:** `m.animate.shift(...)/scale(...)/set_color(...)/rotate(...)`,
  `Indicate(m)`, `Circumscribe(m)`, `Flash(point)`, `Wiggle(m)`, `FocusOn(point)`.
- **`self.wait(0.5)`** — pacing beats. Insert generous waits between logical steps (3b1b pacing).
- `rate_func=` — `linear`, `smooth` (default), `there_and_back`, `rush_into`, `ease_in_out_sine`.
  **Bar-chart-race uses `linear`.**

### Live / data-bound updates (the 3b1b self-updating idiom)
- **`ValueTracker(0)`** — a single animatable scalar. `tracker.animate.set_value(10)`; read with
  `tracker.get_value()`.
- **`always_redraw(lambda: axes.plot(lambda x: tracker.get_value()*x))`** — rebuilds the mobject
  every frame from current state. Best for graphs/labels that track a `ValueTracker` or model state.
- **`m.add_updater(lambda mob, dt: mob.rotate(dt))`** — per-frame mutation (dt = seconds since last
  frame). Remove with `m.clear_updaters()`. Use `always_redraw` for "rebuild from scratch",
  `add_updater` for "nudge in place".

---

## 3. FAILURE-MODE → FIX (consult on render error or bad-looking frame)

| Symptom (from error_tail or vision review) | Likely cause | Fix |
|---|---|---|
| `NameError: name 'ShowCreation'/'get_graph'/...` | ManimGL/deprecated API | Swap per §1 delta table, regenerate. |
| `error converting to dvi` / LaTeX compile error | bad LaTeX, unescaped char, missing pkg | Use **raw strings** `r"..."`; escape `% & # _ { }`; simplify; for non-math use `Text()` not `MathTex`. Compile the exact string standalone to isolate. |
| `LaTeX ... not found` / `latex failed` | MiKTeX package missing | Avoid exotic packages; stick to amsmath/amssymb; use `Text()` where possible. |
| `AttributeError: 'Axes' object has no attribute 'get_graph'` | old plotting API | `axes.plot(...)` not `get_graph`. |
| Output is a **black/empty frame** | nothing added, or all FadeOut'd, or off-screen | Ensure `self.add`/`self.play(Create(...))`; check positions are within frame (~`-7..7` x, `-4..4` y). |
| Mobjects **overlap / collide** (vision) | absolute positioning, no buff | Use `next_to(..., buff=0.5)`, `arrange`, `VGroup().arrange`; add `SurroundingRectangle` spacing. |
| Mobjects **off-screen / clipped** (vision) | too large / placed past frame edge | `m.scale_to_fit_width(config.frame_width-1)`; `to_edge` with `buff`; `self.camera` zoom; group + `.move_to(ORIGIN)`. |
| Text **unreadable / too small** (vision) | font_size too small, low contrast | Raise `font_size` (≥30 for body, ≥40 titles); ensure contrast vs black bg (avoid dark-on-black). |
| **Wrong color semantics** (vision) | arbitrary colors | Use the semantic palette (focal TEAL, force/emphasis RED/YELLOW, secondary GREEN, scaffolding GREY) — prefer `arcanum_manim` theme. |
| `TypeError: ... missing required positional` | wrong CE signature | Re-read the class row in §2; do not guess kwargs. |
| Animation **too fast / abrupt** (vision) | no waits, short run_time | Add `self.wait()` beats; raise `run_time=`; reveal incrementally. |
| Render **times out** | LaTeX first-compile or heavy scene | Keep caching ON; `-ql` + `-s` for layout checks; split into `-n start,end`; reduce MathTex count. |
| Edits **don't show** | stale cache | During the loop pass `--disable_caching`; `--flush_cache` to reset. |
| `Transform` looks like a **jump cut** | mismatched submobject counts | Use `ReplacementTransform`/`TransformMatchingTex`; split Tex into matching tokens. |
| `ImportError`/`ModuleNotFoundError` | wrong interpreter or invented import | Render with `.venv-manim` python; only `from manim import *` (+ `from arcanum_manim import *`, numpy). Never invent helper modules. |

---

## 4. CLI flags the loop uses (via scripts/render.py)

| Flag | Meaning |
|---|---|
| `-ql` / `-qm` / `-qh` / `-qk` | quality 854×480 / 720p / 1080p / 4k. **Loop with `-ql`; final `-qh`.** |
| `-s` | render & save only the **last frame** as PNG (fast layout + vision-review check). |
| `-n A,B` | render only animations A..B (partial scene, fast debugging). |
| `--format gif` | output GIF. |
| `--transparent` | alpha background — **Cairo renderer only** (OpenGL drops alpha). |
| `--disable_caching` | force re-render (use during the loop so edits show). |
| `--flush_cache` | clear the cache. |
| `--media_dir DIR` | where videos/images land. |
| `--fps 10` | low fps to speed loop iterations. |

Frame extent (1080p/default): x ≈ −7.1..7.1, y ≈ −4..4, `config.frame_width≈14.2`,
`config.frame_height=8`. Keep content inside with a margin.
