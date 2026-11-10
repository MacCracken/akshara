# akshara — Roadmap

> Remaining work only; shipped work is in [`CHANGELOG.md`](../../CHANGELOG.md),
> current state in [`state.md`](state.md).
>
> **1.x** takes only changes that keep every output on a valid input
> bit-identical ([ADR 0001](../adr/0001-optimizations-preserve-tokenizer-output.md))
> and every frozen signature ([`docs/api.md`](../api.md)) intact. Each item's gate
> is the suite's golden digests and 1.1.1 oracle, the fuzz harness, the aarch64
> CI job, and before/after rows in `bench-history.csv`. Anything that needs more
> than that is in the **2.0.0 backlog**.

## 1.x.x — standing work

- **Toolchain and dependency realigns** as patch releases whenever the cyrius
  pin moves. The pin, `lib/` (`cyrius lib sync --full`) and `cyrius.lock`
  (regenerated from scratch) change in one commit.
- **CI action pins.** Keep third-party actions on their current releases,
  pinned by commit SHA (the tag's `^{}` commit from `git ls-remote`). akshara
  moved softprops/action-gh-release to v3.0.4 ahead of rosnet and tyche, which
  still pin v3.0.3, and they follow when they are next touched.

## 2.0.0 backlog — needs a semantics or surface change

- **Remove the per-merge O(n) pass.** At the 64 MiB loader cap with K = 512,
  `bpe_learn` takes 45.7 s and the corpus rebuild 31.5 s, because each merge
  rewrites the whole store. A per-pair position index removes that pass but
  costs several bytes per token (gigabytes at the cap), breaking the
  fixed-allocation budget consumers size for. The rebuild half already exists
  for short inputs: 1.3.0's rank-ordered encoder
  ([architecture 003](../architecture/003-rank-ordered-encoder.md)) is
  O(n + K). Running it on the corpus needs its 12 bytes of scratch per byte
  (768 MiB at the cap) instead of the fixed `ENC_MAX` block.
- **`stream_tok` returns an error instead of exiting.** It exits on a bad
  index, a failed seek, a short read or (since 1.3.1) an id outside the
  vocabulary, because its callers (the training samplers) have no error path
  today; changing that changes their contract.
- **A tokenizer handle instead of process-global `g_*` state.** Today one corpus
  and one tokenizer exist per process, and consumers read and write the globals
  directly and serialize them into checkpoints.
- **Larger caps.** `BPE_MAX_MERGES` 512, `BPE_VMAX` 768 and `BPE_MAX_TOKLEN` 64
  are frozen constants that consumers size fixed buffers from (attn11's
  checkpoint loader keeps a 768-slot stack table) and that checkpoints and
  shards encode.
- **Tokenization semantics.** Any change to merge selection, pair counting or
  encode order, plus pre-tokenization (word splitting), Unicode normalization
  and special tokens. Each changes which ids a text encodes to, and so
  invalidates every trained checkpoint.
