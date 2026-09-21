---
name: multi-persona-repo-review
description: Adversarial multi-persona review of a repo, CLI tool, or skill package — code + UX + config + onboarding — using a hands-on heuristic eval plus parallel persona subagents, merged into one severity-ranked report.
version: 1.0.0
metadata:
  hermes:
    tags: [code-review, ux, adversarial, personas, onboarding, heuristic-eval, fan-out]
    related_skills: [adversarial-ux-test, requesting-code-review]
---

# Multi-Persona Repo Review

For reviewing SOMEONE ELSE'S artifact (GitHub repo, CLI tool, skill package, standalone HTML document, configuration reference, architecture visualization, implementation plan / RFC / design doc) across code, config, adversarial, and onboarding — not your own pre-commit diff (use requesting-code-review for that) and not a live web app (use adversarial-ux-test for that).

Three artifact classes, three workflows: **repo/tool** (the numbered workflow below), **standalone document** (the Document audit variant), and **implementation plan against a live codebase** (see `references/implementation-plan-review.md` — premise verification, layer-confusion probes, rubric alignment, and the review + revised-plan double deliverable).

## Workflow

1. **Clone to a scratch dir** (`rm -rf /tmp/<name> && git clone --depth 1 <url> /tmp/<name>`). Never review a stale copy.
   - **Private repos:** if the anonymous clone 404s, don't stop and don't declare the repo missing — bootstrap auth first. Check env (`$GITHUB_TOKEN`), `gh auth status`, then the user's secrets manager. Recipe that works in this environment: `export BWS_ACCESS_TOKEN=$(grep -E '^BWS_ACCESS_TOKEN=' ~/.hermes/.env | cut -d= -f2-)`, `bws secret list` to find the GITHUB_TOKEN id, `bws secret get <id>`, then clone via `https://x-access-token:${GITHUB_TOKEN}@github.com/...`. Gotcha: Hermes does NOT export .env values into the terminal tool's shell, so tokens must be re-derived each session unless the user has exported them (e.g. in ~/.bashrc). Before falling back to "repo not found," also check GitHub search and the owner's public repo list to distinguish private vs renamed vs deleted.

2. **Run your own hands-on heuristic eval FIRST** — this grounds the persona reports in verified fact:
   - Read README / manifest / config example / main entry points.
   - **Fresh-HOME trick:** run the tool with `HOME=/tmp/fhX` to simulate a first-time user. This exposes silent default paths, stray writes outside the project dir, and hardcoded author paths instantly.
   - Walk the documented install/onboarding commands VERBATIM, copy-paste style. Doc/code path mismatches (cp flattening a dir the next command expects) are the #1 dead-on-arrival finding.
   - Probe failure paths: `</dev/null` stdin (EOF in interactive wizards → raw EOFError traceback?), bad config values, malformed config file, missing optional deps, out-of-range CLI args. Raw Python tracebacks for routine user errors are a top-3 recurring finding.
   - Check exit codes on failure paths (`echo $?`) — a crash with exit 0 is worse than a crash.
   - grep for the author's home dir / username in shipped code (`grep -rn "/home/" scripts/`) — hardcoded paths in tests and secret-resolution code are a recurring CRITICAL.

3. **Fan out persona subagents in parallel** (background, one batch + overflow singles). Proven personas:
   - adversarial senior dev (bugs, crash paths, doc/code mismatches)
   - fresh-grad junior (first-10-minutes clarity)
   - config perfectionist (undocumented keys, validation, credential handling)
   - struggling user (onboarding walkthrough, jargon, error recovery)
   Give each: the scratch path, the persona brief, "cite file:line, severity-ranked, max N findings, plain text."

4. **Merge into ONE severity-ranked report** (Nielsen 0–4: CRITICAL/HIGH/MEDIUM/LOW), dedup across personas, and ALWAYS include a "What works" section with verified positives — the report is evidence, not vibes. End with top-3 highest-impact-per-line fixes.

5. **Cross-reference open PRs (post-review).** Users routinely ask "do any of these already have fixes submitted?" — run it proactively; it is cheap. `curl -s https://api.github.com/repos/<owner>/<repo>/pulls?state=open` to list, then `.../pulls/<n>/files` per PR to get changed files. Map each finding's touchpoint file(s) against the PR file lists; skim PR titles AND bodies — feature titles can hide embedded fixes. Report per-finding status (fix submitted / no fix), not a blanket answer. Bonus: note adjacent surface area introduced by open PRs (new key-extraction scripts, browser extensions, native code) — new trust surface worth flagging even though it is not a fix. Recipe + worked example: `references/post-review-pr-cross-reference.md`.

6. **Deliverable: also write a durable evaluation document** (markdown, e.g. `<scratch>-review/evaluation.md`) alongside the terminal report: reviewer experience summary, verified hands-on evidence tables, merged findings, what-works, top-3 fixes, PR cross-reference, scope/limitations. The user asked for this explicitly and it survives the session; the terminal report is the summary, the doc is the record.

### Document audit variant (standalone files — HTML, config references, architecture diagrams)

When the artifact under review is a standalone document (e.g. a standalone HTML file serving as an architecture reference), the repo workflow does not apply directly. Adapt as follows:

1. **Parse all claims.** Read the document and extract every factual assertion — hostnames, file paths, config keys, precedence rules, boundary statements. Catalog them as (claim, location-in-document) pairs.
2. **Verify claims against live state.** For each claim, run the cheapest verification available:
   - Hostname claim → `hostname` and compare.
   - User/label claim → `id` and `grep -r <label> ~/.hermes/` to check if the label appears in config.
   - Config key claim → `grep -n <key> ~/.hermes/config.yaml` to verify presence/values.
   - Filesystem artifact claim → `ls <path>` and compare against the document's listed files.
   - Config precedence claim → check whether `use_gateway`, `provider`, or equivalent keys in config.yaml match the stated precedence.
3. **Assess document integrity.** A standalone reference file has no built-in integrity check. Flag whether the document has a checksum, hash, timestamp, or version marker. If absent, note it — a stale or tampered reference is a P2 finding.
4. **Fan out the same 4 review lenses** (code quality, config correctness, adversarial resistance, onboarding clarity) as structured checks, not as parallel subagents. A document is read-only and small enough for a single thorough pass, but the 4 lenses ensure nothing is missed.
5. **Produce structured findings** in the same severity-ranked format (P0–P4, category, title, description, evidence line refs, suggested fix) so results are comparable across document types.

A document audit does not need the full persona subagent fan-out of a repo review — the artifact is small, static, and self-contained. But the same evidentiary standard applies: every finding needs line references, every config claim needs live verification, and the confidence column should distinguish "verified from live state" from "asserted without evidence."

## Pitfalls

- **delegate_task batch cap:** `max_concurrent_children` (often 3) rejects larger batches as a WHOLE — split N personas into batch-of-cap + single dispatches.
- **Batch results arrive async.** Live transcripts (`cache/delegation/live/<id>/task-N.log`) truncate assistant text with "(+N chars)" — do NOT synthesize the final report from truncated transcripts if the full batch payload is still pending. Headline findings are fine for triage; full merge waits for the payload.
- **The consolidated batch message is ALSO truncated — but the full payload is on disk.** Separate from the live-transcript issue above: when the batch finally returns, each persona summary is head+tail trimmed ("Showing 1,194 chars (head) + 500 chars (tail) of 10,169 total"), and the omitted MIDDLE is where most findings live. The complete text sits at `cache/delegation/subagent-summary-<N>-<timestamp>.txt`; read it with `read_file(path=..., offset=11)` per the footer's own hint. ALWAYS do this before finalizing. Verified cost of skipping it: one session's truncated head exposed 2 of 11 findings, and 6 genuinely new HIGH/MEDIUM items sat unread in the omitted middle.
- **Don't ship raw persona complaints.** The pragmatism filter from adversarial-ux-test applies: every finding needs file:line evidence + user impact + minimal fix.
- **Verify persona claims yourself when cheap.** Subagent self-reports can be wrong; re-run the one-liner before ranking a finding CRITICAL.
- **A persona may correct YOU — re-verify, amend, and disclose.** Your own heuristic eval is not privileged over a subagent's. Worked example: my eval concluded "this abstraction already exists, the plan should just consume it"; the adversarial persona said it does not exist. Re-checking proved the persona right — the package I found was a *startup-time provisioner*, not the *runtime query API* the artifact assumed, and the top finding inverted from "you're duplicating existing infra" to "your premise is unfounded." Grepping a package name and reading its ABC is NOT enough to establish what a layer does: check who actually CALLS it (`grep -rn "\.fetch(" --include=*.py`) to tell an injection layer from a query layer. When you amend, note the correction in the report's limitations section rather than quietly rewriting — that the review caught its own reviewer is itself evidence of the method working.
- **CI-green + local-red test failure = suspect a timezone-dependent test before ranking it a product bug.** GitHub Actions runners are UTC, so a TZ bug sails through CI and only breaks contributors west of UTC. Diagnose by re-running the failing test under `TZ=UTC`, `TZ=Asia/Tokyo`, `TZ=America/New_York` — if TZ changes the outcome, the TEST is wrong, not (necessarily) the product. Classic trigger: the test hardcodes a UTC timestamp at a period boundary (`2026-08-01T00:20:00.000Z` is July 31 20:20 in UTC−4), while the code intentionally uses local-calendar windows (users think in local time). Verified example: token-monitor `sessionUsageArchive.test.js:199` — pass UTC/Tokyo, fail New York; product logic correct. Rank as HIGH (dev-experience) but never as a product correctness bug.
- **Batch interruption recovery.** If a persona batch returns `status=interrupted` (not pending), the live transcripts still hold the full tool trace — complete each persona's lens yourself from the traces, re-verify every claim you rank above LOW, and disclose the interruption in the report. The "don't synthesize from truncated transcripts" rule applies only while the payload is still pending; a definitively interrupted batch leaves the transcripts as the only evidence.
- **`pgrep`/`pkill` self-match kills your own shell.** A `-f` pattern that appears literally in your own command line (e.g. `pkill -f "node src/hub/server"` in a command that contains that string) matches the shell running the command — exit −15, your whole probe dies. Use a self-non-matching regex: `pgrep -f "src/hub/serve[r]"`. Bit me twice in one session before the fix.
- **npm's install-scripts guard can leave `npm ci` green with native modules missing.** Newer npm blocks postinstall scripts by default and logs `npm warn install-scripts` — exit 0, review proceeds, but native-only paths (e.g. koffi on Windows) silently lack their binary. Check the install log for the warning, note the limitation, don't stall.
- **Never check exit codes through a pipe.** `cmd 2>&1 | head -3; echo $?` reports the exit of `head`, not `cmd` — a real exit-1 error looks like exit 0 and you'll misrank a finding (reported a false HIGH this way once). Redirect instead: `cmd >/dev/null 2>&1; echo $?`.
- **Fresh-HOME runs may trigger security-scan warnings** on `rm -rf /tmp/...` — use distinct scratch dirs per probe.
- **shellcheck may not be installed** — don't stall. Read the scripts with read_file and hunt manually; the bash-pitfall checklist lives in `references/bash-setup-script-review.md`.
- **Repos-in-flux hardware/version stories:** when docs and scripts disagree about a moving target (e.g. GPU migration mid-flight: README says CUDA, script builds ROCm, user says both cards soon), check whether the user still considers the doc "stale" before ranking CRITICAL — mid-session user context can reframe "stale docs" as "prematurely true docs." The durable finding is then "no agreed story for the upcoming state," not "docs are wrong."
- **Unverified hostnames and labels in standalone documents.** When reviewing a document that names hosts, users, or labels (e.g. "yogaman", "gateway host"), verify those entities exist in the live environment. If a name is unverified, flag it explicitly rather than silently accepting it as ground truth — unverified labels propagate downstream errors in config precedence, access rules, and onboarding docs.

See `references/repo-review-playbook.md` for the full command recipes and report template.
See `references/implementation-plan-review.md` for reviewing an implementation plan / RFC / design doc against a live codebase — premise verification, the provisioning-vs-query layer-confusion probe, self-contradiction cross-reads, contribution-rubric grading, severity calibration, and the review + revised-plan double deliverable with its "what changed from rev. 1" evidence table.
See `references/post-review-pr-cross-reference.md` for the open-PR fix-mapping recipe, timezone-dependent-test diagnosis, full-history secret scan, and live server-probe checklists.
See `references/bash-setup-script-review.md` for the static bash/Python review checklist (SystemExit-string exit-code trap, SUDO_USER-as-root traps, doc-vs-code matrix, destructive-networking guards, idempotency probes) — load it for any review of system-setup/provisioning repos.
See `references/document-audit-techniques.md` for cross-reference verification patterns when reviewing standalone documents (HTML, config references, architecture diagrams) against live system state.
See `references/onboarding-review-methodology.md` for the focused framework for reviewing first-10-minutes onboarding clarity (6 focus areas, severity-ranked findings, heuristics for cold-start and empty-state checks).
