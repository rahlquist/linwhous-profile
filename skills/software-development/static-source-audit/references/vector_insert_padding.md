# C++ `vector(n)` + `insert(begin(), ...)` padding pitfall

## The idiom (as seen in FastFlowLM Whisper `generate()`)
```cpp
std::vector<float> audio_chunk(l_this_round);                                   // n zeros
audio_chunk.insert(audio_chunk.begin(),
                   audio_buffer.data() + current_idx,
                   audio_buffer.data() + current_idx + l_this_round);           // insert n real samples
```

## Step-by-step semantics
1. `std::vector<float> audio_chunk(l_this_round);`
   Single `size_t` ctor → **value-initializes** `l_this_round` elements. For
   numeric `float`, value-init = **zero** (`0.0f`).
   State: `[ 0, 0, ..., 0 ]`  (length `n`).
2. `audio_chunk.insert(audio_chunk.begin(), first, last);`
   Inserts `n` real samples at position 0. Existing elements shift right.
   State: `[ real[0..n), 0, 0, ..., 0 ]`  (length `2n`).

## Verdict
- **NOT leading zeros / zero-prepend.** Data is at the **front**; zeros are
  **trailing**.
- Resulting length before any downstream step = `2 * l_this_round`
  (e.g. `960000` for a 30 s / 480000-sample chunk).
- A naive reader expecting a zero-prepended buffer is wrong here.

## Whether it corrupts the input — depends on the consumer
Trace to where the vector is used. In the FastFlowLM case the consumer is
`_preprocess_audio`, which does:
```cpp
std::vector<float> x = audio;                       // copy, length 2n
if ((int)x.size() > N_SAMPLES)   x.resize(N_SAMPLES);          // 480000, keep FRONT
else if ((int)x.size() < N_SAMPLES) x.resize(N_SAMPLES, 0.0f); // grow w/ trailing zeros
```
Because real audio sits at the **front**:
- If `2n > 480000` (chunk > 15 s, incl. every full 30 s window): `resize(480000)`
  keeps the first 480000 samples = the entire real audio; trailing zeros dropped.
  No data loss.
- If `2n <= 480000` (chunk <= 15 s): grow with trailing zeros →
  `[ real(n), zeros(n), zeros(480000-2n) ]`. Audio still front, trailing zeros.

Final mel input = **audio-at-front + trailing zero pad**, exactly 480000 samples
→ 3000 mel frames. This is the **correct** Whisper convention. The idiom is
**wasteful/confusing** (2x alloc + O(n) insert shift) but **not a correctness
bug** for this consumer.

## How to report it
State BOTH:
1. What the local idiom does (front-data, trailing-zeros, length `2n`).
2. What the downstream renormalization makes of it (keeps front → harmless;
   would only corrupt if consumer kept the back or assumed length `n`).
Never label a pattern a bug on local appearance alone — trace to the consumer.
