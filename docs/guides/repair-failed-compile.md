# Repair a Failed Compile

Compile runs are long-running Claude sessions that can fail mid-way through — a timeout, a lint loop that didn't converge, a partial artifact from an interrupted session. Repair mode diagnoses the failure and resumes from the first broken step.

---

## When to use repair mode

Use repair mode instead of restarting from scratch whenever:

- The compile timed out (the console showed `Process timed out after N seconds`)
- A lint (Step 7) failed after multiple repair rounds
- The process was interrupted (Ctrl+C, system shutdown)
- A `*_mellea/` directory exists but is incomplete (missing `pipeline.py`, missing `melleafy.json`, etc.)
- Exit code 11 or 12 from a previous compile

---

## Run repair mode

```bash
mellea-skills compile <spec_path> --repair-mode
```

Or the short form:

```bash
mellea-skills compile <spec_path> -r
```

Point it at the same path as the original compile — the skill root, a single spec file, or the `*_mellea/` package directory.

---

## What repair mode does

The `/mellea-fy-repair` slash command (invoked by `--repair-mode`) runs in four phases:

**Phase 1: Artifact audit** — reads every intermediate artifact (`classification.json`, `inventory.json`, `element_mapping.json`, `dependency_plan.json`, `config_emission.json`, `fixtures_emission.json`) and every generated Python file. Classifies each as `valid`, `partial`, `missing`, or `corrupt`.

**Phase 2: First broken step detection** — identifies the earliest step whose output is not `valid`. Steps before it can be reused; this step and everything after must be re-run.

**Phase 3: Resume** — re-runs the pipeline from the first broken step, consuming the valid earlier outputs as grounding. Does not restart from Step 0.

**Phase 4: Halt on unrecoverable failures** — if the failure is not auto-recoverable (spec-level contradictions, `session-boundary` lint failures, `category-specific` security findings), repair halts and writes a diagnostic report. These require human intervention before resuming.

---

## Understanding failure modes

### Timeout failures

The most common failure type. The compile session hit the 75-minute limit mid-way through code generation (Step 5 is typically the longest).

Repair mode picks up from the last valid step. Often one more repair run is enough to complete.

Increase the timeout for very large specs:

```bash
mellea-skills compile <spec_path> --timeout 9000  # 150 minutes
```

### Lint failures (exit code 11)

The Step 7 lints found issues the repair loop (2 rounds) couldn't fix. Check `intermediate/step_7_report.json` for which lints failed:

```bash
cat <package_dir>/intermediate/step_7_report.json | python -m json.tool
```

Common fixable lint failures:
- `import-soundness` — a generated file imports a symbol that doesn't exist (rename mismatch). Repair mode usually fixes this.
- `stub-presence` — stubs exist but aren't documented in `SETUP.md §8`. Repair mode regenerates `SETUP.md`.

Non-auto-recoverable lint failures:
- `session-boundary` — `run_pipeline` mixes Mellea sessions in a way that would cause silent context leakage. Repair halts and explains. Requires spec restructuring.
- `category-specific` — a generated tool call violates a security constraint (e.g., arbitrary command execution). Repair halts. Requires spec revision.

### Smoke check failures (exit code 12)

A fixture raised during the post-lint run. Check the error:

```bash
mellea-skills run <package_dir> --fixture <fixture_id>
```

If it's a `NotImplementedError`, a stub was hit — see [Fill stubs](fill-stubs.md).

If it's a `ValidationError`, the model didn't produce output matching the schema. Try increasing `LOOP_BUDGET` in `config.py` and re-running `mellea-skills validate`.

---

## The `.melleafy-partial/` directory

If a compile fails at Step 3 or later (after skeleton files exist), the compiler preserves a `.melleafy-partial/` directory alongside the spec. This contains whatever was produced before failure.

Repair mode can work from this directory directly:

```bash
mellea-skills compile skills/myskill/.melleafy-partial --repair-mode
```

---

## Diagnostic: which step failed?

Read the intermediate artifacts manually to understand what completed:

```
intermediate/
  classification.json   → exists + valid?  Step 0 complete
  inventory.json        → exists + valid?  Steps 1a + 1b complete
  element_mapping.json  → exists + valid?  Step 2 complete
  dependency_plan.json  → exists + valid?  Step 2.5 complete
  fixtures_emission.json → exists?         Step 4 complete
  step_7_report.json    → exists?          Step 7 ran
```

If `classification.json` is missing or empty, the entire compile failed at Step 0 — check Claude connectivity.

If `step_7_report.json` exists but Python files are malformed, Step 5 (code generation) produced broken output. Repair mode targets this step specifically.

---

## Changing the model for repair

Use a more capable model for a compile that previously produced low-quality output:

```bash
mellea-skills compile <spec_path> --repair-mode --model claude-opus-4-5
```

Repair mode with a more capable model often resolves lint failures that Sonnet couldn't fix in the repair round.
