# Extension guide

Extend Mellea Skills Compiler by preserving one principle: the compiled package is the stable center.

New features should either improve how source specs compile into that package, how the package is validated, how it is governed at runtime, or how it is wrapped for other harnesses.

## Add a new source skill

1. Create a folder under `skills/<skill-name>/`.
2. Add `spec.md` or `SKILL.md`.
3. Put companion runtime assets in sibling directories named `scripts/`, `references/`, or `assets/` when needed.
4. Compile it.

```bash
mellea-skills compile skills/<skill-name>/spec.md --no-run
```

5. Inspect the generated package:

| File | Check |
|---|---|
| `SETUP.md` | Env vars, stubs, external services. |
| `mapping_report.md` | Whether source requirements mapped to the intended components. |
| `intermediate/dependency_plan.json` | Whether dependency dispositions are acceptable. |
| `intermediate/step_7_report.json` | Whether structural lints passed. |
| `fixtures/` | Whether fixture inputs cover realistic cases. |

6. Run validation and at least one fixture once the backend is available.

```bash
mellea-skills validate skills/<skill-name>/<skill_name>_mellea
```

## Add a compiler lint

Add Python lints when a rule can be checked deterministically.

Current lints live in `src/mellea_skills_compiler/compile/lints.py` and return `LintResult`.

1. Write a function with signature:

```python
def lint_new_rule(package_dir: Path) -> LintResult:
    ...
```

2. Populate `LintFailure` with file, line, column, message, and rule reference.
3. Add it to `ALL_LINTS`.
4. Add tests under `tests/mellea_skills_compiler/compile/test_lints.py`.
5. Confirm `run_lints(...)` writes the new result into `intermediate/step_7_report.json`.

Good lint candidates:

| Candidate | Why it should be deterministic |
|---|---|
| `melleafy.json` schema validation | The JSON Schema already exists in `.claude/schemas/melleafy.schema.json`. |
| No unresolved `NotImplementedError` in T1 examples | AST/text scan is enough. |
| Entry signature matches `pipeline.py` | Parse `melleafy.json` and AST. |
| Required files exist | Filesystem check. |
| No CWD-based companion asset paths | AST check like `bundled-asset-path-resolution`. |

## Add a deterministic writer

Use deterministic writers for files where structural drift is common or costly.

Current writer pipeline:

| File | Writer |
|---|---|
| `config.py` | `.claude/melleafy/writers/config_writer.py` |
| `fixtures/` | `.claude/melleafy/writers/fixtures_writer.py` |

To add another writer:

1. Define a JSON IR schema under `.claude/schemas/`.
2. Update the slash-command spec so Claude emits `intermediate/<name>_emission.json`.
3. Add a writer module under `.claude/melleafy/writers/`.
4. Add a `WriterSpec` in `compile/writer_renderer.py:default_writer_specs(...)`.
5. Add the output path to `_WRAPPER_RENDERED_PATHS` in `compile/claude_directives.py`.
6. Add lint tests that catch bypass or drift.

Good writer candidates:

| Output | Why |
|---|---|
| `melleafy.json` | It is a downstream API consumed by export. |
| `SETUP.md` | Stub, env var, and dependency documentation should be complete and consistent. |
| `dependencies.yaml` | Structured dependency metadata is better rendered deterministically. |
| `README.md` | Could be generated from manifest plus known templates. |

## Add a new export target

The export pipeline is intentionally staged.

1. Add a target name to `SUPPORTED_TARGETS` in `export/exporter.py`.
2. Create `src/mellea_skills_compiler/export/targets/<target>.py`.
3. Implement `translate_<target>(loaded: LoadedContext) -> TranslationPlan`.
4. Add dispatch in `stage3_translate(...)`.
5. Add target-specific structural checks in `stage5_lint(...)`.
6. Add tests with a minimal generated package fixture.
7. Document modality support and lost semantics.

Every target translator should preserve:

| Contract | Why |
|---|---|
| Bundled compiled package | The Mellea package remains the source of runtime truth. |
| `run_pipeline(...)` invocation | Avoid reimplementing generated logic. |
| Env var declarations | Users need deploy-time setup visibility. |
| Output serialization | Pydantic models must cross the target boundary predictably. |
| Reverse manifest | Users should know what source package and export version produced the adapter. |

## Add a new governance control

Governance extension can happen at three levels.

| Level | Code surface | Example |
|---|---|---|
| Risk identification | `certification/nexus_policy.py` | Include a new taxonomy or risk source. |
| Runtime observation | `guardian/guardian_hook.py` and `guardian/audit_trail.py` | Add a hook for a new Mellea event or richer tool metadata. |
| Compliance evidence | `certification/classification.py` and `certification/report.py` | Map new controls to audit evidence extractors. |

Add tests for every new mapping. Governance code is easy to make look plausible while being semantically weak, so tests should cover both positive and conservative fallback cases.

## Add a new source dialect

The slash-command compiler already discusses source-runtime detection and dialect docs live under `docs/dialects/`.

A new dialect should define:

| Question | Where to encode it |
|---|---|
| How is the source recognized? | `.claude/commands/mellea-fy-inventory.md` or dialect reference docs. |
| What files are source of truth? | Dialect doc and inventory rules. |
| What maps to C1-C9 categories? | Mapping rules and examples. |
| What becomes deterministic code vs stub? | Dependency disposition rules. |
| What fixtures prove it works? | Fixture generation rules and examples. |
| What export target, if any, should round-trip? | Export target docs and tests. |

Do not add a dialect only as prose. Add at least one example source and one compile expectation.

## Improve tests

The most valuable missing tests are end-to-end but bounded:

| Test | Value |
|---|---|
| Compile a tiny fixture skill with `--no-run` using a fake Claude output harness | Proves wrapper, writers, and lints cooperate without real model calls. |
| Validate all examples with `--no-run` | Catches structural drift in committed examples. |
| Export examples for all supported targets | Keeps target adapters from silently breaking. |
| Load every `melleafy.json` against schema | Stabilizes downstream contract. |
| Certify with a mocked Guardian and Nexus | Proves governance flow without Ollama. |

## Contribution rules of thumb

1. Prefer structured IR plus deterministic rendering over asking the LLM to write load-bearing source directly.
2. Prefer AST or schema checks over text regexes when validating Python or JSON.
3. Keep generated package contracts backward-compatible unless you also update exporter and docs.
4. Treat examples as executable documentation.
5. When a feature touches compile output, update tests, `melleafy.json` expectations, and generated example docs together.

