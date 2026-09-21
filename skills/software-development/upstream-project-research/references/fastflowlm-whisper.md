# Worked example: FastFlowLM Whisper ASR (Aug 2026)

Question: documented Whisper chunking behavior, latency, known bugs.

## Repo topology gotcha
- `FastFlowLM/FastFlowLM` → transferred to `ROCm/FastFlowLM` as of v0.9.46.
- `api.github.com/repos/FastFlowLM/FastFlowLM/*` returns `{"message":"Moved Permanently"}`; must query `ROCm/FastFlowLM`.
- Issue creation appears disabled on the new repo (per issue #651), so old-repo URLs still circulate.

## Key sources
- Docs: `https://raw.githubusercontent.com/ROCm/FastFlowLM/main/docs/docs/models/whisper.md` — usage only. **No latency numbers, no chunking description.** Context length listed "NA".
- Source: `src/common/whisper/modeling_whisper.cpp` (`Whisper::generate`, ~L111-248) and `src/include/whisper/modeling_whisper.hpp`.

## Implementation facts (read from source, not documented)
```cpp
static constexpr int FS = 16000;
static constexpr int WINDOW_LENGTH = 30;              // seconds
static constexpr int WINDOW_SAMPLES = FS * WINDOW_LENGTH;
mel_feature = buffer<bf16>(128 * 3000);               // fixed 30 s mel buffer
```
- Every chunk, including a short final one, is padded into the full 30 s mel window → a 3 s clip still costs a full 30 s encoder pass. Structural latency floor.
- Sliding window is **timestamp-driven**, not fixed-stride:
```cpp
if (enable_time_stamp){
    float end_time = _get_time(last_time_stamp);
    if (end_time == 0){ end_time = 30; }
    l_this_round = end_time * FS;
}
current_idx += l_this_round;
```
- `int overlapping_samples = 5 * FS;` declared at L114 and **never used** — dead overlap logic, no seam overlap. Unverified by maintainers.
- Language re-detected and `clear_context()` called per chunk.

## Notable issues/PRs
| # | Note |
|---|---|
| 637 | `response_format` ignored; timestamps computed then discarded (`return_time_stamp=false` hardcoded in `rest_handler.cpp`) |
| 510 | Open PR: `verbose_json` + multipart dispatcher dropped everything but `model`/`file` |
| 545 | `assert(hidden_size > 0)` crashes `flm serve --asr 1` and `flm list` after whisper pull |
| 585 | Standalone ASR server exits code -1 after 2.5–2.8 h uptime, consistently |
| 651 | v0.9.46 `flm serve whisper-v3:turbo` → "Unsupported model family", silently falls back to Llama-3.2-1B |
| 234 / 313 | No `--language` flag; poor German recognition |
| 636 | Community XDNA2 perf analysis, self-declared unvalidated |

## Changelog
- v0.9.14 introduced whisper-large-v3-turbo + `/v1/audio/transcriptions`; v0.9.21 standalone ASR server.
- v0.9.40: memlock fix affecting standalone ASR. v0.9.41: Gemma4 audio "clipped"→"split into chunks" wording.
- v0.9.42–v0.9.46: **zero Whisper/ASR entries.**
- Trap: v0.9.40 "Chunk Prefill" / `--prefill-chunk-len` is LLM prompt prefill, unrelated to audio chunking. Don't conflate.
