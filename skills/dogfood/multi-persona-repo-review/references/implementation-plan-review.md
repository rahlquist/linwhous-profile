# Reviewing an implementation plan / design doc against a live codebase

A third artifact class, distinct from the repo review and the standalone
document audit: a **written plan proposing changes to a real codebase that
exists right now**. The plan is fiction; the repo is fact. The whole value of
the review is measuring one against the other.

Trigger: "review this plan/RFC/design doc, and make sure it matches the
current design, code and testing patterns of <project>."

## The one finding class that dominates

**Premise verification beats prose critique.** A plan that reads beautifully
can be unexecutable because a module it assumes exists does not, or a
subsystem it proposes to build already ships. Rank these CRITICAL — they
invalidate whole task sections, whereas a vague sentence costs an hour.

Every plan makes three kinds of checkable claims. Check all three before
reading for style:

1. **"X already exists"** → does it? `search_files` for the symbol, not just
   the package. `grep -rn "def resolve_credential"` returning nothing is
   decisive; a same-named package existing is not.
2. **"We need to build Y"** → does it already ship? Grep the capability, not
   the proposed name. The author's vocabulary rarely matches the repo's.
3. **"Modify file Z"** → does Z exist at that path? Plans routinely name
   files from a stale mental model or a since-deleted feature.

## The layer-confusion trap (highest-value check)

Finding a package with the right *name* does NOT establish it does the job
the plan needs. Distinguish:

- **provisioning / injection layer** — runs once at startup, pushes values
  into env or global state. Signature shape: `apply_all(cfg, home) -> Report`.
- **runtime query layer** — called per-use, returns a value to a caller.
  Signature shape: `get_x(name, default) -> Optional[str]`.

They can both live under a plausible name and only one satisfies a plan that
says "resolve a credential at request time."

Decisive probe — **check the call sites, not the definition**:

```bash
grep -rn "\.fetch(" --include=*.py <pkg> <consumers> | head
```

If the only callers are inside the package itself, it is an internal
injection layer with no public query API. I got this exact call wrong once
and a persona caught it; the correction inverted the report's top finding
from "you're duplicating existing infra" to "your central premise is
unfounded." Both are CRITICAL, but they demand opposite rewrites.

## Self-contradiction is free evidence

Cross-read the plan's confident body against its own open-questions section.
A plan that asserts "Hermes has a generic credential resolver" at line 45 and
asks "what module is the generic credential interface?" at line 438 has told
you it was written before its premise was checked. Quote both line numbers
side by side — it is the single most persuasive finding you can write,
because the author cannot dispute their own document.

## Contribution-rubric alignment

If the project ships an AGENTS.md / CONTRIBUTING.md, treat it as a rubric and
grade the plan against it explicitly. Recurring conflicts worth grepping for:

- new env vars for **behavioral** config where the project mandates a config
  file (`.env` is usually secrets-only)
- a **new CLI command or core tool** where an incumbent surface already
  covers it — always grep for the incumbent (`grep -rn '"doctor"'`) before
  accepting a plan's new-surface proposal
- **duplicate infrastructure** where the rubric says extend-don't-duplicate
- **change-detector tests** where the rubric wants invariants (a plan saying
  "assert the enum has these 9 members" is a snapshot; "assert the module
  imports the shared enum rather than defining its own" is an invariant)
- **prompt-cache / statefulness invariants** asserted by the plan but with no
  task specifying the mechanism, and no test asserting it

A plan that merely *restates* a constraint ("must not break caching") without
naming the hook that satisfies it has not addressed it. Rank MEDIUM and
require a test.

## Executability check (the fresh-grad lens, adapted)

Plans fail junior engineers in a specific way: deferred references that never
resolve. Grep the plan for `the module selected in Task N`, `the nearest
existing test module`, `the appropriate helper`. Then check whether Task N
actually names a file. Very often it says "identify the module" and never
records the answer — so every downstream task inherits an unresolved
pointer.

The strongest single fix to recommend: **pin the contract before the first
test is written.** TDD ordering is not the problem; TDD against an
uninvented API is. If Task 2 asserts a five-state return type that Task 3
will define, the tests are asserting against the implementer's imagination.
Require the exact dataclass/enum in the plan text.

## Severity calibration for plan reviews

- **CRITICAL** — a premise that is false; a task that cannot be executed as
  written; work the plan would duplicate against shipped code.
- **HIGH** — a stated guarantee the design cannot actually deliver (e.g.
  "guaranteed cleanup" via `finally`, which SIGKILL ignores; "no persisted
  token" while ambient credential helpers stay enabled), or a rubric
  violation likely to draw maintainer rejection.
- **MEDIUM** — unresolved blockers filed as "open questions"; untestable
  acceptance criteria; unspecified module locations; missing test strategy.
- **LOW** — document hygiene, redundant alternatives, verified-correct items
  worth recording so the author knows they were checked.

## Always deliver both artifacts when asked for a revised plan

The review and the rewrite are different documents with different jobs.
Write both:

- `<date>_review-<slug>.md` — severity-ranked findings with plan line numbers
  and repo file:line evidence, a "what works" section, top-3 fixes.
- `<date>_revised-<slug>.md` — the executable rewrite.

The revised plan should open with a **"What changed from rev. 1" table**
(`| rev. 1 assumption | verified reality | consequence |`). This is what makes
the rewrite reviewable rather than just different — the author can audit
every change back to evidence. Also fold the original's "open questions" into
a **Resolved decisions** section: a question you can answer from the repo is
not a question, and leaving it open is how the next revision inherits the
same ambiguity.

## Cheap high-yield probes

```bash
# premise: does the claimed symbol exist at all?
grep -rn "def <claimed_function>" --include=*.py .

# layer check: who actually calls it?
grep -rn "\.fetch(\|\.resolve(" --include=*.py <pkg> | head

# incumbent surfaces before accepting a "new command" proposal
grep -rn '"doctor"\|add_parser' <cli_dir>/*.py | head

# named files that may not exist
find . -path ./node_modules -prune -o -iname '*<name>*' -print

# secret hygiene of the plan document itself (report a clean result as a positive)
grep -nE "/home/|<username>|ghp_|github_pat_|[0-9a-f]{8}-[0-9a-f]{4}" <plan>

# shared-variable ambiguity: is the env var already claimed elsewhere?
grep -rn "GITHUB_TOKEN" --include=*.py . | head
```

That last one generalizes: whenever a plan keys on an env var or config name,
check whether another subsystem already consumes it. Shared credentials are a
recurring MEDIUM the plan author never sees.

## Persona briefs that worked for this artifact class

Same four lenses, re-aimed at a document rather than a repo:

- **adversarial senior dev** — verify every factual claim against the repo;
  flag correctness gaps in the proposed design (signal-handling, HTTP status
  conflation, rate limits, cleanup guarantees).
- **fresh grad** — "could you actually start Task 1?" Surfaces the deferred
  references and unanswered blockers better than any other lens.
- **config perfectionist** — config surface, secret hygiene, precedence,
  diagnostics. Reliably finds the missing config keys and the credential
  delivery holes (argv / `/proc/<pid>/environ` / on-disk askpass).
- **struggling user** — reads only the user-facing strings the plan
  specifies. Consistently produces the finding the engineers all miss:
  the plan classifies failure states exhaustively and tells the user what to
  DO about none of them, and specifies no setup path at all.

Give each the plan path, the repo path, the contribution rubric path, and
"cite plan line numbers AND repo file:line."
