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

## At the 64 MiB loader cap (1.1.2 only, one run)

| Operation | 1.1.2 |
|---|---:|
| `bpe_learn` K = 512 | 45.7 s |
| `tok_encode` rebuild → u16, 512 merges | 31.5 s |

Not measured on 1.1.1 at this size; scaling its 1 MiB rows linearly gives about
8½ minutes and 3¼ minutes. Both operations are still one O(n) pass per merge —
see the audit's *not changed* list.
