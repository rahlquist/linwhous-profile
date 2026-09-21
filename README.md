# Linwhous — Hermes Agent Profile

A focused code-review and engineering profile for [Hermes Agent](https://hermes-agent.nousresearch.com/docs). Embodies a blunt, evidence-first reviewer persona (Linus Torvalds-inspired) that attacks code defects with surgical precision while remaining patient with genuine learners.

## Identity

- **Persona:** Torvalds-style code reviewer — hostile to bad code, not to people
- **Decision hierarchy:** Correctness → user impact → simplicity → maintainability → performance (measured) → style → process
- **Tone:** Blunt, evidence-first, no hedging. "I measured" beats "I think."

## What's included

| Path | Purpose |
|------|---------|
| `SOUL.md` | Identity document — NEO-PI-R personality model, decision hierarchy, communication rules |
| `profile.yaml` | Profile metadata (bot chat IDs) |
| `config.yaml` | Hermes runtime config — model provider, toolset, delegation, display |
| `skills/` | 60+ bundled skills (code review, devops, creative, research, mlops, etc.) |
| `assets/avatar.png` | Generic cartoon avatar (no personal imagery) |

## What's excluded (and why)

- `.env`, `auth.json` — credentials / per-profile secrets
- `state.db`, `projects.db`, `cron/executions.db` — runtime state, session history, cron runs
- `memories/`, `logs/`, `cache/`, `runtime/`, `sessions/` — personal memory and ephemeral runtime
- `bin/tirith` — 38 MB precompiled security-scanner binary (platform-specific)
- `backups/`, `.curator_backups/`, `state-snapshots/` — local backup artifacts
- Lock files, `.tick.lock`, `.jobs.lock`, `lock.json` — runtime coordination

## Installation

This profile is designed to live inside `~/.hermes/profiles/`. Clone and symlink, or copy:

```bash
# Option A: symlink (recommended — tracks upstream)
git clone https://github.com/rahlquist/linwhous-profile.git ~/hermes-profiles/linwhous-profile
ln -s ~/hermes-profiles/linwhous-profile ~/.hermes/profiles/linwhous

# Option B: copy
cp -r ~/hermes-profiles/linwhous-profile ~/.hermes/profiles/linwhous
```

Then restart Hermes or reload profiles. The profile appears as `linwhous` in the profile switcher.

## Configuration notes

- `config.yaml` sets `nous` provider with `tencent/hy3:free` default — change to your preferred model
- `command_allowlist` restricts shell execution — tighten or loosen per your environment
- `session_reset: mode: none` — conversations persist across restarts
- `delegation.max_concurrent_children: 4` — cap parallel subagents

## Requirements

- [Hermes Agent](https://hermes-agent.nousresearch.com/docs) installed and configured
- A valid model provider key (Nous, OpenRouter, OpenAI, etc.) for the configured default model

## License

MIT — see [LICENSE](LICENSE).
