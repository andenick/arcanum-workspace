---
name: manim
description: "Generate a Manim animation/video of an economic or mathematical concept from a natural-language description, via the manim-animate skill's render→read→fix (RITL) loop."
when-to-use: "User typed /manim <description>, or asked to animate/visualize/make a video of a model, equation, chart, or economic concept."
search-hints: "manim animate animation video render economic math visualization explainer chart equation derivation mp4 gif"
allowed-tools: Skill, Bash, Read, Write, Edit, Glob, Grep
argument-hint: "<natural-language description of the animation to create>"
---

<!--
COMMAND STUB — /manim
=====================
Thin entry point into the `manim-animate` skill (Manim Framework v1.0).
It does NOT reimplement the workflow — it just routes the user's description into the skill.
Kept deliberately minimal.
-->

# /manim — natural-language → Manim animation

Invoke the **`manim-animate`** skill on the user's description (the `$ARGUMENTS`).

The skill drives the agentic Renderer-In-The-Loop (RITL) workflow end to end:
1. (optional) plan a short storyboard for non-trivial asks;
2. generate a contract-compliant `GenScene` (Manim **CE 0.19** idioms, `from arcanum_manim import *`,
   subclass `ArcanumScene`, relative positioning, explicit z-index, `FadeOut` cleanup, `self.wait()` beats);
3. render with `scripts/render.py` using the canonical interpreter
   `python.exe`;
4. on failure, inject the relevant `KNOWLEDGE/manim_ce_0190_api.md` deltas + the last-~10-line
   traceback and regenerate (cap **N ≤ 5**);
5. on success, render the last frame and **Read it back** (multimodal vision review) to catch
   overlap / off-screen / legibility / color issues, fix if needed;
6. step up to `-qh` and write the final mp4 to `Outputs`.

**Action:** call the `manim-animate` skill, passing `$ARGUMENTS` as the animation description.
Then report the final artifact path (and any plan/critique notes) back to the user.

If `$ARGUMENTS` is empty, ask the user what concept they want animated before proceeding.
