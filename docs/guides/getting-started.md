# Getting started with akshara

## Build and check

```sh
cyrius deps                              # resolve stdlib + rosnet into lib/
cyrius build src/main.cyr build/akshara  # the pure-path smoke
./build/akshara
cyrius test                              # [build].test + tests/*.tcyr
CYRIUS_DCE=1 cyrius bench tests/akshara.bcyr
```

## Layout

- `src/tok.cyr` — the library (`[lib].modules`): corpus + vocab, packed store,
  streaming read, BPE, encode / decode. `cyrius distlib` bundles it into
  `dist/akshara.cyr`, which is what consumers import.
- `src/main.cyr` — a smoke entry that stubs the consumer hooks and runs vocab →
  BPE → encode round-trip. Not part of the bundle.
- `src/test.cyr` — the `[build].test` entry referenced by `cyrius.cyml`.
- `tests/akshara.tcyr` — the suite: golden digests (`tests/golden.cyr`), the
  1.1.1 oracle (`tests/reference.cyr`), contract tests. Shared helpers:
  `tests/corpora.cyr` (seeded corpus generators), `tests/hooks.cyr`
  (programmable consumer hooks), `tests/props.cyr` (invariant checks).
- `tests/akshara.fcyr` — seeded property fuzzer.
- `tests/akshara.bcyr` — benchmarks; `scripts/bench-history.sh` records them.

## Using it from a consumer

```cyrius
g_data_w = 2;                         # u16 store (needed for BPE ids)
var V = corpus_set(text, len);        # byte vocab + packed store; V < 0 is an error
var Vb = bpe_learn(256);              # optional: learn up to 256 merges
var ids = alloc(plen * 8);            # room for one i64 id per prompt byte
var n = tok_encode(prompt, plen, ids, 8);
```

Error codes for every function are in [`../api.md`](../api.md).

## Changing the library

1. Edit `src/tok.cyr`; keep every output bit-identical
   ([ADR 0001](../adr/0001-optimizations-preserve-tokenizer-output.md)).
2. Add contract tests to `tests/akshara.tcyr` (and a property to the fuzzer if
   the change is a kernel).
3. Run `cyrius test`, the fuzzer, and the benchmarks; `cyrius distlib`.
4. Bump `VERSION` and add a CHANGELOG entry before tagging.

See [`../adr/template.md`](../adr/template.md) when a non-trivial design choice
deserves an ADR.
