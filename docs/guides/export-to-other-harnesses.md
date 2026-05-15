# Export to Other Harnesses

A compiled Mellea package is a pip-installable Python library. Its `run_pipeline` function is the entry point. You can use it anywhere Python runs.

The `mellea-skills export` command (experimental) generates framework-specific adapter code for three targets. For all other harnesses, you write the wrapper.

For a comprehensive walkthrough of every export option, see [`docs/EXPORTING.md`](../EXPORTING.md).

---

## The three supported export targets

```bash
mellea-skills export --target <target> <package_path>
```

| Target | Status | Output |
|--------|--------|--------|
| `mcp` | Experimental | `server.py` (FastMCP), `mcp.json`, `pyproject.toml`, `README.md` |
| `langgraph` | Experimental | `graph.py`, `state.py`, `langgraph.json`, `pyproject.toml`, `README.md` |
| `claude-code` | Experimental | `SKILL.md`, `scripts/run.sh`, `pyproject.toml`, `README.md` |

"Experimental" means: the exporter runs end-to-end and produces deployable artifacts, but output file layouts and adapter conventions may change between releases without a deprecation period.

---

## MCP export

Produces a FastMCP server with one `@mcp.tool()` per pipeline entry:

```bash
mellea-skills export --target mcp skills/weather/weather_mellea
```

Output is written to `skills/weather/weather_mellea/weather_mellea-mcp/`. The generated `server.py` looks like:

```python
from mcp.server import FastMCP
from weather_mellea.pipeline import run_pipeline

mcp = FastMCP("weather")

@mcp.tool()
def weather(query: str) -> dict:
    """Get current weather and forecasts via wttr.in."""
    return run_pipeline(query=query)
```

The compiled Mellea package is bundled inside the export directory, making it self-contained.

**What's preserved**: typed inputs (Pydantic-validated at the tool boundary), structured outputs, env-var contract.

**What's lost**: Guardian hooks (no native MCP hook surface), token-by-token streaming to MCP clients.

---

## LangGraph export

Produces a LangGraph `StateGraph` with the compiled pipeline as a single async node:

```bash
mellea-skills export --target langgraph skills/weather/weather_mellea
```

The generated `graph.py`:

```python
from langgraph.graph import StateGraph
import asyncio
from weather_mellea.pipeline import run_pipeline

def weather_node(state):
    result = asyncio.to_thread(run_pipeline, query=state["user_query"])
    return {"weather_result": result}

graph = StateGraph(WeatherState)
graph.add_node("weather", weather_node)
```

Modality-aware: streaming modalities use `graph.astream_events()`; `conversational_session` adds `MemorySaver` checkpointing.

**What's preserved**: typed I/O via bundled Pydantic schemas, modality-aware async behavior.

**What's lost**: phase-level node decomposition (the entire Mellea pipeline runs inside one LangGraph node), Guardian hooks.

---

## Claude Code export

Produces a `SKILL.md` with a bash entry point:

```bash
mellea-skills export --target claude-code skills/weather/weather_mellea
```

The generated `scripts/run.sh` invokes the Python pipeline with arguments forwarded as JSON. Claude Code loads the `SKILL.md` and shells out to `run.sh` when the skill is invoked.

---

## Manual integration (any harness)

The simplest integration doesn't need the `export` command:

```python
from weather_mellea.pipeline import run_pipeline

result = run_pipeline(query="Will it rain in Tokyo tomorrow?")
print(result)
```

Install the compiled package:

```bash
pip install -e skills/weather/
```

This works from any Python process: a FastAPI handler, a Slack bot, an AutoGen agent, a custom CLI. The compiled package installs via its own `pyproject.toml`.

Mellea-specific behaviors (Guardian hooks, typed validators, repair loops) run inside `run_pipeline` and are not exposed at the harness boundary.

---

## What's stable to build against

These are safe to depend on across releases:

- **`run_pipeline`** is the entry point. The signature is defined in `melleafy.json:entry_signature`.
- **Pydantic schemas in `schemas.py`** define the I/O contract. Output models have non-Optional fields for required outputs.
- **`melleafy.json`** is versioned (`manifest_version ≥ 1.0.0`). Read it to introspect the package without importing it.
- **`fixtures/`** all follow the `(inputs_dict, fixture_id, description)` tuple contract.

What is not stable yet:

- `mellea-skills export` output file layouts (experimental)
- `intermediate/` JSON shapes (tied to compiler internals)
- The `Invocation` / `LoadedContext` / `TranslationPlan` Python API in `export/exporter.py` (invoke via CLI, not import)

---

## Harnesses with no native export path

The following are not current export targets: OpenClaw, NanoClaw, CrewAI, Letta, AutoGen, smolagents, OpenAI Agents SDK.

The compiler can **compile from** some of these formats (see `docs/dialects/`), but cannot export **to** them. For these harnesses, use the manual integration pattern above: `from <package_name>.pipeline import run_pipeline`.
