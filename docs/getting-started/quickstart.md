# Quickstart

Compile and run the `weather` skill in under 15 minutes. No stubs, no external API keys, no credentials — just Ollama and a Claude API key.

This uses the pre-compiled example from the repository. If you want to compile the skill yourself, see step 5.

---

## Before you start

Confirm the prerequisites:

```bash
python3 --version          # ≥ 3.11, < 3.14.4
claude --version           # Claude Code installed
ollama list                # granite3.3:8b is listed
echo $OLLAMA_API_URL       # http://localhost:11434 (or similar)
```

If anything is missing, see [Prerequisites](prerequisites.md).

---

## Step 1: Install the compiler

```bash
git clone https://github.com/generative-computing/mellea-skills-compiler
cd mellea-skills-compiler

python3 -m venv .venv
source .venv/bin/activate

pip install -e .
```

---

## Step 2: Run the pre-compiled weather skill

The repository ships pre-compiled examples. Run one immediately:

```bash
mellea-skills run examples/weather/weather_mellea --fixture rain_check_city
```

Expected output (exact weather data varies; format is stable):

```
Tokyo: ⛅️  Partly cloudy +14°C
```

This runs in 30–90 seconds on a modern laptop with `granite3.3:8b` locally.

What happened: the pipeline ran two LLM sessions (location extraction, then intent classification into a `WeatherQueryType` enum), then made a deterministic HTTP call to `wttr.in`. The LLM classified intent; Python constructed the URL.

---

## Step 3: Try a different fixture

The weather skill ships seven fixtures covering different query types:

```bash
mellea-skills run examples/weather/weather_mellea --fixture forecast_week
mellea-skills run examples/weather/weather_mellea --fixture json_output
```

List all available fixtures:

```bash
ls examples/weather/weather_mellea/fixtures/
# clean_text.py  forecast_week.py  json_output.py  rain_check_city.py  ...
```

---

## Step 4: Certify the skill

Run the full governance pipeline:

```bash
mellea-skills certify examples/weather/weather_mellea
```

This runs in 2–5 minutes. When complete, inspect the artifacts:

```bash
ls examples/weather/audit/
# CERTIFICATION.md  POLICY.md  audit_trail.jsonl  pipeline_report.json  policy_manifest.json
```

Open `CERTIFICATION.md` to see the Guardian verdict summary and compliance classification.

---

## Step 5 (optional): Compile it yourself

Compile the weather spec from scratch:

```bash
mellea-skills compile skills/weather/spec.md
```

This takes 5–15 minutes (one Claude session for spec decomposition). When complete, a new `skills/weather/weather_mellea/` directory contains the compiled package. Run it the same way:

```bash
mellea-skills run skills/weather/weather_mellea --fixture rain_check_city
```

---

## What's next

- **Try another pre-compiled example** — see `examples/` for `sentry-find-bugs`, `superpowers-systematic-debugging`, and `clawdefender`. The four-skill tutorial at [`docs/README.md`](../README.md) walks each one.
- **Compile your own skill** — read [Write a skill spec](../guides/write-a-skill-spec.md), then [Your first skill](your-first-skill.md) for the full zero-to-hero walkthrough.
- **Understand what was generated** — read [Compiled package anatomy](../concepts/compiled-package-anatomy.md) to understand every file in `weather_mellea/`.
