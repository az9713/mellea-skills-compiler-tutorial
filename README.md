<h1 align="center">Mellea Skills Compiler — Tutorial & Documentation</h1>

<p align="center">
  <strong>Comprehensive documentation and onboarding guide for the Mellea Skills Compiler</strong><br>
  <em>IBM Research AI agent certification pipeline — research preview v0.1</em>
</p>

<p align="center">
  <a href="#original-repository">Original Repo</a> &middot;
  <a href="#what-is-the-mellea-skills-compiler">What is it?</a> &middot;
  <a href="#two-command-workflow">Quick Start</a> &middot;
  <a href="#documentation">Docs</a> &middot;
  <a href="#pre-compiled-examples">Examples</a> &middot;
  <a href="docs/Mellea_Skills_Compiler-tech_report.pdf">Tech Report</a> &middot;
  <a href="FAQ.md">FAQ</a>
</p>

---

## Original Repository

> **This repository is a tutorial and documentation companion to the official IBM Research project.**
>
> **Original source:** [**github.com/generative-computing/mellea-skills-compiler**](https://github.com/generative-computing/mellea-skills-compiler)
>
> This fork adds a comprehensive, structured documentation set covering every aspect of the compiler — what it is, how it works, how to extend it, and how to build on top of it. All original code, examples, and skills are preserved unchanged. The docs are the addition.

Additional reference: **[YouTube — AI Skills Security, Open AI Deployment Company & Zero Days](https://www.youtube.com/watch?v=YCWwh70FZtQ&t=16s)**

---

## What is the Mellea Skills Compiler?

The Mellea Skills Compiler is a **certification pipeline for AI agent skills**. It takes a natural-language skill specification — a `.md` file describing what an agent should do — and produces a typed, instrumented Python program with policy-driven guardrails and auditable execution traces.

It composes three IBM Research technologies:

| Component | Role |
|-----------|------|
| **[Mellea](https://github.com/generative-computing/mellea)** | Structured LLM generation — typed schemas, `@generative` extraction, repair loops, hook system |
| **[Granite Guardian 3.3 8B](https://huggingface.co/ibm-granite/granite-guardian-3.3-8b)** | Runtime risk detection — checks every LLM generation for harm, bias, jailbreak, hallucination |
| **[AI Atlas Nexus](https://github.com/IBM/ai-atlas-nexus)** | Governance knowledge graph — maps use cases to NIST AI RMF and Credo UCF requirements |

It addresses three governance gaps in current AI agent deployment:

- **Specification opacity** — contradictions in `.md` specs are silently resolved by LLMs; decomposition surfaces them as testable failures
- **Runtime unobservability** — no audit trail of what the agent generated or whether outputs were safe; Guardian hooks and JSONL audit trails fix this
- **Compliance disconnect** — no standard way to map governance requirements to runtime capabilities; the `certify` pipeline produces structured NIST AI RMF evidence

---

## Two-command workflow

```bash
# Install
git clone https://github.com/az9713/mellea-skills-compiler-tutorial
cd mellea-skills-compiler-tutorial
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# Step 1: run a pre-compiled skill immediately (no compile needed)
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city

# Step 2: compile your own skill
mellea-skills compile skills/weather/spec.md

# Step 3: certify it
mellea-skills certify skills/weather/weather_mellea
```

**Prerequisites:** Python ≥ 3.11, Claude Code (`npm install -g @anthropic-ai/claude-code`), Ollama with `granite3.3:8b` and `ibm/granite3.3-guardian:8b` pulled. See [docs/getting-started/prerequisites.md](docs/getting-started/prerequisites.md).

---

## Documentation

This repository adds 22 documentation files organized across seven sections. Start with [docs/index.md](docs/index.md) for the full navigation hub.

### Overview

| Doc | What's inside |
|-----|--------------|
| [What is the Mellea Skills Compiler?](docs/overview/what-is-mellea.md) | The compiler metaphor, three component technologies, the critical name disambiguation (Mellea library vs. Mellea Skills Compiler), and the two-step workflow |
| [Key concepts](docs/overview/key-concepts.md) | Complete glossary — every term used across all docs defined in one place: archetype, audit trail, certification, disposition, fixture, modality, PolicyManifest, stub, tier, and more |

### Getting started

| Doc | What's inside |
|-----|--------------|
| [Prerequisites](docs/getting-started/prerequisites.md) | Exact dependencies with verify commands: Python version bounds, Claude Code install, Ollama setup, Granite models, environment variables |
| [Quickstart](docs/getting-started/quickstart.md) | Run the pre-compiled weather skill in under 15 minutes — no compilation required |
| [Your first skill](docs/getting-started/your-first-skill.md) | Zero-to-hero: write a spec from scratch, compile it, inspect the output, fill stubs, run fixtures, certify |

### Concepts

> **New to Mellea? Start with [The skill compiler](docs/concepts/the-skill-compiler.md).** It traces the full compilation of the `weather` skill step by step — showing the actual intermediate JSON, generated Python, and lint results — so you can see exactly how a `.md` spec becomes a typed, governed pipeline.

| Doc | What's inside |
|-----|--------------|
| ⭐ [**The skill compiler — with running example**](docs/concepts/the-skill-compiler.md) | **Start here to understand how Mellea works.** Walks the full 10-step compilation pipeline using the `weather` skill as a concrete running example — showing the actual `classification.json`, `inventory.json`, `element_mapping.json`, `dependency_plan.json`, generated Python code, and lint results produced by the real compiler. Every step is grounded in real output you can verify in `examples/weather/weather_mellea/intermediate/`. |
| [Compiled package anatomy](docs/concepts/compiled-package-anatomy.md) | Every file in a `<name>_mellea/` package: what it does, whether it's LLM-generated or wrapper-rendered, and what's safe to modify |
| [Skill archetypes](docs/concepts/skill-archetypes.md) | The five reasoning patterns (A/B/C/D1/D2/E) with real pipeline code examples for each, and a table of how archetype affects compilation |
| [Dependency resolution](docs/concepts/dependency-resolution.md) | C1–C9 dependency categories, the eight dispositions (`bundle`, `real_impl`, `stub`, `mock`, etc.), and how stubs are generated and documented |
| [Certification pipeline](docs/concepts/certification-pipeline.md) | Nexus risk identification, Guardian hooks (audit vs. enforce mode), PolicyManifest structure, AUTOMATED/PARTIAL/MANUAL compliance classification, audit trail format |

### Guides

| Doc | What's inside |
|-----|--------------|
| [Write a skill spec](docs/guides/write-a-skill-spec.md) | How to author a `SKILL.md` that compiles cleanly — frontmatter fields, phase naming, output type specification, constraint declaration, common mistakes, and a spec template |
| [Fill stubs](docs/guides/fill-stubs.md) | Step-by-step: find stubs, read them, implement them, verify — with troubleshooting for the common failure modes |
| [Run and test fixtures](docs/guides/run-and-test-fixtures.md) | Fixture system, Guardian flags, smoke check exit codes, T1/T2/T3 tier system, adding custom fixtures, timing expectations |
| [Export to other harnesses](docs/guides/export-to-other-harnesses.md) | `mellea-skills export` for LangGraph, MCP, and Claude Code; manual integration pattern; what's stable to build against |
| [Repair a failed compile](docs/guides/repair-failed-compile.md) | `--repair-mode` — what it audits, how it resumes, which failures are auto-recoverable vs. require spec revision |

### Reference

| Doc | What's inside |
|-----|--------------|
| [CLI reference](docs/reference/cli.md) | Every `mellea-skills` subcommand (`compile`, `validate`, `run`, `ingest`, `certify`, `export`) with all options, defaults, and exit codes |
| [melleafy.json reference](docs/reference/melleafy-json.md) | Full manifest schema, every field with type and description, modality values table, stable contract for integrations |
| [Skill tiers](docs/reference/skill-tiers.md) | T1/T2/T3 definitions in depth, pre-compiled example tier table, how to determine your skill's tier after compile |

### Architecture

| Doc | What's inside |
|-----|--------------|
| [System design](docs/architecture/system-design.md) | Component boundaries (the Claude boundary, Mellea library boundary, Nexus boundary), full compilation data flow diagram, the proxy layer, source package structure |
| [Pain points and mitigations](docs/architecture/pain-points-and-mitigations.md) | Honest 360° assessment: specification opacity (strong mitigation), runtime unobservability (strong mitigation), compliance disconnect (early-stage). What remains unsolved and why. |
| [Extension guide](docs/architecture/extension-guide.md) | Every extension point: new skill specs, export targets (Adapter protocol), dialect docs, Guardian risk checks, compliance mappings. What can be built on top: skill registries, per-phase model routing, spec quality tooling, ecosystem-scale governance. Stable vs. unstable interface table. |

### Troubleshooting

| Doc | What's inside |
|-----|--------------|
| [Common issues](docs/troubleshooting/common-issues.md) | Top issues with exact fix commands: installation, compilation timeouts, lint failures, smoke check failures, Ollama connectivity, stub errors, certification gaps, export issues |

---

## Pre-compiled examples

Four reference skills run immediately from `examples/`:

| Skill | Tier | Archetype | Run command |
|-------|------|-----------|-------------|
| `weather` | T1 | D1 — Integration | `mellea-skills run examples/weather/weather_mellea --fixture rain_check_city` |
| `sentry-find-bugs` | T1/T2 | A — Analysis | `mellea-skills run examples/sentry-find-bugs/sentry_find_bugs_mellea --fixture clean_secure_parameterized` |
| `superpowers-systematic-debugging` | T1 | C — Diagnosis | `mellea-skills run examples/superpowers-systematic-debugging/superpowers_systematic_debugging_mellea --fixture architectural_issue_detected` |
| `clawdefender` | T3 | A — Adversarial | `mellea-skills run examples/clawdefender/clawdefender_mellea --fixture prompt_injection_critical` |

For a five-minute walkthrough of all four with real captured output, see [docs/README.md](docs/README.md).

---

## Skills library

16 skill specifications in `skills/` — four pre-compiled, twelve compilable locally:

```bash
mellea-skills compile skills/checklist/spec.md
mellea-skills compile skills/slack/spec.md
mellea-skills compile skills/github/spec.md
# ... and 9 more
```

See `skills/README.md` for the full tier and source attribution table.

---

## License

Apache 2.0 — see [LICENSE](LICENSE).

Original project: [IBM Research](https://www.ibm.com/research) — Elizabeth M. Daly, Dhaval Salwala, Inge Vejsbjerg, Seshu Tirupathi, Rebecka Nordenlöw, Jessica He, Kush R. Varshney, Jordan McAfoose.
