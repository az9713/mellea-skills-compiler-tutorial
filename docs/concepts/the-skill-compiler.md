# The Skill Compiler

`mellea-skills compile` runs a 10-step pipeline. Seven steps involve LLM calls (via Claude Code). Three are deterministic. This page explains each step and shows exactly what happens when the **`weather` skill** (`skills/weather/spec.md`) is compiled — using the actual intermediate artifacts from `examples/weather/weather_mellea/intermediate/`.

---

## The running example: `weather`

The weather skill's spec starts like this:

```markdown
---
name: weather
description: "Get current weather and forecasts via wttr.in or Open-Meteo.
  Use when: user asks about weather, temperature, or forecasts for any location.
  NOT for: historical weather data, severe weather alerts, or detailed
  meteorological analysis. No API key needed."
metadata: { "openclaw": { "emoji": "☔", "requires": { "bins": ["curl"] } } }
---

# Weather Skill

Get current weather conditions and forecasts.

## When NOT to Use
- Historical weather data → use weather archives/APIs
- ...

## Commands
### Current Weather
```bash
curl "wttr.in/London?format=3"
```
### Forecasts
```bash
curl "wttr.in/London?format=v2"
```
...
```

By the end of 10 steps, this Markdown file becomes a typed Python package with two LLM sessions, a `WeatherQueryType` enum with 10 values, a `@generative` location extractor, and a deterministic `fetch_weather` HTTP tool. Let's follow the compiler through each step.

---

## The two-layer architecture

Before the steps: two layers do different work.

**Layer 1 — `/mellea-fy` (LLM-driven)**: Claude reads the spec and writes Python files and JSON IR to disk.

**Layer 2 — deterministic wrapper**: Before and after the slash command, the Python wrapper pre-populates grounding artifacts and — critically — renders `config.py` and `fixtures/` from the compiler's JSON IR, never from LLM text. Claude Code deny rules block the LLM from writing those paths.

---

## Step 0: Five-axis classification

**Goal**: understand the spec before touching any of its content.

The compiler classifies the spec on five axes. For `weather`:

```json
// intermediate/classification.json
{
  "archetype": "D1",
  "archetype_confidence": 0.92,
  "shape": "One-shot",
  "tool_involvement": "Pipeline calls tools (deterministic)",
  "tool_involvement_variant": "P2",
  "source_runtime": "agent_skills_std",
  "source_runtime_scores": { "agent_skills_std": 3.5, "claude_code": 0.0, ... },
  "modality": "synchronous_oneshot",
  "modality_confidence": 0.90
}
```

**What this means for compilation:**

| Axis | Value | Consequence |
|------|-------|-------------|
| Archetype | `D1` (Integration) | Decompose only the intent classification layer; the HTTP call is deterministic |
| Shape | One-shot | Single decision session, not a sequential multi-phase pipeline |
| Tool involvement | `P2` | LLM classifies intent; Python constructs and executes the tool call |
| Source runtime | `agent_skills_std` | Use the `agent_skills_std` dialect doc for Steps 1a/1b |
| Modality | `synchronous_oneshot` | `run_pipeline(query: str) -> str` — one call, one return |

The compiler flagged one warning: *"Axis 5 inferred — no explicit modality signals found in frontmatter or body."* The modality was correctly inferred even without an explicit declaration.

These five values drive every subsequent step. A spec that scores high on `D2` would produce completely different decomposition logic.

---

## Steps 1a + 1b: Inventory

**Goal**: read all spec files, extract every meaningful element, tag each one.

Step 1a reads the files (for `agent_skills_std` dialect: just `spec.md`). Step 1b tags every element it finds.

The weather spec yields **15 elements**. Here are four representative ones:

```json
// intermediate/inventory.json (excerpt)
{
  "elements": [
    {
      "element_id": "elem_001",
      "source_lines": "1-3",
      "tag": "CONFIG",
      "category": "C1",
      "content_summary": "Skill name 'weather' and description defining in-scope/out-of-scope weather query types",
      "content_full": "---\nname: weather\ndescription: \"Get current weather and forecasts...\""
    },
    {
      "element_id": "elem_006",
      "source_lines": "24-30",
      "tag": "DECIDE",
      "category": "C2",
      "content_summary": "Out-of-scope gate: five query categories must be rejected before processing",
      "notes": "Implements early-exit branch in pipeline"
    },
    {
      "element_id": "elem_007",
      "source_lines": "34",
      "tag": "EXTRACT",
      "category": "—",
      "content_summary": "Extract a city name, region, or airport code from the user's query",
      "notes": "Downstream TOOL_TEMPLATE URL construction depends on this value"
    },
    {
      "element_id": "elem_008",
      "source_lines": "40-49",
      "tag": "TOOL_TEMPLATE",
      "category": "C6",
      "content_summary": "HTTP fetch templates for current weather conditions",
      "content_full": "curl \"wttr.in/London?format=3\"  ...",
      "aggregation_hint": "Unification candidate: elem_008/009/010/012/013/014 all resolve to HTTP fetches — consider single fetch_weather(location, endpoint_params)"
    }
  ]
}
```

**Key compiler insight at this step**: the aggregation hint on `elem_008` is the compiler spotting that six separate `curl` examples in the spec are all the same underlying tool with different parameters. Instead of generating six functions, Step 2.5 will unify them into one `fetch_weather(location, endpoint_params)` function. This is how the compiler avoids code bloat from repetitive spec sections.

---

## Step 2: Element mapping

**Goal**: route each element to a specific Mellea primitive.

Each element now gets a target file, a target symbol, and a Mellea primitive type:

```json
// intermediate/element_mapping.json (excerpt)
{
  "mappings": [
    {
      "element_id": "elem_001",
      "target_file": "config.py",
      "target_symbol": "SKILL_NAME",
      "primitive": "Final",
      "step_2_rationale": "CONFIG C1 → Final[str] constants in config.py"
    },
    {
      "element_id": "elem_006",
      "target_file": "pipeline.py",
      "target_symbol": "run_pipeline",
      "primitive": "m.instruct",
      "primitive_details": {
        "format_schema": "WeatherScopeDecision",
        "inline_position": "early_exit_gate",
        "grounding_context_keys": ["query"]
      },
      "step_2_rationale": "DECIDE C2 → m.instruct(format=WeatherScopeDecision) as early-exit gate"
    },
    {
      "element_id": "elem_007",
      "target_file": "slots.py",
      "target_symbol": "extract_location",
      "primitive": "@generative",
      "primitive_details": { "return_type": "str" },
      "step_2_rationale": "EXTRACT → @generative slot returning str"
    },
    {
      "element_id": "elem_008",
      "target_file": "tools.py",
      "target_symbol": "fetch_weather",
      "primitive": "function",
      "primitive_details": {
        "signature": "fetch_weather(location: str, endpoint_params: str) -> str",
        "implementation": "subprocess_curl"
      },
      "final_target_file": "pending_step_2.5"
    }
  ]
}
```

Notice `"final_target_file": "pending_step_2.5"` for the tool entries. The mapping is provisional — Step 2.5 must confirm the disposition (should it be a real implementation, a stub, or something else?) before code can be generated.

---

## Step 2.5: Dependency audit (deterministic — no LLM)

**Goal**: classify every external dependency and commit a disposition. This step has no LLM calls.

The compiler reads every element tagged C1–C9 and assigns a disposition from a fixed table. For the weather skill, five dependencies are audited:

```json
// intermediate/dependency_plan.json
{
  "generation_mode": "auto",
  "plan": [
    {
      "entry_id": "dep_001",
      "category": "c1_identity",
      "disposition": "bundle",
      "source_elements": ["elem_001", "elem_005"],
      "target": "config.py:SKILL_NAME"
    },
    {
      "entry_id": "dep_002",
      "category": "c2_operating_rules",
      "disposition": "bundle",
      "source_elements": ["elem_006"],
      "target": "config.py:OUT_OF_SCOPE_CATEGORIES"
    },
    {
      "entry_id": "dep_003",
      "category": "c8_runtime_environment",
      "disposition": "bundle",
      "source_elements": ["elem_003", "elem_015"],
      "target": "config.py:BACKEND",
      "prerequisites": ["curl binary must be installed"]
    },
    {
      "entry_id": "dep_004",
      "category": "c6_tools",
      "disposition": "real_impl",
      "source_elements": ["elem_008", "elem_009", "elem_010"],
      "target": "tools.py:fetch_weather",
      "subtype": "http",
      "prerequisites": ["HTTP access to wttr.in (public, no API key required)"]
    },
    {
      "entry_id": "dep_005",
      "category": "c6_tools",
      "disposition": "bundle",
      "source_elements": ["elem_011", "elem_012", "elem_013", "elem_014"],
      "target": "config.py:WEATHER_FORMAT_CODES"
    }
  ]
}
```

**The key decision**: `dep_004` gets `real_impl` because the spec contains actual `curl` commands with a known public endpoint. If the spec had said "call our internal weather service" without an endpoint, the disposition would be `stub` and Step 5 would generate `raise NotImplementedError(...)` instead.

**The aggregation payoff**: `elem_008`, `elem_009`, and `elem_010` — three separate spec sections describing different `curl` commands — are unified into a single `dep_004` entry, which produces a single `fetch_weather(location, endpoint_params)` function. Six spec sections → one Python function.

After Step 2.5, `element_mapping.json` is amended: `"pending_step_2.5"` entries are replaced with their final target files.

---

## Step 3: Skeleton emit

**Goal**: lock the `run_pipeline` signature and create empty Python file stubs.

The compiler writes empty Python files with correct imports and function signatures. The `run_pipeline` signature is fixed here and cannot change — fixtures in Step 4 are generated against it as grounding.

For weather, the skeleton locks in:

```python
# pipeline.py skeleton — bodies are empty at this point
def run_pipeline(query: str) -> str:
    ...

# slots.py skeleton
@generative
def extract_location(query: str) -> str:
    ...

# tools.py skeleton
def fetch_weather(location: str, endpoint_params: str) -> str:
    ...

# schemas.py skeleton
class WeatherQueryType(str, Enum): ...
class WeatherIntent(BaseModel): ...
```

The signature `run_pipeline(query: str) -> str` captures the D1/synchronous_oneshot classification: one string input, one string output.

---

## Step 4: Fixture generation

**Goal**: generate 5–8 sample inputs that cover happy paths, edge cases, and scope boundaries.

Fixtures are generated now — before code bodies — so they serve as grounding for Step 5. The compiler generates them from the skeleton's `run_pipeline` signature and the spec's `When to Use` / `When NOT to Use` sections.

The weather skill ships seven fixtures. Here is the actual generated fixture IR for `rain_check_city`:

```python
# fixtures/rain_check_city.py (wrapper-rendered from fixtures_emission.json)
def make_rain_check_city():
    inputs = {
        'query': 'Will it rain in Tokyo tomorrow?'
    }
    return (
        inputs,
        'rain_check_city',
        'Rain-probability query — pipeline classifies as rain_check, '
        'uses WEATHER_PRESET_RAIN_CHECK endpoint params, returns precipitation '
        'format response. Exercises C2 (query-type routing), C6 (tool with rain_check params).'
    )
```

Other fixtures cover: `current_summary` (London, standard query), `forecast_week` (Paris), `json_output`, `out_of_scope` (a historical weather query that must be rejected), `airport_code` (ORD), and `empty_location` (no city specified — edge case).

> **Note:** The fixture files you see in `fixtures/` are not written by the LLM. They are rendered by the deterministic wrapper from `intermediate/fixtures_emission.json` after the slash command completes.

---

## Step 5: Code body generation

**Goal**: fill the skeleton files with real, working Python code.

This is the longest step — Claude writes the actual bodies for every function, using the fixtures (Step 4) as grounding for expected behavior and `intermediate/mellea_api_ref.json` (live Mellea API introspection pre-populated by the wrapper) to avoid hallucinating API signatures.

**`slots.py`** — the `extract_location` slot becomes:

```python
# slots.py (generated)
from mellea import generative

@generative
def extract_location(query: str) -> str:
    """Set `result` to the city name, region, or IATA airport code mentioned
    in the query. If no location is mentioned, set `result` to an empty string."""
    ...
```

The `@generative` decorator handles everything. There is no `m.instruct` call here — the decorator provides it. The docstring is the prompt: the Mellea library uses it as the generation instruction.

**`schemas.py`** — the `WeatherIntent` and `WeatherQueryType` types, built from the DECIDE element (elem_006) and the dispatch table:

```python
# schemas.py (generated)
class WeatherQueryType(str, Enum):
    current_summary = "current_summary"
    current_detailed = "current_detailed"
    forecast_3day = "forecast_3day"
    forecast_week = "forecast_week"
    forecast_day0 = "forecast_day0"
    forecast_day1 = "forecast_day1"
    forecast_day2 = "forecast_day2"
    json_output = "json_output"
    rain_check = "rain_check"
    out_of_scope = "out_of_scope"

class WeatherIntent(BaseModel):
    query_type: WeatherQueryType = Field(
        description="Classify the weather query type. Use 'out_of_scope' for: "
                    "historical weather data, climate analysis, hyper-local data, "
                    "severe weather alerts, or aviation/marine weather."
    )
    location: Optional[str] = Field(default=None,
        description="City name, region, or IATA airport code if stated; otherwise null.")
    out_of_scope_reason: Optional[str] = Field(default=None, ...)
```

The 10-value enum comes directly from the spec's `When NOT to Use` section (5 out-of-scope categories → `out_of_scope` sentinel) combined with the `Commands` section (9 distinct `curl` patterns → 9 `WeatherQueryType` values).

**`tools.py`** — `fetch_weather`, generated with a domain guard because the spec said "rate limited; don't spam requests":

```python
# tools.py (generated)
def fetch_weather(location: str, endpoint_params: str) -> str:
    url = f"https://{WTTR_BASE_URL}/{location}{endpoint_params}"

    # Domain guard — only wttr.in is permitted
    parsed = urlparse(url)
    if parsed.hostname != WTTR_BASE_URL:
        raise ValueError(f"Constructed URL targets unexpected host '{parsed.hostname}'")

    result = subprocess.run(
        ["curl", "-s", "--max-time", str(HTTP_TIMEOUT), url],
        capture_output=True, text=True,
        timeout=HTTP_TIMEOUT + CURL_EXTRA_TIMEOUT,
    )
    if result.returncode != 0:
        raise RuntimeError(f"curl exited with code {result.returncode}: {result.stderr.strip()}")
    return result.stdout.strip()
```

The domain guard — a security control that prevents the compiled tool from being redirected to an attacker-controlled host — was inferred from the spec's "wttr.in only" intent, not from an explicit spec instruction.

**`pipeline.py`** — the orchestrator that calls all the above:

```python
# pipeline.py (generated) — abridged
def run_pipeline(query: str) -> str:
    # Session 1: extract location
    with start_session(BACKEND, MODEL_ID) as m:
        location_raw: str = extract_location(m, query=query)

    # Session 2: classify intent + scope check
    with start_session(BACKEND, MODEL_ID) as m:
        intent_thunk = m.instruct(
            "Classify this weather query and determine whether it is in scope.",
            model_options={ModelOption.SYSTEM_PROMPT: PREFIX_TEXT},
            grounding_context={
                "query": query,
                "extracted_location": location_raw,
                "out_of_scope_categories": OUT_OF_SCOPE_CATEGORIES,
            },
            format=WeatherIntent,
            strategy=RepairTemplateStrategy(loop_budget=LOOP_BUDGET),
        )
        intent = WeatherIntent.model_validate_json(intent_thunk.value)

    # Scope gate (deterministic Python — not another LLM call)
    if intent.query_type == WeatherQueryType.out_of_scope:
        return f"Out of scope ({intent.out_of_scope_reason}). Please consult a specialized source."

    # Deterministic dispatch table
    endpoint_params = _ENDPOINT_PARAMS[intent.query_type]
    return fetch_weather(_url_encode_location(intent.location), endpoint_params)
```

Two separate sessions, not one. The D1/P2 archetype pattern: first session extracts an entity (the `@generative` slot); second session classifies intent into a typed enum; Python does the rest. The LLM never touches the URL or the HTTP response.

---

## Step 6: Supporting artifacts

**Goal**: produce the human-readable and machine-readable package metadata.

The compiler emits four files:

**`mapping_report.md`** — a table showing every element and where it landed:

```
| elem_007 | EXTRACT | slots.py:extract_location | @generative slot |
| elem_008 | TOOL_TEMPLATE C6 | tools.py:fetch_weather | function real_impl |
| elem_012 | TOOL_TEMPLATE C6 | config.py:WEATHER_PRESET_RAIN_CHECK | bundle |
```

**`melleafy.json`** — the package manifest:

```json
{
  "manifest_version": "1.1.0",
  "package_name": "weather_mellea",
  "source_runtime": "agent_skills_std",
  "modality": "synchronous_oneshot",
  "archetype": "D1",
  "entry_signature": "run_pipeline(query: str) -> str",
  "pipeline_parameters": [
    { "name": "query", "type": "str", "required": true }
  ],
  "declared_env_vars": []
}
```

**`SETUP.md`** — backend setup instructions. For weather (T1, no stubs): "Pull `granite3.3:8b`, set `OLLAMA_API_URL`, install `curl`."

---

## Step 7: Structural lints (14 checks)

**Goal**: static validation before anything runs. The compiler verifies its own output.

For the weather skill, all lints pass:

```json
// intermediate/step_7_report.json
{
  "overall_verdict": "pass",
  "lints": [
    { "lint_id": "fixtures-loader-contract", "verdict": "pass" },
    { "lint_id": "bundled-asset-path-resolution", "verdict": "pass" },
    { "lint_id": "runtime-defaults-bound", "verdict": "pass" }
  ]
}
```

If a lint had failed, the compiler would enter a repair loop — regenerate only the failing files, re-run lints, up to 2 rounds. The most common lint failures and their causes:

| Lint | What it checks | Common cause |
|------|---------------|-------------|
| `import-soundness` | All imports resolve | Renamed symbol in one file, old name in another |
| `stdlib-arity` | Mellea API calls use correct signatures | LLM hallucinated a non-existent parameter |
| `schema-non-optional` | Required output fields are not `Optional` | LLM made required field nullable |
| `fixture-shape` | Every fixture returns `(dict, str, str)` | Missing the description string in the tuple |
| `runtime-defaults-bound` | `config.py` BACKEND/MODEL_ID match compile-time injection | LLM overwrote the wrapper-injected constants |

---

## Post-lint: wrapper rendering and smoke check

**Goal**: render `config.py` and `fixtures/` deterministically, then run one fixture.

After the slash command exits, the Python wrapper:

1. **Renders `config.py`** from `intermediate/config_emission.json` — not from LLM text. The actual weather `config.py`:

```python
# config.py — wrapper-rendered, not LLM-written
SKILL_NAME: Final[str] = 'weather'
PREFIX_TEXT: Final[str] = 'You are a weather assistant...'
OUT_OF_SCOPE_CATEGORIES: Final[str] = 'historical weather data, climate analysis...'
WEATHER_PRESET_RAIN_CHECK: Final[str] = '?format=%l:+%c+%p'
WEATHER_PRESET_WEEK_FORECAST: Final[str] = '?format=v2'
BACKEND: Final[str] = 'ollama'          # injected by wrapper from runtime_defaults.json
MODEL_ID: Final[str] = 'granite3.3:8b'  # injected by wrapper
WTTR_BASE_URL: Final[str] = 'wttr.in'
LOOP_BUDGET: Final[int] = 3
```

2. **Renders `fixtures/*.py`** from `intermediate/fixtures_emission.json` — the seven fixture files including `rain_check_city.py` above.

3. **Runs the smoke check** — executes `rain_check_city` against the pipeline. If the pipeline returns without raising, exit code 0. If a `NotImplementedError` stub is hit, exit code 12.

For weather, the smoke check passes. Running it yourself:

```bash
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city
# → Tokyo: ⛅️  Partly cloudy +14°C
```

---

## The complete transformation

Looking at what the 10 steps produced from a 113-line Markdown spec:

| Spec element | Step that processed it | What it became |
|-------------|----------------------|----------------|
| `description:` field (frontmatter) | Step 0 → Step 2 → Wrapper | `SKILL_DESCRIPTION` constant in `config.py` |
| `When NOT to Use` section (5 categories) | Step 1b (elem_006: DECIDE) → Step 2 → Step 5 | `OUT_OF_SCOPE_CATEGORIES` constant + early-exit gate in `pipeline.py` |
| `Always include a city...` sentence | Step 1b (elem_007: EXTRACT) → Step 2 → Step 5 | `extract_location()` `@generative` slot in `slots.py` |
| 6 `curl` command blocks | Step 1b (elem_008–014: TOOL_TEMPLATE) → Step 2.5 (unified) → Step 5 | `fetch_weather(location, endpoint_params)` in `tools.py` + 3 preset constants in `config.py` |
| `"Will it rain?"` quick response | Step 1b (elem_013) → Step 2.5 (bundle) → Wrapper | `WEATHER_PRESET_RAIN_CHECK = '?format=%l:+%c+%p'` in `config.py` |
| `metadata.openclaw.requires.bins: [curl]` | Step 1b (elem_003: CONFIG C8) → Step 2.5 → Wrapper | `REQUIRED_BINS = 'curl'` in `config.py` |
| (Compiler inferred from D1/P2 classification) | Step 0 → Step 5 | Two-session pipeline structure — never explicitly stated in the spec |

---

## What can go wrong

| Symptom | Cause | Fix |
|---------|-------|-----|
| Compile times out after 75 min | Large spec or slow Claude session | `--repair-mode` |
| Exit code 11 | Step 7 lints failed and repair loop didn't converge | `--repair-mode`; check `intermediate/step_7_report.json` |
| Exit code 12 | Smoke check failed — a fixture raised | Likely a stub; see [Fill stubs](../guides/fill-stubs.md) |
| No `*_mellea/` directory after compile | The slash command failed before Step 3 | Check Claude session output; `--repair-mode` |

See [Repair a failed compile](../guides/repair-failed-compile.md) for the full diagnosis flow.
