# Skill Tiers

Every compiled skill is classified into one of three tiers based on how much setup is needed to run any fixture end-to-end. Tier is determined at compile time and documented in `SETUP.md §1`.

---

## Tier definitions

| Tier | Name | What's needed | Who can run it |
|------|------|---------------|---------------|
| **T1** | Zero friction | Only Ollama with the default models pulled. No stubs, no env vars, no external services, no extra binaries. | Anyone with Ollama installed |
| **T2** | One gap | One stub or one missing artifact blocks one code path. Other paths run unchanged. | Anyone who fills one stub or provides one file |
| **T3** | External integration | Requires an external service (HTTP API, CLI tool, OAuth, secret store) before any fixture completes. | Users with access to the external service |

---

## Pre-compiled example tiers

| Skill | Tier | What blocks full execution |
|-------|------|--------------------------|
| `weather` | T1 | Nothing — runs immediately with Ollama |
| `superpowers-systematic-debugging` | T1 | Nothing — runs immediately with Ollama |
| `sentry-find-bugs` | T1 / T2 | T1 for most fixtures (stubs are guarded); T2 for full codebase-scan paths (`search_fn`, `read_file_fn`) |
| `clawdefender` | T3 | `audit`, `sanitize`, `scan_skill` modes require `jq`, `npx`, `clawhub` on PATH and `chmod +x` on scripts |

---

## Tier classifications for the full skills library

See `skills/README.md` for the per-skill tier table with source attribution.

---

## T1 in detail

A T1 skill:
- Has no `NotImplementedError` stubs in any file
- Declares no environment variables that must be set before first run
- Makes no subprocess calls to external binaries
- Makes no HTTP calls requiring API keys (or makes only calls to public, no-auth APIs like `wttr.in`)

All four examples in `docs/README.md` can be run as T1 — the stubs in `sentry-find-bugs` are guarded so the T1 fixtures don't trigger them.

---

## T2 in detail

A T2 skill has exactly one thing blocking one branch:
- One stub (a `NotImplementedError` function) that blocks specific fixture paths
- One reference file (`load_from_disk`) that needs to be provided
- One env var that only one branch reads

Other branches run without that thing. A T2 skill is partially runnable out of the box.

Converting T2 to T1:
1. Identify the stub or missing artifact from `SETUP.md §8`
2. Fill the stub or provide the file
3. Run `mellea-skills validate --all` to confirm all fixtures pass

---

## T3 in detail

A T3 skill requires an external integration before any fixture can complete. Examples:
- An OAuth-authenticated API (Slack, GitHub, Sentry)
- A local CLI tool that must be installed (`jq`, `clawhub`, custom scripts)
- An API key for a service with no public access tier
- A running service that can't be mocked at the fixture level

`SETUP.md §1–§7` documents the full setup procedure for T3 skills, including prerequisite installation, credential configuration, and service setup.

---

## How tier affects development workflow

When you compile your own skill:

1. Check `SETUP.md §8` immediately after compile — it lists all stubs
2. If there are no stubs and no required env vars: you're T1. Run `mellea-skills run --fixture <name>` immediately.
3. If there are stubs: you're T2 or T3 depending on what the stubs need. Fill them per [Fill stubs](../guides/fill-stubs.md).
4. If the stubs require external services: you're T3. Follow `SETUP.md §1–§7` before running.

The tier is not encoded in `melleafy.json` yet (planned for a future manifest version) — check `SETUP.md` directly.
