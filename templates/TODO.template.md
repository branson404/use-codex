# TODO: <one-line goal>
model: sol            # sol = default | astra = only hard design/debug
effort: high          # low = mechanical | medium | high | xhigh = rare, justify it
write: src/foo/, tests/test_foo.py   # ONLY these paths are writable. Dirs end with /. New files need a listed dir.
run: pytest -q -p no:cacheprovider   # extra commands Codex may run (prefix match; no commas inside one command)
max-minutes: 20
max-commands: 60
# backend: auto            # auto | docker (Linux) | local (macOS/Linux, Codex's native sandbox)
# image: python:3.12-slim   # docker backend only: change if the job needs node/git/etc.
# web: no                   # yes only if a step truly needs web search

## Context
- <2-4 bullets: what exists and where. Paths + conventions, no prose.>

## Steps
1. `src/foo/bar.py` - <exact change: signature / behaviour>. Done when: <observable check>.
2. `tests/test_foo.py` - <tests to add>. Done when: `pytest -q tests/test_foo.py` passes.

## Forbidden
- git commit/push, installing packages, network, files outside `write:`, refactors not listed above.
- If a step can't be done as written: print `BLOCKED: <reason>` and stop. Do not improvise.

## Acceptance (the orchestrator re-runs these)
- `pytest -q tests/test_foo.py` -> all pass

## Report
Print `DONE:` then at most 8 lines: files changed, checks run, results.
