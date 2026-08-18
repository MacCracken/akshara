# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Changed

- **`[deps.rosnet]` `0.2.0` -> `1.1.1`.** akshara shipped 1.0.1 still pinned to a pre-1.0 rosnet:
  it sits downstream of rosnet in the release order, and 1.0.1 went out before rosnet 1.1.1 existed,
  so the tag could not have been current at the time. Corrected now that rosnet 1.1.1 is published.
  Verified the bump took — vendored `lib/rosnet.cyr` moved to `1.1.1`. Suite **2/2**, unchanged.

## [1.0.1] - 2026-08-17

### Changed

- **Cyrius pin `6.2.36` -> `6.5.27`** (2026-08-17, ecosystem-wide ML/AI-arc realign ahead of
  the arc reopening). `cyrius lib sync --full` re-vendored the whole version-matched stdlib
  snapshot, clearing the toolchain-drift and `./lib/ shadows version-pinned` warnings.
  Suite unchanged and green at the new pin: **2/2 assertions**, identical to the pre-bump baseline.

## [1.0.0] - 2026-07-05

**v1.0 — a clean freeze.** No behavior change from 0.1.0: this cut freezes
what the shipping consumers (attn11 — the origin, 1060-suite — and tarka)
have exercised unchanged since the 2026-06-22 extraction; the multi-consumer
soak is the readiness evidence. One tokenizer, shared for real.

### Added
- **`docs/api.md`** — the frozen 1.x surface with contract notes: corpus +
  vocab (`corpus_set`/`corpus_build`/loaders + `g_V`/`g_vocab`/`g_char2id`),
  the width-generic packed store (`gd_ld`/`gd_st`/`stream_tok` +
  `g_data`/`g_datalen`/`g_data_w`), opt-in BPE (`bpe_learn` + the `g_bpe_*`
  state consumer checkpoints serialize), encode/decode (`tok_encode` /
  `tok_emit` / `decode_char`). Caller-buffer/caller-guarantees: I/O stays in
  consumer hooks, unresolved in the dist bundle by design. The tokenizer
  `g_*` state is process-global (one corpus at a time) and freezes WITH the
  surface — consumer checkpoints depend on it.

## [0.1.0] - 2026-06-22

**Extraction from attn11 (M1).** akshara is the text → token-id data layer carved
out of attn11's `src/train.cyr` (via the intermediate `src/tok.cyr` split) — the
third attn11 extraction after `rosnet` (tensors) and `tyche` (PRNG). One shared
tokenizer behind attn11 and [tarka](https://github.com/MacCracken/tarka).

### Added
- **`src/tok.cyr`** (`[lib]`): the corpus + tokenizer layer — adaptive byte vocab,
  opt-in BPE (Sennrich et al. 2016: learn / encode / decode, pure i64), the
  width-generic packed token store (u8 / u16 / i64) with a streaming read path, and
  the checkpoint-facing tokenizer state. Byte-identical to attn11's proven code.
- **`dist/akshara.cyr`** — the `cyrius distlib` bundle consumers import.
- `[deps.rosnet]` 0.2.0 for `t_alloc` / `t_zero`. The bundle leaves the consumer
  symbols `puts` / `_putc` / `secure_read_file` / `read_stdin` / `file_seek`
  unresolved (rosnet→tyche pattern); the pure path needs only rosnet + stdlib.
- Pure-path smoke (`src/main.cyr`): byte vocab + BPE + encode round-trip — green.

### Validation
- attn11 re-pointed to `[deps.akshara]` (was `include "src/tok.cyr"`): **full suite
  1060/1060 green, fuzz + bench compile** — the byte-identity gate. The lib is
  validated transitively by attn11's finite-difference grad-checks (the rosnet/tyche
  precedent); akshara's own richer test suite is a follow-on.
