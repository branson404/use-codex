# use-codex

A [Claude Code](https://claude.com/claude-code) skill that hands planned development work to an
[OpenAI Codex](https://github.com/openai/codex) agent, so Claude spends its usage on planning and review
instead of its own bash and edit calls.

Claude writes one short TODO. `codex_run.py` runs `codex exec --json` for that TODO in a confined environment
(a locked-down Docker container, or Codex's native sandbox on a scratch copy) and checks every command and file edit Codex makes against the TODO's allowlist. If Codex steps outside
the plan, the container is killed, and Claude can steer the same session with `resume`. Claude then verifies the
result itself.

```
Claude: plan -> TODO.md ──> codex_run.py start ──> confined run (docker | local)
                                │                     └─ codex exec --json (gpt-6.2 sol / astra)
                                └─ live monitor: unlisted command / edit / tool? kill
Claude: read summary, re-run acceptance checks, commit
```

## Requirements

- **macOS or Linux** (Windows: use WSL2; native Windows is not supported)
- **[Codex CLI](https://github.com/openai/codex) installed and logged in** (`codex login`). That is all the default setup needs.
- Python 3.9+ (standard library only)
- Optional: Docker on Linux. If it is available the runner uses it for stronger isolation (see backends below).

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

## Does it save Claude usage?

Honest answer: **only on large jobs.** Measured on 2026-11-22 with Claude Sonnet 5.5 (high effort) doing the same four
tasks directly vs through this skill (one run per cell, tokens = a subagent's total, which includes about 49k of fixed
baseline, so the differences are small but real):

| Task (new code written) | Claude directly | Claude via use-codex | Difference | Codex input (cached) / output tokens |
|---|---|---|---|---|
| S, 0.7 KB | 49,212 | 54,474 | **+10.7%** | 84,084 (76,928) / 765 |
| M, 3.6 KB | 51,043 | 55,885 | **+9.5%** | 87,826 (82,560) / 1,657 |
| L, 8.5 KB | 53,950 | 56,304 | **+4.4%** | 217,895 (198,272) / 5,666 |
| XL, 17 KB | 58,407 | 57,840 | **-1.0%** | 218,689 (200,064) / 7,096 |

- Claude's tool calls went from 2-3 to 6-7 per job, and wall time from 9-54 s to 47-194 s.
- What Claude has to *write* is what shrinks: the TODO stays about 1-4 KB while the code it replaces grows to 17 KB.
  That is where the saving comes from, and why break-even is near 17 KB. Extrapolated from these four points (not
  measured), a 50 KB job would save roughly 15%.
- Quality was equal: every run passed its unit tests and an independent hidden check.
- **Codex side:** each job used 82k-219k input tokens (about 91% cached) and 0.5k-7k output tokens, and moved the
  Codex quota meters by 0-1 points of the 5-hour window and 0-1 of the weekly window. A fixed ~76k cached prompt
  overhead per run means many tiny jobs are wasteful; batch work into fewer, larger TODOs.
- Limits of this test: one run per cell, one model, tokens rather than dollars, Linux Docker backend only.

## Backends

`backend: auto` (the default) picks one per run. Override with `backend: docker` or `backend: local` in the TODO header.

| | `docker` (auto on Linux with a working Docker) | `local` (auto on macOS, or Linux without Docker) |
|---|---|---|
| Where Codex runs | locked-down container (`--cap-drop ALL`, non-root, resource limits) | on your machine, inside Codex's own sandbox (`workspace-write`: Seatbelt on macOS, bwrap on Linux), never `danger-full-access` |
| Project | mounted **read-only**; only `write:` paths are writable | **scratch copy** (no `.git`, dependency dirs or secret files); after the run only `write:` paths are copied back |
| Out-of-scope edits | impossible (read-only mount) | detected by diff, **reverted** and reported as a violation |
| Secrets | `.env`, `*.pem`, `*.key`, `.npmrc`, ... blanked | not copied into the scratch copy |
| Login | your `~/.codex/auth.json` is copied to a short-lived run folder, deleted afterwards (a refreshed token is copied back; Codex rotates refresh tokens) | uses your normal Codex login in place, nothing copied |

Both backends share the same live monitor:

| Rule | Effect |
|---|---|
| Commands | read-only inspection plus the TODO's `run:` entries; `$(...)`, backticks, unlisted binaries and paths outside the project are violations |
| Tools | MCP and web search are violations unless `web: yes` |
| Budget | `max-commands`, `max-minutes`, and a stall timer (no events for 300 s) |

## Limits (please read)

- **The monitor is not a security boundary against a hostile model.** The container (docker) or Codex's native sandbox
  (local) is. The monitor parses shell text and is a policy and cost guard; unusual but legitimate commands may need a
  `run:` entry.
- The `local` backend is weaker than `docker`: it relies on Codex's own sandbox plus the scratch copy. Use `docker`
  on Linux when you can.
- `docker`: network stays on (Codex needs its API); the monitor blocks `curl` and friends, but an allowlisted command
  can still make network calls. The run folder holds a short-lived copy of your login that the container user can
  read; the monitor blocks reads of it, the filesystem does not.
- **Tested:** `docker` backend live on Linux/arm64 with codex-cli 0.170 and `gpt-6.2-sol`. The `local` backend's
  logic (scratch copy, diff, apply/revert, secret exclusion) is tested offline with a stand-in `codex` in CI on
  Linux and macOS; a real Codex job through `local` on macOS has **not** been verified yet. On Linux hosts that block
  user namespaces Codex's native sandbox cannot start, so use `docker` there.
- Event parsing follows Codex's `--json` output; if it changes, the runner prints `WARN` instead of silently not
  policing.
- Using Codex costs Codex usage. Keep TODOs short and `effort` low; a tiny job still used tens of thousands of
  (mostly cached) input tokens.
- Check the Codex CLI's current terms before running automation against your account.

## Development

```bash
python3 scripts/codex_run.py selftest          # offline: policy engine, monitor, local backend (runs in CI on Linux + macOS)
python3 scripts/codex_run.py start TODO.md PROJECT --probe   # docker backend: verifies confinement, no model call
```

## License

MIT, see [LICENSE](LICENSE).
