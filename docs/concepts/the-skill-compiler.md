# The Skill Compiler

`mellea-skills compile` runs a 10-step pipeline. Seven steps involve LLM calls (via Claude Code). Three are deterministic. Understanding the flow helps you diagnose failures, interpret intermediates, and predict what the compiler will do with a new spec.

---

## The two-layer architecture

The compile command has two layers:

**Layer 1 — `/mellea-fy` (LLM-driven)**: A Claude Code slash command that runs Steps 0–7. This is where the spec is read, classified, decomposed, and converted into Python. Claude does the reasoning; the spec at `.claude/commands/mellea-fy.md` (and nine sub-command files) tells it exactly how.

**Layer 2 — deterministic wrapper (`mellea_skills.py`)**: Before and after the slash command runs, the Python wrapper does deterministic work that the LLM must not do: companion-directory mirroring, Mellea API reference pre-population, runtime-defaults injection, `config.py` rendering from JSON IR, `fixtures/` rendering from JSON IR, and structural lints.

This separation is important. It ensures two critical files — `config.py` and `fixtures/` — are always generated the same way regardless of how Claude phrases a request. Claude Code deny rules block the LLM from writing those paths during the slash-command phase.

---

## The 10 steps

```
Source spec (.md)
    │
    ▼
Step 0: Five-axis classification
    │   → intermediate/classification.json
    │
    ▼
Steps 1a + 1b: Inventory
    │   → intermediate/inventory.json
    │
    ▼
Step 2: Element mapping
    │   → intermediate/element_mapping.json
    │
    ▼
Step 2.5: Dependency audit (deterministic)
    │   → intermediate/dependency_plan.json
    │   → intermediate/element_mapping_amendments.json
    │
    ▼
Step 3: Skeleton emit (Python file stubs)
    │
    ▼
Step 4: Fixture generation
    │   → fixtures/ subpackage (5–8 fixtures)
    │
    ▼
Step 5: Code body generation (LLM fills skeleton)
    │   → pipeline.py, schemas.py, slots.py, tools.py, requirements.py, ...
    │
    ▼
Step 6: Supporting artifacts
    │   → mapping_report.md, melleafy.json, SETUP.md, README.md
    │
    ▼
Step 7: Structural lints (14 checks)
    │   → intermediate/step_7_report.json
    │
    ├── [PASS] → deterministic wrapper renders config.py + fixtures/
    │            → smoke check (one fixture)
    │
    └── [FAIL] → repair loop (max 2 rounds) → partial preserved
```

---

### Step 0: Five-axis classification

The compiler classifies the spec along five axes:

| Axis | What it determines | Example values |
|------|-------------------|---------------|
| Reasoning archetype | What kind of thinking the skill does | A (Analysis), B (Generation), C (Diagnosis), D1 (Integration), D2 (Orchestration), E (Knowledge) |
| Pipeline shape | How phases relate | Sequential (multi-phase), One-shot (flat) |
| Tool involvement | Whether the pipeline calls tools and how | P0 (none), P2 (LLM classifies, Python calls), P3 (LLM-directed tool calls), P4 (tools provide input) |
| Source runtime | What format the spec comes from | `agent_skills_std`, `claude_code`, `openclaw`, `crewai`, `langgraph`, `letta`, `hybrid` |
| Interaction modality | How the pipeline is invoked | `synchronous_oneshot`, `streaming`, `conversational_session`, `scheduled`, `event_triggered` |

The output is `classification.json`. Every later step reads it.

The axes run in a specific order because each informs the next: source runtime first (determines dialect rules), then modality, then tool involvement, then archetype, then pipeline shape.

---

### Steps 1a + 1b: Inventory

Step 1a reads all spec files based on the source runtime dialect. For `agent_skills_std`, this is typically one `.md` file. For `crewai`, it's `agents.yaml`, `tasks.yaml`, and `crew.py`.

Step 1b tags every element with:
- A **category** (C1–C9, see below)
- An **element type** (`EXTRACT`, `CLASSIFY`, `GENERATE`, `SCHEMA`, `TOOL`, `CONSTRAINT`, etc.)
- Whether it should be aggregated with another element

The output is `inventory.json`. It is the compiler's complete understanding of what the spec contains.

**The C1–C9 categories:**

| Category | What it covers |
|----------|---------------|
| C1 | Configuration constants (persona text, model ID, loop budgets) |
| C2 | Schema fields (output types, intermediate types) |
| C3 | Domain knowledge (rules, decision tables) |
| C4 | Templates (prompt templates, URL templates) |
| C5 | Credentials and secrets |
| C6 | Tool calls (HTTP APIs, SDKs, CLIs, abstract tool references) |
| C7 | Memory and session state |
| C8 | Scope and routing logic |
| C9 | Interaction modality (streaming behavior, session carry-forward) |

---

### Step 2: Element mapping

Each inventoried element is mapped to a Mellea primitive:

| Element type | Mellea primitive |
|-------------|-----------------|
| EXTRACT / CLASSIFY | `@generative` slot in `slots.py` or `m.instruct(format=Schema)` in `pipeline.py` |
| GENERATE | `m.instruct(format=Schema)` with grounding context |
| SCHEMA | Pydantic model field in `schemas.py` |
| TOOL (C6, `real_impl`) | Function in `tools.py` |
| TOOL (C6, `stub`) | `NotImplementedError` in `constrained_slots.py` |
| CONSTRAINT | `@requirement` validator in `requirements.py` |
| DECIDE | Deterministic Python control flow in `pipeline.py` |
| CONFIG | Constant in `config.py` (via `config_emission.json`) |

The output is `element_mapping.json`. `TOOL_TEMPLATE` entries are marked `"final_target_file": "pending_step_2.5"` until Step 2.5 commits dispositions.

---

### Step 2.5: Dependency audit (deterministic)

This step is entirely deterministic — no LLM calls.

Every element with category C1–C9 gets a **disposition** that determines how the compiler generates code for it. In `auto` mode (the default), dispositions are assigned from a fixed table per category. In `ask` mode, the user can override each one interactively.

**Dispositions:**

| Disposition | Generated code |
|-------------|---------------|
| `bundle` | Constant in `config.py` |
| `real_impl` | Full implementation in `tools.py` (HTTP call, SDK usage, etc.) |
| `stub` | `NotImplementedError` in `constrained_slots.py` |
| `mock` | Mock implementation in `fixtures/mock_tools.py` for testing |
| `delegate_to_runtime` | No code generated; the host runtime provides this |
| `external_input` | CLI flag or environment variable declaration |
| `load_from_disk` | `open(Path(__file__).parent / "...")` at runtime |
| `remove` | No code generated; element is a cross-reference artifact |

The output is `dependency_plan.json` and `element_mapping_amendments.json`. After this step, every element knows exactly where it will land in the generated package.

---

### Steps 3 + 5: Code generation

Step 3 emits Python file skeletons — empty files with correct imports and `run_pipeline` signature. The signature is locked at this point; fixtures in Step 4 use it as grounding.

Step 4 generates 5–8 fixtures covering normal, edge, and adversarial inputs. Fixtures are JSON IR that the wrapper later renders into Python files under `fixtures/`.

Step 5 fills the skeletons with code bodies. The LLM writes `pipeline.py`, `schemas.py`, `slots.py`, `tools.py`, `requirements.py`, `mobjects.py` (if needed), and `loader.py` (if needed). It grounds itself in `mellea_api_ref.json` (the live Mellea library introspection pre-populated by the wrapper before the slash command ran) to avoid hallucinating API signatures.

---

### Step 6: Supporting artifacts

The compiler emits:
- `mapping_report.md` — human-readable table of element-to-primitive mapping
- `melleafy.json` — the package manifest
- `SETUP.md` — backend setup instructions and dependency catalogue
- `README.md` — per-package usage instructions
- `SKILL.md` — CLI compatibility shim (for non-`.md` source formats)

---

### Step 7: Structural lints (14 checks)

Static validation before anything runs. Checks include:

- **import-soundness** — all imports resolve to real symbols
- **stdlib-arity** — all Mellea API calls use the correct signatures (from `mellea_api_ref.json`)
- **schema-non-optional** — required output fields are non-Optional
- **fixture-shape** — all fixtures return `(inputs_dict, fixture_id, description)`
- **melleafy-schema** — `melleafy.json` conforms to the versioned schema
- **runtime-defaults-alignment** — `config.py` constants match what was injected by the wrapper
- **stub-presence** — stubs that exist are documented in `SETUP.md §8`

If any lint fails, the compiler enters a repair loop (up to 2 rounds): regenerate only the failing files, re-run lints. If it still fails after 2 rounds, the partial compile is preserved under `.melleafy-partial/`.

---

### Post-lint: wrapper rendering

After Step 7 passes, the Python wrapper (not Claude) renders:

- `config.py` — from `intermediate/config_emission.json`
- `fixtures/*.py` — from `intermediate/fixtures_emission.json`

Then the smoke check runs one fixture against the compiled pipeline. Exit code 0 means everything worked.

---

## What can go wrong

| Symptom | Cause | Fix |
|---------|-------|-----|
| Compile times out after 75 min | Long spec or slow Claude session | `--repair-mode` |
| Exit code 11 | Step 7 lints failed and repair loop didn't converge | `--repair-mode`; check `intermediate/step_7_report.json` |
| Exit code 12 | Smoke check failed — a fixture raised | Likely a stub; see [Fill stubs](../guides/fill-stubs.md) |
| No `*_mellea/` directory after compile | The slash command failed before Step 3 | Check Claude session output; `--repair-mode` |

See [Repair a failed compile](../guides/repair-failed-compile.md) for the full diagnosis flow.
