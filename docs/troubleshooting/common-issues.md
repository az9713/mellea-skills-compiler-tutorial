# Common Issues

Top issues and their exact fixes. Check the symptom, find your issue, apply the fix.

---

## Installation and setup

### `mellea-skills: command not found`

The package is not installed or the virtual environment is not activated.

```bash
source .venv/bin/activate
pip install -e .
mellea-skills --help
```

### `claude: command not found` during compile

Claude Code is not installed.

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

### `No claude models available with your API key`

The Anthropic API key is not set or is invalid.

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# or for LiteLLM gateway:
export ANTHROPIC_BASE_URL="https://your-gateway"
export ANTHROPIC_AUTH_TOKEN="your-token"
```

### `Invalid Claude model provided`

The `--model` you specified is not available under your API key. Run without `--model` to let the compiler auto-select:

```bash
mellea-skills compile skills/weather/spec.md
# (removes --model)
```

Or list your available models:

```python
from anthropic import Anthropic
print([m.id for m in Anthropic().models.list()])
```

---

## Compilation

### Compile hangs or times out

The default timeout is 4500 seconds (75 minutes). Large or complex specs can exceed this.

```bash
mellea-skills compile skills/myskill/spec.md --timeout 9000
```

If the compile was already running and timed out, use repair mode instead of restarting:

```bash
mellea-skills compile skills/myskill/spec.md --repair-mode
```

### Exit code 11 — lint failed

Step 7 structural lints found issues the repair loop couldn't fix. Check the lint report:

```bash
cat skills/myskill/myskill_mellea/intermediate/step_7_report.json | python -m json.tool
```

Then use repair mode with a more capable model:

```bash
mellea-skills compile skills/myskill/spec.md --repair-mode --model claude-opus-4-5
```

If the lint report mentions `session-boundary` or `category-specific`: the issue requires spec revision. Read the error message for what to change in `spec.md`.

### Exit code 12 — smoke check failed

A fixture raised during the post-compile smoke check. Run the fixture directly to see the error:

```bash
mellea-skills run skills/myskill/myskill_mellea --fixture <fixture_id>
```

If the error is `NotImplementedError`: a stub was hit. See [Fill stubs](../guides/fill-stubs.md).

If the error is `ValidationError` from Pydantic: the model didn't follow the schema. Increase `LOOP_BUDGET` in `config.py` and re-run `mellea-skills validate`.

### No `*_mellea/` directory after compile

The Claude session failed before Step 3 (skeleton generation). Check the compile output for errors. Common causes:

- Claude Code not authenticated — set `ANTHROPIC_API_KEY`
- Spec file not found — check the path
- Spec file isn't `.md` extension — only `.md` specs are supported

```bash
# Verify the spec path is correct:
ls skills/myskill/spec.md
```

---

## Running skills

### `ConnectionError` when running a fixture

Ollama is unreachable. Fix:

```bash
# Check Ollama is running:
ollama list

# Check the URL is set correctly:
echo $OLLAMA_API_URL       # should be http://localhost:11434 or similar

# Start Ollama if not running:
ollama serve
```

### `model not found` from Ollama

The required model hasn't been pulled:

```bash
ollama pull granite3.3:8b
ollama pull ibm/granite3.3-guardian:8b
```

### `NotImplementedError` at runtime

A stub was called. Find the stub:

```bash
grep -n "raise NotImplementedError" <package_dir>/constrained_slots.py
```

See [Fill stubs](../guides/fill-stubs.md) for the implementation walkthrough.

### `ValidationError` from Pydantic

The LLM produced output that doesn't match the expected schema. Try:

1. Increase `LOOP_BUDGET` in `config.py` (allows more repair attempts)
2. Try a more capable model by changing `MODEL_ID` in `config.py`
3. Inspect the schema in `schemas.py` — if a field is too constrained, relax it

### `PluginViolationError` in enforce mode

Guardian detected a risk and blocked execution. This is expected behavior in `--enforce` mode. To see what was generated:

```bash
mellea-skills run <package_dir> --fixture <name>
# (without --enforce — runs in audit mode, logs but doesn't block)
```

Check `audit/audit_trail.jsonl` for the verdict details.

---

## Certification

### `No policy manifest found` during certify

Either `ingest` hasn't been run yet, or the manifest file is missing. Certify runs `ingest` automatically — if it's failing, check Ollama connectivity and that `granite3.3:8b` is available.

### Nexus returns an empty or partial PolicyManifest

`granite3.3:8b` may have produced a malformed response. Try:

```bash
# Use a different model for Nexus:
mellea-skills certify <package_dir> --model granite3.3:8b --inference-engine ollama
```

Or rerun:

```bash
mellea-skills ingest skills/myskill/spec.md
```

### `CERTIFICATION.md` shows all MANUAL requirements

This is expected for skills in domains not yet covered by the static YAML action-to-control mapping. The classification is indicative — "MANUAL" means no automated technical control was mapped, not that the requirement is unaddressable.

---

## Export

### `mellea-skills export` produces an error about `manifest_version`

The compiled package's `melleafy.json` has `manifest_version` < `1.0.0`. This usually means the package was compiled with a very old version of the compiler. Recompile the skill.

### Output directory already exists

Use `--force` to overwrite:

```bash
mellea-skills export --target mcp <package_dir> --force
```
