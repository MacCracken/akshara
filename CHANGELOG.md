# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Changed

- **CI: softprops/action-gh-release v3.0.3 → v3.0.4** in `release.yml`,
  pinned by commit SHA to `8a0cd651e5b2365bd640854a3fa5dfc70e4875f3` (the
  tag's commit, checked with `git ls-remote`). v3.0.4 is a maintenance release:
  13 dependency bumps and the release commit, with `action.yml` byte-identical
  (same inputs, `node24` runtime). The 1.1.2 audit had kept v3.0.3 for parity
  with rosnet and tyche. akshara now moves ahead of them, and they still pin
  v3.0.3. CI only, so no library change and no version bump.

## [1.3.1] - 2026-11-09

A hardening release that closes
[the `stream_tok` out-of-vocabulary issue](docs/development/issues/2026-11-09-stream-tok-unchecked-ids.md)
found by attn11's 1.15.2 audit. No signature changed, and every output on a
valid input is bit-identical to 1.1.1 on x86_64 and aarch64.

### Fixed

- **`stream_tok` checks every chunk against the installed vocabulary**
  (MEDIUM; defense in depth). Through 1.3.0 it returned whatever u8 / u16 / i64
  a shard stored. A consumer cannot afford to scan a GB-scale payload at open,
  so a corrupt or hostile shard handed it ids past its embedding table. In
  attn11 ≤ 1.15.1 that was an out-of-bounds gradient write; attn11 1.15.2
  checks downstream.
  - After each chunk read, the whole chunk is compared with the model
    vocabulary: `g_bpe_base + g_bpe_nmerges` under BPE, `g_corpus_vlen` at byte
    level. That is the bound `tok_emit` already enforces, and for attn11's
    `shard_open` it equals the issue's proposal. At width 8 a negative id fails
    too.
  - A chunk holding a bad id ends in
    `akshara: shard token id out of vocabulary -- corrupt shard` (stderr,
    exit 1), the existing fail-loud contract, before any of its ids is
    returned.
  - The cache-hit path is unchanged.
  - The issue's repro now exits 1 with that message instead of returning two
    bad ids.
- **Consumers must install the shard's tokenizer before the first read.**
  `docs/api.md` now says so. attn11's `shard_open`, the only consumer that
  streams, already does. With no tokenizer installed, every chunk fails.

### Added

- **`stream_tok: ids outside the vocabulary fail loud`** in the suite. It
  covers the issue's repro, the V − 1 / V boundary at u8, u16 and i64, a bad id
  as a chunk's last id at each width, a negative i64 id, a bad id in a later
  chunk (earlier chunks stream, any read of its chunk fails, a later chunk
  streams), and no tokenizer installed. Each
  read that must fail runs in a forked child, and the test checks both its exit
  code and its stderr (`tests/props.cyr` `_stream_child`). `test_stream`
  installs the 768-id vocabulary its ids need. The suite is 161 assertions.
- **Fuzz property P9**, every third round. A token shard, with a random
  header length and chunk cap, streams back identically. The shard is the
  learned store at u16 or i64, or in half the rounds the corpus at byte level
  as u8.
  With one id overwritten (in range, at V, just past it, or negative at i64), a
  forked child fails loud exactly when the chunk it reads holds an id outside
  the vocabulary.
- **Mutation run.** 11 mutants of the check, including a dropped check, a V
  that passes, a skipped first or last id, a wrong BPE vocabulary and the
  previous window's length. The suite and the fuzz harness each kill all 11.
  The first run missed three: the fuzz harness never streamed a u8 shard, and
  the suite never put the bad id at a chunk's end.
- **A `stream_tok sweep 1Mi u16` benchmark row**, the in-memory `gd_ld`
  sweep's streamed twin at the real 64 Ki-token chunk size.

### Performance

- A full sweep of a 4 Mi-token u16 shard through `gd_ld` takes 48.1 → 54.4 ms
  (+13%, about 1.5 ns a token; interleaved A/B of the 1.3.0 and 1.3.1 sources,
  median of 5). That is the chunk scan, paid once per 64 Ki-token read; a
  training step reads ctx + 1 tokens per window, so it is invisible there. A
  4-way unrolled scan saved only a sixth of it and was not kept.

## [1.3.0] - 2026-11-09

The 1.3.x roadmap: `tok_encode` is 5.35× faster on a 4 KiB prompt. No signature
changed, and every output is bit-identical to 1.1.1 on x86_64 and aarch64
([ADR 0001](docs/adr/0001-optimizations-preserve-tokenizer-output.md)).

### Changed

- **`tok_encode` merges by rank for inputs of 2 to `ENC_MAX` (8,192) bytes**
  ([architecture 003](docs/architecture/003-rank-ordered-encoder.md)).
  Through 1.2.0 every encode ran one pass over the input per learned merge.
  When every merge is well-founded (it uses only ids that exist before it),
  merging every occurrence of the lowest-rank pair present, rank by rank,
  gives the same ids: a pass creates only pairs that hold its new id, and
  every merge using that id ranks higher. The new path files each node of a
  linked list over the input under its pair's rank and drains the ranks in
  order, pairing each run of `a`s under an `(a, a)` merge left to right. That
  is O(n + K) instead of O(n × K).
  - Longer inputs, which includes the corpus rebuild, still run the passes.
  - So do tables the rank path cannot reproduce: a merge that is not
    well-founded (possible only when a table is installed without
    `bpe_build_spans`), a base outside [1, 256], or a `g_char2id` id past 1023.
  - If its scratch allocation is refused, it uses the passes rather than
    failing.
  - The u8 refusal is the same on both paths.
- **The scratch is about 113 KiB**, allocated on the first BPE encode and kept
  for the process lifetime, so the prompt path still allocates nothing per
  call. It holds u16 node and bucket links, plus a rank hash that is rebuilt
  from `g_merges` on every call.
- **Benchmarks** (interleaved A/B: the 1.2.0 and 1.3.0 sources built in one
  session, alternating processes, median of 7):
  - `tok_encode` 4 KiB prompt → i64, 512 merges: 2.023 → 0.378 ms (5.35×);
  - the 1 MiB corpus rebuild runs the passes and is unchanged (−0.1%);
  - every other row runs unchanged code and is within ±2%.

### Added

- **Tests for both encoder paths.** The suite's oracle matrix now also encodes
  8,191, 8,192, 8,193 and 12,000 bytes. A new group covers:
  - runs under self-merges at every width;
  - the lengths 1, 2, `ENC_MAX` − 1, `ENC_MAX`, `ENC_MAX` + 1 and
    `ENC_MAX` + 3000;
  - the u8 refusal past `ENC_MAX`, and a pass whose only hit is the last pair;
  - a duplicate-merge table;
  - a table that is not well-founded, which the rank path would encode
    differently;
  - a `g_char2id` id past 1023.

  The suite is 143 assertions.
- **`tests/corpora.cyr` `cp_runs`**, a generator of runs of `a`s that makes
  BPE learn self-merges. It is the suite's runs corpus and the fuzz harness's
  seventh generator.
- **The fuzz harness covers both paths.**
  - Every 12th round's probe is longer than `ENC_MAX`, sometimes exactly at the
    boundary, and some of those repeat the corpus so its merges fire through
    the passes.
  - P7 compares `tok_encode` with the oracle under each hostile table. It has
    two new kinds of hostile state: a base that shrinks but stays in range,
    and an overwritten `g_char2id` entry.
  - Every round whose vocab passes 257 ids checks the u8 boundary on the
    passes, under the table cut to exactly 257 ids.
  - 1,500 rounds take about 10 s on x86_64. The 500-round quick build takes
    31 s under qemu.
- **Mutation run.** There are 43 mutants: the 18 new ones aimed at the rank
  encoder (listed in architecture 003) and the 25 from 1.2.0. The suite and
  the fuzz harness each kill all 42 that are not equivalent; M4 is equivalent,
  as before.

### Fixed

- **The fuzz harness wrote past a buffer** (test code only, since 1.1.2). P6's
  u8 check and P8's invalid-width call encoded the probe (up to 512 tokens at
  8 bytes) into the buffer sized for the corpus, which can be 24 bytes. It
  overran the bump arena harmlessly, since nothing still in use came after it.
  The probe now has its own buffer.

### Verified

- Suite 143/143 on x86_64 and on aarch64 (qemu). Fuzz holds at 1,500 rounds on
  both, and at 500 with `-D AK_FUZZ_QUICK`. The full 461,435-value golden dump
  is `cmp`-identical to 1.1.1 on both targets.

## [1.2.0] - 2026-11-09

The 1.2.x roadmap: `bpe_learn` is 2.8× faster on the K = 512, 64 KiB
benchmark, and the fuzz harness now runs on aarch64 in CI. No signature
changed, and every output is bit-identical to 1.1.1 on x86_64 and aarch64
([ADR 0001](docs/adr/0001-optimizations-preserve-tokenizer-output.md)).

### Changed

- **`bpe_learn`'s argmax reads a per-row cache**
  ([architecture 002](docs/architecture/002-bpe-argmax-row-cache.md)). Through
  1.1.3 it rescanned the whole live V × V pair-count table before every merge,
  which was 68% of a K = 512 learn on 64 KiB. Now each row's best eligible count
  and the smallest column holding it are cached. The pick takes the row maxima
  in ascending row order with strict `>`, which selects exactly the cell 1.1.1's
  row-major scan selected. After each merge pass, `_bpe_rowfix` revalidates the
  cache in O(V) and rescans only the rows whose maximum cell the pass lowered.
  On the benchmark corpus that is 3.2 rows a merge, or about 1.1 M cell reads
  over the learn where 1.1.3 read 52 M. Measured by running it twice, the
  argmax now costs about 1.3 ms of the 45 ms learn, where it cost 88 of 129 ms
  in 1.1.2. The merge pass is byte-identical to 1.1.3's.
- The cache is a 12 KiB block allocated on the first learn. If the allocator
  refuses it, `bpe_learn` returns -3 before changing anything, as it does for
  its other allocations.
- **Benchmarks** (interleaved A/B: the 1.1.3 and 1.2.0 sources built in one
  session, alternating processes, median of 5):
  - `bpe_learn` K = 512 on 64 KiB: 137.4 → 48.3 ms (2.84×);
  - `bpe_learn` K = 512 on 1 MiB: 815 → 753 ms (−7.6%). The O(n) merge pass
    dominates from there up, so the 64 MiB cap is essentially unchanged;
  - `bpe_learn` K = 64 on 64 KiB: +1.2% in the full benchmark binary. A driver
    that times only this learn measures 12.32 → 11.93 ms (−3.2%);
  - the other rows run unchanged code, and move between −2.7% and +7.1% from
    code placement.

  Details are in [`docs/benchmarks.md`](docs/benchmarks.md).

### Added

- **The fuzz harness runs on aarch64 in CI.** Building it with
  `-D AK_FUZZ_QUICK` runs the first 500 of its 1,500 seeded rounds (about 19 s
  under qemu); the aarch64 job cross-builds and runs that build. The harness
  now prints the round count it ran.
- **A row-cache invariant check.** `tests/props.cyr` `_rowcache_exact`
  compares the cache with an independent right-to-left scan of the counts. The
  suite runs it after each of its 432 oracle learns and its six invariant
  learns (123 assertions), and the fuzz harness's P4 runs it after every
  round's learn.
- **The oracle comparison poisons the state the oracle leaves behind.**
  `ref_learn_matches` ran the 1.1.1 oracle learn first, which left the correct
  span lengths in the shared `g_tok_len`. A library learn that read a new id's
  length before writing it therefore passed the fuzz harness (mutation R11).
  `_ref_poison` now overwrites every span length and the row cache with values
  that change a merge if read, before the library learn runs.
- **Mutation run.** All 25 mutants: the 12 new ones aimed at the cache and the
  13 earlier ones that still apply. The suite kills 24/24 non-equivalent ones,
  and so does the fuzz harness. M4 survives both, as in 1.1.2, because it is
  equivalent.

### Verified

- Suite 123/123 on x86_64 and on aarch64 (qemu). Fuzz holds at 1,500 rounds on
  both, and at 500 with `-D AK_FUZZ_QUICK`. The full 461,435-value golden dump
  is `cmp`-identical to 1.1.1 on both targets.

## [1.1.3] - 2026-11-09

A dependency update. No source change.

### Changed

- **`[deps.rosnet]` `1.2.2` → `1.4.1`**, the latest rosnet. The toolchain stays
  6.7.18 and tyche stays 1.2.2. akshara calls only `t_alloc` and `t_zero`,
  which are byte-identical between the two releases, as is their failure path.
  rosnet's 1.3.0–1.4.1 changes are GPU attention and AVX2 paths for its linear
  and BLAS-1 kernels, none of which akshara calls. Vendored `lib/rosnet.cyr`
  byte-matches the 1.4.1 bundle.
- **`cyrius.lock` regenerated from scratch**: rosnet 1.4.1 (`af2bb1a`), tyche
  1.2.2 (`cdb74e2`), 115 file hashes. `cyrius deps --verify` and `--locked` are
  clean.
- **rosnet's bundle now begins with `include "lib/simd.cyr"`** (a new leaf in
  its sidecar). akshara vendors the whole 6.7.18 snapshot, so `lib/simd.cyr` was
  already present and is unchanged. akshara's own bundle changes only in its
  version line; its sidecar is still `io`, and it still leaves the same four
  hooks unresolved.
- **Every output is unchanged.** The suite passes 122/122 on x86_64 and on
  aarch64 (qemu), and the fuzz harness passes 1,500 rounds. The full
  461,435-value golden dump is `cmp`-identical to 1.1.1 on both targets.
- **Benchmarks** (interleaved A/B in one session: the same akshara source built
  against the rosnet 1.2.2 and 1.4.1 bundles, alternating processes, median of
  3):
  - slower: `corpus_set` +4.0%, `bpe_learn` K = 512 on 64 KiB +5.0%,
    `tok_encode` 4 KiB prompt +2.3%;
  - faster: the `gd_ld` sweep −7.1%;
  - the other four rows within ±1%.

  Both builds execute the same functions from the same source, so the
  differences come from where the larger bundle places code in the binary. The
  smoke binary grows from 204,112 to 225,800 B (without DCE).

## [1.1.2] - 2026-11-09

The P(-1) hardening, optimization and repair sweep, recorded in
[`docs/audit/2026-11-09-audit.md`](docs/audit/2026-11-09-audit.md) —
akshara's first standalone audit. No public signature changed, and every output
on a valid input is bit-identical to 1.1.1 on x86_64 and aarch64: 15 golden
cases (461,435 printed values) are `cmp`-identical across the two releases on
both targets, and the suite now asserts their digests
([ADR 0001](docs/adr/0001-optimizations-preserve-tokenizer-output.md)).

### Fixed

- **Store and output widths are validated** (MEDIUM). `corpus_set` accepted any
  `g_data_w`, sized the store `len * w`, and wrote 8-byte tokens for any width
  but 1 and 2 — w = 4 wrote 64 bytes past a 16-byte corpus's store into the next
  heap block. `tok_encode`'s `ow` had the same flaw, and `ow = 1` under a vocab
  past 256 ids truncated ids without an error. Now `corpus_set` → -6,
  `bpe_learn` → -5 and `tok_encode` → -1 for a width other than 1, 2 or 8, and
  `tok_encode` at `ow = 1` → -1 exactly when a merge minting an id past 255
  applies to the text; any text whose encoding stays below 256 ids encodes as
  before.
- **A second `bpe_learn` no longer corrupts the tokenizer** (MEDIUM). Over a
  store that was already BPE it restarted from the byte base and minted ids that
  already meant something else in the store (a learn of 40 then 20 merges left a
  47-id table over a store holding ids up to 66, and re-encoding no longer
  reproduced the store). It now returns -4 and changes nothing; learn again
  after a fresh `corpus_set`.
- **`bpe_build_spans` validates the table before writing** (LOW). A forward or
  negative merge id, a base outside [1, 256], a count past 512, or a span past
  64 bytes read and wrote outside the span tables. It now returns -1 with every
  span unchanged. attn11 validates tables itself, so it was never exposed.
- **A refused `corpus_set` keeps the loaded corpus** (LOW). It used to replace
  the corpus before allocating, so a refusal (`len = 0`, a refused allocation)
  left `g_data = 0` and empty tables. It now validates (`len` in
  [1, 2^60 − 1], so `len * width` cannot wrap), allocates, and only then
  replaces anything; the codes are unchanged (-5).
- **`tok_emit` refuses ids outside the table** (LOW) — it printed a stale span
  from an earlier tokenizer, or read past the span table. -1, nothing emitted.
- **`stream_tok` fails loud on a bad index or an unset cache** (LOW) instead of
  returning a stale token or header bytes, and retries a read interrupted by a
  signal instead of exiting.
- **`tok_encode` with no tokenizer loaded** returns -1 instead of reading address
  0; **`corpus_load_file`** rejects a hook that reports more than the cap (-2);
  **`build_corpus`** checks its buffer allocation (-4).
- **Fatal messages go to stderr** with an "akshara:" prefix (they went to stdout
  through the consumer's `puts`, saying "attn11:"). The bundle no longer needs
  the consumer to define `puts`: it leaves four hooks unresolved, not five.

### Changed — faster, same bits

Same session, median of three runs ([`docs/benchmarks.md`](docs/benchmarks.md)):

- **`bpe_learn`** 9.8× (K = 64, 64 KiB: 114.2 → 11.6 ms), 5.8× (K = 512,
  64 KiB: 738.5 → 127.3 ms), 10.2× (K = 512, 1 MiB: 7.93 → 0.77 s). The pair
  counts are counted once and updated by exactly the pairs each merge destroys
  and creates, instead of being zeroed and recounted before every merge
  ([`docs/architecture/001-bpe-pair-counts.md`](docs/architecture/001-bpe-pair-counts.md));
  the pass works on u16 directly and writes nothing before the first hit.
- **`tok_encode`** 6.9× (4 KiB prompt: 11.71 → 1.69 ms), 5.9× (1 MiB corpus
  rebuild: 3.05 → 0.52 s): one specialized pass per width.
- **`corpus_set`** 2.7× (1 MiB: 10.25 → 3.75 ms) and **`gd_ld`** 1.8× (7.28 →
  4.04 ms per 1 Mi-token sweep): the width dispatch is inlined.
- `bpe_build_spans` is 6.5% slower (13.5 → 14.4 µs) from the new validation.

### Changed

- **Tests: 2 → 122 assertions**, every public function covered (`cyrius
  coverage`: 0/21 → 21/21). `tests/golden.cyr` digests 15 fixed corpora against
  1.1.1's outputs; `tests/reference.cyr` keeps the 1.1.1 kernels verbatim as an
  oracle (432 learns, 111 encodes); contract and hardening tests for every error
  code. Test entries exit through `sys_exit` (a literal `syscall(60, …)` is not
  `exit` on aarch64).
- **The fuzz harness is real**: 1,500 seeded rounds × 8 properties (oracle
  equality, table well-foundedness, decode identity, exact pair counts, the
  re-encode round-trip, hostile tables, invalid arguments), ~10 s. Mutation
  testing: the suite and the harness each kill all 13 non-equivalent injected
  bugs (8 kernel, 5 guard).
- **Benchmarks**: 8 hot-path benchmarks (`tests/akshara.bcyr`, were a no-op);
  `scripts/bench-history.sh` and `bench-history.csv` with the before and after
  rows.
- **CI**: fmt, lint (any warning fails, `src/` and `tests/`), `vet`,
  `doc --check`, `deps --verify` / `--locked`, benchmarks, fuzz, and a `distlib`
  drift check; an aarch64 job runs the suite and the smoke under qemu; a docs
  job checks required files and version consistency. Read-only token by
  default, write only on the release job; actions pinned by commit SHA
  (checkout v7.0.1, action-gh-release v3.0.3); `persist-credentials: false`;
  tag filter `X.Y.Z*`. The release ships the bundle, a source archive and
  `SHA256SUMS` (it shipped only the smoke binary).
- `.gitattributes` (`lib/** -text`): `cyrius.lock` records `lib/`'s bytes, so a
  CRLF checkout would fail `deps --verify`.

### Documentation

- **`docs/api.md` rewritten**: it documented a `g_V` akshara does not have and
  omitted the streaming globals, several tables, the constants and every error
  code. Now every public fn and global and every return code.
- `SECURITY.md` rewritten for 1.x and this audit (it described an unaudited
  0.1.0). New: `docs/sources.md`, ADR 0001, architecture note 001,
  `docs/benchmarks.md`, the audit. Every public fn has a doc comment
  (`cyrius doc --check` clean; `cyrius audit` docs complete).
- `src/tok.cyr`'s header no longer calls itself attn11's; `CLAUDE.md` and
  `CONTRIBUTING.md` state `var buf[N]` for both scopes (module scope is N i64
  slots); the README pins the current tag; the roadmap and getting-started
  guide replace their scaffold templates; `cyrius.cyml` no longer lists the
  stdlib's `file_read` as a consumer hook.

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
