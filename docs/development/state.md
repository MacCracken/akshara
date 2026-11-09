# akshara — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.1.0** (2026-11-09) — moves onto the cyrius 6.7 series (6.7.9), no source
change; every tokenizer value is byte-identical to 1.0.4 @ 6.6.20 (see CHANGELOG).
The 1.x surface is the one frozen at 1.0.0. Prior: **1.0.4** (2026-11-09) —
toolchain + dependency realign to cyrius 6.6.20.

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

- **Cyrius pin**: `6.7.9` (in `cyrius.cyml [package].cyrius`)

**Pin bumped to `6.7.9` 2026-11-09** (from 6.6.20, itself bumped from 6.6.2 the same day). `cyrius lib sync --full` re-vendored the whole version-matched stdlib snapshot (113 files) and `cyrius.lock` was regenerated from scratch; suite re-verified green at the new pin, smoke green on x86_64 and aarch64 (qemu), and a 2,192-value tokenizer dump is byte-identical to 6.6.20 on both. Run the suite with bare `cyrius test` (`cyrius tests` is deprecated from 6.7.6, removed in 6.7.12).

## Source

- `src/tok.cyr` (`[lib]`) — the corpus + tokenizer layer: adaptive byte vocab,
  opt-in BPE (Sennrich et al. 2016: learn / encode / decode, pure i64,
  bit-reproducible x86_64 / aarch64), the width-generic packed token store
  (u8 / u16 / i64) + streaming read path, and the checkpoint-facing tokenizer
  state. Byte-identical to attn11's proven code.
- `dist/akshara.cyr` — the `cyrius distlib` bundle consumers import. Leaves the
  consumer symbols `puts` / `_putc` / `secure_read_file` / `read_stdin` /
  `file_seek` unresolved (rosnet→tyche pattern).
  Its sidecar `dist/akshara.deps` lists only the compile-verified stdlib leaves
  the bundle needs (`io`, emitted as an `include` at the bundle top); consumers
  declare any other leaf they use.
- `src/main.cyr` — pure-path smoke (byte vocab + BPE + encode round-trip),
  excluded from the bundle.

## Tests

- `tests/akshara.tcyr` — primary suite (smoke + math; passes on `cyrius test`)
- `tests/akshara.bcyr` — benchmark stub (no-op)
- `tests/akshara.fcyr` — fuzz stub
- Validated transitively by attn11's finite-difference suite (1060/1060 green at
  extraction — the byte-identity gate). akshara's own richer suite is a
  follow-on (v1.0 criterion).

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench, math
- [`rosnet`](https://github.com/MacCracken/rosnet) 1.2.0 — `t_alloc` / `t_zero`
  (the typed bump-allocator the vocab / packed store / BPE tables use)

Transitive (via rosnet; locked in `cyrius.lock`):

- [`tyche`](https://github.com/MacCracken/tyche) 1.2.0 — PRNG for rosnet's
  `t_randn`; no akshara path uses it

## Consumers

- [attn11](https://github.com/MacCracken/attn11) — re-pointed to `[deps.akshara]`
  (was `include "src/tok.cyr"`); full suite green on the bundle.
- [tarka](https://github.com/MacCracken/tarka) — intended shared consumer (one
  tokenizer behind SFT/diffusion and RL/reasoning).

## Next

See [`roadmap.md`](roadmap.md).
