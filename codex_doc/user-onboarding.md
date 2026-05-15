# User onboarding

Use Mellea Skills Compiler when you have a skill spec and want a runnable, typed, governable Mellea package.

This guide assumes you are in the repository root.

## Prerequisites

| Dependency | Why it is needed | Verify |
|---|---|---|
| Python 3.11 to below 3.14.4 | Package requirement in `pyproject.toml`. | `python --version` |
| Editable package install | Provides the `mellea-skills` CLI. | `pip install -e .` |
| Claude Code and Anthropic-compatible credentials | `mellea-skills compile` invokes `claude -p` with the `/mellea-fy` command. | `claude --version` and `ANTHROPIC_API_KEY` or gateway env vars |
| Ollama endpoint | Default risk and runtime backend. | `ollama list` and `OLLAMA_API_URL` |
| `granite3.3:8b` | Default risk model and compiled-skill model. | `ollama pull granite3.3:8b` |
| `ibm/granite3.3-guardian:8b` | Default Guardian risk assessment model. | `ollama pull ibm/granite3.3-guardian:8b` |

## Install

1. Create and activate a virtual environment.

```bash
python -m venv .venv
```

2. Install the package in editable mode.

```bash
pip install -e .
```

3. Confirm the CLI is available.

```bash
mellea-skills --help
```

Expected result: Typer prints commands including `compile`, `validate`, `run`, `ingest`, `certify`, and `export`.

## Fastest productive path

1. Run a precompiled example first.

```bash
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city --no-guardian
```

This loads `examples/weather/weather_mellea/pipeline.py`, loads fixtures from `fixtures/`, selects `rain_check_city`, and calls `run_pipeline(query=...)`.

2. Compile a spec.

```bash
mellea-skills compile skills/weather/spec.md --no-run
```

`--no-run` skips the post-compile fixture smoke check. Use it when the backend is unavailable or you only want generated files first.

3. Validate the compiled package.

```bash
mellea-skills validate skills/weather/weather_mellea --no-run
```

This runs Python structural lints in `src/mellea_skills_compiler/compile/lints.py` and writes `intermediate/step_7_report.json`.

4. Run a fixture.

```bash
mellea-skills run skills/weather/weather_mellea --fixture rain_check_city --no-guardian
```

5. Generate risk policy and certification artifacts.

```bash
mellea-skills certify skills/weather/weather_mellea --fixture rain_check_city
```

This writes artifacts under `skills/weather/audit/`.

## What the main commands do

| Command | Input | Output | Use it when |
|---|---|---|---|
| `mellea-skills compile` | A `.md` spec or skill directory. | A `<name>_mellea/` generated package. | You want to turn a skill spec into runnable Mellea code. |
| `mellea-skills validate` | A compiled package directory. | `intermediate/step_7_report.json` and optionally `step_7b_report.json`. | You changed generated code or want to confirm package structure. |
| `mellea-skills run` | A compiled package plus fixture id. | Console output from `run_pipeline(...)`; optional Guardian/audit hooks. | You want to execute a known example input. |
| `mellea-skills ingest` | A raw spec file. | `audit/policy_manifest.json`, `POLICY.md`, `CERTIFICATION.md` from static analysis. | You want risk policy without running the compiled package. |
| `mellea-skills certify` | A compiled package. | Policy, audit trail, pipeline report, certification report. | You want end-to-end governance evidence. |
| `mellea-skills export` | A compiled package and target. | A target-specific adapter folder. | You want to wrap the compiled package for LangGraph, MCP, or Claude Code. |

## What a compiled package contains

| File or directory | Meaning |
|---|---|
| `pipeline.py` | The runnable orchestration. It defines `run_pipeline(...)`. |
| `schemas.py` | Pydantic data models for structured inputs, outputs, and phase outputs. |
| `slots.py` | `@generative` slots used for typed extraction or classification. |
| `requirements.py` | Generated validators and behavioral requirements when the spec needs them. |
| `tools.py` or `constrained_slots.py` | Deterministic helpers, external tool wrappers, or stubs. |
| `config.py` | Runtime constants, rendered by deterministic writer from `intermediate/config_emission.json`. |
| `fixtures/` | Runnable fixture factories, rendered by deterministic writer from `intermediate/fixtures_emission.json`. |
| `melleafy.json` | Package manifest consumed by exporter and downstream tooling. |
| `mapping_report.md` | Explanation of how source spec elements mapped to generated components. |
| `SETUP.md` | Backend, dependency, stub, and integration notes for that package. |
| `intermediate/` | Compiler IR and diagnostics. |

## The first thing to inspect after compile

Start with these files:

1. `intermediate/step_7_report.json` for structural lint results.
2. `intermediate/step_7b_report.json` for fixture smoke-check result if `--no-run` was not used.
3. `SETUP.md` for unresolved stubs, env vars, and external tools.
4. `mapping_report.md` for the compiler's source-to-code decisions.
5. `melleafy.json` for entry signature, package name, modality, env vars, and categories.

## Common first-run failures

| Symptom | Likely cause | Fix |
|---|---|---|
| `No claude models available with your API key` | Anthropic-compatible credentials are missing or invalid. | Set `ANTHROPIC_API_KEY` or the gateway variables documented in `README.md`. |
| Smoke check is skipped as backend unreachable | Ollama endpoint is not running or auth failed. | Start Ollama, set `OLLAMA_API_URL`, and retry `mellea-skills validate <pkg>`. |
| `No fixtures directory found` | The path points to the skill root instead of the generated package, or compile did not finish. | Pass the `<name>_mellea/` directory. |
| `No run_* function found` | Generated `pipeline.py` does not expose the expected entry point. | Re-run compile or inspect `pipeline.py` and `melleafy.json`. |
| Fixture raises `NotImplementedError` | The compiler emitted a host-integration stub. | Read `SETUP.md` and `docs/FROM_STUBS_TO_RUNNING.md`, then implement the stub. |
| Guardian warning says policy manifest is missing | `run` was called before `ingest` or `certify`. | Run `mellea-skills certify <pkg>` or `mellea-skills ingest <spec>`. |

## Recommended learning order

1. Read [Mellea demystified](mellea-demystified.md).
2. Run `examples/weather/weather_mellea` with `--no-guardian`.
3. Inspect `examples/weather/weather_mellea/pipeline.py`, `schemas.py`, `fixtures/`, and `melleafy.json`.
4. Read [Compiler pipeline](compiler-pipeline.md).
5. Compile `skills/weather/spec.md`.
6. Run `validate`, then a fixture.
7. Run `certify` and inspect the `audit/` directory.
8. Read [Extension guide](extension-guide.md) before adding a new dialect, writer, lint, or export target.

