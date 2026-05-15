# melleafy.json Reference

`melleafy.json` is the compiled package manifest. It is the versioned contract that the `mellea-skills export` command, runtime tooling, and external integrations consume to understand a compiled skill package.

---

## Schema

```json
{
  "format_version": "1.0",
  "manifest_version": "1.1.0",
  "package_name": "weather_mellea",
  "generated_at": "2026-04-30T00:00:00Z",
  "melleafy_version": "4.3.2",
  "source_runtime": "agent_skills_std",
  "modality": "synchronous_oneshot",
  "archetype": "D1",
  "categories_resolved": ["C1", "C2", "C6", "C8"],
  "entry_signature": "run_pipeline(query: str) -> str",
  "pipeline_parameters": [
    {
      "name": "query",
      "type": "str",
      "description": "Natural-language weather request",
      "required": true
    }
  ],
  "declared_env_vars": []
}
```

---

## Fields

| Field | Type | Description |
|-------|------|-------------|
| `format_version` | string | JSON schema version of this file. Currently `"1.0"`. |
| `manifest_version` | string | Semver version of the manifest contract. `mellea-skills export` requires ≥ `1.0.0`. Current: `1.1.0`. |
| `package_name` | string | Python package name. Matches the directory name (`<name>_mellea`). Used by the exporter to locate the package. |
| `generated_at` | ISO 8601 string | Timestamp of when this file was generated. |
| `melleafy_version` | string | Version of the `/mellea-fy` command spec used to compile this package. |
| `source_runtime` | string | Dialect of the source spec. One of: `agent_skills_std`, `claude_code`, `openclaw`, `crewai`, `langgraph`, `letta`, `hybrid`, `unknown`. |
| `modality` | string | How the pipeline is invoked. See [Modality values](#modality-values) below. |
| `archetype` | string | Reasoning archetype. One of: `A`, `B`, `C`, `D1`, `D2`, `E`. See [Skill archetypes](../concepts/skill-archetypes.md). |
| `categories_resolved` | array of strings | C1–C9 dependency categories present in this skill. Tells integrations what kinds of external dependencies exist. |
| `entry_signature` | string | Python function signature of `run_pipeline` as a string. The exporter parses this to generate adapter code. |
| `pipeline_parameters` | array | Structured parameter list. Each entry: `name`, `type`, `description`, `required`. |
| `declared_env_vars` | array | Environment variables the pipeline reads at runtime. Each entry: `name`, `description`, `required`. |

---

## Modality values

| Value | Meaning | `run_pipeline` shape |
|-------|---------|---------------------|
| `synchronous_oneshot` | Single request → single response | `run_pipeline(**kwargs) -> ResultType` |
| `streaming` | Single request → streamed response | `run_pipeline(**kwargs) -> Generator[str, None, None]` |
| `conversational_session` | Multi-turn conversation | `run_pipeline(**kwargs, session_id: str) -> ResultType` |
| `scheduled` | Runs on a schedule with no user input | `run_pipeline() -> ResultType` |
| `event_triggered` | Triggered by an external event | `run_pipeline(event: dict) -> ResultType` |
| `heartbeat` | Periodic check or monitor | `run_pipeline() -> ResultType` |

---

## Stable contract

These fields are stable and safe to build against across releases:

- `entry_signature` — the callable surface of the compiled package
- `pipeline_parameters` — the typed input schema
- `declared_env_vars` — the environment contract
- `manifest_version` — semver-versioned; the exporter checks ≥ 1.0.0

These fields are informational and may change:

- `melleafy_version` — tied to internal compiler versioning
- `categories_resolved` — compiler heuristic, not contractual

---

## Programmatic access

Read `melleafy.json` without importing the package:

```python
import json
from pathlib import Path

manifest = json.loads((Path("skills/weather/weather_mellea") / "melleafy.json").read_text())
print(manifest["entry_signature"])  # run_pipeline(query: str) -> str
print(manifest["modality"])         # synchronous_oneshot
```

This is the recommended way to introspect a compiled package before deciding how to integrate it — no Python import, no Ollama, no dependencies beyond stdlib.
