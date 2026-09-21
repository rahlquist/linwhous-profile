---
name: upstream-project-research
description: Use when researching an upstream repo's docs/issues/source.
---

# Upstream Project Research

Answering "what does upstream actually say/do about X?" for a third-party repo — documented behavior, known bugs, changelog history, and the real implementation.

## When to use
- "Research <project> for <behavior/latency/bug>"
- Verifying a vendor claim before building on it
- Locating the issue/PR/commit that touches a specific code path

## Workflow

1. **Fetch the doc page first** (raw.githubusercontent, not the rendered site):
   `curl -sL https://raw.githubusercontent.com/<org>/<repo>/main/docs/.../<page>.md`
   Note explicitly what the docs *omit* — absence of a documented number is itself a finding.

2. **Pull releases + issues + PRs via the unauthenticated GitHub API.** Public repos work at low rate.
   - `GET /repos/<org>/<repo>/releases?per_page=100&page=N`
   - `GET /repos/<org>/<repo>/issues?state=all&per_page=100&page=N` — returns **both issues and PRs**; `item.get("pull_request")` distinguishes them.
   - Paginate until an empty page. Real projects have 600+ items.

3. **Keyword-filter titles+bodies locally**, then fetch full bodies only for the hits. Print `number | state | title | html_url`.

4. **Read the actual source** for the behavior in question:
   `curl -sL https://raw.githubusercontent.com/<org>/<repo>/main/src/path/file.cpp`
   Constants and loop structure beat prose. Header files carry the magic numbers (window lengths, buffer sizes).

5. **Label every claim** as *documented*, *maintainer-stated in a release note*, *reported in an issue*, or *my reading of the source*. Never blur them. Community perf analyses in issues are frequently unvalidated — quote their own disclaimer.

## Pitfalls

- **Renamed/transferred repos return `{"message": "Moved Permanently"}` from the API** — `curl -L` does NOT resolve this for `api.github.com` (it returns a JSON body, not a followable redirect to the same path). Query the **new** org path directly. Issues/PRs move with the repo even when the old URL still works in a browser.
- **Terminal tool output is truncated**, so `curl … | python -c json.load` fails to parse on large responses. Always `curl -o /tmp/x.json` then load the file. Single most common time-waster here.
- Do the whole fan-out inside one `execute_code` block (loop over pages/endpoints, filter, print only hits) instead of one tool call per fetch.
- Grep the fetched source with `search_files` against the local `/tmp` copy — faster than re-curling.
- Watch for **declared-but-unused variables** in the code path (e.g. an overlap constant never referenced) — strong bug signal, but report as "unverified by maintainers".
- Changelog searches must span a *range* of versions; a feature's bugs are often fixed in a release whose notes never name the feature.

## Output shape

Lead with the answer, then: documented behavior → implementation reality (quoted code + line numbers) → table of relevant issues/PRs with URLs → changelog entries by version → explicit caveats separating fact from inference. Every claim carries a URL. Keep it tight; no process replay.

See `references/fastflowlm-whisper.md` for a worked example.
