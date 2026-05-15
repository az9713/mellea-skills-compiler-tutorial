# System Design

How the Mellea Skills Compiler is architected: component boundaries, data flows, and why things are split the way they are.

---

## High-level architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  COMPILE LAYER                                                  │
│                                                                 │
│  mellea-skills compile                                          │
│       │                                                         │
│       ├─ [deterministic] companion-dir mirror                   │
│       ├─ [deterministic] Mellea API ref pre-population          │
│       ├─ [deterministic] runtime-defaults injection             │
│       │                                                         │
│       ├─ [LLM]  Claude Code + /mellea-fy                        │
│       │         10-step decomposition                           │
│       │         → intermediate JSON IR                          │
│       │         → Python source files                           │
│       │                                                         │
│       ├─ [deterministic] config.py render (from config_emission.json)
│       ├─ [deterministic] fixtures/ render (from fixtures_emission.json)
│       ├─ [deterministic] 14 structural lints                    │
│       └─ [LLM + backend] fixture smoke check                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                    ↓ compiled package
┌─────────────────────────────────────────────────────────────────┐
│  RUNTIME LAYER                                                  │
│                                                                 │
│  <name>_mellea/                                                 │
│       pipeline.py ─── Mellea library ─── Ollama / granite3.3:8b │
│       schemas.py                                                │
│       config.py        ↑ hooks                                  │
│                        │                                        │
│               Granite Guardian 3.3 8B                           │
│               (audit or enforce mode)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                    ↓ runtime data
┌─────────────────────────────────────────────────────────────────┐
│  CERTIFICATION LAYER                                            │
│                                                                 │
│  mellea-skills certify                                          │
│       │                                                         │
│       ├─ AI Atlas Nexus → PolicyManifest                        │
│       ├─ Guardian hook configuration                            │
│       ├─ Fixture execution (with Guardian)                      │
│       ├─ Compliance classification                              │
│       └─ Report generation                                      │
│                                                                 │
│  audit/                                                         │
│    CERTIFICATION.md  POLICY.md  audit_trail.jsonl               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component boundaries

### The Claude boundary

Claude Code is the compiler's LLM engine. It runs as a subprocess (`claude -p --allowed-tools Read,Write,Edit ...`), not as a library import. This is the largest architectural boundary in the system.

**What Claude does**:
- Reads the spec and intermediate artifacts
- Writes Python files (`pipeline.py`, `schemas.py`, `slots.py`, `tools.py`, etc.)
- Writes intermediate JSON IR (`classification.json`, `inventory.json`, etc.)

**What Claude cannot do** (enforced by deny rules in the `--settings` passed to Claude):
- Write `config.py` (wrapper-rendered from `config_emission.json`)
- Write any file under `fixtures/` (wrapper-rendered from `fixtures_emission.json`)

This means the two files that have the strongest correctness contract — the constants file and the test inputs — are always generated the same way regardless of how Claude phrases a request.

### The Mellea library boundary

The compiled skill runs on the Mellea library at runtime. Mellea is the boundary between the compiled code and the LLM backend.

**Mellea's role at runtime**:
- `start_session(BACKEND, MODEL_ID)` — creates and manages an LLM session
- `m.instruct(format=Schema, strategy=RepairStrategy)` — executes structured generation with typed output
- `@generative` — decorates extraction slots to run as single-purpose LLM calls
- Hook system — allows Guardian to intercept every generation before it returns

The compiled skill calls Mellea APIs; Mellea calls Ollama. The skill never calls Ollama directly.

### The Nexus boundary

AI Atlas Nexus is queried during certification. It is a governance knowledge graph that accepts a text description and returns a structured `PolicyManifest`. It runs locally via Ollama.

Nexus is queried once per certification run (or per `mellea-skills ingest` call). Its output (the `PolicyManifest`) is persisted to `audit/policy_manifest.json` and reused for compliance classification without re-querying Nexus.

---

## Data flow through compilation

```
spec.md
    │ parse frontmatter + body
    ▼
classification.json (5 axes)
    │ source_runtime → dialect selection
    ▼
inventory.json (elements with C1-C9 categories)
    │ element types + aggregation hints
    ▼
element_mapping.json (element → Mellea primitive)
    │ TOOL_TEMPLATE entries: "pending_step_2.5"
    ▼
dependency_plan.json (dispositions)
element_mapping_amendments.json (final targets)
    │ disposition → generated file
    ▼
[Step 3] Skeleton Python files (run_pipeline signature locked)
    │ signature as grounding
    ▼
[Step 4] fixtures_emission.json (fixture IR)
    │ fixture inputs as grounding for Step 5
    ▼
[Step 5] Populated Python files
    │ grounded by mellea_api_ref.json (real API signatures)
    ▼
[Step 6] melleafy.json, mapping_report.md, SETUP.md
    │ artifacts
    ▼
[Step 7] step_7_report.json (lint results)
    │ pass
    ▼
[Wrapper] config.py (from config_emission.json)
[Wrapper] fixtures/*.py (from fixtures_emission.json)
    │
    ▼
Compiled package on disk
```

---

## Why the intermediate representation exists

The IR files (`classification.json`, `inventory.json`, etc.) serve three purposes:

**Reproducibility**: a failed compile can be resumed from the last valid IR file. Without IR, every failure is a full restart.

**Auditability**: the IR records every decision the compiler made. `mapping_report.md` renders this for human review; `element_mapping_amendments.json` records every disposition that was applied.

**Separation of concerns**: the LLM produces typed JSON IR; the deterministic wrapper consumes it to render Python. The correctness of `config.py` and `fixtures/` depends only on the wrapper's behavior, not the LLM's.

---

## The proxy layer

`mellea_skills.py:compile()` starts a local HTTP proxy that strips the `context_management` field from Anthropic API requests before forwarding them. This field is automatically added by Claude Code but rejected by IBM LiteLLM gateways. The proxy is transparent to all other traffic.

This is a workaround for a specific IBM gateway incompatibility, not a general feature. It runs as a daemon thread during compilation and shuts down when the Claude session ends.

---

## Package structure

```
src/mellea_skills_compiler/
  cli.py                  ← Typer CLI — all subcommands
  compile/
    mellea_skills.py      ← compile() and validate() implementation
    claude_directives.py  ← system prompt building, grounding pre-population
    grounding.py          ← Mellea API ref and doc index generation
    lints.py              ← 14 structural lints
    smoke_check.py        ← post-compile fixture execution
    proxy.py              ← context_management stripping proxy
    writer_renderer.py    ← deterministic config.py / fixtures/ rendering
  certification/
    pipeline.py           ← full_pipeline() and skill_pipeline()
    ingest.py             ← ingest_one() — Nexus PolicyManifest generation
    nexus_policy.py       ← Nexus client and manifest serialization
    report.py             ← CERTIFICATION.md generation
    classification.py     ← AUTOMATED / PARTIAL / MANUAL classification
    data/                 ← YAML action-to-control mappings
  guardian/               ← Granite Guardian Mellea hook plugin
  export/
    exporter.py           ← run_export(), Invocation, LoadedContext, TranslationPlan
    adapters/             ← per-target adapter (langgraph, mcp, claude_code)
  toolkit/
    file_utils.py         ← parse_spec_file, load_fixtures, load_skill_pipeline
    logging.py            ← configure_logger()
  enums.py                ← InferenceEngineType, SpecFileFormat, InferenceModel, ...
```
