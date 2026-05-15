# Dependency Resolution

The compiler can't implement every tool a skill spec references. An API key you don't have, a service not installed on your machine, an abstract "send a message" function with no implementation details — these can't be auto-implemented. The dependency audit (Step 2.5) handles this systematically.

---

## The C1–C9 category system

Every external dependency in a spec is assigned one of nine categories during inventory (Step 1b). The category drives the default disposition in auto mode.

| Category | What it covers | Auto-mode default |
|----------|---------------|-------------------|
| **C1** | Configuration constants — persona text, model ID, loop budgets, personas | `bundle` (embed in `config.py`) |
| **C2** | Schema fields — output types, intermediate types | Mapped to Pydantic fields in `schemas.py` |
| **C3** | Domain knowledge — rules, decision tables, reference data | `bundle` (embed as constants) or `load_from_disk` |
| **C4** | Templates — prompt templates, URL templates, format strings | `bundle` (embed in `config.py`) |
| **C5** | Credentials and secrets — API keys, auth tokens | `external_input` (environment variable) |
| **C6** | Tool calls — HTTP APIs, SDKs, CLIs, abstract tool references | Varies by subtype (see below) |
| **C7** | Memory and session state | `delegate_to_runtime` |
| **C8** | Scope and routing logic | Encoded as deterministic Python in `pipeline.py` |
| **C9** | Interaction modality — streaming, session carry-forward | `delegate_to_runtime` |

---

## C6: The category that matters most

C6 (tool calls) is where stubs come from. The compiler subdivides C6 tools by implementation type:

| C6 subtype | What it is | Auto-mode default |
|------------|-----------|-------------------|
| `http` | HTTP API call to a known endpoint | `real_impl` — generates actual HTTP code |
| `graphql` | GraphQL query | `real_impl` — generates actual query code |
| `mcp` | Model Context Protocol tool | `real_impl` (if schema available) or `stub` |
| `sdk` | Pip-installable SDK | `real_impl` — generates SDK import and call |
| `abstract` | Tool reference without implementation details ("send a message") | `stub` — generates `NotImplementedError` |

The critical distinction: tools with known endpoints and schemas get real implementations; tools with only a description get stubs. The compiler can implement `curl "wttr.in/London?format=3"` from the spec. It cannot implement "send a message to the user" without knowing the messaging platform, credentials, and API.

---

## The eight dispositions

| Disposition | Generated code | When to use |
|-------------|---------------|-------------|
| `bundle` | Constant in `config.py` | The value is known at compile time and doesn't change per invocation |
| `real_impl` | Full implementation in `tools.py` | The API, endpoint, and usage are documented in the spec |
| `stub` | `NotImplementedError` in `constrained_slots.py` | Tool needs real integration you supply |
| `mock` | Test double in `fixtures/mock_tools.py` | Demo or test coverage without a real service |
| `delegate_to_runtime` | No code generated | The host runtime (Mellea session, memory backend) provides this |
| `external_input` | Env var declaration + assertion in `config.py` | Secret or per-invocation value the caller supplies |
| `load_from_disk` | `open(Path(__file__).parent / "path")` call | Reference data bundled with the package |
| `remove` | No code generated | Cross-reference artifact — doesn't need to be implemented |

---

## How dispositions affect what you receive

A skill spec that references three things — a persona (C1), a weather API (C6 HTTP), and an abstract "notify team" function (C6 abstract) — produces:

- `config.py` — `PERSONA_TEXT = "..."` (bundled C1)
- `tools.py` — `def fetch_weather(location: str, params: str) -> str:` with real HTTP code (`real_impl`)
- `constrained_slots.py` — `def notify_team(message: str):` with `raise NotImplementedError(...)` (`stub`)

The pipeline runs end-to-end for any fixture that doesn't call `notify_team`. Fixtures that do call it raise immediately until you fill the stub.

---

## Auto mode vs. ask mode

**Auto mode** (the default) assigns dispositions from the fixed table above. No user interaction. Fast and reproducible.

**Ask mode** (roadmap — not yet fully implemented in v0.1) walks the user through each dependency interactively, showing the default disposition and allowing overrides. Planned for future releases.

**Config mode** (roadmap) reads a JSON file of disposition overrides. Same override options as ask mode, but non-interactive.

For most use cases, auto mode produces sensible defaults. The cases where you want to override:

- An `abstract` C6 tool that you want mocked instead of stubbed (for demo/testing)
- A reference document you want bundled rather than loaded from disk
- An API key you want bundled (for internal tools with non-secret keys)

Override dispositions by editing the intermediate artifacts manually and running `mellea-skills compile --repair-mode` — or wait for ask/config mode.

---

## Reading the dependency plan

After compile, inspect `intermediate/dependency_plan.json`:

```json
{
  "plan": [
    {
      "entry_id": "dep_001",
      "category": "c6_tools",
      "disposition": "real_impl",
      "subtype": "http",
      "target": "tools.py:fetch_weather",
      "prerequisites": ["HTTP access to wttr.in"]
    },
    {
      "entry_id": "dep_002",
      "category": "c6_tools",
      "disposition": "stub",
      "subtype": "abstract",
      "target": "constrained_slots.py:notify_team"
    }
  ]
}
```

Each entry tells you: which dependency, what category, what disposition, and where the generated code lands. This is the authoritative record of what the compiler decided.

Also check `SETUP.md §8` — the compiler generates a human-readable stub catalogue with each stub's signature, docstring, and example implementation. This is the fastest way to see what needs implementing.

---

## What "stub" means in practice

A stub is a Python function that ends with:

```python
raise NotImplementedError(
    "TO IMPLEMENT: Replace this stub with a real implementation.\n"
    "Expected signature: notify_team(message: str) -> None\n"
    "Example: ...\n"
)
```

The docstring explains what the function should do. `SETUP.md §8` lists every stub with its expected inputs, return type, and a sample implementation.

Until a stub is filled, any pipeline branch that calls it raises `NotImplementedError`. Whether that causes the whole run to fail depends on how the call site is guarded:

- **Unguarded** (`result = notify_team(...)`) — run fails immediately
- **Guarded** (`try: notify_team(...) except NotImplementedError: pass`) — run degrades gracefully, that branch treated as "not verified"

The compiler wraps many stub call sites with guards to allow partial runs. See [Fill stubs](../guides/fill-stubs.md) for the walkthrough.
