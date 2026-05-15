# Write a Skill Spec

A skill spec is a `.md` file with YAML frontmatter and a natural-language body. The body is what the compiler decomposes; the frontmatter is metadata the compiler reads first.

---

## Frontmatter

The frontmatter block must appear at the top of the file. The required fields:

```yaml
---
name: my-skill
description: "One or two sentences. What the skill does. When to use it. When NOT to use it."
---
```

Optional but useful fields:

```yaml
---
name: my-skill
description: "..."
homepage: https://example.com/api
metadata:
  openclaw:
    emoji: "🔍"
    requires:
      bins: ["curl", "jq"]
      env: ["MY_API_KEY"]
---
```

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Used to derive the package name (`<name>_mellea`) |
| `description` | Yes | Used as the use-case description for Nexus during certification |
| `homepage` | No | Linked in generated README |
| `metadata.openclaw.requires.bins` | No | Informs the T3 tier classification and SETUP.md |
| `metadata.openclaw.requires.env` | No | Informs the dependency audit (C5 credentials) |

---

## Body structure

The body is a natural-language description of what the skill does. Structure matters — it determines which archetype the compiler selects and how cleanly the decomposition succeeds.

### Name your phases explicitly

The compiler performs better when phases are named. Instead of:

> Extract the location from the query, then classify the intent, then fetch the weather.

Write:

> ## Phase 1: Extract location
> Parse the user's query and identify the geographic location.
>
> ## Phase 2: Classify intent
> Determine the type of weather information requested (current, forecast, rain check).
>
> ## Phase 3: Fetch and return
> Make a deterministic HTTP call based on the classified intent.

Named phases produce cleaner element inventories and more predictable archetype classification.

### Specify outputs with concrete types

Instead of:

> Return weather information about the requested location.

Write:

> Return a `WeatherResult` with these fields:
> - `location` (string): the resolved city name
> - `summary` (string): one-line weather description
> - `temperature_celsius` (float): current temperature
> - `forecast_days` (list[str]): next 3 days, one line each

Concrete output types compile to Pydantic models. Vague output descriptions compile to less structured schemas.

### Specify constraints as rules

The compiler converts constraints into `@requirement` validators. State them clearly:

> **Constraints:**
> - Do not invent issues not present in the diff
> - Every output field in `FindingsReport` must be populated
> - If severity is `critical`, the `mitigation` field must not be empty

Constraints embedded in prose are harder to detect than constraints listed explicitly.

### Describe tools with enough detail to implement

The key question for every tool: can the compiler implement it without asking you?

**Implementable** (the compiler can generate real code):

> ```bash
> curl "https://wttr.in/{location}?format=3"
> ```

> Call `requests.get("https://api.openweathermap.org/data/2.5/weather", params={"q": location, "appid": API_KEY})`

**Stub-generating** (abstract — the compiler generates `NotImplementedError`):

> Send a notification to the team
> Search the codebase for occurrences of a pattern
> Post the result to the internal dashboard

If a tool will be a stub, that's fine — just be aware of it. The compiler will generate the stub signature from the description.

### State when NOT to use the skill

The "When NOT to use" section compiles to an `out_of_scope_categories` constant in `config.py` and a scope-gate at the top of the pipeline. This is important for D1 integration skills.

```markdown
## When NOT to Use

- Historical weather data → use weather archives
- Aviation/marine weather → use METAR/specialized services
```

---

## Frontmatter-compatible formats

The compiler's primary dialect is `agent_skills_std` — a single `.md` file with YAML frontmatter. This format is compatible with:

- Anthropic Agent Skills standard (Claude Code skills)
- OpenClaw / NanoClaw skill libraries

For other formats (CrewAI, LangGraph, Letta), see `docs/dialects/`. The compiler detects these automatically but support is less mature.

---

## Common mistakes

**Mistake: output described in prose only**

```markdown
The skill returns weather information including the current temperature,
conditions, humidity, wind speed, and a three-day forecast.
```

The compiler extracts fields from prose but is more reliable with explicit lists or TypedDict-style definitions. Prefer structured output descriptions.

**Mistake: phases not named**

A flat spec with no phase headers defaults to one-shot pipeline shape. If your skill genuinely has sequential phases, name them.

**Mistake: constraints buried in prose**

```markdown
The skill is careful not to invent security issues and always provides
evidence for any finding it reports.
```

This is harder to extract as a validator than:

```markdown
**Constraints:**
- Do not report issues not supported by evidence in the diff
- Every reported finding must include at least one `evidence_line` reference
```

**Mistake: abstract tool names without detail**

```markdown
Use the search tool to find relevant code.
```

Add the signature, example, and the API or CLI it wraps. The more detail, the better the `real_impl` or stub docstring.

---

## Template

A starting point for a well-structured spec:

```markdown
---
name: skill-name
description: "One line: what it does. Use when: X. Not for: Y."
---

# Skill Name

One paragraph describing the skill's purpose.

## When to Use

- Use case A
- Use case B

## When NOT to Use

- Case C → use alternative
- Case D → use different service

## Inputs

- `input_field` (type): description

## Output

A `ResultModel` with:
- `field_one` (str): description
- `field_two` (list[str]): description

## Phases

### Phase 1: [Name]
What this phase does, what it reads, what it produces.

### Phase 2: [Name]
...

## Tools

### [Tool Name]
How to call it. Example:
```bash
curl "https://api.example.com/endpoint?param={value}"
```

## Constraints

- Constraint 1
- Constraint 2
```
