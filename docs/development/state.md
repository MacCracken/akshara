# akshara — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

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

Consumers are on the 1.x line: attn11, tentib, prajna and tarka all declare
`[deps.akshara] tag = "1.0.2"` (checked 2026-11-09).

## Toolchain

- **Cyrius pin**: `6.7.18` (in `cyrius.cyml [package].cyrius`)

**Pin bumped to `6.7.18` 2026-11-09** (6.6.2 → 6.6.20 → 6.7.9 → 6.7.18 the same day). `cyrius lib sync --full` re-vendored the whole version-matched stdlib snapshot (113 files) and `cyrius.lock` was regenerated from scratch; suite re-verified green at the new pin, smoke green on x86_64 and aarch64 (qemu), and a 2,192-value tokenizer dump is byte-identical to 6.6.20 and 6.7.9 on both. From 6.7.18 a moved pin is refused until `lib/` is re-synced, so the pin, `lib/` and `cyrius.lock` must land in one commit. Run the suite with bare `cyrius test` (`cyrius tests` is deprecated from 6.7.6, removed in 6.7.12).

## Source

- `src/tok.cyr` (`[lib]`, 666 lines) — the corpus + tokenizer layer: adaptive
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

- `tests/akshara.tcyr` — **122 assertions**, ~6 s on x86_64 (~45 s under qemu
  aarch64): 15 golden digests recorded from 1.1.1 (`tests/golden.cyr`), the
  1.1.1 oracle (`tests/reference.cyr`: 432 learns, 111 encodes), and contract /
  hardening tests for every public fn and error code. `cyrius coverage`: 21/21
  of `tok.cyr`'s public fns referenced.
- `tests/akshara.fcyr` — seeded property fuzzer: 1,500 rounds × 8 properties,
  ~10 s. Mutation run (1.1.2): suite and fuzzer each kill 13/13 non-equivalent
  injected bugs.
- `tests/akshara.bcyr` — 8 hot-path benchmarks; history in `bench-history.csv`
  (`scripts/bench-history.sh`), summary in [`docs/benchmarks.md`](../benchmarks.md).
- Shared helpers: `tests/corpora.cyr`, `tests/hooks.cyr`, `tests/props.cyr`.
- CI: fmt, lint (hard gate), vet, doc --check, deps --verify / --locked, suite,
  bench, fuzz, distlib drift, and an aarch64 (cross + qemu) job.

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

## Performance (1.1.3, x86_64, median of 3)

`bpe_learn` K = 512: 134 ms / 64 KiB, 798 ms / 1 MiB (45.7 s at the 64 MiB cap,
measured on 1.1.2). `tok_encode`: 1.82 ms per 4 KiB prompt (512 merges); corpus
rebuild 526 ms / 1 MiB (31.5 s at the cap, 1.1.2). See
[`docs/benchmarks.md`](../benchmarks.md).

## Next

See [`roadmap.md`](roadmap.md).
