# Codebase map

This map links the high-level concepts to concrete files.

## Root files

| File | Role |
|---|---|
| `README.md` | Public project overview, install steps, quickstart, examples, limitations, citation. |
| `pyproject.toml` | Package metadata, dependencies, Python version, CLI entry point. |
| `FAQ.md` | User-facing questions. |
| `CONTRIBUTING.md` | Contribution guidance. |
| `LICENSE` | Apache 2.0 license. |

## CLI and shared models

| File | Role |
|---|---|
| `src/mellea_skills_compiler/cli.py` | Typer app and command options. |
| `src/mellea_skills_compiler/models.py` | Dataclasses for `NexusRisk`, `GovernanceAction`, `PolicyManifest`, `RequirementClassification`, and `ComplianceSummary`. |
| `src/mellea_skills_compiler/enums.py` | Enums for Claude stream records, inference engines, model defaults, spec file names, taxonomies, and coverage levels. |
| `src/mellea_skills_compiler/inference.py` | Ollama inference service factory and cache for risk and Guardian models. |
| `src/mellea_skills_compiler/toolkit/file_utils.py` | Spec parser, dynamic pipeline import, fixture loader. |
| `src/mellea_skills_compiler/toolkit/logging.py` | Rich colored logger configuration. |

## Compiler implementation

| File | Role |
|---|---|
| `src/mellea_skills_compiler/compile/mellea_skills.py` | Main compile and validate implementation. |
| `src/mellea_skills_compiler/compile/claude_directives.py` | Package naming, companion mirroring, runtime defaults, compile settings, system prompt. |
| `src/mellea_skills_compiler/compile/grounding.py` | Writes cached Mellea API reference and docs index. |
| `src/mellea_skills_compiler/compile/proxy.py` | Local proxy that strips `context_management` from Anthropic API requests. |
| `src/mellea_skills_compiler/compile/writer_renderer.py` | Runs deterministic writer modules against emitted IR. |
| `src/mellea_skills_compiler/compile/lints.py` | Python structural lints and lint report writer. |
| `src/mellea_skills_compiler/compile/smoke_check.py` | Post-lint fixture execution and smoke report writer. |

## Slash-command compiler spec

| File or directory | Role |
|---|---|
| `.claude/commands/mellea-fy.md` | Main 10-step compiler orchestrator. |
| `.claude/commands/mellea-fy-classify.md` | Source classification. |
| `.claude/commands/mellea-fy-inventory.md` | Source inventory and element tagging. |
| `.claude/commands/mellea-fy-map.md` | Source element to Mellea primitive mapping. |
| `.claude/commands/mellea-fy-deps.md` | Dependency audit and disposition rules. |
| `.claude/commands/mellea-fy-fixtures.md` | Fixture generation rules. |
| `.claude/commands/mellea-fy-generate.md` | Skeleton and code body generation. |
| `.claude/commands/mellea-fy-artifacts.md` | `mapping_report.md`, `melleafy.json`, setup docs, README. |
| `.claude/commands/mellea-fy-validate.md` | Formal lint specification and repair loop. |
| `.claude/commands/mellea-fy-repair.md` | Repair workflow. |
| `.claude/commands/mellea-fy-behaviours.md` | Known Mellea behavior reference. |
| `.claude/schemas/*.json` | JSON schemas for compiler IR and manifest artifacts. |
| `.claude/melleafy/writers/*.py` | Deterministic code writers for generated artifacts. |
| `.claude/data/runtime_defaults.json` | Default backend and model id for compiled skills. |

## Governance and certification

| File | Role |
|---|---|
| `src/mellea_skills_compiler/certification/__init__.py` | Converts parsed skill metadata into use-case text. |
| `src/mellea_skills_compiler/certification/ingest.py` | Static risk and policy generation for raw specs. |
| `src/mellea_skills_compiler/certification/pipeline.py` | Full certification pipeline and runtime `run` implementation. |
| `src/mellea_skills_compiler/certification/nexus_policy.py` | AI Atlas Nexus risk/action identification and policy markdown. |
| `src/mellea_skills_compiler/certification/classification.py` | Skill sensitivity and governance requirement classification. |
| `src/mellea_skills_compiler/certification/report.py` | Certification report and audit evidence extraction. |
| `src/mellea_skills_compiler/certification/data/*.yaml` | AI Atlas Nexus pipeline control data and mappings. |

## Guardian hooks

| File | Role |
|---|---|
| `src/mellea_skills_compiler/guardian/__init__.py` | Register and deregister Guardian plus audit plugins. |
| `src/mellea_skills_compiler/guardian/guardian_hook.py` | Guardian audit/enforce plugins and risk check calls. |
| `src/mellea_skills_compiler/guardian/audit_trail.py` | JSONL audit plugin for generation, component, validation, and tool events. |

## Export

| File | Role |
|---|---|
| `src/mellea_skills_compiler/export/exporter.py` | Five-stage export pipeline: validate, load, translate, emit, lint. |
| `src/mellea_skills_compiler/export/targets/langgraph.py` | LangGraph adapter renderer. |
| `src/mellea_skills_compiler/export/targets/mcp.py` | MCP FastMCP adapter renderer. |
| `src/mellea_skills_compiler/export/targets/claude_code.py` | Claude Code skill adapter renderer. |

## Examples

| Example | What it demonstrates |
|---|---|
| `examples/weather/weather_mellea/` | Fetch and summarize pattern. Typed intent classification followed by deterministic HTTP fetch. |
| `examples/sentry-find-bugs/sentry_find_bugs_mellea/` | Structured code-review analysis, security checklist, and optional host stubs. |
| `examples/superpowers-systematic-debugging/superpowers_systematic_debugging_mellea/` | Multi-phase constrained reasoning with architectural issue detection. |
| `examples/clawdefender/clawdefender_mellea/` | Adversarial classification, local scripts, URL/prompt validation, and T3 external dependencies. |

## Tests

| Test file | Coverage |
|---|---|
| `tests/mellea_skills_compiler/toolkit/test_file_utils.py` | Spec parsing and fixture/pipeline loader behavior. |
| `tests/mellea_skills_compiler/compile/test_lints.py` | Fixture contract lint, bundled asset path lint, runtime default lint, lint runner. |
| `tests/mellea_skills_compiler/compile/test_smoke_check.py` | Smoke-check exception classification and fixture execution reports. |
| `tests/mellea_skills_compiler/compile/test_grounding.py` | Atomic write, API reference generation, docs index caching/fallback. |
| `tests/mellea_skills_compiler/compile/test_mellea_skills_helpers.py` | Compile helper behavior. |
| `tests/mellea_skills_compiler/certification/test_nexus_policy.py` | Policy dataclasses and manifest serialization. |
| `tests/mellea_skills_compiler/certification/test_compliance.py` | Coverage levels, compliance summaries, audit trail loading, report generation. |
| `tests/mellea_skills_compiler/guardian/test_audit_trail.py` | AuditTrailPlugin initialization, event writes, hooks, summary. |

## High-risk files to edit carefully

| File | Why |
|---|---|
| `compile/mellea_skills.py` | Orchestrates external Claude process, proxy, writers, validation, and cleanup. Small mistakes can break compile end to end. |
| `compile/claude_directives.py` | Controls writer deny rules and runtime defaults. Mis-scoped paths can make Claude write to the wrong package. |
| `toolkit/file_utils.py` | Dynamic import and fixture loading are used by run, certify, and smoke check. |
| `guardian/guardian_hook.py` | Runtime enforcement path. Blocking behavior must be precise. |
| `export/exporter.py` | Manifest compatibility and output emission. Breaking this affects all export targets. |
| `.claude/commands/mellea-fy*.md` | The compiler spec. Prose changes can alter generated code behavior. |

## Known code-level observations

These are implementation observations from the current tree, not product guarantees.

| Observation | Impact |
|---|---|
| Several docs and source comments contain mojibake characters from non-ASCII punctuation. | Published docs may render oddly in some terminals. New docs in this folder use ASCII. |
| Some formal validation rules remain in slash-command prose rather than Python. | Validation quality partly depends on model self-application. |
| `compile/mellea_skills.py` calls `subprocess.call("clear")`. | On Windows PowerShell this may not behave the same as Unix shells. |
| `Guardian` registration writes `runtime_audit.jsonl`, while the README artifact tree calls it `audit_trail.jsonl`. | User-facing docs and actual output naming should be reconciled. |
| Export docs mention branch-specific history, while the current package includes `export` command code. | Some existing docs may be stale relative to the checked-out source. |

