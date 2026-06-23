# Contributing to akshara

## Development

1. Install the Cyrius toolchain at the version pinned in `cyrius.cyml`
   (`[package].cyrius`). The pin is the single source of truth — never hardcode
   a version elsewhere.
2. `cyrius deps` — resolve the stdlib + `rosnet` deps into `lib/`.
3. `cyrius distlib` — regenerate the consumer bundle `dist/akshara.cyr` from
   `[lib].modules` (`src/tok.cyr`). Do this after any change to the lib surface.
4. `cyrius build src/main.cyr build/akshara && ./build/akshara` — the pure-path
   smoke (adaptive byte vocab + BPE learn + encode round-trip).
5. `cyrius test` — runs the `[build].test` entry (`src/test.cyr`) plus
   `tests/akshara.tcyr`.
6. `cyrius build tests/akshara.fcyr build/fuzz && ./build/fuzz` — fuzz harness.
7. `cyrius build tests/akshara.bcyr build/bench && ./build/bench` — benchmarks.

See [`CLAUDE.md`](CLAUDE.md) for the full development loop and
[`docs/development/state.md`](docs/development/state.md) for the live surface
(versions, module sizes, consumers, dep gaps).

## Round-trips are the contract

akshara is a tokenizer: every byte it ingests must come back unchanged. The
correctness contract is **encode → decode round-trip identity** and
**bit-reproducibility** of the learned BPE merge table across `x86_64` /
`aarch64` (pure i64 arithmetic, no float, no platform-dependent ordering). A
change to `corpus_set`, `bpe_learn`, `tok_encode`, or the packed/streaming store
must preserve both. The smoke in `src/main.cyr` pins the simplest case
(re-encoding `"abracadabra"` reproduces the stored corpus length); richer cases
belong in `tests/akshara.tcyr`. Until akshara grows its own deep suite, it is
also validated **transitively** by attn11's full finite-difference suite (the
byte-identity gate recorded in `CHANGELOG.md` 0.1.0) — do not break that.

## Invariants you must not violate

- **Well-founded merges by construction.** `bpe_learn` mints merge ids in order,
  so every merge references only ids that already exist (`< base + m`). This
  rules out forward/cyclic references and is what lets `bpe_build_spans` decode
  as a flat table instead of recursing. Do not reorder the merge scan or mint
  ids out of order.
- **Bounded expansion.** A token's byte expansion is capped at
  `BPE_MAX_TOKLEN` (64); merges that would exceed it are skipped. Keep that
  guard — it is the anti-bomb bound.
- **Fixed, cap-sized allocations.** Vocab, packed store, merge tables, and the
  encode scratch are allocated once at cap size and reused; the tokenizer loop
  allocates nothing. Do not introduce per-call or per-merge heap growth.
- **Consumer-supplied IO stays unresolved.** The loader / streaming / emit paths
  reference `puts` / `_putc` / `secure_read_file` / `read_stdin` / `file_seek`,
  which the consumer provides (attn11: `tensor.cyr` / `fileio.cyr`). Keep these
  out of the `[lib]` bundle; `src/main.cyr` stubs them only so akshara builds
  standalone.

## Cyrius rules

- `var buf[N]` is a contract: **N bytes**, not N entries. Token stores are
  width-packed (u8 / u16 / i64) via the `g_data_w` accessors — respect them.
- akshara is integer-only (token ids); there is no float math here. If you ever
  add it, remember Cyrius has no float type — an `f64` is its bit pattern in an
  `i64`; use the `f64_*` builtins, never `+`/`*` on float values.
- Build with `cyrius build`, never raw `cat file | cycc` — the manifest
  auto-resolves deps and prepends includes.

## Process

- One change at a time. Never bundle unrelated changes in a single PR.
- Test after every change; round-trip after every tokenizer-touching change;
  benchmark after every perf-touching change.
- Performance claims must include numbers — `before → after` with the bench name.
- Breaking changes get a `Breaking` section in [`CHANGELOG.md`](CHANGELOG.md)
  with a migration paragraph. akshara is a shared lib — a surface change breaks
  attn11 and tarka, so flag it loudly.
- Do not commit/push or use `gh` — the maintainer handles git operations.

## License

GPL-3.0-only.
