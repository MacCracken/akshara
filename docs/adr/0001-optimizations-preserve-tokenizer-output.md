# 0001 — Optimizations preserve every tokenizer output bit for bit

**Status**: Accepted
**Date**: 2026-11-09

## Context

akshara's outputs are persisted state, not just intermediate values. attn11
serializes the merge table into checkpoints and token shards, then re-encodes
the retained corpus bytes through it on resume and expects the result to equal
the store it trained on. A model's embedding rows are indexed by token id, so
a change in which pair a merge picks — or in which id a byte sequence encodes
to — silently mis-indexes a trained model. The 1.x API is frozen, and the
frozen part includes the meaning of every id.

The 1.1.2 sweep made `bpe_learn` 5–10× faster and `tok_encode` 6× faster by
restructuring the kernels: incremental pair counts instead of a per-merge
recount, width-specialized passes, writes starting at the first hit. Each such
change could, if wrong, change a merge choice that no single unit test notices.

## Decision

An optimization of a tokenizer kernel ships only if every output on every
valid input is identical to the previous release's — return values, vocab and
char2id tables, the packed store, the merge table, every span, and encode
output — on x86_64 and aarch64. The check is threefold:

- **Golden digests** (`tests/golden.cyr`): 15 fixed corpora whose every output
  is folded into an FNV-1a digest, recorded from the last release before the
  change (1.1.1) and asserted by the suite on both targets.
- **An in-process oracle** (`tests/reference.cyr`): the previous release's
  kernels kept verbatim under `ref_*` names; the suite compares a fixed matrix
  and the fuzz harness random corpora against them.
- **A full dump** at review time: every golden value printed and compared with
  `cmp(1)` across the old and new builds on both targets.

Invalid inputs are outside the rule: a hardening change may turn undefined
behaviour (an out-of-bounds write, a silently corrupt table) into a documented
error return.

## Consequences

- **Positive** — consumers can take any 1.x release without re-learning a
  tokenizer or invalidating a checkpoint; a kernel bug that changes an output
  fails the suite's golden or oracle group (in the 1.1.2 mutation run the suite
  and the fuzz harness each caught all 8 injected kernel bugs).
- **Negative** — the argmax tie-break, the overlap-counting rule and the greedy
  left-to-right replace are fixed forever in 1.x; an encoder that is faster but
  merges differently (for example by merge rank rather than in passes, if it
  ever disagreed) is a 2.0 change. `tests/reference.cyr` must never be "fixed".
- **Neutral** — each optimization release re-records nothing: the digests stay
  the 1.1.1 values until a deliberate 2.0 redefinition.

## Alternatives considered

- **Tolerance-class equivalence** (same compression ratio, different merges):
  rejected — ids are indices into trained weights, so "close" is wrong.
- **Re-recording digests per release**: rejected — it would pin each release to
  itself, not to the outputs consumers persisted.
