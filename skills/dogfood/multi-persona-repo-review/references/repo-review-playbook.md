# Repo Review Playbook — command recipes & report template

Condensed from real reviews of a Python CLI skill package and a bash provisioning repo (2026-07). All recipes verified.

## Access: private repos & auth bootstrap

```bash
# Anonymous clone 404s? Don't stop — bootstrap credentials first.
env | grep -i github          # is GITHUB_TOKEN already exported?
gh auth status                # is gh logged in?
# Bitwarden Secrets Manager fallback (bws). Hermes does NOT export .env values
# into the terminal tool's shell — load the access token yourself:
export BWS_ACCESS_TOKEN=$(grep -E '^BWS_ACCESS_TOKEN=' ~/.hermes/.env | cut -d= -f2-)
bws secret list | grep -i github           # find the token's id
export GITHUB_TOKEN=$(bws secret get <id> | python3 -c "import json,sys; print(json.load(sys.stdin)['value'])")
git clone --depth 1 "https://x-access-token:${GITHUB_TOKEN}@github.com/OWNER/REPO" /tmp/REPO
```
- Distinguish private vs renamed vs deleted before reporting "not found": `curl -s https://api.github.com/users/<owner>/repos?per_page=100` and GitHub search.
- If the user wants the bootstrap to survive sessions: `export BWS_ACCESS_TOKEN=...` appended to ~/.bashrc (verified working; interactive bash only — systemd user services need their own Environment).

## Fresh-HOME probes (the highest-yield trick)

```bash
# First-time user simulation — catches silent defaults, stray writes, hardcoded paths
HOME=/tmp/fh1 python3 tool.py some-command
# EOF in interactive wizards — raw EOFError traceback?
HOME=/tmp/fh2 python3 tool.py setup </dev/null; echo "exit=$?"
# Bad config value — friendly error or KeyError traceback?
printf '[storage]\nbackend = "bogus"\n' > /tmp/fh3/.tool/config.toml
HOME=/tmp/fh3 python3 tool.py run; echo "exit=$?"
# Malformed config — friendly error or parser traceback?
printf 'not = = toml' > config.toml
# Missing optional dep — actionable message?
# Out-of-range validated arg — friendly or ValueError traceback?
python3 tool.py add --rating 99; echo "exit=$?"
# Exit codes: NEVER through a pipe — `cmd 2>&1 | head -3; echo $?` reports
# head's exit (0), masking cmd's real failure. Always: `cmd >/dev/null 2>&1; echo $?`
# Hardcoded author paths in shipped code (recurring CRITICAL)
grep -rn "/home/" scripts/ | grep -v __pycache__
# Author repo paths hardcoded in test tooling
grep -rn "REPO = \|SKILL = " scripts/
```

## Doc/code mismatch checks (dead-on-arrival class)

- Walk install docs VERBATIM in a fresh dir. `cp -r scripts/* dest/` flattens — if the next documented command is `dest/scripts/tool.py`, install is broken as written.
- mkdir-before-cp ordering in copy-paste blocks.
- Documented flags vs argparse reality: grep the docstrings for `--flag` mentions and check they exist in the parser (`tool.py --help`). Error-recovery hints pointing at dead commands are worse than no hint.
- Version-compat claims ("Python 3.10+") vs unconditional `import tomllib` (3.11+) — check every entry-point script, not just the main one.
- Two install guides in one repo (README vs manifest) with diverging steps — flag as HIGH consistency finding.

## Report template

```
<ARTIFACT> — COMBINED ADVERSARIAL REVIEW
Sources: <self heuristic eval>, <persona1>, <persona2>, ... Severity = Nielsen 0–4.

CRITICAL (sev 4) — blocks or breaks real use
  C1. <title> [<personas that found it>]
      file:line — evidence. Impact. Minimal fix.
HIGH (sev 3) / MEDIUM (sev 2) / LOW (sev 1) — same shape.

WHAT WORKS (evidence, not vibes)
  - verified positives: --help coverage, friendly FK errors, passing test suite, nonzero exit codes.

TOP 3 FIXES (highest user-impact per line changed)
  1. ...
```

## Persona dispatch briefs (copy-adapt)

- adversarial dev: "bugs, crash paths, silent failures, sharp-edge CLI behavior, doc/code mismatches. file:line + minimal fix, max 12, worst first."
- fresh-grad: "first-read; what confused you, what's missing, what you'd give up on. max 10."
- config perfectionist: "undocumented keys, missing validation, silent fallbacks, DSN/credential handling, driver selection flaws. max 12."
- struggling user: "walk the README literally with fresh HOME; where do you get stuck; jargon; error recovery. max 10, worst onboarding blockers first."

Dispatch: one `delegate_task` batch of up to `max_concurrent_children` (check — often 3), remaining personas as single dispatches. All run in background; results re-enter as async messages.
