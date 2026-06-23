# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
