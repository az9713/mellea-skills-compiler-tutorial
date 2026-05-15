# Compiler pipeline

The compiler turns a source skill into a generated Python package with a typed `run_pipeline(...)` entry point.

The implemented compile command is a hybrid:

1. Deterministic wrapper setup in Python.
2. LLM-assisted decomposition through Claude Code slash commands.
3. Deterministic writer rendering and validation in Python.

## Inputs

`mellea-skills compile` accepts either:

| Input | Resolution |
|---|---|
| `path/to/spec.md` | The file is used directly. The skill root is `path/to/`. |
| `path/to/SKILL.md` | The file is used directly. The skill root is `path/to/`. |
| `path/to/skill-dir/` | The wrapper looks for `SKILL.md`, then `spec.md`. |

`compile/mellea_skills.py:_get_spec_md_path(...)` implements this resolution.

## Wrapper setup before Claude Code

Before invoking Claude Code, the wrapper does several important things.

| Step | Code | Purpose |
|---|---|---|
| Validate path | `compile/mellea_skills.py` | Fail early if the input is missing or not Markdown. |
| Parse frontmatter | `toolkit/file_utils.py:parse_spec_file` | Display skill metadata and derive package name. |
| Select Claude model | `Anthropic().models.list()` in `compile/mellea_skills.py` | Use provided model or first available model containing `sonnet`. |
| Start Anthropic proxy | `compile/proxy.py` | Strip `context_management` from Claude API requests for IBM LiteLLM compatibility. |
| Derive package name | `compile/claude_directives.py:derive_package_name` | Apply lowercase, hyphen/space to underscore, `_mellea` suffix. |
| Mirror companion dirs | `mirror_companion_dirs` | Copy `scripts/`, `references/`, and `assets/` into the package before generation. |
| Write grounding | `compile/grounding.py` | Write `mellea_api_ref.json` and `mellea_doc_index.json` for the slash command. |
| Resolve runtime defaults | `resolve_runtime_defaults` | Choose backend and model for the generated package. |
| Write runtime directive | `write_runtime_directive` | Save expected `BACKEND` and `MODEL_ID` for post-compile linting. |
| Write Claude settings | `write_compile_settings` | Deny Write/Edit on wrapper-rendered paths. |

## The Claude Code phase

The wrapper invokes:

```bash
claude -p --model <model> --append-system-prompt <prompt> --allowed-tools Read,Write,Edit --output-format stream-json --verbose --permission-mode acceptEdits ./mellea-fy <spec_path>
```

The slash-command orchestrator is `.claude/commands/mellea-fy.md`. It defines a 10-step workflow:

| Step | Output |
|---|---|
| Step 0: classify source | `classification.json` |
| Steps 1a and 1b: inventory files and tag elements | `inventory.json`, `step_1b_trace.json` |
| Step 2: map elements to Mellea primitives | `element_mapping.json` |
| Step 2.5: dependency audit and dispositions | `dependency_plan.json`, `element_mapping_amendments.json` |
| Step 3: emit skeleton files | Initial package structure and locked `run_pipeline` signature |
| Step 4: generate fixtures | `fixtures_emission.json` for deterministic rendering |
| Step 5: generate code bodies | `pipeline.py`, `schemas.py`, `slots.py`, optional helpers |
| Step 6: emit supporting artifacts | `mapping_report.md`, `melleafy.json`, `README.md`, `SETUP.md` |
| Step 7: static validation | `step_7_report.json` |

The spec describes some steps as deterministic and some as scoped LLM invocations. The wrapper currently enforces part of that contract in Python.

## Intermediate representation

The compiler IR lives in `<package>/intermediate/`.

| Artifact | Meaning |
|---|---|
| `classification.json` | Source runtime, modality, archetype, and other classification output. |
| `inventory.json` | Source elements extracted from the input skill files. |
| `element_mapping.json` | Mapping from source elements to Mellea primitives and generated components. |
| `dependency_plan.json` | External dependencies and their dispositions: bundle, real implementation, stub, mock, external input, and similar. |
| `config_emission.json` | IR consumed by deterministic `config_writer.py`. |
| `fixtures_emission.json` | IR consumed by deterministic `fixtures_writer.py`. |
| `runtime_directive.json` | Backend/model values the wrapper instructed the package to use. |
| `mellea_api_ref.json` | Introspected installed Mellea symbols and compatibility data. |
| `mellea_doc_index.json` | Cached or fetched Mellea docs page index. |
| `step_7_report.json` | Structural lint report. |
| `step_7b_report.json` | Fixture smoke-check report. |

The IR is a major reason this project deserves the word "compiler": it records the decisions between source Markdown and target Python.

## Deterministic writers

Two output surfaces are intentionally not LLM-authored:

| Output | IR | Writer |
|---|---|---|
| `config.py` | `intermediate/config_emission.json` | `.claude/melleafy/writers/config_writer.py` |
| `fixtures/` | `intermediate/fixtures_emission.json` | `.claude/melleafy/writers/fixtures_writer.py` |

`compile/writer_renderer.py` loads writer modules from disk and either compares or overwrites generated output. The current wrapper calls `render_writers(..., enforce=True)`, so writer output is authoritative.

The system prompt from `build_system_prompt(...)` tells Claude Code to emit the IR JSON and avoid writing these paths. The compile settings add path-scoped deny rules for Write/Edit.

## Structural lints

Python lints live in `compile/lints.py`.

| Lint | What it checks |
|---|---|
| `fixtures-loader-contract` | `fixtures/__init__.py` exports `ALL_FIXTURES` or `FIXTURES`. |
| `bundled-asset-path-resolution` | Runtime code resolves `scripts/`, `references/`, or `assets/` relative to `Path(__file__).parent`. |
| `runtime-defaults-bound` | `config.py` `BACKEND` and `MODEL_ID` match `intermediate/runtime_directive.json`. |

The slash-command validation spec describes more lints, but only these are currently implemented in Python. Implementing the remaining lints in Python is one of the best ways to improve reliability.

## Smoke check

`compile/smoke_check.py` executes one fixture by default, or all fixtures when requested.

| Verdict | Meaning | Exit behavior |
|---|---|---|
| `passed` | Fixture executed without exception. | Exit code 0. |
| `skipped` | Backend unreachable, timeout, or auth failure. | Exit code 0 with warning. |
| `failed` | Fixture raised another exception. | Exit code 12. |

The smoke check is deliberately not a correctness test. It is an execution test: imports, fixture loading, argument passing, model backend reachability, and basic runtime path are validated.

## Generated package anatomy

The generated package is the compiler target.

```text
<skill-root>/
  spec.md or SKILL.md
  <package_name>/
    __init__.py
    __main__.py
    pipeline.py
    schemas.py
    slots.py
    config.py
    main.py
    tools.py or constrained_slots.py
    requirements.py
    fixtures/
    scripts/ or references/ or assets/
    melleafy.json
    mapping_report.md
    README.md
    SETUP.md
    intermediate/
```

The most important file is `pipeline.py`. It exposes the callable runtime contract. The most important metadata file is `melleafy.json`, because exporters and downstream tooling consume it.

## Where the compiler can fail

| Failure point | Typical symptom | Debug artifact |
|---|---|---|
| Bad input path | Immediate CLI error. | None. |
| Claude model selection | No available model or invalid model. | CLI logs. |
| Claude Code generation | Subprocess error or timeout. | Partial generated files if any. |
| Missing IR | Writer warnings or errors. | `intermediate/*_emission.json`. |
| Writer render failure | Compile warning or generated file mismatch. | Wrapper logs. |
| Structural lint failure | Compile fails after generation. | `intermediate/step_7_report.json`. |
| Fixture runtime failure | Compile or validate fails unless backend unreachable. | `intermediate/step_7b_report.json`. |
| Unresolved stub | Runtime `NotImplementedError` or degraded output. | `SETUP.md`, `dependency_plan.json`, generated stub docstring. |

## Why the design works

The compiler works best when each layer does the thing it is good at.

| Layer | Good at | Weak at |
|---|---|---|
| Claude Code slash command | Semantic decomposition of natural language into program structure. | Perfectly stable source rendering and self-validation. |
| Python wrapper | Deterministic file operations, cache generation, validation, writer rendering. | Understanding ambiguous natural-language intent. |
| Mellea runtime | Typed generation, requirements, hooks, repair loops, sessions. | Discovering host-specific integrations from thin specs. |
| Fixtures and lints | Catching concrete drift and runtime breakage. | Proving behavioral correctness across all inputs. |

The strategic direction is clear: keep Claude Code for semantic decomposition, but move more load-bearing emission and validation into deterministic Python.

