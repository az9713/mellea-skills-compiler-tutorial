# Skill Archetypes

The compiler classifies every skill into one of five reasoning archetypes. The archetype determines how the spec is decomposed, how many LLM sessions the pipeline uses, and which Mellea primitives are dominant.

Understanding archetypes helps you predict what a compiled skill will look like, write specs that compile well, and debug unexpected compilation results.

---

## Archetype overview

| Archetype | Name | Core pattern | Key signals in spec |
|-----------|------|-------------|---------------------|
| **A** | Analysis | Sequential phases → structured findings | "extract", "classify", "review", "verify", OWASP checklist, multi-stage |
| **B** | Generation | Intent + constraints → structured artifact | "draft", "generate", "compose", "write", schema-heavy output |
| **C** | Diagnosis | Hypothesis-driven investigation with gating | "investigate", "root cause", "reproduce", "hypothesis", conditional phases |
| **D1** | Integration | Thin wrapper around a single service/API | Single external service, minimal decision logic, intent → URL/call |
| **D2** | Orchestration | Coordinate multiple tools or subagents | "spawn", "dispatch", "CI/CD", multi-service coordination |
| **E** | Knowledge | Passive rules, conventions, best practices | Checklists, guidelines, "when X do Y", decision tables |

---

## Archetype A: Analysis

**What it does**: takes input data, runs it through sequential analytical phases, and returns a structured findings report. Each phase refines or enriches the previous one.

**Dominant Mellea primitives**: multiple `m.instruct(format=PhaseSchema)` calls in sequence, `@requirement` validators for correctness constraints.

**Example**: `sentry-find-bugs` — four-phase OWASP security review: extract attack surface → checklist per file → verify issues → structured report. Each phase is a separate `m.instruct()` call with the previous phase's output as `grounding_context`.

**What the pipeline looks like**:

```python
def run_pipeline(diff: str) -> FindingsReport:
    with start_session(BACKEND, MODEL_ID) as m:
        attack_surface = m.instruct("Extract attack surface...", format=AttackSurface, ...)

    with start_session(BACKEND, MODEL_ID) as m:
        checklist = m.instruct("Run checklist...",
            grounding_context={"attack_surface": attack_surface.model_dump_json()},
            format=SecurityChecklist, ...)

    # ... two more phases

    return findings_report
```

**Key insight**: decomposition into phases prevents the LLM from silently skipping checklist items. The `@requirement` validators enforce constraints like "no invented issues" across the full pipeline.

---

## Archetype B: Generation

**What it does**: takes a user intent and produces a structured artifact — a document, checklist, plan, or other output. Often involves a "capture intent" phase followed by a "generate artifact" phase.

**Dominant Mellea primitives**: `m.instruct(format=OutputSchema)` with rich grounding context, `@requirement` validators for quality constraints.

**Example**: `checklist` — generate a structured checklist from a topic description. Two phases: classify the domain and desired depth, then generate the checklist with the classification as grounding.

**What the pipeline looks like**:

```python
def run_pipeline(topic: str, detail_level: str = "standard") -> Checklist:
    with start_session(BACKEND, MODEL_ID) as m:
        domain = m.instruct("Classify domain and depth...", format=DomainClassification, ...)

    with start_session(BACKEND, MODEL_ID) as m:
        checklist = m.instruct("Generate checklist...",
            grounding_context={"domain": domain.model_dump_json(), "topic": topic},
            format=Checklist, ...)

    return checklist
```

**Key insight**: separating intent capture from artifact generation lets the compiler apply different constraints to each phase. The intent phase can be permissive; the generation phase can be strict.

---

## Archetype C: Diagnosis

**What it does**: investigates a problem through hypothesis formation, testing, and conditional branching. The pipeline's control flow depends on what each phase finds.

**Dominant Mellea primitives**: `m.instruct(format=...)` calls with conditional Python gates, `@requirement` validators for "no fixes without root cause" style invariants, branching based on runtime data.

**Example**: `superpowers-systematic-debugging` — four-phase debugger: analyze error → reproduce → find root cause → (if `fix_attempts_count < threshold`) generate fix plan. The fix plan phase is gated by a runtime counter, not by LLM decision.

**What the pipeline looks like**:

```python
def run_pipeline(error: str, fix_attempts_count: int = 0) -> DebuggingReport:
    with start_session(BACKEND, MODEL_ID) as m:
        error_analysis = m.instruct("Analyze error...", format=ErrorAnalysis, ...)

    with start_session(BACKEND, MODEL_ID) as m:
        reproduction = m.instruct("Reproduce...", format=ReproductionResult, ...)

    # ... root cause phase

    fix_plan = None
    architectural_issue = fix_attempts_count >= MAX_FIX_ATTEMPTS
    if not architectural_issue:
        with start_session(BACKEND, MODEL_ID) as m:
            fix_plan = m.instruct("Generate fix...", format=FixPlan, ...)

    return DebuggingReport(fix_plan=fix_plan, architectural_issue_detected=architectural_issue, ...)
```

**Key insight**: the compiler encodes spec constraints like "no fixes without root cause" as `@requirement` validators. The branching logic (`fix_attempts_count >= threshold`) is deterministic Python, not LLM choice — making behavior predictable and testable.

---

## Archetype D1: Integration

**What it does**: thin wrapper around a single external service. The LLM classifies the user's intent; Python code constructs and executes the service call.

**Dominant Mellea primitives**: one or two `m.instruct(format=...)` calls for intent extraction/classification, deterministic Python for URL construction and HTTP dispatch, `@generative` slot for entity extraction.

**Example**: `weather` — extract location, classify query type (one of nine `WeatherQueryType` values), then deterministically build a `wttr.in` URL and make the HTTP call. The LLM never touches the URL or the HTTP response.

**What the pipeline looks like**:

```python
def run_pipeline(query: str) -> str:
    with start_session(BACKEND, MODEL_ID) as m:
        location = extract_location(m, query=query)  # @generative slot

    with start_session(BACKEND, MODEL_ID) as m:
        intent = m.instruct("Classify query type...", format=WeatherIntent, ...)

    if intent.query_type == WeatherQueryType.out_of_scope:
        return "Out of scope"

    params = _ENDPOINT_PARAMS[intent.query_type]  # deterministic lookup table
    return fetch_weather(location, params)         # HTTP call in tools.py
```

**Key insight**: the compiler's decomposition of D1 skills prevents the LLM from hallucinating tool calls. It classifies intent into a finite enum; Python dispatches from that enum to the real API. This is more reliable than letting the LLM construct the URL directly.

---

## Archetype D2: Orchestration

**What it does**: coordinates multiple tools, subagents, or services across a multi-step workflow.

**Dominant Mellea primitives**: decision logic `m.instruct()` calls, multiple `tools.py` functions (often stubbed), conditional Python dispatch.

**Example**: `appdeploy` — deploy an application by coordinating build, test, push, and notify steps across different services. Each service call is a stub the user implements.

**Key insight**: for D2 skills, the compiler focuses decomposition on the decision logic (which service to call, in what order, with what error handling) rather than the services themselves (which are stubs or real implementations). The decision logic can be tested without the services being available.

---

## Archetype E: Knowledge

**What it does**: encodes domain knowledge — rules, conventions, guidelines, decision tables — and applies it to inputs.

**Pipeline shape depends on internal structure**: flat rules compile to one-shot pipelines; named stages compile to sequential pipelines.

**Example**: `coding-agent` — a set of coding conventions and when-to-apply rules. The pipeline applies the relevant conventions to a given code change and produces a structured review.

**Key insight**: knowledge skills benefit most from specification linting (an incidental benefit of decomposition). When "rules" are decomposed into typed schema fields and validators, contradictions that the LLM would silently resolve become explicit failures.

---

## How archetype affects compilation

| Decision | Archetype A | Archetype B | Archetype C | Archetype D1 | Archetype E |
|----------|------------|------------|------------|--------------|------------|
| Number of sessions | 3–5 | 2–3 | 3–6 | 1–2 | 1–3 |
| `requirements.py` present | Usually | Often | Usually | Rarely | Often |
| `tools.py` stubs | Rarely | Rarely | Often | Sometimes | Rarely |
| Pipeline shape | Sequential | Sequential | Sequential (conditional) | One-shot (decision) | One-shot or Sequential |
| Fixture C-categories | Analysis inputs | Generation prompts | Error scenarios | Query types | Rule edge cases |

---

## Choosing between archetypes when authoring

The compiler assigns archetypes based on signal detection in the spec. If your spec compiles to the wrong archetype, adjust the language:

- **Getting D1 when you want A**: add more named phases ("Phase 1: Extract...", "Phase 2: Classify...")
- **Getting E when you want B**: add output schema examples and generation constraints
- **Getting D1 when you want C**: add hypothesis and investigation language
- **Unexpected A**: remove explicit phase headers; flatten the spec structure

The `classification.json` intermediate artifact shows which signals the compiler detected and why it chose the archetype it did.
