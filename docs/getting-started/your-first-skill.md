# Your First Skill

## What you'll build

A skill that takes a natural-language query and returns a structured response. By the end, you'll have:

- A `SKILL.md` specification you wrote
- A typed Python package compiled from it
- Fixtures that verify it works
- A certification report you can read

This walkthrough takes 30–60 minutes the first time (most of that is Claude compiling).

---

## What is a skill spec?

A skill spec is a Markdown file with YAML frontmatter. It describes what an agent should do in natural language. The compiler reads this and produces a typed Python pipeline.

Think of the spec as the "source code" and the compiled package as the "executable." The spec is easy to write and read; the compiled package is verifiable and governed.

---

## Step 1: Create a skill directory

```bash
mkdir -p skills/my-first-skill
```

---

## Step 2: Write the spec

Create `skills/my-first-skill/spec.md`:

```markdown
---
name: sentence-analyzer
description: "Analyze a sentence and classify its sentiment, extract key entities, and summarize its main claim. Use when: user wants structured analysis of a text snippet. NOT for: long documents, audio, or multilingual text."
---

# Sentence Analyzer Skill

Analyze a single sentence and produce a structured report.

## What this skill produces

Given an input sentence, return:
- **Sentiment**: positive, negative, or neutral
- **Confidence**: 0.0–1.0 score for the sentiment classification
- **Key entities**: named persons, organizations, locations, or concepts
- **Main claim**: a one-sentence restatement of what the sentence asserts

## Constraints

- Handle sentences up to 500 characters
- If the input is empty or nonsensical, return sentiment=neutral with confidence=0.0 and empty entities
- Do not invent entities that are not present in the sentence
```

This is a minimal but complete spec. It names the skill, describes inputs and outputs, and states constraints.

---

## Step 3: Compile the spec

```bash
mellea-skills compile skills/my-first-skill/spec.md
```

This takes 5–15 minutes. Claude reads the spec and generates:

- `skills/my-first-skill/sentence_analyzer_mellea/pipeline.py`
- `skills/my-first-skill/sentence_analyzer_mellea/schemas.py`
- `skills/my-first-skill/sentence_analyzer_mellea/config.py`
- `skills/my-first-skill/sentence_analyzer_mellea/slots.py`
- `skills/my-first-skill/sentence_analyzer_mellea/fixtures/`
- ... and more

You'll see Claude's output stream to the terminal as it works through each step.

---

## Step 4: Inspect what was generated

Open `pipeline.py` and find `run_pipeline`:

```python
def run_pipeline(sentence: str) -> SentenceAnalysis:
    ...
```

The function signature matches what your spec described. The return type (`SentenceAnalysis`) is defined in `schemas.py` as a Pydantic model.

Open `schemas.py` to see the output type:

```python
class SentenceAnalysis(BaseModel):
    sentiment: Literal["positive", "negative", "neutral"]
    confidence: float
    key_entities: list[str]
    main_claim: str
```

Open `mapping_report.md` for a human-readable table of which spec elements became which code constructs.

---

## Step 5: Look for stubs

Some compiled skills contain `NotImplementedError` stubs — functions the compiler couldn't auto-implement:

```bash
grep -r "raise NotImplementedError" skills/my-first-skill/sentence_analyzer_mellea/
```

For a simple analytical skill like this one, there are typically no stubs. If there are stubs, see [Fill stubs](../guides/fill-stubs.md).

---

## Step 6: Run a fixture

```bash
mellea-skills run skills/my-first-skill/sentence_analyzer_mellea --fixture <fixture_name>
```

List available fixtures:

```bash
ls skills/my-first-skill/sentence_analyzer_mellea/fixtures/
```

Pick one and run it. A successful run prints the `SentenceAnalysis` output.

If the run fails with `ConnectionError`, Ollama is unreachable — check `OLLAMA_API_URL` and `ollama list`.

---

## Step 7: Certify the skill

```bash
mellea-skills certify skills/my-first-skill/sentence_analyzer_mellea
```

When complete:

```bash
ls skills/my-first-skill/audit/
# CERTIFICATION.md  POLICY.md  audit_trail.jsonl  pipeline_report.json  policy_manifest.json
```

Read `CERTIFICATION.md`. It will show:
- Which Guardian risk checks were run against your skill
- Which checks triggered (if any)
- Which NIST AI RMF requirements are AUTOMATED, PARTIAL, or MANUAL

---

## What just happened

Your `spec.md` went through 10 compiler steps:

1. **Classify** — the compiler identified this as a Type A (Analysis), Sequential, P0 (no tools) skill
2. **Inventory** — it extracted elements: the output fields (sentiment, confidence, entities, main_claim) and the constraints
3. **Map** — each element was routed to a Mellea primitive (schema fields, `m.instruct` calls, requirement validators)
4. **Dependency audit** — no external dependencies found; nothing to stub
5. **Generate** — Python files were written
6. **Fixtures** — 5–8 sample inputs were generated covering normal, edge, and error cases
7. **Validate** — 14 structural lints were run
8. **Smoke check** — one fixture was executed to verify the package runs

For the full detail on each step, see [The skill compiler](../concepts/the-skill-compiler.md).

---

## Common issues at first compile

**Compile times out (>75 minutes)**

Rare with Sonnet. If it happens, use repair mode:

```bash
mellea-skills compile skills/my-first-skill/spec.md --repair-mode
```

**Smoke check fails with NotImplementedError**

The fixture hit a stub. See [Fill stubs](../guides/fill-stubs.md).

**`granite3.3:8b` not available**

The post-compile smoke check couldn't reach Ollama. The compile still succeeded — exit code 0, smoke check marked as skipped. Run the fixture manually when Ollama is available:

```bash
mellea-skills run skills/my-first-skill/sentence_analyzer_mellea --fixture <name>
```

---

## Next steps

- **Try a more complex spec** — add tools, conditions, or multi-phase reasoning. See [Write a skill spec](../guides/write-a-skill-spec.md).
- **Export to another harness** — `mellea-skills export --target mcp skills/my-first-skill/sentence_analyzer_mellea`. See [Export to other harnesses](../guides/export-to-other-harnesses.md).
- **Read the four pre-compiled examples** — [`docs/README.md`](../README.md) covers weather, sentry-find-bugs, superpowers-systematic-debugging, and clawdefender with real captured output.
