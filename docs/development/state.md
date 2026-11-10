# akshara — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.3.1** (2026-11-09) — hardening. `stream_tok` checks every chunk it reads
against the installed vocabulary and fails loud on an id outside it, closing
[the issue](issues/2026-11-09-stream-tok-unchecked-ids.md) attn11's 1.15.2 audit
filed. Consumers install the shard's tokenizer before streaming (attn11's
`shard_open` does). No signature changed; every output on a valid input is
identical to 1.1.1 on x86_64 and aarch64.

**1.3.0** (2026-11-09) — the 1.3.x roadmap. `tok_encode` merges by rank for
inputs of 2 to `ENC_MAX` bytes ([architecture 003](../architecture/003-rank-ordered-encoder.md)),
so a 4 KiB prompt encodes 5.35× faster; longer inputs, such as the corpus
rebuild, run the per-merge passes as before. Every output is identical to 1.1.1
on x86_64 and aarch64 (the full golden dump). No signature changed. This
completes the planned 1.x feature work; what is left is standing maintenance
and the 2.0.0 backlog.

**1.2.0** (2026-11-09) — the 1.2.x roadmap. `bpe_learn`'s argmax reads a
per-row maximum cache ([architecture 002](../architecture/002-bpe-argmax-row-cache.md)),
so the K = 512, 64 KiB learn is 2.84× faster. The fuzz harness also runs on
aarch64 in CI (500 rounds). Every output is identical to 1.1.1 on x86_64 and
aarch64 (the full golden dump). No signature changed.

**1.1.3** (2026-11-09) — rosnet `1.2.2` → `1.4.1`, no source change; every
output is identical to 1.1.1 on x86_64 and aarch64 (the full golden dump).

**1.1.2** (2026-11-09) — the first P(-1) hardening, optimization and repair
sweep ([`docs/audit/2026-11-09-audit.md`](../audit/2026-11-09-audit.md)): two
MEDIUM and ten LOW findings fixed, `bpe_learn` 5.8–10.2× and `tok_encode`
5.9–6.9× faster, every output on a valid input bit-identical to 1.1.1 on x86_64
and aarch64 ([ADR 0001](../adr/0001-optimizations-preserve-tokenizer-output.md)).
No signature changed; invalid arguments now get documented error codes. The 1.x
surface is the one frozen at 1.0.0. Prior: **1.1.1** (2026-11-09) — realign to
cyrius 6.7.18 / rosnet 1.2.2; **1.1.0** — onto the cyrius 6.7 series (6.7.9);
**1.0.4** — realign to cyrius 6.6.20.

**1.0.0** — **stable, API frozen** (cut 2026-07-05). A clean freeze with no
behavior change from 0.1.0: what the shipping consumers (attn11 — the origin,
1066-assertion suite — and tarka) had exercised unchanged since the 2026-06-22
extraction. One tokenizer, shared for real. Frozen 1.x surface in
[`docs/api.md`](../api.md): corpus + vocab, the width-generic packed store, opt-in
BPE, encode/decode. Caller-buffer / caller-guarantees; I/O stays in consumer hooks,
unresolved in the dist bundle by design. The tokenizer `g_*` state is
process-global (one corpus at a time) and freezes WITH the surface — consumer
checkpoints depend on it. Prior: **0.1.0** — extraction from attn11 (M1),
2026-06-22, the text → token-id data layer carved out of attn11's `src/train.cyr`
(via `src/tok.cyr`); third attn11 extraction after `rosnet` (tensors) and `tyche`
(PRNG).

Consumers are on the 1.x line (`[deps.akshara] tag`, checked 2026-11-09):
attn11 `1.1.3`, tentib `1.2.0`, tarka `1.1.2`, prajna `1.0.2`. Only attn11
streams shards; its `shard_open` installs the tokenizer before streaming, as
1.3.1's vocabulary check requires.

## Toolchain

- **Cyrius pin**: `6.7.18` (in `cyrius.cyml [package].cyrius`)

**Pin bumped to `6.7.18` 2026-11-09** (6.6.2 → 6.6.20 → 6.7.9 → 6.7.18 the same day). `cyrius lib sync --full` re-vendored the whole version-matched stdlib snapshot (113 files) and `cyrius.lock` was regenerated from scratch; suite re-verified green at the new pin, smoke green on x86_64 and aarch64 (qemu), and a 2,192-value tokenizer dump is byte-identical to 6.6.20 and 6.7.9 on both. From 6.7.18 a moved pin is refused until `lib/` is re-synced, so the pin, `lib/` and `cyrius.lock` must land in one commit. Run the suite with bare `cyrius test` (`cyrius tests` is deprecated from 6.7.6, removed in 6.7.12).

## Source

- `src/tok.cyr` (`[lib]`, 982 lines) — the corpus + tokenizer layer: adaptive
  byte vocab, opt-in BPE (Sennrich et al. 2016: learn / encode / decode, pure
  i64, bit-reproducible x86_64 / aarch64), the width-generic packed token store
  (u8 / u16 / i64) + streaming read path, and the checkpoint-facing tokenizer
  state. Outputs bit-identical to attn11's proven code (the 1.1.2 golden digests
  pin 1.1.1's).
- `dist/akshara.cyr` — the `cyrius distlib` bundle consumers import. Leaves four
  consumer hooks — `_putc` / `secure_read_file` / `read_stdin` / `file_seek` —
  unresolved (rosnet→tyche pattern); `puts` is no longer needed (1.1.2).
  Its sidecar `dist/akshara.deps` lists only the compile-verified stdlib leaves
  the bundle needs (`io`, emitted as an `include` at the bundle top); consumers
  declare any other leaf they use.
- `src/main.cyr` — pure-path smoke (byte vocab + BPE + encode round-trip),
  excluded from the bundle.

## Tests

- `tests/akshara.tcyr` — **161 assertions**, ~5 s on x86_64 (~40 s under qemu
  aarch64): 15 golden digests recorded from 1.1.1 (`tests/golden.cyr`), the
  1.1.1 oracle (`tests/reference.cyr`: 432 learns; encodes on both sides of
  `ENC_MAX`, so both encoder paths; the library learn runs over tables the
  oracle left poisoned), the argmax row-cache invariant after every oracle
  learn, the encoder paths' edge cases (runs under self-merges, the `ENC_MAX`
  boundary, duplicate and not well-founded tables), `stream_tok`'s vocabulary
  check (each failing read in a forked child, exit code and stderr checked),
  and contract / hardening tests for every public fn and error code. `cyrius coverage`: 21/21 of
  `tok.cyr`'s public fns referenced.
- `tests/akshara.fcyr` — seeded property fuzzer: 1,500 rounds × 9 properties,
  ~10 s; `-D AK_FUZZ_QUICK` builds the first 500 rounds for CI's aarch64 job
  (~31 s under qemu). Every 12th round's probe is longer than `ENC_MAX` (the
  passes); the rest take the rank-ordered encoder; every third round streams a
  shard (P9). Mutation runs: suite and fuzzer each kill 42/42 non-equivalent
  injected bugs (1.3.0; 18 aimed at the rank encoder, 12 at the row cache) and
  11/11 in `stream_tok`'s vocabulary check (1.3.1).
- `tests/akshara.bcyr` — 8 hot-path benchmarks; history in `bench-history.csv`
  (`scripts/bench-history.sh`), summary in [`docs/benchmarks.md`](../benchmarks.md).
- Shared helpers: `tests/corpora.cyr`, `tests/hooks.cyr`, `tests/props.cyr`.
- CI: fmt, lint (hard gate), vet, doc --check, deps --verify / --locked, suite,
  bench, fuzz, distlib drift, and an aarch64 (cross + qemu) job running the
  suite, the smoke and the 500-round fuzz.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench, math
- [`rosnet`](https://github.com/MacCracken/rosnet) 1.4.1 — `t_alloc` / `t_zero`
  (the typed bump-allocator the vocab / packed store / BPE tables use)

Transitive (via rosnet; locked in `cyrius.lock`):

- [`tyche`](https://github.com/MacCracken/tyche) 1.2.2 — PRNG for rosnet's
  `t_randn`; no akshara path uses it

## Consumers

- [attn11](https://github.com/MacCracken/attn11) — re-pointed to `[deps.akshara]`
  (was `include "src/tok.cyr"`); full suite green on the bundle.
- [tarka](https://github.com/MacCracken/tarka) — intended shared consumer (one
  tokenizer behind SFT/diffusion and RL/reasoning).

## Performance (1.3.0, x86_64, median of 7, host under other load)

`bpe_learn` K = 512: 52 ms / 64 KiB, 808 ms / 1 MiB. At the 64 MiB cap it is
45.7 s (measured on 1.1.2; the O(n) merge pass dominates there). `tok_encode`:
0.38 ms per 4 KiB prompt (512 merges, the rank-ordered encoder); corpus
rebuild 581 ms / 1 MiB (the passes; 31.5 s at the cap, 1.1.2). See
[`docs/benchmarks.md`](../benchmarks.md).

## Next

See [`roadmap.md`](roadmap.md).
