---
name: use-codex
description: Delegate a planned development or infra job to a Codex agent (gpt-6.2 sol/astra) running in a confined Docker container, so Claude spends its usage on planning and review instead of bash/edit tool calls. Claude writes one crisp TODO, a runner executes Codex against it and kills or steers it the moment it runs an unlisted command or touches an unlisted file. Use when the user says "use codex", "delegate to codex", "/use-codex", "save claude usage", or hands over a multi-step build/refactor/test-writing job. Skip for tiny edits (under ~3 tool calls).
---

# use-codex

Claude plans and reviews. Codex does the typing. A runner enforces the plan.

## 0. Gate (10 seconds)
- Tiny job (a one-line fix, a single read)? Do it yourself; Codex has startup cost.
- `python3 ~/.claude/skills/use-codex/scripts/codex_run.py check TODO PROJECT` prints `login: present|MISSING`.
  If Codex answers `refresh token was revoked`, stop and tell the user to run `! codex login` (interactive; you cannot).
- Docker must work (`docker ps`). Image default `python:3.12-slim` has no git/node; set `image:` if needed.

## 1. Write the TODO (this is where usage is saved or wasted)
Copy `templates/TODO.template.md`. Rules:
- You have already read the code; the TODO says WHAT changes, not how to explore. Exact paths, signatures, behaviours.
- One step = one file or one behaviour, each with a `Done when:` check. Aim for under ~60 lines total.
- `write:` is the whole write scope. Dirs end with `/`. Never `.`. Codex cannot touch anything else (read-only mount).
- `run:` lists the only non-read-only commands it may use (tests, linters, builds). Anything else = violation.
- Forbid git commits, installs, network. Claude commits after review.
- Put acceptance commands in `## Acceptance`; you re-run them yourself afterwards.
- Do not paste large code or write essays. Short, literal, checkable.

Model/effort: `sol` + `high` default. `astra` only for hard architecture or gnarly debugging. `effort: low` for mechanical edits,
`medium` for straightforward features. `xhigh` or above only with a reason you can state in one sentence. Aliases resolve to the
newest listed model in `~/.codex/models_cache.json`.

## 2. Dry check
`python3 ~/.claude/skills/use-codex/scripts/codex_run.py check TODO PROJECT` shows model, scopes, masked secret files (.env, keys), limits.

## 3. Run (background) and watch
```
python3 ~/.claude/skills/use-codex/scripts/codex_run.py start TODO PROJECT      # run_in_background: true
```
The runner starts a container (`--cap-drop ALL`, project mounted read-only, only `write:` paths writable, `.env`/keys blanked,
login copy deleted afterwards) and checks every Codex event live. It prints one compact line per action and writes
`~/.cache/use-codex/<run>/summary.log`. You do not poll. Optionally attach a Monitor to
`tail -F <run>/summary.log | grep --line-buffered -E 'VIOLATION|STALL|TIMEOUT|BUDGET|WARN|BLOCKED|RESULT'`.

Auto-stopped (container killed) on: command not in the allowlist or touching paths outside `/work`, edit outside `write:`,
unlisted tool use (MCP/web), more than `max-commands`, over `max-minutes`, no events for `stall-seconds`.

## 4. Streamline (steer) on a stop
Exit codes: 0 done, 3 violation, 4 timeout/stall, 5 failed/blocked. Read only the last ~8 lines of `summary.log` and `result.json`.
- Harmless drift (e.g. it tried `pip install` or wandered): `codex_run.py resume <run> "one-line correction"`. Same session, same limits.
- Needs something not in the TODO: add it to the TODO (new `write:`/`run:` entry) and start a fresh run, or do that bit yourself.
- Same violation twice, or anything that looks adversarial (reading `/codexhome`, curl, git push): `stop`, take over or rewrite the TODO.
- `BLOCKED:` means the step was wrong or underspecified. Fix the TODO; do not argue with it.
Max two resumes per job. After that, take over.

## 5. Verify (never trust the report alone)
- `git -C PROJECT diff --stat` (or `ls`) to confirm only expected files changed.
- Run the `## Acceptance` commands yourself. Read `last_message.md` (not the event stream).
- Then commit/PR as the user wants. Codex never commits.

## 6. Cleanup
Automatic at the end of every run (container removed, login copy deleted, empty placeholders removed). If the runner itself was
killed: `codex_run.py cleanup <run>`. Runs live under `~/.cache/use-codex/`; delete old ones freely.

## Keeping usage under control
- Short TODO, narrow `write:`, `effort` no higher than needed, `max-minutes`/`max-commands` set per job.
- One worker per job. Parallel workers only with disjoint `write:` sets and independent TODOs.
- Do not re-explain context to Codex that it can read in the files; do not re-read what Codex already summarised.
- Check `tokens=` on the RESULT line; if a job type is costly, lower effort or tighten the TODO next time.

## Known limits (be honest about them)
- Network stays on (Codex needs its API). The monitor, not the network, blocks `curl`; a command that is allowlisted can still call out.
- The policy parses shell text. It blocks `$(...)`, backticks and nested shells by design; unusual-but-legit commands may need a `run:` entry.
- Codex's own login lives in `/codexhome` inside the container. Reading it is blocked by the monitor, not by the filesystem.
- Codex rotates refresh tokens. The runner copies a refreshed login back to `~/.codex/auth.json` after each run; never copy auth files by hand.
- `codex_run.py selftest` re-validates the policy engine offline. `--probe` on `start` verifies the container confinement without a model call.
- Verified live (2026-11-21, codex-cli 0.170, gpt-6.2-sol): a real job ran end to end, `BLOCKED` -> `resume` -> `DONE` worked, files stayed inside `write:`, `.env` untouched, container and login copy removed. The monitor was also replayed on the real `events.jsonl` with stricter policies and flagged the right command/edit. Codex usually obeys the allowlist in its own prompt and prints `BLOCKED`, so the kill path is a backstop, not the common case.
- Codex needs its sibling binaries (`codex-code-mode-host`, `codex-exec-server`) next to `codex`; the runner mounts all `codex*` files. If a run says the tool host is missing, that is why.
- Underspecified TODOs cost a resume: e.g. `unittest discover -t .` needs `tests/__init__.py`, which must be in `write:`/Steps. Name every file a step implies.
- The runner prints `WARN` if it sees command events it cannot parse (format drift); then read `events.jsonl` before trusting the run.
