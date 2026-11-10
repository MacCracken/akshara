# Benchmarks

`tests/akshara.bcyr`, run as CI runs it (`CYRIUS_DCE=1 cyrius bench
tests/akshara.bcyr`) and recorded by `./scripts/bench-history.sh` into
[`bench-history.csv`](../bench-history.csv): each value is the median of three
runs. Every corpus comes from the seeded generators in `tests/corpora.cyr`
(pseudo-English text from a 148-word list), so each row times the same bytes on
every run. Compare rows only when they were recorded in the same session on the
same host.

## 1.1.1 → 1.1.2 (2026-11-09, one session, one x86_64 host, cyrius 6.7.18)

Every 1.1.2 output is bit-identical to 1.1.1's
([ADR 0001](adr/0001-optimizations-preserve-tokenizer-output.md)).

| Benchmark | 1.1.1 | 1.1.2 | Speedup |
|---|---:|---:|---:|
| `corpus_set` 1 MiB, u16 store | 10.25 ms | 3.75 ms | 2.7× |
| `bpe_learn` K = 64, 64 KiB | 114.2 ms | 11.60 ms | 9.8× |
| `bpe_learn` K = 512, 64 KiB | 738.5 ms | 127.3 ms | 5.8× |
| `bpe_learn` K = 512, 1 MiB | 7,925 ms | 773.7 ms | 10.2× |
| `tok_encode` 4 KiB prompt → i64, 512 merges | 11.71 ms | 1.694 ms | 6.9× |
| `tok_encode` 1 MiB corpus rebuild → u16, 512 merges | 3,053 ms | 519.2 ms | 5.9× |
| `gd_ld` sequential sweep, 1 Mi tokens u16 | 7.28 ms | 4.04 ms | 1.8× |
| `bpe_build_spans`, 512 merges | 13.53 µs | 14.41 µs | 0.94× |

What moved each row:

- **`bpe_learn`** — the pair counts are counted once and then updated by the
  pairs each merge destroys and creates
  ([architecture 001](architecture/001-bpe-pair-counts.md)), instead of zeroing
  a V × V block and recounting the whole store before every merge. The rewrite
  pass and the count run on u16 directly rather than through two accessor calls
  per token, write nothing before the first hit, and the argmax hoists its
  constants and row pointer. What remains per merge is one rewrite pass over the
  store (O(n)) and one argmax over the live V × V counts (O(V²)). Measured by
  running the argmax twice: it is 68% of the K = 512, 64 KiB learn and 10% of
  the 1 MiB one — the O(n) pass dominates from there up.
- **`tok_encode`** — one specialized pass per width (u8 / u16 / i64) instead of
  a width dispatch on every load and store; each pass scans read-only to the
  first hit.
- **`corpus_set`** / **`gd_ld`** — the width dispatch is inlined.
- **`bpe_build_spans`** is 6.5% slower: it now validates the whole table before
  writing a span (security, not speed; it runs once per checkpoint or shard
  install).

## 1.1.2 → 1.1.3: rosnet 1.2.2 → 1.4.1 (2026-11-09, interleaved A/B)

No akshara source changed, and the two rosnet functions akshara calls
(`t_alloc`, `t_zero`) are byte-identical in both releases. The benchmark was
built twice in the same session, against the rosnet 1.2.2 and 1.4.1 bundles,
and the two binaries were run alternately three times each (median):

| Benchmark | rosnet 1.2.2 | rosnet 1.4.1 | Change |
|---|---:|---:|---:|
| `corpus_set` 1 MiB, u16 store | 3.565 ms | 3.709 ms | +4.0% |
| `bpe_learn` K = 64, 64 KiB | 11.69 ms | 11.70 ms | +0.1% |
| `bpe_learn` K = 512, 64 KiB | 125.6 ms | 132.0 ms | +5.0% |
| `bpe_learn` K = 512, 1 MiB | 772.7 ms | 779.7 ms | +0.9% |
| `tok_encode` 4 KiB prompt → i64 | 1.755 ms | 1.796 ms | +2.3% |
| `tok_encode` 1 MiB rebuild → u16 | 518.1 ms | 516.4 ms | −0.3% |
| `gd_ld` sweep, 1 Mi tokens u16 | 4.060 ms | 3.770 ms | −7.1% |
| `bpe_build_spans`, 512 merges | 14.4 µs | 14.3 µs | −0.7% |

Both builds execute the same functions from the same source; only where the
larger rosnet bundle places code in the binary differs. Read the ±5–7% rows as
code-placement effects, not as more or less work. The `bench-history.csv`
1.1.3 row was recorded 35 minutes after the 1.1.2 row, so it also carries
machine-state drift; this table is the like-for-like comparison.

## 1.1.3 → 1.2.0: the argmax row cache (2026-11-09, interleaved A/B)

Every 1.2.0 output is bit-identical to 1.1.1's. The benchmark was built from the
1.1.3 and the 1.2.0 source in the same session, and the two binaries were run
alternately five times each (median):

| Benchmark | 1.1.3 | 1.2.0 | Change |
|---|---:|---:|---:|
| `corpus_set` 1 MiB, u16 store | 3.926 ms | 3.820 ms | −2.7% |
| `bpe_learn` K = 64, 64 KiB | 12.17 ms | 12.32 ms | +1.2% |
| `bpe_learn` K = 512, 64 KiB | 137.4 ms | 48.30 ms | **2.84×** |
| `bpe_learn` K = 512, 1 MiB | 815.3 ms | 753.5 ms | −7.6% |
| `tok_encode` 4 KiB prompt → i64 | 1.882 ms | 1.894 ms | +0.6% |
| `tok_encode` 1 MiB rebuild → u16 | 538.0 ms | 542.3 ms | +0.8% |
| `gd_ld` sweep, 1 Mi tokens u16 | 3.969 ms | 4.249 ms | +7.1% |
| `bpe_build_spans`, 512 merges | 14.9 µs | 15.0 µs | +0.7% |

- **`bpe_learn`** — the argmax reads a cache of each row's maximum
  ([architecture 002](architecture/002-bpe-argmax-row-cache.md)) instead of
  scanning the live V × V counts before every merge. Over the K = 512, 64 KiB
  learn it reads about 1.1 M cells where 1.1.3 read 52 M. Measured by running
  it twice, the argmax is now about 1.3 ms of that learn. The rest is the O(n)
  merge pass, unchanged, which is why the 1 MiB row gains only 7.6% and the
  64 MiB cap is essentially unchanged.
- **K = 64** has the fewest merges for the cache to save on, and its +1.2% is
  within the placement noise of this binary. A driver that builds only this
  learn, alternating the two sources four times, measures 12.32 → 11.93 ms
  (−3.2%), and 12.27 → 12.02 ms when the code is shifted by a padding function.
- **Every other row** runs code that did not change, so its movement is code
  placement.

## 1.2.0 → 1.3.0: the rank-ordered encoder (2026-11-09, interleaved A/B)

Every 1.3.0 output is bit-identical to 1.1.1's. The benchmark was built from the
1.2.0 and the 1.3.0 source in the same session, and the two binaries were run
alternately seven times each (median). Other jobs were running on the host
throughout; the two builds alternated, so both saw the same load.

| Benchmark | 1.2.0 | 1.3.0 | Change |
|---|---:|---:|---:|
| `corpus_set` 1 MiB, u16 store | 4.174 ms | 4.161 ms | −0.3% |
| `bpe_learn` K = 64, 64 KiB | 13.21 ms | 13.16 ms | −0.4% |
| `bpe_learn` K = 512, 64 KiB | 52.92 ms | 51.93 ms | −1.9% |
| `bpe_learn` K = 512, 1 MiB | 809.9 ms | 808.3 ms | −0.2% |
| `tok_encode` 4 KiB prompt → i64 | 2.023 ms | 0.378 ms | **5.35×** |
| `tok_encode` 1 MiB rebuild → u16 | 582.0 ms | 581.3 ms | −0.1% |
| `gd_ld` sweep, 1 Mi tokens u16 | 4.533 ms | 4.538 ms | +0.1% |
| `bpe_build_spans`, 512 merges | 16.1 µs | 16.0 µs | −0.5% |

- **`tok_encode` 4 KiB prompt** — the input fits `ENC_MAX`, so it runs the
  rank-ordered encoder ([architecture 003](architecture/003-rank-ordered-encoder.md)):
  one pass to file every pair under its rank, then a constant number of list
  and bucket updates per merge performed, instead of 512 passes over the
  prompt. The seven 1.3.0 runs fall between 367 and 384 µs.
- **The 1 MiB rebuild** is longer than `ENC_MAX` and runs the same passes as
  1.2.0.
- **Every other row** runs unchanged code. These two builds place it so that
  every row is within ±2%, tighter than the 1.2.0 comparison.

## 1.3.0 → 1.3.1: the stream vocabulary check (2026-11-09, interleaved A/B)

1.3.1 checks each chunk `stream_tok` reads against the installed vocabulary.
Before this release no benchmark streamed, so a driver was built from the 1.3.0
and 1.3.1 sources: a full `gd_ld` sweep of a 4 Mi-token u16 shard (a memfd) at
the real 64 Ki-token chunk size, alternating five times (median).

| Benchmark | 1.3.0 | 1.3.1 | Change |
|---|---:|---:|---:|
| `stream_tok` sweep, 4 Mi tokens u16 | 48.1 ms | 54.4 ms | +13% |

The difference is the chunk scan: about 1.5 ns a token, paid once per 64 Ki
tokens read. A training step reads ctx + 1 tokens per window, so the check does
not show there. A 4-way unrolled scan measured 52.9 ms, saving about a sixth of
the cost, and was not kept. From 1.3.1 the benchmark file has a
`stream_tok sweep 1Mi u16` row, the streamed twin of the `gd_ld` sweep, so
`bench-history.csv` tracks this path.

## At the 64 MiB loader cap (1.1.2 only, one run)

| Operation | 1.1.2 |
|---|---:|
| `bpe_learn` K = 512 | 45.7 s |
| `tok_encode` rebuild → u16, 512 merges | 31.5 s |

Not measured on 1.1.1 at this size; scaling its 1 MiB rows linearly gives about
8½ minutes and 3¼ minutes. Both operations are still one O(n) pass per merge —
see the audit's *not changed* list.
