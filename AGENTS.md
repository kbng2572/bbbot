# AGENTS.md

## Cursor Cloud specific instructions

### Overview

nanobot is an ultra-lightweight personal AI assistant framework (~4,000 lines of Python). It has a single Python package (`nanobot/`) and an optional Node.js WhatsApp bridge (`bridge/`).

### Development commands

| Task | Command |
|------|---------|
| Install (dev) | `pip install -e ".[dev]"` |
| Install (with Matrix) | `pip install -e ".[dev,matrix]"` |
| Lint | `ruff check .` |
| Tests | `pytest` |
| CLI help | `nanobot --help` |
| Initialize config | `nanobot onboard` |
| Run agent (CLI) | `nanobot agent` or `nanobot agent -m "Hello"` |
| Run gateway | `nanobot gateway` |
| Check status | `nanobot status` |
| Build WhatsApp bridge | `cd bridge && npm install && npm run build` |

### Non-obvious caveats

- **PATH**: After `pip install -e .`, the `nanobot` CLI binary is installed to `~/.local/bin`. Ensure `~/.local/bin` is on `PATH`.
- **LLM API key required**: The agent (`nanobot agent`) will not work without at least one LLM provider API key configured in `~/.nanobot/config.json`. The easiest provider to configure is OpenRouter (`providers.openrouter.apiKey`). Without a key, the CLI will print "No API key configured" and exit gracefully.
- **`nanobot onboard`** must be run once before using the agent. It creates `~/.nanobot/config.json` and workspace files. It is safe to re-run (it will not overwrite existing config).
- **Matrix optional dependency**: Tests in `tests/test_matrix_channel.py` require the `[matrix]` optional dependency (`pip install -e ".[dev,matrix]"`). Without it, pytest will fail to collect that test module.
- **Pre-existing test failures**: As of this writing, 5 tests in `test_matrix_channel.py` fail due to test-vs-code mismatches (KeyError on `attachments` metadata, wrong mock signature). These are pre-existing issues in the repository, not caused by environment setup.
- **Ruff lint findings**: The existing codebase has ~529 ruff lint findings (mostly import ordering). These are pre-existing and not blockers for development.
- **No external services required**: nanobot stores state in local JSON files under `~/.nanobot/`. It does not require databases, Redis, or Docker for local development. The only external dependency is an LLM API key.
- **`nanobot/templates/AGENTS.md`** is a template file shipped with the nanobot agent product — it is NOT a development instruction file. Do not confuse it with this root-level `AGENTS.md`.
