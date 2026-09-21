---
name: static-source-audit
description: Audit source code for a specific bug; cite exact file:line.
category: software-development
---

# Static Source Audit

Fetch source files and statically analyze them for a specific pattern or bug,
producing a technical report with **exact file:line citations** and **verbatim**
code. Common triggers: "analyze <repo> for how X is handled", "find the chunking
logic", "is this buffer leading- or trailing-padded", "report the exact code at
lines N-M with citations".

## When to use
- A delegated/standalone task to investigate a specific code path in a repo you
  do NOT have cloned (fetch raw files), or a local repo.
- The deliverable is a *report* (findings + citations), not a code change.
- You must determine precise behavior (e.g. padding direction, loop bounds,
  frame counts) by reading actual source, not by guessing.

## Workflow
1. **Fetch raw files** (remote repos). Use `curl -sL -o <name> <raw_url>` and run
   multiple fetches **sequentially in one terminal command** (separate `&&`
   lines). Do NOT use shell `&` backgrounding inside `terminal()` — it is
   rejected; use `background=true` only for genuinely long downloads.
   - GitHub raw pattern:
     `https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path>`
   - Create a dedicated workdir (e.g. `mkdir -p ~/audit_tmp && cd ...`) and
     `wc -l` the files to confirm they downloaded (a 404 returns an HTML page,
     not source — check the first lines).
2. **Locate patterns** with `search_files` (pattern=regex, target=content) over
   the workdir. Use a broad alternation of suspect symbols
   (`audio_chunk|l_this_round|n_rounds|_preprocess_audio|insert\(|480000|3000|EOS|encode_audio`).
   Read the matches with `context=` to see surrounding lines.
3. **Get exact line numbers** with `read_file` (1-indexed; use offset/limit to
   page large files, and re-run with higher `offset` when the tool reports
   truncation at N lines). Capture the `LINE|CONTENT` output verbatim.
4. **Reason about language semantics precisely** — see Pitfalls. Trace the full
   data flow from construction → consumer before concluding a pattern is a bug.
5. **Write the report** with `file:line` citations and fenced verbatim snippets.
   Prefer a table of "question → finding" at the end. Save the full report to a
   file in the workdir AND summarize.

## Pitfalls
- **`std::vector<T> v(n)` is value-initialized.** For numeric `T` (e.g. `float`)
  that means **zeros**, not uninitialized memory. So
  `std::vector<float> audio_chunk(l_this_round);` = `l_this_round` zeros.
- **`v.insert(v.begin(), first, last)` puts data at the FRONT**, shifting the
  existing zeros to the **back**. Result = `[real_data (n), zeros (n)]`,
  length `2*n`. This is a **trailing** pad, NOT a leading/zero-prepend. Do not
  assume "constructor + insert at begin = leading zeros" — verify by tracing.
- **Whether a doubled buffer corrupts input depends on the consumer.** If a
  downstream step does `x.resize(N)` (keeping the FRONT), real data at the front
  is preserved and only trailing zeros are trimmed → harmless (just wasteful).
  It would only corrupt if the consumer kept the BACK or assumed length `n`.
  Always trace to the consumer before labeling a pattern a bug.
- **Don't stop at "it looks wrong."** A confusing/inefficient idiom can be
  functionally correct because of a later normalization step. Report BOTH: what
  the local pattern does, and what the downstream renormalization makes of it.
- **Declared-but-unused variables** (e.g. `overlapping_samples = 5*FS` never
  referenced) mean a *documented* behavior (overlap between chunks) is NOT
  actually implemented — state the gap explicitly rather than assuming overlap.
- **Frame/padding counts**: confirm the fixed buffer size (e.g. `128*3000`) and
  the resize/trim target (e.g. `N_SAMPLES = 480000`) to answer "is it always
  padded to N frames / does the encoder always see full frames".

## References
- `references/vector_insert_padding.md` — worked C++ example of the
  constructor+insert pitfall and how to decide leading vs trailing zero pad.
- `references/remote_fetch_workflow.md` — exact curl/search/read commands for a
  multi-file remote-source audit.

## Verification
After writing the report, re-open the cited files with `read_file` at the exact
line ranges you quoted to confirm the snippet and line numbers are verbatim —
citations must be reproducible by the reader.
