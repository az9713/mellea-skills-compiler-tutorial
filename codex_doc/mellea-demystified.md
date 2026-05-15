# Mellea demystified

Mellea Skills Compiler is a research-preview compiler that converts agent skill specifications into typed Mellea programs, then validates, runs, certifies, and exports those programs.

It is not just a prompt template generator. It is a bridge from "an LLM reads a Markdown instruction file and improvises" to "a Python package runs a typed, instrumented, auditable workflow."

## What Mellea is in this repo

Mellea is the generative-programming runtime used by compiled skills. The generated packages use Mellea concepts such as `m.instruct(...)`, `@generative` slots, typed Pydantic schemas, model backends, requirements, validation, sessions, and plugin hooks.

This repo does not implement the Mellea runtime itself. It builds around Mellea:

| Layer | Code surface | Responsibility |
|---|---|---|
| Mellea runtime | External `mellea[hooks]` dependency in `pyproject.toml` | Execute generative programs, run hooks, validate outputs, manage sessions. |
| Compiler wrapper | `src/mellea_skills_compiler/compile/mellea_skills.py` | Invoke Claude Code, prepare grounding artifacts, render deterministic outputs, run lints and smoke checks. |
| Slash-command compiler spec | `.claude/commands/mellea-fy*.md` | Define the LLM-assisted decomposition workflow from source spec to package. |
| Generated skill packages | `examples/*/*_mellea/` | Runnable Mellea programs produced by the compiler. |
| Governance layer | `src/mellea_skills_compiler/certification/` and `src/mellea_skills_compiler/guardian/` | Identify risks, register Guardian hooks, log runtime evidence, classify controls, emit certification reports. |
| Export layer | `src/mellea_skills_compiler/export/` | Wrap compiled packages for LangGraph, MCP, and Claude Code. |

## Why this is a "skill compiler"

A normal compiler translates source code into a target program while preserving meaning and surfacing errors. Mellea Skills Compiler tries to do the same for agent skills.

| Compiler concept | Mellea Skills Compiler equivalent |
|---|---|
| Source language | Markdown or multi-file agent skill specs. |
| Parsing and classification | Frontmatter/body parsing plus source-runtime, modality, archetype, and category classification. |
| Intermediate representation | JSON files under `intermediate/`: `classification.json`, `inventory.json`, `element_mapping.json`, `dependency_plan.json`, `config_emission.json`, `fixtures_emission.json`, and related reports. |
| Code generation | LLM-emitted Python package plus deterministic writer-rendered `config.py` and `fixtures/`. |
| Static checks | Step 7 lints in `.claude/commands/mellea-fy-validate.md`, with Python implementations in `compile/lints.py`. |
| Runtime test | Fixture smoke check in `compile/smoke_check.py`. |
| Metadata | `melleafy.json`, consumed by the export pipeline. |
| Target program | A Python package with `pipeline.py:run_pipeline(...)`. |

The unusual part is that the compiler has both deterministic and LLM-assisted phases. The wrapper performs deterministic setup, path mirroring, grounding, runtime default binding, writer rendering, linting, and smoke checks. Claude Code performs the semantic decomposition from natural language into Mellea primitives.

## The pain points it tries to solve

### Pain point 1: Markdown skills are flexible but opaque

Raw skill specs are easy to author, but they leave important choices implicit. A model may silently decide what counts as state, a tool, a credential, a runtime assumption, or an operating rule.

Mellea Skills Compiler mitigates this by forcing decomposition into categories and artifacts. The slash-command workflow inventories the source, maps elements to Mellea primitives, writes a dependency plan, and emits a mapping report. The generated package makes these decisions inspectable.

How well it works: good for surfacing structure in the included examples. The generated packages expose schemas, fixtures, dependency metadata, and mapping reports. The remaining weakness is that much of the semantic decomposition is still performed by an LLM, so compiler quality depends on the slash-command spec, model behavior, and validation coverage.

### Pain point 2: Agent behavior is hard to test

A prompt-only skill often has no stable callable contract. You can ask the agent to do something, but you cannot easily run fixtures or assert typed behavior.

Mellea Skills Compiler mitigates this by generating `run_pipeline(...)`, Pydantic models, fixture factories, and smoke-check infrastructure. `toolkit/file_utils.py` loads the package and fixture contract dynamically; `compile/smoke_check.py` executes one or all fixtures and records `step_7b_report.json`.

How well it works: good for first-order runtime verification. It proves that imports, fixture shape, and at least one execution path can run. It does not yet provide golden-output assertions, regression comparison, property tests, cost budgets, or deterministic model mocking.

### Pain point 3: Runtime risks are under-observed

Agent outputs can contain harmful content, sensitive leakage, prompt-injection effects, or policy violations. Prompt-only execution rarely produces a durable audit trail.

Mellea Skills Compiler mitigates this by using AI Atlas Nexus to identify relevant risks, Granite Guardian to check model inputs and outputs, Mellea hooks to intercept runtime events, and JSONL audit trails to preserve evidence.

How well it works: stronger than a pure prompt workflow because every generation can be logged with Guardian verdicts. The code supports audit and enforce modes through `GuardianAuditPlugin` and `GuardianEnforcePlugin`. The practical limitation is that quality depends on Guardian model availability, taxonomy mapping quality, and whether all relevant tool paths flow through Mellea hooks.

### Pain point 4: Compliance evidence is disconnected from execution

Compliance often becomes a document written after the fact. The runtime behavior and the governance report are not connected.

Mellea Skills Compiler mitigates this by generating `policy_manifest.json`, `POLICY.md`, `runtime_audit.jsonl`, `pipeline_report.json`, and `CERTIFICATION.md` under an `audit/` directory. `certification/report.py` extracts evidence from audit entries and maps it to compliance classifications.

How well it works: useful as an evidence package for research and onboarding. It is not a full enterprise GRC system. The compliance classifier is mostly mapping-driven and conservative; some control coverage is still manual or partial.

### Pain point 5: Agent skills are hard to move between harnesses

A useful skill may need to run under LangGraph, MCP, Claude Code, or another harness. Prompt formats and runtime contracts differ.

Mellea Skills Compiler mitigates this by treating the compiled package as the canonical core and generating adapters around it. `export/exporter.py` validates `melleafy.json`, loads the package contract, dispatches to target translators, emits a bundled output, and lints the result.

How well it works: promising but explicitly experimental. LangGraph, MCP, and Claude Code are supported. Other harnesses require hand-written wrappers or new exporters.

## What this project is not

This project is not a universal agent runtime. It generates and governs Mellea-based Python packages.

This project is not a static formal verifier. It combines structured generation, lints, fixtures, typed schemas, Guardian checks, and certification artifacts, but it does not prove semantic correctness.

This project is not a stable production platform yet. The README marks the project as a research preview, and the code includes experimental surfaces such as export targets and evolving deterministic writers.

## What can be built on top

| Product or capability | Why Mellea Skills Compiler helps |
|---|---|
| Skill registries with certification badges | Compiled packages produce manifests, audit trails, and certification reports that can back registry metadata. |
| Enterprise skill onboarding portals | Users can upload a skill, compile it, inspect generated artifacts, run fixtures, and review risk coverage before use. |
| Governance CI for agent specs | Repos can require compile, lint, smoke check, risk manifest generation, and certification artifacts before merging a new skill. |
| Multi-harness deployment tooling | The exporter can become a plugin architecture for LangGraph, MCP, Claude Code, OpenAI Agents SDK, CrewAI, Letta, and others. |
| Skill quality dashboards | Intermediate artifacts expose source coverage, dependency dispositions, stubs, fixtures, runtime risks, and audit evidence. |
| Human-in-the-loop repair workflows | Failed lints, smoke-check errors, Guardian verdicts, and `NotImplementedError` stubs can feed guided repair queues. |

## How it can improve

The highest-leverage improvements are structural, not cosmetic.

| Area | Current state | Improvement |
|---|---|---|
| Compiler determinism | Some artifacts are LLM-emitted; `config.py` and `fixtures/` are deterministic writer-rendered. | Migrate more generated files to IR plus deterministic writers, especially `melleafy.json`, `SETUP.md`, and dependency wrappers. |
| Validation depth | Python lints cover fixture loader contract, bundled asset paths, and runtime defaults; slash-command spec describes more lints. | Implement the remaining formal lints in Python so validation does not depend on model self-checking. |
| Test assertions | Smoke checks verify execution, not correctness. | Add golden outputs, schema invariants, expected risk labels, cost budgets, and fixture-specific assertions. |
| Stub handling | Stubs are documented and can degrade gracefully, but they still require manual work. | Add stub inventory commands, generated implementation templates, and CI failure modes for unresolved stubs. |
| Governance coverage | Generation and some tool hooks are audited; classification is mapping-driven. | Expand control mappings, validate them against real cases, and add richer tool, credential, memory, and scheduling governance. |
| Export stability | Export works for three targets but is experimental. | Stabilize `melleafy.json`, formalize target contracts, and add target-specific integration tests. |
| Developer ergonomics | The architecture is powerful but spread across wrapper code, slash commands, schemas, examples, and docs. | Keep this doc set current, add sequence diagrams to generated reports, and provide a "first extension" tutorial. |

## The shortest explanation

Mellea Skills Compiler treats an agent skill as source code. It compiles the skill into a typed Mellea program, records the compiler's decisions as JSON, validates the generated package, runs fixtures, attaches governance hooks, emits audit evidence, and can wrap the result for other harnesses.

The value is not just that it generates code. The value is that it makes a formerly implicit agent specification inspectable, runnable, governable, and portable.

