# Prerequisites

Three things are required before you can compile or run any skill.

---

## 1. Python 3.11–3.13

The compiler requires Python ≥ 3.11 and < 3.14.4 (ai-atlas-nexus and Mellea both require 3.11+; the upper bound comes from ai-atlas-nexus).

Verify:

```bash
python3 --version
# Python 3.11.x  ✓
# Python 3.13.x  ✓
# Python 3.10.x  ✗ — too old
# Python 3.14.4+ ✗ — too new
```

---

## 2. Claude Code (for compilation)

`mellea-skills compile` invokes Claude Code under the hood. Claude Code must be installed and authenticated before any compile can run.

Install Claude Code:

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:

```bash
claude --version
```

Authenticate with your Anthropic API key or LiteLLM gateway:

```bash
# Direct Anthropic API:
export ANTHROPIC_API_KEY="sk-ant-..."

# LiteLLM gateway (IBM or other proxy):
export ANTHROPIC_BASE_URL="https://your-litellm-host"
export ANTHROPIC_AUTH_TOKEN="your-token"
```

Claude Code requires `--allowed-tools Read,Write,Edit` for compilation. `mellea-skills compile` sets this automatically. No additional Claude Code configuration is needed.

---

## 3. Ollama with Granite models (for runtime)

Compiled skills run against Ollama at runtime. The default backend is `granite3.3:8b`. The certification pipeline's Guardian checks use `ibm/granite3.3-guardian:8b`.

Install Ollama:

```bash
# macOS / Linux:
curl -fsSL https://ollama.ai/install.sh | sh

# Windows: download from https://ollama.ai
```

Pull the required models:

```bash
# For compiled skill execution (default runtime model):
ollama pull granite3.3:8b

# For Guardian risk assessment during certification:
ollama pull ibm/granite3.3-guardian:8b
```

Set the Ollama API URL:

```bash
export OLLAMA_API_URL="http://localhost:11434"
```

Verify Ollama is running and the models are available:

```bash
ollama list
# NAME                               ID              SIZE      MODIFIED
# granite3.3:8b                      ...             ...       ...
# ibm/granite3.3-guardian:8b         ...             ...       ...
```

> **Note:** Ollama is only needed for running and certifying compiled skills. You can compile a skill (`mellea-skills compile`) without Ollama; the post-compile smoke check will be skipped automatically if the backend is unreachable.

---

## Install the compiler

Clone and install:

```bash
git clone https://github.com/generative-computing/mellea-skills-compiler
cd mellea-skills-compiler

python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -e .
```

Verify:

```bash
mellea-skills --help
# Usage: mellea-skills [OPTIONS] COMMAND [ARGS]...
# ...
```

---

## Summary

| Requirement | Purpose | Verify command |
|------------|---------|---------------|
| Python ≥ 3.11, < 3.14.4 | Running the compiler and compiled skills | `python3 --version` |
| Claude Code | Spec decomposition during compile | `claude --version` |
| `ANTHROPIC_API_KEY` or gateway env vars | Claude API access | (env check) |
| Ollama running | Compiled skill execution | `ollama list` |
| `granite3.3:8b` pulled | Default runtime model | `ollama list` |
| `ibm/granite3.3-guardian:8b` pulled | Guardian checks during certify | `ollama list` |
| `OLLAMA_API_URL` set | Backend connectivity | (env check) |
