# Run and Test Fixtures

Fixtures are sample inputs for a compiled pipeline. The compiler generates 5–8 fixtures per skill; you can add more. They serve as smoke tests, regression checks, and certification inputs.

---

## Running a specific fixture

```bash
mellea-skills run <package_dir> --fixture <fixture_id>
```

The `fixture_id` is the filename without `.py`:

```bash
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city
```

To list available fixtures:

```bash
ls examples/weather/weather_mellea/fixtures/
# rain_check_city.py  forecast_week.py  json_output.py  ...
```

---

## Guardian flags during run

By default, Guardian runs in audit mode — it logs verdicts but doesn't interrupt:

```bash
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city
# [GUARDIAN] harm: SAFE (confidence: 0.98)
# [GUARDIAN] social_bias: SAFE (confidence: 0.99)
```

To block execution when Guardian detects a risk:

```bash
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city --enforce
```

To skip Guardian entirely (useful for debugging without Ollama for Guardian):

```bash
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city --no-guardian
```

---

## The smoke check

`mellea-skills compile` runs one fixture automatically after compile (the "smoke check"). This verifies the compiled package doesn't immediately crash.

Exit codes:
- `0` — lints passed, smoke check passed or skipped (backend unreachable)
- `11` — a lint failed
- `12` — the smoke check fixture raised an exception

Skip the smoke check in CI (lints still run):

```bash
mellea-skills compile spec.md --no-run
```

Run lints and smoke check manually after the fact:

```bash
mellea-skills validate <package_dir>             # first fixture only
mellea-skills validate <package_dir> --all       # all fixtures
```

---

## What fixtures look like

Each fixture file exports an `ALL_FIXTURES` list of factory functions:

```python
def rain_check_city() -> tuple[dict, str, str]:
    inputs = {"query": "Will it rain in Tokyo tomorrow?"}
    fixture_id = "rain_check_city"
    description = "Standard rain-check query for a named city"
    return inputs, fixture_id, description

ALL_FIXTURES = [rain_check_city]
```

The `inputs` dict unpacks as kwargs to `run_pipeline`. The `fixture_id` uniquely identifies the fixture for CLI selection and audit trail logging.

---

## The three fixture C-categories

The compiler generates fixtures across three categories:

| Category | What it covers | Examples |
|----------|---------------|---------|
| Happy path | Normal, well-formed inputs that should succeed | "What's the weather in London?" |
| Edge cases | Boundary inputs, empty inputs, ambiguous cases | Empty query, query with no location |
| Adversarial / error | Inputs designed to trigger error paths or scope gates | Out-of-scope request, injection attempt |

Aim for at least one fixture in each category. The compiler targets this distribution; add more as you discover edge cases.

---

## Adding your own fixtures

Add a new file in `fixtures/`:

```python
# fixtures/extreme_location.py

def extreme_location() -> tuple[dict, str, str]:
    inputs = {"query": "What's the weather at the South Pole?"}
    fixture_id = "extreme_location"
    description = "Test geographic edge case — location at extreme latitude"
    return inputs, fixture_id, description

ALL_FIXTURES = [extreme_location]
```

Run it:

```bash
mellea-skills run <package_dir> --fixture extreme_location
```

---

## The skill tier system

Skills are classified on how much setup is needed before fixtures run:

| Tier | What's needed | Examples |
|------|---------------|---------|
| **T1** | Only Ollama on the host — no stubs, no env vars, no extra services | `weather` |
| **T2** | One stub or one missing reference file blocks one branch | `sentry-find-bugs` (search_fn stub) |
| **T3** | External service, credential, or CLI tool required before any fixture completes | `clawdefender` (requires `clawhub`, `jq`, `npx`) |

T1 skills run immediately after install. T2 skills run partially (guarded stubs degrade gracefully) — full T2 requires filling stubs. T3 skills need external setup first.

Check a skill's tier in `SETUP.md §1` or in `skills/README.md`.

See also the detailed [Skill tiers reference](../reference/skill-tiers.md).

---

## Fixture execution timing

Rough expected runtimes with `granite3.3:8b` on a modern laptop via Ollama:

| Skill type | Per fixture |
|-----------|------------|
| D1 (integration) | 30–90 seconds |
| A (analysis, few phases) | 1–2 minutes |
| A (analysis, many phases) | 2–4 minutes |
| C (diagnosis) | 3–5 minutes |

Certification adds Nexus risk identification (~1 minute) plus Guardian overhead (~10–30% per invocation).

---

## Debugging a failing fixture

**`NotImplementedError`** — a stub was hit. See [Fill stubs](fill-stubs.md).

**`ConnectionError`** — Ollama is unreachable. Check `OLLAMA_API_URL` and `ollama list`.

**`ValidationError`** from Pydantic — the LLM returned a response that doesn't match the expected schema. This can mean:
- A model quality issue (the chosen model doesn't follow the schema reliably)
- A repair loop that exhausted its budget
- A schema field that's too constrained

Check `config.py` for `LOOP_BUDGET` — increasing it allows more repair attempts.

**`PluginViolationError`** — Guardian blocked the generation in enforce mode. Run without `--enforce` to see what was generated before the block.
