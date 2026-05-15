# Pain Points and Mitigations

The Mellea Skills Compiler was built to address three specific governance gaps in the current state of AI agent deployment. This page describes each gap, how the compiler addresses it, how well the mitigation works today, and what remains to be improved.

---

## Pain point 1: Specification opacity

**The problem**: AI agent skills are increasingly authored as Markdown files and executed by LLMs without formal verification. When an LLM executes a spec directly, contradictions in the spec are silently resolved through the model's implicit judgement. There is no way to observe what the model decided, no way to test whether it decides consistently, and no way to prove that the behaviour matches the author's intent.

**How the compiler addresses it**: The 10-step decomposition surfaces latent contradictions. When a spec is broken into typed schema fields, `@requirement` validators, and distinct extraction slots, contradictions that a monolithic prompt resolves silently become explicit validation failures.

For example, a spec that says both "always provide a mitigation for each finding" and "only report findings with confirmed evidence" creates a tension: what if a finding is confirmed but no mitigation is known? A monolithic LLM execution silently picks one constraint to satisfy. The compiled pipeline, with separate schema fields for `findings` and `mitigations` and a `@requirement` validator that checks both, either fails validation (surfacing the contradiction) or forces the spec author to resolve it.

**How well it works**: This is the strongest and most consistent benefit of compilation. The `mapping_report.md` gives the spec author a precise record of what was extracted and how it was interpreted. The intermediate IR (`element_mapping.json`) is auditable.

**Limitations**:
- The benefit depends on the spec being well-structured. A flat, narrative spec produces a weaker decomposition than a spec with named phases, explicit output types, and stated constraints.
- The compiler does not yet have a standalone "specification linter" that detects contradictions before code generation. Contradictions are only surfaced when a validator fires at runtime.
- The repair loop (up to 2 rounds) means some lint failures that reflect spec ambiguity are patched by the LLM rather than surfaced to the author.

---

## Pain point 2: Runtime unobservability

**The problem**: agent outputs are typically unmonitored. When an agent runs, the LLM generates text that may be harmful, biased, factually wrong, or manipulated — and no system records this or alerts on it. Enterprise deployments need audit trails.

**How the compiler addresses it**: every `m.instruct()` call in a compiled pipeline is automatically instrumented with Granite Guardian hooks. Guardian checks the input and output of each generation against the risk checks identified for the skill's use case. Every check result is written to `audit/audit_trail.jsonl` with timestamp, verdict, confidence, and payload.

The audit trail is:
- **Append-only** — multiple runs accumulate evidence in the same file
- **Structured** — JSONL, queryable with standard tools
- **Comprehensive** — covers every LLM generation in the pipeline, not just the final output

**How well it works**: Guardian integration is the most technically mature component of the system. The JSONL audit trail is machine-readable and linked to the certification report. Running certify on a skill produces meaningful Guardian verdicts for risk categories relevant to the skill's domain.

**Limitations**:
- Guardian only runs during `certify` or `run` (with Guardian enabled). A compiled pipeline run without `certify` produces no audit trail.
- Guardian is currently limited to Ollama as an inference engine. Cloud-hosted Guardian is not supported.
- The audit trail is append-only but not tamper-evident. It's a log file, not a cryptographically signed audit record.
- "Audit-only" mode (the default) doesn't prevent harmful outputs — it only records them. Enforce mode does block, but raises a Python exception that the calling code must handle.

---

## Pain point 3: Compliance disconnect

**The problem**: enterprise AI governance frameworks — NIST AI RMF, EU AI Act, ISO/IEC 42001 — require documented evidence that risks were identified, managed, and monitored. No standard tooling bridges from "we ran an agent skill" to "here is the governance evidence." Teams either produce compliance documentation manually (expensive, error-prone) or skip it (risky).

**How the compiler addresses it**: the `certify` command maps a skill's use case to applicable governance requirements (via AI Atlas Nexus) and classifies each requirement as AUTOMATED, PARTIAL, or MANUAL based on what the pipeline's runtime controls can demonstrate. The output is `CERTIFICATION.md` — a structured document organized around NIST AI RMF categories with evidence chains linking requirements to audit trail data.

**How well it works**: this is the least mature component, and the one the team is most explicit about limitations for:

- The AUTOMATED / PARTIAL / MANUAL classification uses a static YAML mapping of governance action IDs to pipeline controls. It has not been validated against ground truth — treat it as indicative, not authoritative.
- The compliance classification has been tested primarily on security and utility skills. Coverage of other domains is thinner.
- Nexus maps the use-case description to risk taxonomies using `granite3.3:8b` locally. The quality of the PolicyManifest depends on how well the skill's description matches patterns Nexus was trained to recognize.

The compliance output is most useful as a starting point for internal governance reviews, not as a regulatory submission. It documents what was technically feasible to automate; it does not replace organizational governance processes.

---

## What the compiler does not address

**Spec quality before compile**: the compiler cannot detect a poorly-written spec before attempting decomposition. A spec that is vague, self-contradictory, or underspecified will produce a compiled package that technically runs but doesn't faithfully implement the author's intent. Specification linting (detecting contradictions before code generation) is on the roadmap.

**Cross-model evaluation**: the compiled pipeline's behavior depends on the runtime model (`granite3.3:8b` by default). A pipeline that works well with one model may behave differently with another. Systematic cross-model evaluation is planned but not yet available.

**Cost-benefit of decomposition**: decomposition increases LLM call count compared to monolithic execution (multiple sessions per pipeline run vs. one). For simple tasks, this adds latency and cost without correctness gains. The project is working on quantifying this tradeoff.

**Closed-loop repair**: Guardian currently flags risky outputs but cannot fix them. "Guardrails that fix" — feeding Guardian verdicts back into Mellea's repair loop — is on the roadmap.

---

## Summary

| Pain point | Mitigation strength | Key remaining gap |
|-----------|--------------------|--------------------|
| Specification opacity | Strong | No standalone spec linter; contradictions only surfaced at runtime |
| Runtime unobservability | Strong | Audit trail only for certified/guarded runs; no tamper-evidence |
| Compliance disconnect | Early-stage | Static YAML classification; not validated against ground truth |
