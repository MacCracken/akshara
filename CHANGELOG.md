# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [1.1.1] - 2026-11-09

A toolchain and dependency realign to the latest releases. No source change.

### Changed

- **Toolchain `6.7.9` → `6.7.18`** (through 6.7.10–6.7.17), the latest cyrius
  release. `src/` is unchanged, and so is every value akshara computes: 1.1.0's
  2,192-value tokenizer dump is byte-identical at 6.7.18, on x86_64 and on
  aarch64 (under qemu), so it matches 6.6.20 and 6.7.9 too. Suite **2/2**
  unchanged; fmt, lint (including 6.7.11's and 6.7.12's new warnings, over
  `src/` and `tests/`), `vet`, bench and fuzz pass. The smoke binary grows from
  188,048 to 195,744 B, all of it unreachable stdlib code; with DCE it shrinks
  from 32,400 to 31,904 B.

  Upstream changes checked against akshara's source (`src/`, `tests/`), from
  the 6.7.10–6.7.18 sections of cyrius's `ecosystem-migration.md`. Nearly all of
  it concerns structs, generics, traits, vectors, multi-value returns, `Str`,
  net/TLS/threads and platform ports, which akshara does not use. The rest:
  - **6.7.11** and **6.7.12** add lint rules for buffer overruns, write lengths
    and array sizing. All six source and test files lint with 0 warnings.
  - **6.7.11** refuses a top-level `return` and stops running `main` twice in a
    program with no explicit exit. Every akshara entry ends with an explicit
    exit (`sys_exit` or `syscall`) and has no top-level `return`.
  - **6.7.14** bakes constant global initializers into the image instead of
    storing them at startup (`var g_stream_fd = 0 - 1;` and the rest of
    `src/tok.cyr`'s globals). Same values; nothing writes them earlier.
  - **6.7.17** refuses an include line with code after the file name, and a
    statement that is only a value builtin. `src/main.cyr`'s includes carry only
    a trailing `#` comment, which is allowed, and akshara has no bare
    `load*(…);` statement.
  - No akshara name (including the five consumer-hook stubs in `src/main.cyr`)
    collides with a fn or global in the 6.7.18 stdlib, rosnet or tyche, so the
    new duplicate-fn warnings do not fire.
- **`[deps.rosnet]` `1.2.0` → `1.2.2`**, the latest rosnet, built against 6.7.18.
  It brings tyche `1.2.0` → `1.2.2` transitively. rosnet 1.2.2's CPU results are
  bit-identical to 1.2.1. Its one change on akshara's surface is `t_alloc`:
  `t_alloc(0)` now returns an empty tensor instead of exiting, a negative or
  wrapping size exits with "invalid size", and its errors go to stderr. akshara
  calls `t_alloc` only with fixed positive sizes (256, 1,024, 768, 8,192 and
  589,824), so none of that reaches it. tyche 1.2.2 gives the same bits as
  1.2.0 for every state its API can reach. Vendored `lib/rosnet.cyr` and
  `lib/tyche.cyr` byte-match the tagged `dist/` bundles.
- **Vendored `lib/` re-synced** (`cyrius lib sync --full`). It byte-matches the
  6.7.18 snapshot: the same 113 files as 6.7.9, 66 of them with new content.
- **`lib/agnosys.cyr` removed**, as rosnet 1.2.2 did. It was a scaffold-era file
  that no cyrius snapshot from 6.6.0 on ships, that nothing in akshara used, and
  that `lib sync --full` never prunes. `lib/` is now exactly the 6.7.18 snapshot
  plus rosnet and tyche.
- **`cyrius.lock` regenerated from scratch.** One line each for rosnet 1.2.2
  (`5c27e7d`) and tyche 1.2.2 (`cdb74e2`), 115 file hashes (116 less
  `agnosys`), trailer `cyrius 6.7.18`. `cyrius deps --verify` and
  `cyrius deps --locked` are clean.
- `dist/akshara.cyr` changes only in its version line, and `dist/akshara.deps`
  is unchanged (`io`).

### Note — the pin, `lib/` and `cyrius.lock` land together

From 6.7.18, with a fully vendored `lib/`, cyrius refuses a moved pin until
`lib/` follows it: `cyrius deps` and `cyrius build` exit 1 ("cyrius.lock not
re-locked") until `cyrius lib sync --full` runs. A commit that moves the pin in
`cyrius.cyml` without `lib/` and `cyrius.lock` fails CI at its `cyrius deps`
step.

## [1.1.0] - 2026-11-09

Moves akshara onto the cyrius 6.7 series. No library or API change.

### Changed

- **Toolchain `6.6.20` → `6.7.9`** (through 6.7.0–6.7.8). `src/` is unchanged,
  and so is every value akshara computes. A dump of 2,192 tokenizer values is
  byte-identical between 1.0.4 @ 6.6.20 and 1.1.0 @ 6.7.9, on x86_64 and on
  aarch64 (under qemu). It covers the embedded corpus at byte level (vocab,
  `g_char2id`, the u8 packed store); BPE on it at the full merge budget (45
  merges learned, the u16 store, the merge table, every id's span length and
  bytes); `tok_encode` of the corpus (u16) and of a probe with unseen bytes
  (i64); and a 512-byte corpus holding all 256 byte values. Suite **2/2**
  unchanged; fmt, lint, bench and fuzz pass. The smoke binary stays at 188,048 B.

  Upstream changes checked against akshara's source (`src/`, `tests/`):
  - **6.7.0–6.7.8** reserve `trait`, `const`, `true`, `false`, `do` and `dyn`,
    and **6.7.3** makes `!` a real operator (before, a lone `!x` compiled as
    `x`). akshara uses none of them outside comments and strings.
  - **6.7.9** makes unary `-` and `~` on a constant bind tighter, and
    **6.7.8–6.7.9** change how typed `*T` pointers step and dereference.
    akshara writes negatives as `0 - n`, uses no `~`, and has no typed
    pointers: its tables are untyped addresses read with `load8` / `load16` /
    `load64`.
  - **6.7.6** rounds every write into an `f32`. akshara is pure i64.
  - **6.7.6** deprecates `cyrius tests` (removed in 6.7.12). CI and the docs
    already use bare `cyrius test`, which still runs the suite.
- **`[deps.rosnet]` `1.1.3` → `1.2.0`**, the rosnet release built against 6.7.9.
  It brings tyche `1.1.1` → `1.2.0` transitively. Neither changed source or
  values; vendored `lib/rosnet.cyr` and `lib/tyche.cyr` byte-match the tagged
  `dist/` bundles.
- **Vendored `lib/` re-synced** (`cyrius lib sync --full`). It byte-matches the
  6.7.9 snapshot: the same 113 files as 6.6.20, 67 of them with new content.
  Nothing was added or removed.
- **`cyrius.lock` regenerated from scratch**, as in 1.0.4. One line each for
  rosnet 1.2.0 (`f065c58`) and tyche 1.2.0 (`2c393fe`), the same 116 file
  hashes, trailer `cyrius 6.7.9`. `cyrius deps --verify` and 6.7.6's
  `cyrius deps --locked` are both clean.
- `dist/akshara.cyr` changes only in its version line, and `dist/akshara.deps`
  is unchanged (`io`).

## [1.0.4] - 2026-11-09

A toolchain and dependency realign. No source change.

### Changed

- **Toolchain `6.6.2` → `6.6.20`.** Suite **2/2** unchanged; fmt and lint clean; the
  smoke binary runs on x86_64 and on aarch64 (under qemu). The x86_64 smoke binary
  grows from 122,656 to 188,048 B, nearly all of it unreachable stdlib code that DCE
  would drop (95,382 → 155,977 B unreachable).
- **`[deps.rosnet]` `1.1.2` → `1.1.3`**, the rosnet release built against 6.6.20. It
  brings tyche `1.0.1` → `1.1.1` transitively (also 6.6.20-built). akshara uses only
  `t_alloc` / `t_zero` from rosnet, so tyche 1.1.0's portable-log `rng_normal` change
  does not reach any akshara path. Vendored `lib/rosnet.cyr` and `lib/tyche.cyr`
  byte-match the tagged `dist/` bundles.
- **Vendored `lib/` re-synced** (`cyrius lib sync --full`). It byte-matches the 6.6.20
  snapshot, 113 files; five are new (`alloc_cx`, `boxed`, `hashseed`, `poison`,
  `tls_hostid`). This clears the `./lib/ shadows version-pinned` warning (12 bundled
  libs) that 1.0.3 carried because it bumped the pin without a full re-sync.
  `lib/agnosys.cyr`, a scaffold-era file in no snapshot from 6.6.2 on, is left as-is.
- **`cyrius.lock` regenerated from scratch.** One line each for rosnet 1.1.3
  (`5aa4ad7`) and tyche 1.1.1 (`68bb5db`), 116 file hashes, and the toolchain line
  the 6.6.20 format records (`cyrius 6.6.20`). `cyrius deps --verify` is clean.
- **`dist/akshara.deps` lists only what the bundle needs.** From 6.6.18,
  `cyrius distlib` writes only the stdlib leaves a bundle provably needs
  (compile-verified) and emits them as `include` lines at the top of the bundle:
  10 → 1 leaf (`io`). `cyrius distlib --check` passes. The five names it reports
  undefined (`puts`, `_putc`, `secure_read_file`, `read_stdin`, `file_seek`) are
  the consumer-supplied I/O hooks, unresolved by design.

  **Consumers:** a build that used `string`, `fmt`, `alloc`, `vec`, `str`,
  `syscalls`, `assert`, `bench` or `math` only because akshara's sidecar brought it
  in must now declare that leaf in its own `[deps] stdlib`. All four consumers
  (attn11, tarka, tentib, prajna) already declare all ten (checked in their
  manifests 2026-11-09), so none is affected.

## [1.0.3] - 2026-09-11

### Changed

- **Toolchain `6.5.27` → `6.6.2`**, and rosnet `1.1.1` → `1.1.2`. No source change,
  and no format drift despite the pin sitting below the 6.5.28 cyrfmt change.
  Zero compiler rejections, zero fail-open sites, no `callptr`.


## [1.0.2] - 2026-08-18

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
