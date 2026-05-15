# Fill Stubs

After compilation, some skills contain `NotImplementedError` stubs in `constrained_slots.py` or `tools.py`. A stub is a function the compiler couldn't auto-implement — typically an abstract tool reference ("search the codebase") or a service requiring credentials the compiler doesn't have. Filling stubs converts a partial skill into a fully functional one.

For a worked example, see [`docs/FROM_STUBS_TO_RUNNING.md`](../FROM_STUBS_TO_RUNNING.md), which walks through filling two stubs in the `sentry-find-bugs` skill.

---

## Step 1: Find every stub

```bash
grep -rn "raise NotImplementedError" <package_dir>/
```

Or check `SETUP.md §8` — the compiler generates a stub catalogue listing every stub with its name, signature, and description.

---

## Step 2: Read the stub

Open `constrained_slots.py`. Each stub follows this structure:

```python
def search_fn(pattern: str) -> list[str]:
    """Search the codebase for files containing the given pattern.

    TO IMPLEMENT: Replace this stub with a real implementation.
    Expected behavior: return a list of file paths matching the pattern.
    Example:

        import subprocess
        result = subprocess.run(
            ["grep", "-rl", pattern, "."],
            capture_output=True, text=True
        )
        return result.stdout.strip().split("\n") if result.stdout.strip() else []
    """
    raise NotImplementedError(
        "search_fn is not implemented. See SETUP.md §8 for implementation guidance."
    )
```

The docstring tells you:
- What the function should do
- The expected return type
- A sample implementation (often copy-paste ready)

---

## Step 3: Implement the body

Replace the `raise NotImplementedError(...)` with your implementation. Keep the function signature unchanged — `pipeline.py` and `fixtures/` call this with specific arguments.

```python
def search_fn(pattern: str) -> list[str]:
    """Search the codebase for files containing the given pattern."""
    import subprocess
    result = subprocess.run(
        ["grep", "-rl", pattern, "."],
        capture_output=True, text=True
    )
    return result.stdout.strip().split("\n") if result.stdout.strip() else []
```

---

## Step 4: Verify the implementation

Run a fixture that exercises the stub:

```bash
mellea-skills run <package_dir> --fixture <fixture_name>
```

Which fixture exercises which stub? Check `SETUP.md §8` — it lists which fixtures touch each stub. Or run all fixtures:

```bash
mellea-skills validate <package_dir> --all
```

A successful run (exit 0) means the fixture completed without raising.

---

## How to identify fixtures that use a stub

The compiler typically wraps stub call sites with exception guards:

```python
try:
    files = search_fn(pattern)
except (NotImplementedError, Exception):
    files = []  # degrade gracefully
```

This means a run might succeed (exit 0) even with unfilled stubs — but those paths return empty or "unverified" results. To see which paths were guarded, check `pipeline.py` for `except NotImplementedError` guards around calls.

To force the stub to be exercised without guards, run a fixture specifically designed for it (check `SETUP.md §8` for the fixture name).

---

## Multiple stubs

If a skill has multiple stubs, fill them in order:

1. Start with stubs that the most fixtures depend on
2. Verify each stub before moving to the next
3. Run the full fixture suite once all stubs are filled

Running `mellea-skills validate <package_dir> --all` after filling all stubs is the final check.

---

## When stub implementations need external services

Some stubs need credentials, running services, or installed tools (making the skill Tier T3). Set those up before filling:

```bash
# Example: Sentry API key for sentry-find-bugs
export SENTRY_API_KEY="..."

# Example: local git repository for search_fn
cd /path/to/repo
```

The skill's `SETUP.md §1–§7` documents prerequisites. The stub catalogue in `§8` lists which stubs need which services.

---

## After filling stubs

Run certification to include the stub-covered paths in the compliance report:

```bash
mellea-skills certify <package_dir>
```

The audit trail now includes Guardian verdicts for generations that go through the filled stub paths.

---

## Common issues

**`TypeError: unexpected keyword argument`**

The stub's parameter names changed between your edit and the call site. Restore the original signature from the docstring.

**`AttributeError` on the return value**

Your implementation returns a different type than the docstring specifies. The call site in `pipeline.py` expects the documented return type.

**Stub passes, but output is "unverified"**

The fixture hit a guarded call site (`except NotImplementedError: continue`). Your stub is working — but the fixture didn't trigger the guarded path. Check which fixtures exercise the unguarded paths in `SETUP.md §8`.

**`ConnectionError` or `AuthenticationError`**

Your implementation hits a real service. Check credentials and connectivity before the Mellea session starts.
