# Mellea Skills Compiler — Documentation

A certification pipeline that turns a natural-language agent skill specification (`.md` file) into a typed, instrumented Python program with policy-driven guardrails and auditable execution traces.

Produced by IBM Research. Research preview — v0.1.

---

## Resources

| Resource | Link |
|----------|------|
| **Original repository** | [generative-computing/mellea-skills-compiler](https://github.com/generative-computing/mellea-skills-compiler) |
| **Tutorial repository** | [az9713/mellea-skills-compiler-tutorial](https://github.com/az9713/mellea-skills-compiler-tutorial) |
| **YouTube — AI Skills Security, Open AI Deployment Company & Zero Days** | [Watch on YouTube](https://www.youtube.com/watch?v=YCWwh70FZtQ&t=16s) |
| **Technical report (PDF)** | [docs/Mellea_Skills_Compiler-tech_report.pdf](Mellea_Skills_Compiler-tech_report.pdf) |
| **IBM Research** | [ibm.com/research](https://www.ibm.com/research) |

---

## Documentation

| Doc | Section | What's inside |
|-----|---------|--------------|
| [What is it?](overview/what-is-mellea.md) | Overview | The compiler metaphor, three component technologies, how they fit together, and the critical name disambiguation (Mellea library vs. Mellea Skills Compiler) |
| [Key concepts](overview/key-concepts.md) | Overview | Complete glossary — every term used across these docs defined in one place |
| [Prerequisites](getting-started/prerequisites.md) | Getting started | Exact dependencies with verify commands: Python, Claude Code, Ollama, Granite models |
| [Quickstart](getting-started/quickstart.md) | Getting started | Compile and run the weather skill in under 15 minutes using the pre-compiled example |
| [Your first skill](getting-started/your-first-skill.md) | Getting started | Zero-to-hero: write a spec from scratch, compile it, run fixtures, certify |
| [The skill compiler](concepts/the-skill-compiler.md) | Concepts | How the 10-step compilation works: classification → inventory → mapping → dependency audit → code generation → lints → smoke check |
| [Compiled package anatomy](concepts/compiled-package-anatomy.md) | Concepts | Every file the compiler produces, what it does, and what's safe to modify |
| [Skill archetypes](concepts/skill-archetypes.md) | Concepts | The five reasoning patterns (A/B/C/D1/D2/E) with pipeline code examples for each |
| [Dependency resolution](concepts/dependency-resolution.md) | Concepts | C1–C9 dependency categories, the eight dispositions, and how stubs are generated |
| [Certification pipeline](concepts/certification-pipeline.md) | Concepts | Nexus risk identification, Guardian hooks, PolicyManifest, and compliance classification (AUTOMATED/PARTIAL/MANUAL) |
| [Write a skill spec](guides/write-a-skill-spec.md) | Guides | How to author a `SKILL.md` that compiles cleanly — structure, frontmatter, common mistakes |
| [Fill stubs](guides/fill-stubs.md) | Guides | Converting `NotImplementedError` stubs into real implementations |
| [Run and test fixtures](guides/run-and-test-fixtures.md) | Guides | Fixture system, smoke checks, Guardian flags, and the T1/T2/T3 tier system |
| [Export to other harnesses](guides/export-to-other-harnesses.md) | Guides | LangGraph, MCP, and Claude Code export — and how to hand-wrap for any other harness |
| [Repair a failed compile](guides/repair-failed-compile.md) | Guides | Diagnosing and resuming a broken compile with `--repair-mode` |
| [CLI reference](reference/cli.md) | Reference | Every `mellea-skills` subcommand (`compile`, `validate`, `run`, `ingest`, `certify`, `export`) with all options and exit codes |
| [melleafy.json reference](reference/melleafy-json.md) | Reference | Manifest schema, every field, modality values, and the stable contract for integrations |
| [Skill tiers](reference/skill-tiers.md) | Reference | T1/T2/T3 definitions, the pre-compiled example tier table, and how to determine your skill's tier |
| [System design](architecture/system-design.md) | Architecture | Component boundaries, data flow through compilation, the Claude boundary, and source package layout |
| [Pain points and mitigations](architecture/pain-points-and-mitigations.md) | Architecture | Honest 360° assessment: what problems exist, how well the compiler solves each one, and what remains to be improved |
| [Extension guide](architecture/extension-guide.md) | Architecture | Every extension point, what can be built on top (skill registries, per-phase routing, spec quality tooling), and the stable vs. unstable interface table |
| [Troubleshooting](troubleshooting/common-issues.md) | Troubleshooting | Top issues with exact fix commands: installation, compilation, runtime, certification, and export |

---

## Pre-compiled examples

Four reference skills ship ready to run in `examples/`:

| Skill | Tier | Archetype | Description |
|-------|------|-----------|-------------|
| [`weather`](../examples/weather/) | T1 | D1 — Integration | Intent classification → deterministic `wttr.in` HTTP call. Runs in 30–90s. |
| [`sentry-find-bugs`](../examples/sentry-find-bugs/) | T1/T2 | A — Analysis | Four-phase OWASP security review of a git diff. 1–4 min. |
| [`superpowers-systematic-debugging`](../examples/superpowers-systematic-debugging/) | T1 | C — Diagnosis | Hypothesis-driven debugger with architectural threshold gating. 3–5 min. |
| [`clawdefender`](../examples/clawdefender/) | T3 | A — Adversarial | Prompt injection / SSRF / command injection / credential exfiltration classifier. |

Run the weather skill immediately:

```bash
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city
```

For a five-minute walkthrough of all four skills with real captured output, see [`docs/README.md`](README.md).

---

## Two-command workflow

```bash
# Step 1: compile a spec into a typed pipeline
mellea-skills compile skills/weather/spec.md

# Step 2: govern it — risk identification, Guardian hooks, compliance report
mellea-skills certify skills/weather/weather_mellea
```

See [Quickstart](getting-started/quickstart.md) to run this end-to-end in under 15 minutes.
