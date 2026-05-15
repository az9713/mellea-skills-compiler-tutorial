# Certification Pipeline

`mellea-skills certify` runs end-to-end governance on a compiled skill. It produces evidence that the skill's runtime behavior was monitored and that applicable governance requirements are addressed.

The certification report does not certify that the agent is "safe." It documents what was checked, what was found, and what remains uncovered.

---

## What certification does

In one command:

```bash
mellea-skills certify skills/weather/weather_mellea
```

Five things happen, in order:

1. **Risk identification** — AI Atlas Nexus reads the skill's spec and use-case description and returns a `PolicyManifest`: applicable Guardian risk checks and NIST AI RMF / Credo UCF governance actions
2. **Hook configuration** — Guardian hooks are configured from the manifest and registered with Mellea's plugin system
3. **Fixture execution** — all fixtures (or one if `--fixture` is specified) run through the compiled pipeline; every `m.instruct()` call is intercepted by Guardian
4. **Compliance classification** — each governance action is classified as AUTOMATED, PARTIAL, or MANUAL based on what was detected
5. **Report generation** — `CERTIFICATION.md`, `POLICY.md`, `audit_trail.jsonl`, and `pipeline_report.json` are written to `audit/`

---

## The three components

### AI Atlas Nexus (risk identification)

Nexus is a governance knowledge graph. It maps a skill's use case to applicable risks across three frameworks:

| Framework | What it provides |
|-----------|-----------------|
| Granite Guardian taxonomy | Which specific risk checks to enable (e.g., harm, social bias, jailbreak) |
| NIST AI RMF | Governance actions — MAP, MEASURE, MANAGE categories |
| Credo UCF | Mitigation controls — concrete technical and organizational steps |

The `PolicyManifest` Nexus produces is a JSON document containing:
- Guardian risk checks with system prompts (the exact prompt used to invoke Guardian for this skill)
- Governance actions from NIST and Credo mapped to this skill's use case

Nexus runs via Ollama locally. The default model is `granite3.3:8b`. The manifest is saved to `audit/policy_manifest.json` and rendered as `audit/POLICY.md` for human review.

### Granite Guardian (runtime risk detection)

Guardian is a risk detection model that runs alongside the pipeline, checking every LLM generation. It is integrated as a Mellea hook plugin — the Mellea library calls it automatically before and after each `m.instruct()` call.

Guardian checks for:
- **Harm** (physical, psychological, social harms)
- **Social bias** (demographic group stereotyping)
- **Jailbreak** (attempts to override system instructions)
- **Hallucination** (factually unsupported claims)
- **Groundedness** (output not supported by grounding context)
- **Context relevance** (response to a different question than asked)
- Custom risk criteria specific to the skill's domain

Each check produces a verdict: SAFE or UNSAFE, with a confidence score and explanation. Verdicts are written to `audit/audit_trail.jsonl`.

**Audit mode** (default): Guardian observes and logs. The pipeline continues regardless of Guardian findings. Use this to understand a skill's risk profile without interrupting execution.

**Enforce mode** (`--enforce`): Guardian can halt the pipeline. If any check returns UNSAFE, Mellea raises `PluginViolationError` and execution stops. Use this for high-stakes production scenarios.

```bash
mellea-skills certify skills/myskill/myskill_mellea --enforce
```

### Compliance classification

After fixture execution, each governance action in the `PolicyManifest` is classified:

| Classification | Meaning |
|---------------|---------|
| **AUTOMATED** | ≥2 implemented pipeline controls cover this requirement. Runtime evidence exists. |
| **PARTIAL** | 1 implemented control, but organizational process is also required. |
| **MANUAL** | Requires organizational process beyond what technical instrumentation can provide. |

The classification is based on a static YAML mapping under `src/mellea_skills_compiler/certification/data/`. It is indicative, not authoritative — ground-truth validation is planned for future work.

---

## The audit trail

`audit/audit_trail.jsonl` — one JSON object per line, one per Guardian check. Each entry includes:

```json
{
  "timestamp": "2026-05-15T10:23:41Z",
  "hook_type": "post_generation",
  "policy_id": "guardian:harm",
  "verdict": "SAFE",
  "confidence": 0.97,
  "payload": {
    "input_text": "What is the weather in Tokyo?",
    "output_text": "Tokyo: ⛅️  Partly cloudy +14°C",
    "latency_ms": 312
  }
}
```

The audit trail is the foundation of all evidence chains in `CERTIFICATION.md`. It is append-only — multiple certification runs on the same skill accumulate evidence in the same file.

---

## The certification report (CERTIFICATION.md)

The report combines:

**Guardian verdict summary** — for each risk check: how many invocations were checked, how many returned UNSAFE, and the flag rate. A flag rate of 0% for harm on a weather skill is expected and should be documented.

**Audit trail statistics** — total events, hook type distribution, latency percentiles. Useful for understanding pipeline behavior at scale.

**Per-requirement evidence chains** — for each governance action from NIST / Credo: what controls address it, what evidence was collected, and the AUTOMATED / PARTIAL / MANUAL classification with rationale.

**Known limitations** — sections that couldn't be verified, requirements that are inherently manual, coverage gaps.

---

## Running only part of certification

Ingest only (risk identification without fixture execution):

```bash
mellea-skills ingest skills/myskill/spec.md
```

Produces `policy_manifest.json` and `POLICY.md` in `audit/`. Use this to review the policy before running.

Run only (execute fixtures without certification):

```bash
mellea-skills run skills/myskill/myskill_mellea --fixture rain_check_city
```

No policy, no audit trail — just fixture execution.

Certify a specific fixture:

```bash
mellea-skills certify skills/myskill/myskill_mellea --fixture rain_check_city
```

---

## Using different models for certification

The default models are set in `runtime_defaults.json` (Ollama backend, `granite3.3:8b` for risk identification, `ibm/granite3.3-guardian:8b` for Guardian). Override per run:

```bash
mellea-skills certify skills/myskill/myskill_mellea \
  --model granite3.3:8b \
  --guardian-model ibm/granite3.3-guardian:8b \
  --inference-engine ollama
```

The `--inference-engine` option is currently limited to `ollama`.

---

## What certification evidence is useful for

- **Internal governance reviews** — the CERTIFICATION.md is structured around NIST AI RMF categories. Share it with risk/compliance teams.
- **Incident investigation** — the `audit_trail.jsonl` records every generation with Guardian verdicts. Search it by policy ID or verdict to understand what happened.
- **Skill comparison** — running certification across multiple compiled versions of the same spec lets you compare risk profiles across model updates or spec changes.
- **Regulatory documentation** — the EU AI Act and similar frameworks require documented evidence of risk management. The certification artifacts are designed to map to this requirement.
