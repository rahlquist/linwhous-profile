# Remote source fetch + audit workflow (verified commands)

## 1. Fetch raw files (sequential, one terminal call)
```bash
mkdir -p ~/audit_tmp && cd ~/audit_tmp && \
curl -sL -o a.cpp  https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path_a> && \
curl -sL -o b.cpp  https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path_b> && \
curl -sL -o c.hpp  https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path_c> && \
echo DONE && wc -l *.cpp *.hpp
```
- Do **NOT** use shell `&` inside `terminal()` — it is rejected ("Foreground
  command uses '&' backgrounding"). Chain with `&&` instead, or set
  `background=true`.
- A wrong path/branch returns an HTML 404 page, not source. After download,
  `read_file` the first ~10 lines of each file to confirm it's C++/source, not
  `<html>`.

## 2. Locate suspect symbols
`search_files` (target=content, regex alternation, output_mode=content,
context=2..3). Example:
```
audio_chunk|l_this_round|n_rounds|_preprocess_audio|insert\(|480000|3000|EOS|encode_audio|generate
```
- Results truncate past ~50 matches; narrow the pattern or use `file_glob` to one
  extension, then re-run with `offset`.

## 3. Get exact line numbers
`read_file(path, offset, limit)` — 1-indexed. For files >500 lines the tool
returns `truncated:true` with a `next_offset`; continue with `offset=` until the
relevant function is fully captured. Copy the `LINE|CONTENT` lines verbatim into
the report.

## 4. Write the report
- Open with the constants that bound the behavior (e.g. `FS=16000`,
  `WINDOW_SAMPLES=480000`, fixed buffer `128*3000`).
- Quote the suspect code **verbatim** with `file:line`.
- Trace data flow construction → consumer; conclude leading/trailing/length.
- End with a question→finding table.
- Save full report to the workdir AND give a tight summary.
