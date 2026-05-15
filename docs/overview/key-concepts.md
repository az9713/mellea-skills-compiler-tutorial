# Key Concepts

Every term used across these docs, defined once.

---

**AI Atlas Nexus** — An IBM Research governance knowledge graph (Apache 2.0, [GitHub](https://github.com/IBM/ai-atlas-nexus)). Accepts a skill's use-case description and returns a `PolicyManifest` mapping the use case to Guardian risks, NIST AI RMF actions, and Credo UCF controls. Used during the `certify` step. See [Certification pipeline](../concepts/certification-pipeline.md).

**archetype** — The classification of a skill by what kind of reasoning it does. Five archetypes: **A** (Analysis), **B** (Generation), **C** (Diagnosis), **D1** (Integration), **D2** (Orchestration), **E** (Knowledge). The archetype drives how the 10-step compiler decomposes the spec. See [Skill archetypes](../concepts/skill-archetypes.md).

**audit trail** — A JSONL file (`audit/audit_trail.jsonl`) written during certification. Each line is one Guardian verdict for one `m.instruct()` call: timestamp, hook type, policy identifier, payload, latency. The evidence base for the certification report.

**CERTIFICATION.md** — Human-readable certification report. Documents Guardian verdict summary, audit trail statistics, per-requirement evidence chains, and known gaps. Does not assert safety — it documents what was checked. See [Certification pipeline](../concepts/certification-pipeline.md).

**certify** — The second user-facing command (`mellea-skills certify`). Runs end-to-end governance on a compiled skill: Nexus → `PolicyManifest` → Guardian hook configuration → fixture execution + audit → compliance classification → `CERTIFICATION.md`. See [Certification pipeline](../concepts/certification-pipeline.md).

**classification (Step 0)** — The first step of the 10-step compilation. Classifies the spec along five axes: reasoning archetype, pipeline shape, tool involvement, source runtime, and interaction modality. The output (`classification.json`) drives every subsequent step.

**Claude Code** — Required for the compilation step. `mellea-skills compile` invokes Claude (via the Claude Code CLI) to perform LLM-driven spec decomposition. Must be installed and authenticated.

**compile** — The first user-facing command (`mellea-skills compile`). Invokes the `/mellea-fy` slash command inside Claude Code, then chains deterministic post-processing: companion-directory mirror, grounding pre-population, structural lints (Step 7), fixture smoke check. See [The skill compiler](../concepts/the-skill-compiler.md).

**compiled package** — The output of `mellea-skills compile`: a Python package directory named `<skillname>_mellea/`. Contains `pipeline.py`, `schemas.py`, `config.py`, `fixtures/`, `melleafy.json`, and other files. Pip-installable and runnable independently of the compiler. See [Compiled package anatomy](../concepts/compiled-package-anatomy.md).

**config.py** — A file in every compiled package that holds constants: `BACKEND`, `MODEL_ID`, persona text, loop budgets, and other values extracted from the spec. Written by the deterministic `config_writer.py` from `intermediate/config_emission.json` — not by the LLM directly.

**constrained_slots.py** — A file present in compiled packages that use stubs. Holds `NotImplementedError` stub functions for external tool calls the spec referenced but didn't pin down. Users fill these to unlock full pipeline functionality. See [Fill stubs](../guides/fill-stubs.md).

**Credo UCF** — Credo AI's Unified Control Framework. One of the governance taxonomies mapped by Nexus during certification. Provides mitigation controls linked to specific NIST AI RMF actions.

**dependency audit (Step 2.5)** — The step that classifies every external dependency (credentials, tools, backends, triggers) into one of nine categories (C1–C9) and assigns a disposition. Deterministic — no LLM calls. Drives what code the compiler generates for each dependency. See [Dependency resolution](../concepts/dependency-resolution.md).

**dialect** — The source format of a skill spec. `agent_skills_std` (single `.md` with YAML frontmatter) is the primary dialect. Other dialects — `claude_code`, `openclaw`, `crewai`, `langgraph`, `letta`, `hybrid` — have dedicated dialect docs that adjust how the inventory and mapping steps interpret the spec. See `docs/dialects/`.

**disposition** — The decision made about how to handle an external dependency: `bundle`, `real_impl`, `stub`, `mock`, `delegate_to_runtime`, `external_input`, `load_from_disk`, or `remove`. Set during Step 2.5 (auto mode or interactively in ask mode). See [Dependency resolution](../concepts/dependency-resolution.md).

**element** — A discrete item extracted from a skill spec during Step 1 (inventory). Each element gets a category tag (C1–C9), an element type tag (EXTRACT, CLASSIFY, GENERATE, etc.), and eventually maps to a Mellea primitive (slot, schema field, tool, etc.).

**enforce mode** — A Guardian operating mode (`--enforce` flag). When enabled, any Guardian detection raises `PluginViolationError` and halts the pipeline. Default is audit mode (observe and log only). See [Certification pipeline](../concepts/certification-pipeline.md).

**export** — The `mellea-skills export` subcommand. Translates a compiled package into a deployment adapter for a target harness: `langgraph`, `claude-code`, or `mcp`. Experimental. See [Export to other harnesses](../guides/export-to-other-harnesses.md).

**fixture** — A sample input for a compiled pipeline. Stored as a Python file in `fixtures/` that returns `(inputs_dict, fixture_id, description)`. Used for smoke checks, certify runs, and manual testing. 5–8 fixtures are auto-generated per skill. See [Run and test fixtures](../guides/run-and-test-fixtures.md).

**Granite Guardian** — IBM Research's runtime risk detection model ([HuggingFace](https://huggingface.co/ibm-granite/granite-guardian-3.3-8b), Apache 2.0). Runs as a Mellea hook plugin. Checks every `m.instruct()` call for harm, social bias, jailbreaking, hallucination, and other risks. See [Certification pipeline](../concepts/certification-pipeline.md).

**grounding context** — A dict passed to `m.instruct()` in compiled pipelines that provides structured factual inputs to the LLM (e.g., `{"query": "...", "extracted_location": "..."}`) to constrain generation without embedding facts in the system prompt.

**intermediate/** — A subdirectory inside every compiled package that holds the compiler's intermediate representation (IR) files: `classification.json`, `inventory.json`, `element_mapping.json`, `dependency_plan.json`, `config_emission.json`, `fixtures_emission.json`, `step_7_report.json`, and others. Preserved for debugging and audit.

**inventory (Steps 1a + 1b)** — The step that reads all spec files and tags every element with its category (C1–C9) and type. Outputs `inventory.json`. Dialect-aware — the files read and their interpretation vary by source runtime.

**lint (Step 7)** — Structural validation of the compiled package. 14 formal checks including import soundness, stub presence, fixture shape, melleafy.json schema conformance, and runtime-defaults alignment. Exit code 11 means lints failed.

**mapping (Step 2)** — The step that routes each inventoried element to a Mellea primitive: `@generative` slot, `m.instruct` call, Pydantic schema field, tool function, requirement validator, etc. Outputs `element_mapping.json`. See [The skill compiler](../concepts/the-skill-compiler.md).

**Mellea** — The IBM Research library for structured LLM generation (Apache 2.0, [GitHub](https://github.com/generative-computing/mellea)). Provides `start_session`, `m.instruct`, `@generative`, `RepairTemplateStrategy`, and the plugin hook system. Every compiled skill runs on Mellea. See [What is it?](what-is-mellea.md).

**mellea-fy** — The Claude Code slash command that performs the LLM-driven decomposition steps (Steps 0–7). `mellea-skills compile` invokes this internally. Can be invoked directly inside Claude Code for debugging.

**melleafy.json** — The compiled package manifest. Contains `package_name`, `source_runtime`, `modality`, `archetype`, `entry_signature`, `pipeline_parameters`, and `declared_env_vars`. The contract the `export` command consumes and external integrations build against. See [melleafy.json reference](../reference/melleafy-json.md).

**modality** — How the pipeline is invoked and responds. One of: `synchronous_oneshot`, `streaming`, `conversational_session`, `scheduled`, `event_triggered`, `heartbeat`. Determines the shape of `run_pipeline`'s signature and affects export target adapter generation.

**NIST AI RMF** — National Institute of Standards and Technology AI Risk Management Framework. One of the governance frameworks mapped by Nexus during certification. Provides the governance actions that get classified as AUTOMATED / PARTIAL / MANUAL.

**Ollama** — The default LLM backend for compiled skills at runtime. `granite3.3:8b` is the default model. Separate from Claude (used at compile time). Configured via `OLLAMA_API_URL`.

**pipeline.py** — The entry-point file of every compiled package. Defines `run_pipeline(...)` — the typed function the entire compiled skill exposes. LLM-emitted during compilation.

**PolicyManifest** — A JSON document generated by Nexus from the skill's use-case description. Contains Guardian risks (with system prompts) and governance actions (NIST + Credo). Drives all downstream certification steps. See [Certification pipeline](../concepts/certification-pipeline.md).

**repair mode** — A compile mode (`--repair-mode` / `-r`) that audits a partial or failed compile and resumes from the first broken step, reusing valid intermediate artifacts rather than starting over. See [Repair a failed compile](../guides/repair-failed-compile.md).

**run_pipeline** — The typed Python function at the root of every compiled package. Takes skill-specific inputs and returns a Pydantic model or plain string. The stable contract that export adapters and integrations build on.

**schemas.py** — A file in every compiled package that defines the Pydantic models for intermediate and output types. Every `m.instruct(format=...)` call references a model from this file.

**skill** — A discrete agent capability described in a `.md` specification file. The unit the compiler processes. Called a "skill" to match the Anthropic Agent Skills standard and OpenClaw conventions.

**skill specification** — The `.md` file that describes a skill. Contains YAML frontmatter (`name`, `description`, optionally `allowed-tools`, `metadata.openclaw.requires`) and a natural-language body describing the workflow. The primary input to the compiler.

**slots.py** — A file in compiled packages that holds `@generative`-decorated extraction functions — short, focused LLM slots that extract or classify specific pieces of information (e.g., extract a location from a query string).

**smoke check** — A post-compile validation step that runs one fixture against the compiled pipeline and verifies it doesn't raise. Exit code 12 means a fixture raised. Skippable with `--no-run`. See [Run and test fixtures](../guides/run-and-test-fixtures.md).

**source runtime** — The framework or convention the skill spec was authored in. Determines which dialect rules apply. `agent_skills_std` is the primary supported dialect. Others (`crewai`, `langgraph`, `letta`, `openclaw`, `claude_code`) are detected and supported with varying maturity. See `docs/dialects/`.

**stub** — A `NotImplementedError` function in `constrained_slots.py` or `tools.py` generated when the spec references a tool the compiler can't implement automatically. Users fill stubs before those code paths can execute. See [Fill stubs](../guides/fill-stubs.md).

**tier (T1/T2/T3)** — A classification of how much work is needed to run a compiled skill. T1: runs with only Ollama. T2: one stub or artifact needed. T3: external service or credential required. See [Skill tiers](../reference/skill-tiers.md).

**wrapper-rendered** — Files in a compiled package that are emitted by the deterministic Python wrapper, not by the LLM: `config.py` (from `config_emission.json`) and `fixtures/` (from `fixtures_emission.json`). LLM cannot overwrite these paths during compilation (enforced by Claude Code deny rules).
