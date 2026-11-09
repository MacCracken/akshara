# Sources

Citations for every algorithm and constant akshara (and its test harness)
relies on.

## Library (`src/tok.cyr`)

- **Byte-pair encoding as a compression scheme** — P. Gage, "A New Algorithm
  for Data Compression", *The C Users Journal* 12(2), February 1994. The
  original replace-the-most-frequent-pair loop.
- **BPE for subword tokenization** — R. Sennrich, B. Haddow, A. Birch, "Neural
  Machine Translation of Rare Words with Subword Units", ACL 2016,
  [arXiv:1508.07909](https://arxiv.org/abs/1508.07909). The learn / encode
  procedure `bpe_learn` and `tok_encode` implement: merges learned greedily by
  pair frequency, applied in learned order at encode time. akshara's variant
  works on bytes (no pre-tokenization, no end-of-word marker), counts
  overlapping pairs, breaks frequency ties by smallest (left, right) id, stops
  below a minimum frequency of 2, and caps a token's expansion at 64 bytes.
- **Incremental pair counts** (1.1.2, `_bpe_merge16`) — the standard
  optimization of BPE training: after a merge, only the pairs that touch a
  replaced position change, so the counts are updated by those deltas instead
  of being recounted. The bookkeeping rule akshara uses (which old pairs a hit
  destroys, which new pairs it creates, and how adjacent hits share a pair) is
  derived in the function's comment and in
  [`architecture/001-bpe-pair-counts.md`](architecture/001-bpe-pair-counts.md).
- **Row-maximum cache for the argmax** (1.2.0, `_bpe_rowscan` / `_bpe_rowfix`)
  — akshara's own, not taken from a published source. Production BPE trainers
  usually keep a max-heap of pairs and discard stale entries when they surface
  (the Hugging Face `tokenizers` trainer does). akshara keeps one cached
  maximum per row of its fixed V × V count table instead, which needs no
  per-pair entries and keeps the frozen tie-break a plain ascending scan. Why
  the cache stays exact under the merge pass is in
  [`architecture/002-bpe-argmax-row-cache.md`](architecture/002-bpe-argmax-row-cache.md).

## Test harness (`tests/`)

- **FNV-1a** — G. Fowler, L. C. Noll, K.-P. Vo, the Fowler–Noll–Vo hash
  (draft-eastlake-fnv); the 64-bit offset basis 0xcbf29ce484222325 and prime
  1099511628211 (`tests/golden.cyr`, folded over 64-bit words rather than
  bytes).
- **64-bit LCG** — the linear congruential generator D. E. Knuth gives for
  MMIX, multiplier 6364136223846793005 and increment 1442695040888963407,
  modulus 2^64 (`tests/corpora.cyr`), with draws taken from the high bits
  (an LCG's low bits have short periods; Knuth, *The Art of Computer
  Programming*, Vol. 2, §3.2.1).
