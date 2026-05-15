# What is the Mellea Skills Compiler?

The Mellea Skills Compiler is a certification pipeline for AI agent skills. It takes a natural-language skill specification — a `.md` file describing what an agent should do — and produces a typed, instrumented Python program with policy-driven guardrails and auditable execution traces.

> **Name disambiguation**: "Mellea" refers to two related but distinct things. **Mellea** (the library) is an IBM Research open-source library for structured LLM generation — it provides the typed schemas, extraction slots, repair loops, and hook system that compiled pipelines are built from. The **Mellea Skills Compiler** is this project — it uses the Mellea library, plus two other IBM Research technologies, to compile and certify agent skill specifications. When these docs say "compile a skill," they mean the Mellea Skills Compiler. When they say "a Mellea pipeline," they mean code that uses the Mellea library at runtime.

---

## Why a "compiler"?

The compiler metaphor is deliberate. Traditional compilers translate high-level source code into a lower-level representation that can be executed, verified, and optimized. The Mellea Skills Compiler does the same for agent skills:

| Compiler concept | Mellea Skills Compiler equivalent |
|-----------------|----------------------------------|
| Source language | Natural-language `.md` skill specification |
| Compilation | 10-step LLM-driven decomposition (`/mellea-fy`) |
| Intermediate representation | JSON artifacts (`classification.json`, `inventory.json`, `element_mapping.json`, etc.) |
| Target language | Typed Python package using the Mellea library |
| Linker / dependency resolution | Step 2.5 dependency audit (C1–C9 categories, dispositions) |
| Linting / static analysis | Step 7: 14 formal structural lints |
| Test suite | 5–8 auto-generated fixtures per skill |
| Runtime verification | Granite Guardian hooks on every LLM call |
| Compliance report | NIST AI RMF / Credo UCF certification report |

A spec authored as Markdown is human-readable and fast to write, but it cannot be formally verified, monitored, or audited. The compilation step makes those things possible.

---

## The three component technologies

The Mellea Skills Compiler composes three IBM Research technologies:

### Mellea

[Mellea](https://github.com/generative-computing/mellea) (Apache 2.0) is the structured generative programming library that compiled pipelines run on. It provides:

- **`start_session` / `m.instruct`** — typed LLM extraction with Pydantic schemas as format constraints
- **`@generative`** — decorator for extraction slot functions
- **Repair loops** — `RepairTemplateStrategy` retries failed validations automatically
- **Hook system** — `PluginViolationError`, pre/post-generation hooks — the integration surface for Guardian

Every compiled skill's `pipeline.py` uses Mellea. Without Mellea, a compiled skill does not run.

### Granite Guardian

[Granite Guardian 3.3 8B](https://huggingface.co/ibm-granite/granite-guardian-3.3-8b) (Apache 2.0) is an enterprise-grade risk detection model. The compiler integrates it as a Mellea hook plugin that checks every `m.instruct()` call for risks: harm, social bias, jailbreaking, hallucination, and others.

Guardian operates in two modes:
- **AUDIT** (default) — observe and log, never interrupt
- **ENFORCE** — raise `PluginViolationError` and halt the pipeline on detection

### AI Atlas Nexus

[AI Atlas Nexus](https://github.com/IBM/ai-atlas-nexus) (Apache 2.0) is a governance knowledge graph that consolidates AI risk and capability taxonomies. During certification, Nexus accepts a skill's use-case description and returns a `PolicyManifest` — a document mapping the skill to:
- Guardian risk checks (with system prompts for each risk type)
- NIST AI RMF governance actions applicable to this skill
- Credo UCF mitigation controls

The `PolicyManifest` drives all downstream instrumentation: which Guardian checks to configure, and which controls get classified as AUTOMATED / PARTIAL / MANUAL after execution.

---

## The two-step user workflow

```
SKILL.md
   │
   ▼
mellea-skills compile <spec>
   │
   │  LLM-driven decomposition (Claude)
   │  → Intermediate JSON IR (classification, inventory, mapping, dependencies)
   │  → Typed Python package (<name>_mellea/)
   │  → Structural lints (14 checks)
   │  → Fixture smoke check
   │
   ▼
<name>_mellea/   ← runnable Mellea pipeline
   │
   ▼
mellea-skills certify <name>_mellea/
   │
   │  AI Atlas Nexus → PolicyManifest (risks + governance actions)
   │  Guardian hooks configured from manifest
   │  Fixtures executed + every m.instruct() audited
   │  Compliance classification (AUTOMATED / PARTIAL / MANUAL)
   │
   ▼
audit/
  ├── POLICY.md             ← human-readable policy document
  ├── CERTIFICATION.md      ← certification report with evidence chains
  ├── audit_trail.jsonl     ← every Guardian verdict, timestamped
  └── pipeline_report.json  ← fixture execution results
```

The compile step and the certify step are independent. You can compile and run a skill without ever certifying it. Certification makes sense when you need governance evidence.

---

## What this is not

- **Not a general-purpose agent framework** — the compiler produces Mellea pipelines, not AutoGen graphs or CrewAI crews. It has export paths to LangGraph, MCP, and Claude Code (experimental), but those are adapters, not native targets.
- **Not production-ready** — this is a research preview (v0.1). APIs, CLI, and artifact formats may change.
- **Not a compliance oracle** — the certification report documents what was checked and found. It does not assert that the agent is "safe." The AUTOMATED / PARTIAL / MANUAL classification is indicative, not authoritative.
- **Not model-agnostic at compile time** — the compilation step uses Claude (via Claude Code). The compiled skill's runtime model is configurable (default: `granite3.3:8b` via Ollama).

---

## How it all fits together

A typical developer session looks like this:

1. **Author**: write `skills/myskill/spec.md` describing what the agent should do.
2. **Compile**: `mellea-skills compile skills/myskill/spec.md` — Claude decomposes the spec into a Python package under `skills/myskill/myskill_mellea/`.
3. **Inspect**: read the generated `pipeline.py`, `schemas.py`, and `mapping_report.md`. Fill any `NotImplementedError` stubs.
4. **Run**: `mellea-skills run skills/myskill/myskill_mellea --fixture <name>` — execute against a sample input.
5. **Certify**: `mellea-skills certify skills/myskill/myskill_mellea` — govern the skill and produce compliance artifacts.
6. **Export** (optional): `mellea-skills export --target mcp skills/myskill/myskill_mellea` — package for another harness.

See [Your first skill](../getting-started/your-first-skill.md) for the end-to-end walkthrough.
