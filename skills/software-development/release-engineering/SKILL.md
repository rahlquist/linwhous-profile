---
name: release-engineering
description: "Use when shipping a versioned software release."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [release, changelog, versioning, git, ci, verification]
    category: software-development
---

# Release engineering

Use this skill when completed repository changes must become a versioned release,
especially a patch/point release. Keep the release narrow: include the requested
fixes and their tests, but do not silently pull deferred enhancements or change
an intentional CI policy.

## Procedure

1. **Inspect release scope.** Read the current status, recent commits, version
   sources, existing changelog, README release claims, security/documentation
   scope, and CI workflow. Separate shipped changes from explicitly deferred
   work before editing.
2. **Write the changelog first.** Add the release entry before the release
   commit. Prefer compact `Added`, `Changed`, `Fixed`, `Deferred`, and
   `Verification` sections. Record security-boundary changes and test coverage;
   record intentional omissions such as manual-only integration jobs instead of
   “fixing” them by assumption.
3. **Synchronize metadata.** Update every authoritative version source: plugin
   manifests, package metadata, language package files, generated lockfile
   package entries, README status, and release/security documentation. Search
   tracked files for the old version; dependency versions in lockfiles are not
   release metadata and should remain unchanged.
4. **Close the test loop.** Add regression tests for each corrected finding,
   including rejection/error paths. Run the complete local suite, formatters,
   linters, compilation/build checks, and `git diff --check` before staging.
   Never report remote CI as passed from local execution alone.
5. **Commit coherently.** Stage the release files deliberately and use one
   conventional release commit containing the implementation, tests, changelog,
   metadata, and documentation. Do not mix unrelated cleanup into the release.
6. **Push and verify.** Push the commit, read back the remote branch SHA, and
   confirm it exactly matches the local release commit. If this is a requested
   release, create and push an annotated `vX.Y.Z` tag pointing at that commit;
   read back both the tag object and dereferenced commit.
7. **Report honestly.** Read the remote workflow status after pushing. Say
   `queued` or `in_progress` until checks complete; say `success` only after a
   fresh remote read confirms it. Distinguish local verification from CI.

## Pitfalls

- **Do not call a release complete with a stale changelog.** The changelog is
  the durable contract for what shipped; update it in the same commit as code.
- **Do not bump only the obvious manifest.** Search tracked files because
  plugin metadata, Rust/Python package metadata, README claims, and security
  scope often carry independent versions.
- **Do not rewrite intentional workflow policy.** A manually triggered
  integration suite is a release fact to document, not a defect to “repair”
  without explicit direction.
- **Do not describe an application boolean as operator approval.** Document the
  actual trust boundary; UI/operator confirmation is a separate feature.
- **Do not claim a tag or CI result from a successful push alone.** Read back
  the tag dereference and workflow state because push success proves neither.

## Release output

Report the version, changelog path, commit SHA, tag SHA/dereference when used,
local verification results, and the exact remote CI state. Mention any deferred
work explicitly.

This skill complements the protected GitHub workflow skills: use those for PR,
authentication, and generic remote operations; use this skill for release scope,
changelog, metadata synchronization, and release verification.
