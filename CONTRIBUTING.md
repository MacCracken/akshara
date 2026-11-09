# Contributing to akshara

## Development

1. Install the Cyrius toolchain at the version pinned in `cyrius.cyml`
   (`[package].cyrius`). The pin is the single source of truth — never hardcode
   a version elsewhere.
2. `cyrius deps` — resolve the stdlib + `rosnet` deps into `lib/`.
3. `cyrius build src/main.cyr build/akshara && ./build/akshara` — the pure-path
   smoke (adaptive byte vocab + BPE learn + encode round-trip).
4. `cyrius test` — runs the `[build].test` entry (`src/test.cyr`) plus
   `tests/akshara.tcyr`: golden digests, the 1.1.1 oracle, every public
   function's contract.
5. `CYRIUS_DCE=1 cyrius build tests/akshara.fcyr build/akshara-fuzz && ./build/akshara-fuzz`
   — the seeded property fuzzer (1,500 rounds). Add `-D AK_FUZZ_QUICK` for the
   500-round build CI runs on aarch64.
6. `CYRIUS_DCE=1 cyrius bench tests/akshara.bcyr` — benchmarks;
   `./scripts/bench-history.sh` appends medians to `bench-history.csv`.
7. `cyrius distlib` — regenerate the consumer bundle `dist/akshara.cyr` from
   `[lib].modules` (`src/tok.cyr`). Do this after any change to the lib; CI
   fails on a stale bundle.

CI also runs fmt, lint (any warning fails), `vet`, `doc --check`, and the suite
on aarch64 under qemu. See [`CLAUDE.md`](CLAUDE.md) for the full development
loop and [`docs/development/state.md`](docs/development/state.md) for the live
surface (versions, sizes, consumers).

## Outputs are the contract

akshara is a tokenizer whose outputs are persisted: consumers store the merge
table in checkpoints and shards and index trained weights by token id. Every
byte it ingests must come back unchanged (**encode → decode identity**), and
every output on a valid input must stay **bit-identical** across releases and
across x86_64 / aarch64 (pure i64 arithmetic, no float, no platform-dependent
ordering) — [ADR 0001](docs/adr/0001-optimizations-preserve-tokenizer-output.md).

A change to `corpus_set`, `bpe_learn`, `tok_encode`, `bpe_build_spans` or the
packed / streaming store must keep the suite's golden digests (recorded from
1.1.1) and its oracle comparisons (`tests/reference.cyr`, the 1.1.1 kernels
verbatim) passing. Never edit the digests or the oracle to make a change pass.

## Invariants you must not violate

- **Well-founded merges by construction.** `bpe_learn` mints merge ids in order,
  so every merge references only ids that already exist (`< base + m`). This
  rules out forward/cyclic references and is what lets `bpe_build_spans` decode
  as a flat table instead of recursing. Do not reorder the merge scan or mint
  ids out of order.
- **Bounded expansion.** A token's byte expansion is capped at
  `BPE_MAX_TOKLEN` (64); merges that would exceed it are skipped. Keep that
  guard — it is the anti-bomb bound.
- **Exact pair counts.** `bpe_learn` keeps `g_paircnt` incrementally; see
  [`docs/architecture/001-bpe-pair-counts.md`](docs/architecture/001-bpe-pair-counts.md)
  before touching `_bpe_merge16`.
- **No per-call heap growth on the hot paths.** The BPE tables (merges, spans,
  encode scratch, pair counts) are allocated once at cap size and reused;
  `tok_encode`, `stream_tok`, `gd_ld` and `tok_emit` allocate nothing. The
  one-shot setup calls (`corpus_set`, the loaders, `bpe_learn`'s widen) do
  allocate, once per corpus.
- **Consumer-supplied IO stays unresolved.** The loader / streaming / emit paths
  call `_putc` / `secure_read_file` / `read_stdin` / `file_seek`, which the
  consumer provides (attn11: `tensor.cyr` / `fileio.cyr`). Keep these out of the
  `[lib]` bundle; `src/main.cyr` and `tests/hooks.cyr` stub them so akshara
  builds standalone.

## Cyrius rules

- `var buf[N]` is a contract: inside a function it is **N bytes**; at module
  scope it is N i64 slots (N × 8 bytes). Token stores are
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
  attn11, tarka, tentib and prajna, so flag it loudly.
- Do not commit/push or use `gh` — the maintainer handles git operations.

## License

GPL-3.0-only.
