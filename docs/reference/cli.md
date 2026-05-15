# CLI Reference

All `mellea-skills` subcommands. Run `mellea-skills --help` or `mellea-skills <command> --help` for live help text.

---

## `mellea-skills compile`

Decompose a skill spec into a typed Mellea pipeline.

```bash
mellea-skills compile <spec_path> [OPTIONS]
```

| Argument / Option | Type | Default | Description |
|-------------------|------|---------|-------------|
| `spec_path` | path | required | Path to a `.md` spec file or a skill directory containing `spec.md` / `SKILL.md` |
| `--model` / `-m` | string | Sonnet (auto-detected) | Claude model for compilation. Must be available via your API key. |
| `--timeout` / `-t` | int | 4500 | Claude session timeout in seconds (75 min). Increase for large specs. |
| `--repair-mode` / `-r` | flag | false | Resume a partial or failed compile. See [Repair a failed compile](../guides/repair-failed-compile.md). |
| `--no-run` | flag | false | Skip the post-compile fixture smoke check. Structural lints still run. |
| `--refresh-cache` | flag | false | Force refresh of Mellea API reference grounding caches at `~/.cache/mellea-skills-compiler/`. |
| `--skill-backend` | string | from `runtime_defaults.json` | Override the LLM backend used by the compiled skill at runtime (e.g., `ollama`). |
| `--skill-model` | string | from `runtime_defaults.json` | Override the model ID used by the compiled skill at runtime (e.g., `granite3.3:8b`). |

**Exit codes:**
- `0` — compile succeeded, lints passed, smoke check passed or skipped
- `1` — compile failed (Claude error, timeout, file not found)
- `11` — lints failed
- `12` — smoke check failed (lint pass, fixture raised)

**Example:**

```bash
# Compile with default Sonnet model:
mellea-skills compile skills/weather/spec.md

# Compile with Opus, no smoke check (useful for CI):
mellea-skills compile skills/weather/spec.md --model claude-opus-4-5 --no-run

# Resume a failed compile:
mellea-skills compile skills/weather/spec.md --repair-mode

# Specify the runtime model for the compiled skill:
mellea-skills compile skills/weather/spec.md --skill-model llama3.1:8b
```

---

## `mellea-skills validate`

Run structural lints and optionally the smoke check against a compiled package without recompiling.

```bash
mellea-skills validate <pipeline_dir> [OPTIONS]
```

| Argument / Option | Type | Default | Description |
|-------------------|------|---------|-------------|
| `pipeline_dir` | path | required | Compiled skill directory (contains `melleafy.json`). |
| `--no-run` | flag | false | Skip the fixture smoke check. Run lints only. |
| `--all` | flag | false | Run all fixtures (default: first fixture only). |

**Exit codes:** Same as `compile` (0, 11, 12).

**Example:**

```bash
mellea-skills validate skills/weather/weather_mellea
mellea-skills validate skills/weather/weather_mellea --all
mellea-skills validate skills/weather/weather_mellea --no-run
```

---

## `mellea-skills run`

Execute a compiled pipeline against a specific fixture.

```bash
mellea-skills run <pipeline_dir> --fixture <fixture_id> [OPTIONS]
```

| Argument / Option | Type | Default | Description |
|-------------------|------|---------|-------------|
| `pipeline_dir` | path | required | Compiled skill directory. |
| `--fixture` / `-f` | string | required | Fixture ID to run (filename without `.py`). |
| `--enforce` / `-e` | flag | false | Block execution when Guardian detects a risk (default: audit-only). |
| `--no-guardian` / `-ng` | flag | false | Skip Guardian checks even if a policy manifest exists. |

**Exit codes:**
- `0` — fixture completed (or was blocked by Guardian in enforce mode — that's a valid exit)
- `1` — unexpected error

**Example:**

```bash
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city
mellea-skills run examples/clawdefender/clawdefender_mellea --fixture prompt_injection_critical --enforce
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city --no-guardian
```

---

## `mellea-skills ingest`

Run risk analysis on a skill spec — produce a `PolicyManifest` without running full certification.

```bash
mellea-skills ingest <spec_path> [OPTIONS]
```

| Argument / Option | Type | Default | Description |
|-------------------|------|---------|-------------|
| `spec_path` | path | required | Path to the skill spec file. |
| `--dry-run` | flag | false | Preview what would be ingested without making LLM calls. |
| `--model` / `-m` | string | Ollama default | Model for risk identification via Nexus. |
| `--inference-engine` / `-i` | enum | `ollama` | Inference service. Currently only `ollama` is supported. |

Outputs: `audit/policy_manifest.json` and `audit/POLICY.md` adjacent to the spec.

---

## `mellea-skills certify`

Run the full certification pipeline on a compiled skill.

```bash
mellea-skills certify <pipeline_dir> [OPTIONS]
```

| Argument / Option | Type | Default | Description |
|-------------------|------|---------|-------------|
| `pipeline_dir` | path | required | Compiled skill directory. |
| `--fixture` / `-f` | string | all fixtures | Run only this fixture (by fixture ID). |
| `--enforce` / `-e` | flag | false | Block on risk detection. |
| `--model` / `-m` | string | Ollama default | Model for risk identification (Nexus). |
| `--guardian-model` / `-g` | string | Guardian default | Model for risk assessment (Guardian checks). |
| `--inference-engine` / `-i` | enum | `ollama` | Inference service. Currently only `ollama` is supported. |

Outputs to `audit/` adjacent to the pipeline directory:
- `policy_manifest.json` — PolicyManifest from Nexus
- `POLICY.md` — human-readable policy document
- `CERTIFICATION.md` — certification report with evidence chains
- `audit_trail.jsonl` — per-generation Guardian verdicts
- `pipeline_report.json` — fixture execution results

**Example:**

```bash
mellea-skills certify examples/weather/weather_mellea
mellea-skills certify examples/clawdefender/clawdefender_mellea --enforce
mellea-skills certify examples/weather/weather_mellea --fixture rain_check_city
mellea-skills certify examples/weather/weather_mellea \
  --model granite3.3:8b \
  --guardian-model ibm/granite3.3-guardian:8b \
  --inference-engine ollama
```

---

## `mellea-skills export` [EXPERIMENTAL]

Export a compiled skill package to a deployment harness.

```bash
mellea-skills export <package_path> --target <target> [--force]
```

| Argument / Option | Type | Default | Description |
|-------------------|------|---------|-------------|
| `package_path` | path | required | Compiled skill directory (contains `melleafy.json`), or parent containing one `*_mellea/` subdirectory. |
| `--target` / `-t` | string | required | Deployment target: `mcp`, `langgraph`, or `claude-code`. |
| `--force` / `-f` | flag | false | Overwrite output directory if it already exists. |

Output is written to `<package_name>/<package_name>-<target>/` inside the skill directory. The compiled package is bundled inside the export.

> **Warning:** This command is experimental. Output structure and CLI interface may change between releases without a deprecation period.

**Example:**

```bash
mellea-skills export skills/weather/weather_mellea --target mcp
mellea-skills export skills/weather/weather_mellea --target langgraph --force
```
