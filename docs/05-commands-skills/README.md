# Commands & Skills — Operational Templates

The documents in this folder are **operational runbook templates** exported from the owner's private
research workspace. They document *how each command and skill is invoked and gated* — they are not
runnable out of this repository.

What that means for a reader:

- **Engine scripts are not shipped.** The runbooks reference engine implementations
  (`preparehdarp_v6_engine.py`, `sraffa40_processor.py`, the `hopperline` package, `rdb_*.py`,
  `scripts/render.py`, and similar) that live in the private workspace. They are named so the method
  is reproducible in principle, not downloadable here.
- **Internal standards are not shipped.** Files cited as "canonical standard" or "definitive spec"
  (for example `HOPPER_LINE_V2_PROTOCOL.md`, `KB_INTEGRATION_PIPELINE_BUILD_PLAN.md`,
  `A4_TRIAGE_HANDOFF.md`, `GPU_NAMING_PIPELINE_PLAN.md`) are workspace-internal. Where a readable
  equivalent exists in this export, the runbook says so and links it.
- **Readable overviews live one level up.** [`docs/frameworks/`](../frameworks/) carries the
  framework-level documentation (HDARP, Hopper Line v2, KBIP, Robert DB, Anu, AUKA) written for
  readers outside the workspace.
- **Runnable public code lives in the project repositories.** The curated entries under
  [`projects/`](../../projects/) link the public repos and sites where the methods are applied and
  can actually be cloned and run.

Version note: these templates were exported at HDARP v6.4 / Anu v12.4 (2026-10-02, with 2026-10-04
audit annotations).
