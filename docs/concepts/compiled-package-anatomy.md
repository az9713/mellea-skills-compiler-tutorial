# Compiled Package Anatomy

Every `mellea-skills compile` run produces a Python package directory named `<skillname>_mellea/`. This page describes every file, what it does, and which files are safe to modify.

---

## Directory layout

```
<skillname>_mellea/
├── pipeline.py           ← Entry point — run_pipeline(...)
├── schemas.py            ← Pydantic models for all I/O types
├── config.py             ← Constants: backend, model, persona, budgets
├── slots.py              ← @generative extraction functions
├── tools.py              ← Deterministic tool implementations (if tools present)
├── constrained_slots.py  ← NotImplementedError stubs (if stubs present)
├── requirements.py       ← Requirement validators (if constraints present)
├── mobjects.py           ← Mellea objects for session state (if conversational)
├── loader.py             ← Disk-loaded reference data (if load_from_disk present)
├── main.py               ← CLI entry point (python -m <package>)
├── __init__.py
├── __main__.py
│
├── fixtures/             ← Sample inputs and expected behavior
│   ├── __init__.py
│   ├── <fixture_name>.py
│   └── mock_tools.py     ← Mock tool implementations (if mock disposition used)
│
├── intermediate/         ← Compiler IR — do not edit
│   ├── classification.json
│   ├── inventory.json
│   ├── element_mapping.json
│   ├── element_mapping_amendments.json
│   ├── dependency_plan.json
│   ├── config_emission.json
│   ├── fixtures_emission.json
│   ├── runtime_directive.json
│   ├── mellea_api_ref.json
│   ├── mellea_doc_index.json
│   ├── step_7_report.json
│   └── step_7b_report.json
│
├── melleafy.json         ← Package manifest (versioned contract)
├── mapping_report.md     ← Human-readable element-to-primitive mapping
├── SKILL.md              ← Original spec (copy)
├── SETUP.md              ← Backend setup and stub catalogue
└── README.md             ← Per-skill usage
```

Not all files are present in every package — conditional files only appear when the skill needs them.

---

## Core files

### `pipeline.py`

The entry point. Defines `run_pipeline(...)` — the function everything else wraps.

**LLM-generated.** Safe to read and understand; safe to modify after compile if you know what you're doing. The function signature is stable once generated; changing it breaks fixtures and exports.

The pipeline follows a pattern: sessions using `with start_session(BACKEND, MODEL_ID) as m:`, followed by `m.instruct(format=Schema)` calls with Pydantic schemas as format constraints. Deterministic Python control flow handles scope gates, dispatch, and error paths.

Example (from the weather skill):

```python
def run_pipeline(query: str) -> str:
    with start_session(BACKEND, MODEL_ID) as m:
        location_raw = extract_location(m, query=query)   # @generative slot

    with start_session(BACKEND, MODEL_ID) as m:
        intent_thunk = m.instruct(
            "Classify this weather query...",
            grounding_context={"query": query, "extracted_location": location_raw},
            format=WeatherIntent,
            strategy=RepairTemplateStrategy(loop_budget=LOOP_BUDGET),
        )
        intent = WeatherIntent.model_validate_json(intent_thunk.value)

    if intent.query_type == WeatherQueryType.out_of_scope:
        return "Out of scope: ..."

    return fetch_weather(intent.location, _build_endpoint_params(intent.query_type))
```

### `schemas.py`

Pydantic models for all intermediate and output types. Every `m.instruct(format=Schema)` references a class from this file.

**LLM-generated.** Safe to extend with additional validation logic. Do not rename or remove fields that `pipeline.py` or `fixtures/` reference.

Required output fields are non-Optional (enforced by lint). Nullable fields use `Optional[T]`. Enum types are defined here alongside model classes.

### `config.py`

Constants extracted from the spec: `BACKEND`, `MODEL_ID`, persona text, loop budgets, URL templates, out-of-scope categories, and other values.

**Wrapper-rendered** (from `intermediate/config_emission.json`). Not LLM-written directly. This is intentional — constants are derived from the spec by a deterministic process, not from LLM judgement.

Safe to modify constants after compile (e.g., changing `MODEL_ID` to use a different model). The runtime-defaults lint checks that `BACKEND` and `MODEL_ID` match what the compile specified; if you change them, re-run `mellea-skills validate` to check for new issues.

### `slots.py`

`@generative`-decorated extraction functions. Each function represents a focused LLM extraction: extract a location from a query, classify an intent, normalize a date string.

**LLM-generated.** The `@generative` decorator handles the LLM call internally; you call the function like a normal Python function passing the Mellea session object `m` and any inputs.

```python
from mellea import generative

@generative
def extract_location(m, query: str) -> str:
    """Extract the geographic location from a natural-language weather query."""
```

---

## Conditional files

### `tools.py`

Present when the skill has C6 tool calls with `real_impl` disposition. Contains deterministic Python implementations: HTTP calls, SDK usage, subprocess invocations.

**LLM-generated.** Safe to modify — these are the implementations that make the pipeline useful. If the compiler generated an incorrect implementation (wrong API endpoint, wrong parameter names), fix it here.

### `constrained_slots.py`

Present when the skill has C6 tool calls with `stub` disposition. Contains `NotImplementedError` functions the user must fill.

**Partially LLM-generated.** The stub signatures, docstrings, and example implementations are LLM-generated. You supply the body. See [Fill stubs](../guides/fill-stubs.md).

### `requirements.py`

Present when the spec includes constraints or correctness requirements. Contains `@requirement`-decorated validator functions that the Mellea repair loop uses.

**LLM-generated.** The compiler translates spec constraints like "do not invent entities not present in the input" into Python validators that run after each `m.instruct()` call. If a validator fails, the repair loop retries the generation.

### `mobjects.py`

Present for `conversational_session` modality skills. Defines Mellea session-state objects for carrying context across turns.

### `loader.py`

Present when the skill uses `load_from_disk` disposition for reference data. Loads files from the package directory at runtime.

---

## Fixtures

### `fixtures/<name>.py`

Each fixture file is a Python module defining a factory function:

```python
def fixture() -> tuple[dict, str, str]:
    inputs = {"query": "Will it rain in Tokyo tomorrow?"}
    fixture_id = "rain_check_city"
    description = "Standard rain check query for a named city"
    return inputs, fixture_id, description

ALL_FIXTURES = [fixture]
```

**Wrapper-rendered** (from `intermediate/fixtures_emission.json`). The inputs, fixture_id, and description come from the LLM's fixture generation (Step 4), but the Python code wrapping them is deterministically generated.

Fixtures are categorized by the compiler across at least three "C-categories" (happy path, edge case, adversarial/error case). Inspect `SETUP.md` for the fixture catalogue.

---

## Manifest and metadata

### `melleafy.json`

The package manifest. The stable, versioned contract that export adapters and integrations consume.

```json
{
  "format_version": "1.0",
  "manifest_version": "1.1.0",
  "package_name": "weather_mellea",
  "source_runtime": "agent_skills_std",
  "modality": "synchronous_oneshot",
  "archetype": "D1",
  "entry_signature": "run_pipeline(query: str) -> str",
  "pipeline_parameters": [
    {"name": "query", "type": "str", "required": true}
  ],
  "declared_env_vars": []
}
```

**Do not edit** — this is the contract the exporter and runtime tooling read. See [melleafy.json reference](../reference/melleafy-json.md).

### `mapping_report.md`

A human-readable table showing which spec element became which code construct. Useful for understanding why the compiler made the choices it did, and for spotting elements that were incorrectly mapped.

### `SETUP.md`

Structured setup instructions per section:
- Required environment variables
- Required Ollama models
- Stub catalogue (§8) — every stub with its signature, docstring, and example implementation

The stub catalogue in §8 is the canonical list of what needs implementing before the skill can run end-to-end.

---

## Intermediate directory

The `intermediate/` directory holds the compiler's IR. Don't edit these files — they are the compiler's internal state and will be overwritten on repair runs.

| File | Purpose |
|------|---------|
| `classification.json` | Five-axis spec classification |
| `inventory.json` | All elements with categories and tags |
| `element_mapping.json` | Element → Mellea primitive routing |
| `element_mapping_amendments.json` | Dispositions committed during Step 2.5 |
| `dependency_plan.json` | Dependency dispositions |
| `config_emission.json` | Typed IR the wrapper uses to render `config.py` |
| `fixtures_emission.json` | Typed IR the wrapper uses to render `fixtures/` |
| `runtime_directive.json` | BACKEND/MODEL_ID values injected at compile time |
| `mellea_api_ref.json` | Live Mellea API introspection (grounding for Step 5) |
| `step_7_report.json` | Lint results |
| `step_7b_report.json` | Smoke check results |

---

## What's safe to modify

| File | Safe to modify? | Notes |
|------|----------------|-------|
| `pipeline.py` | Yes | Don't change `run_pipeline` signature |
| `schemas.py` | Yes | Don't rename fields referenced by pipeline or fixtures |
| `config.py` | Yes | Re-run `mellea-skills validate` after changes |
| `slots.py` | Yes | |
| `tools.py` | Yes | |
| `constrained_slots.py` | **Required** | Fill stubs before using those code paths |
| `requirements.py` | Yes | |
| `fixtures/*.py` | Yes | Add new fixtures; don't break `ALL_FIXTURES` list |
| `melleafy.json` | No | Contract consumed by export tooling |
| `intermediate/` | No | Compiler state; overwritten on repair |
| `mapping_report.md` | No | Overwritten on repair |
