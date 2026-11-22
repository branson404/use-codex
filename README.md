# use-codex

A [Claude Code](https://claude.com/claude-code) skill that hands planned development work to an
[OpenAI Codex](https://github.com/openai/codex) agent, so Claude spends its usage on planning and review
instead of its own bash and edit calls.

Claude writes one short TODO. `codex_run.py` runs `codex exec --json` for that TODO inside a locked-down Docker
container and checks every command and file edit Codex makes against the TODO's allowlist. If Codex steps outside
the plan, the container is killed, and Claude can steer the same session with `resume`. Claude then verifies the
result itself.

```
Claude: plan -> TODO.md ──> codex_run.py start ──> Docker container
                                │                     └─ codex exec --json (gpt-6.2 sol / astra)
                                └─ live monitor: unlisted command / edit / tool? kill
Claude: read summary, re-run acceptance checks, commit
```

## Requirements

- Linux with Docker (the user must be able to run `docker`)
- [Codex CLI](https://github.com/openai/codex) on `PATH`, logged in (`codex login`)
- Python 3.9+ (standard library only)
- A container image whose architecture matches the host (the host's `codex` binaries are mounted in, default image `python:3.12-slim`)

## Install

```bash
git clone https://github.com/branson404/use-codex ~/.claude/skills/use-codex
```

Restart Claude Code. Then say "use codex to ..." or run `/use-codex`.

## Use

1. Copy `templates/TODO.template.md` and fill it in. The header is machine-read:

   ```
   model: sol            # sol (default) | astra | any exact model id
   effort: high          # low | medium | high | xhigh
   write: src/foo/, tests/test_foo.py   # the ONLY writable paths (dirs end with /)
   run: pytest -q        # extra commands Codex may run (prefix match, no commas inside one)
   max-minutes: 20
   max-commands: 60
   ```

2. Check, then run (put `start` in the background):

   ```bash
   python3 scripts/codex_run.py check TODO.md /path/to/project
   python3 scripts/codex_run.py start TODO.md /path/to/project
   ```

3. Watch `~/.cache/use-codex/<run>/summary.log` (one compact line per action). Exit codes: `0` done,
   `3` violation, `4` timeout/stall, `5` failed or `BLOCKED`.
4. Steer or stop: `resume <run> "one-line correction"`, `stop <run>`, `status <run>`, `cleanup <run>`.

`model: sol` / `astra` resolve to the newest listed `gpt-X-sol` / `gpt-X-astra` in `~/.codex/models_cache.json`.

## What it enforces

| Layer | Rule |
|---|---|
| Container | `--cap-drop ALL`, `no-new-privileges`, pids/memory/cpu limits, non-root, project mounted **read-only** |
| Writes | only the `write:` paths are mounted read-write; `.env`, `*.pem`, `*.key`, `.npmrc`, ... are blanked |
| Commands | read-only inspection plus the TODO's `run:` entries; `$(...)`, backticks, unlisted binaries, paths outside `/work` are violations |
| Tools | MCP and web search are violations unless `web: yes` |
| Budget | `max-commands`, `max-minutes`, and a stall timer (no events for 300 s) |
| Credentials | your `~/.codex/auth.json` is copied into the run folder and deleted afterwards; a refreshed token is copied back (Codex rotates refresh tokens, so the host copy would otherwise go stale) |

## Limits (please read)

- **The monitor is not a security boundary against a hostile model.** The container is. The monitor parses shell
  text and is a policy and cost guard; unusual but legitimate commands may need a `run:` entry.
- **Network stays on** (Codex needs its API). The monitor blocks `curl` and friends, but an allowlisted command can
  still make network calls.
- The run folder holds a short-lived copy of your Codex login, and the container user can read it. The monitor
  blocks reads of it; the filesystem does not.
- Tested only on **Linux/arm64** with **codex-cli 0.170** and `gpt-6.2-sol`. Event parsing follows Codex's
  `--json` output; if it changes, the runner prints `WARN` instead of silently not policing.
- Using Codex costs Codex usage. Keep TODOs short and `effort` low; a tiny job still used tens of thousands of
  (mostly cached) input tokens.
- Check the Codex CLI's current terms before running automation against your account.

## Development

```bash
python3 scripts/codex_run.py selftest          # offline: policy engine + monitor (also runs in CI)
python3 scripts/codex_run.py start TODO.md PROJECT --probe   # verifies container confinement, no model call
```

## License

MIT, see [LICENSE](LICENSE).
