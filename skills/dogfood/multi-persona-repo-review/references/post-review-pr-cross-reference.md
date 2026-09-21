# Post-Review PR Cross-Reference & Verification Recipes

When to use: after merging a repo review, to answer "do any open PRs already fix my findings?" — and for the diagnostic recipes that de-risk ranking (timezone tests, full-history secret scans, live server probes). All recipes verified on token-monitor (2026-08): 10 open PRs, 13 findings, zero fixes — the mapping table shape below.

## PR cross-reference (GitHub API, no auth needed for public repos)

```bash
# 1. List open PRs (title, author, created, draft)
curl -s "https://api.github.com/repos/<owner>/<repo>/pulls?state=open&per_page=100" \
  | python3 -c "import json,sys; [print(f\"#{p['number']:>4} | {p['title'][:80]} | {p['user']['login']} | {'[DRAFT]' if p.get('draft') else ''}\") for p in json.load(sys.stdin)]"

# 2. Per PR: changed files (this is what you match against finding touchpoints)
curl -s "https://api.github.com/repos/<owner>/<repo>/pulls/<n>/files?per_page=50" \
  | python3 -c "import json,sys; [print(f\"{f['status']:9} {f['filename']} (+{f['additions']}/-{f['deletions']})\") for f in json.load(sys.stdin)]"

# 3. PR body (titles lie; bodies carry the 'fixes #xx' refs)
curl -s "https://api.github.com/repos/<owner>/<repo>/pulls/<n>" \
  | python3 -c "import json,sys; p=json.load(sys.stdin); print(p['title'], '\n', (p.get('body') or '')[:400])"

# 4. Fetch a file from a PR head ref for spot-checks
curl -s "https://raw.githubusercontent.com/<owner>/<repo>/refs/pull/<n>/head/<path>"
```

Procedure:
1. Build the finding→touchpoint map from your report (e.g. H1 → `tests/shared/foo.test.js:199`, M4 → `docs/configuration.md:48`).
2. For each PR, intersect its changed files with the touchpoints. A PR that touches the file is NOT automatically a fix — check the diff hunk; `.env.example`/README edits are usually feature additions, not doc-claim fixes.
3. Report per-finding status: `fix submitted (PR #n)` / `no fix`. Feature-only touches of the same file get a note ("touched by #n for features only").
4. Bonus (still in scope of "evaluate the PRs"): flag adjacent surface area new PRs introduce — a PR shipping a process-memory key-extraction script, a browser extension with content scripts on third-party domains, or native code (WidgetKit) each deserve a one-line risk note even though they are not fixes. Check extension manifests (`permissions`, `host_permissions`, `content_scripts`) and any `scripts/` added.

## Timezone-dependent test diagnosis

Trigger: full suite fails locally but the repo's CI is green (GitHub runners are UTC).

```bash
TZ=UTC          node --test tests/shared/<file>.test.js   # pass?
TZ=Asia/Tokyo   node --test tests/shared/<file>.test.js   # pass?
TZ=America/New_York node --test tests/shared/<file>.test.js  # fail?
```

- TZ-dependent outcome ⇒ the TEST is wrong, not necessarily the product. Pattern that causes it: test hardcodes a UTC timestamp near a period boundary (`2026-08-01T00:20:00.000Z` = July 31 20:20 in UTC−4) while the code uses LOCAL-calendar windows — a user-facing dashboard correctly thinks in local time.
- Verified example: token-monitor `tests/shared/sessionUsageArchive.test.js:199` — pass UTC/Tokyo, fail America/New_York; the month-window assertion failed because locally it was still July. Rank HIGH for dev-experience (red `npm test` for half the world, invisible to maintainers), never as a product correctness bug.
- Fix suggestion to include: construct `now` from local-time components (`new Date(2026, 7, 1, 0, 20)`) or use a non-boundary UTC timestamp (noon UTC).
- Isolate a single failing test file instead of the whole glob: `node --test tests/shared/<file>.test.js`.

## Full-history secret scan without gitleaks

```bash
cd /tmp/<repo> && git fetch --unshallow --quiet   # shallow clone → full history
git log --all -p --full-history 2>/dev/null \
  | grep -nEi "(sk-[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}|ghp_[A-Za-z0-9]{30,}|xox[baprs]-[A-Za-z0-9-]{10,}|AIza[0-9A-Za-z_\-]{30,}|-----BEGIN (RSA|EC|OPENSSH|PRIVATE)-----)"
# Exclude test fixtures / .example files / placeholders before concluding; check
# .gitignore coverage (.env, data/, dist/, .dev.vars) and that workflows use
# secrets.* placeholders only.
```

Shallow-clone secret scans are worthless — unshallow first. Report the count of commits scanned (e.g. 509) so the "clean history" claim is verifiable.

## Live server/hub probe checklist

For any local server (hub, daemon, CLI server): start with `terminal(background=true)` — never `&` in foreground (rejected), and never `pkill -f` with a pattern that appears in your own command line (self-match kills your shell; use `serve[r]` regex trick).

```bash
curl -s http://127.0.0.1:<port>/api/health          # baseline
curl -s -o /dev/null -w "%{http_code}\n" <url>      # status-code probes:
  # wrong secret → 401, right secret → 200, header-auth variants, query-string
  # auth (may be per-surface: Node hub rejected ?secret=, Worker accepted it)
  # malformed input: empty body, bad JSON, malformed path (%ZZ → decodeURIComponent
  # URIError — Node hub caught it, Worker did not → asymmetry finding)
ss -tlnp | grep <port>                               # verify bind address — the
  # security-by-default claim (loopback-only without secret) is provable live
```

## npm install-scripts guard note

`npm ci` can exit 0 while blocking native postinstall scripts — log shows `npm warn install-scripts` (koffi on token-monitor). The shared/JS code under test still works; only native paths (Windows registry access) are affected. Check the log, note the limitation, do not stall the review.
