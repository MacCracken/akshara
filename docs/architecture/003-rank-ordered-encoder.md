# 003 — tok_encode merges by rank for short inputs, and falls back to the passes

Since 1.3.0, `tok_encode` has two paths that produce the same ids. An input of
2 to `ENC_MAX` (8,192) bytes under BPE runs the **rank-ordered encoder**
(`_enc_ranked`), which is O(n + K). Everything else runs the **per-merge
passes**, one O(n) pass per learned merge, exactly as in 1.2.0. The corpus
rebuild is longer than `ENC_MAX` and so always runs the passes.

## Why merging by rank gives the passes' output

Call merge `m` **well-founded** when both of its ids are below `B + m`, the
ids that exist before it. In a table where every merge is well-founded:

1. Pass `m` turns every non-overlapping occurrence of `(ba, bb)`, left to
   right, into `B + m`. No occurrence is left: a copied `ba` is never followed
   by `bb` in the output.
2. The only new adjacencies a pass creates hold `B + m`, and every merge that
   uses `B + m` has a rank above `m`. So when pass `m` starts, no pair with a
   rank below `m` is present. Pass `m` is "merge every occurrence of the
   lowest-rank pair present".
3. Within one rank, occurrences of `(a, b)` with `a ≠ b` cannot overlap, so the
   order they are merged in does not matter. Occurrences of `(a, a)` overlap
   in a run of `a`s; the pass pairs a run from its first `a`, left to right,
   and the encoder does the same for the whole run at once.

So the encoder processes ranks in ascending order and merges every occurrence
of each before moving on, and gets the passes' result without a pass per merge.
Mutants E1 (a run paired from the wrong node) and E2 (`(a, a)` taken in
arbitrary order) break exactly point 3.

## Data structures

- A doubly linked list of nodes, one per input byte. A merge keeps the left
  node and unlinks the right one, so node 0 is always the head and the output
  is the list read in order.
- Each node sits in the **bucket** of its current pair's rank, or in none.
  Buckets are doubly linked, so a node moves between them in O(1). A merge
  changes the pairs of exactly three nodes: the merged node, the node before
  it, and the unlinked node, which leaves its bucket (E7, E8, E9). A rank's
  bucket is drained until it is empty; merges never file a node under the
  rank being drained (point 2).
- A **rank hash** maps a pair to the first merge with it: 2,048 open-addressed
  slots for at most 512 entries, rebuilt from `g_merges` on every call (O(K)).
  A duplicate merge keeps the earlier rank (E4). In the passes the later copy
  never fires either, since pass `m` leaves no occurrence and none can be
  created.
- The scratch is one block of about 113 KiB (`12 × ENC_MAX` bytes of u16 node
  fields, the bucket heads, and the hash), allocated on the first BPE encode
  and reused for the process lifetime. The prompt path still allocates nothing
  per call.

## When it falls back to the passes

`_enc_ranked` returns -2, and `tok_encode` runs the 1.2.0 passes instead, when:

- **a merge is not well-founded.** A consumer can install such a table by
  writing `g_merges` without calling `bpe_build_spans`, which would refuse it.
  Point 2 then fails. Example from the suite: a base of 2 under a 3-byte
  corpus makes `c` (id 2) look like merge 0's output, merge 0 = `(a, 2)` uses
  it, and `aac` is `(a, 2)` after one pass, but merging by rank would merge it
  again (E3);
- **`g_bpe_base` is outside [1, 256]**;
- **a `g_char2id` entry is past 1023.** `corpus_set` writes only ids below 256,
  but the table is a public global. A node's token is a u16 and the pair key is
  `a * 1024 + b` (E13);
- **the scratch allocation is refused.** Nothing fails; the input runs the
  passes. The tests do not exercise this branch, since the allocator cannot be
  made to refuse 113 KiB on demand.

The u8 refusal is the same on both paths: -1 when a merge minting an id past 255
applies to the text (E6 on the rank path, M6b and M6c on the passes).

## Other differences that do not change any output

The rank path reads all of `text` before it writes `out`, so `text` and `out`
may overlap. On `-1` it leaves `out` untouched, where the passes leave a
partial encoding. Neither is part of the contract.

## Mutation coverage (1.3.0)

Each mutant was injected into `src/tok.cyr` alone, and the suite and the fuzz
harness were rebuilt and run against it.

| Mutant | Bug injected |
|---|---|
| E1 | a run of `a`s is paired from the filed node, not its first `a` |
| E2 | `(a, a)` occurrences are taken as filed, like `(a, b)` |
| E3 | the well-founded check is dropped |
| E4 | a duplicate pair keeps the last rank, not the first |
| E5 | the hash lookup gives up at the first collision |
| E6 | the rank path's u8 refusal lets id 256 through |
| E7 | the node before a join is not re-filed |
| E8 | the joined node is not re-filed |
| E9 | the unlinked node stays in its bucket |
| E10 | unlinking a bucket's head empties the bucket |
| E11 | an input of `ENC_MAX + 1` bytes takes the rank path |
| E12 | the last node's link is not terminated |
| E13 | the `g_char2id` range check is dropped |
| E14 | only the first `(a, b)` occurrence of a rank merges |
| E15 | the initial filing misses the last pair |
| E16 | ranks past 255 are truncated |
| E17 | the bucket heads are not reset per call |
| E18 | the nodes' bucket marks are not reset per call |

The suite and the fuzz harness each kill all 18 mutants. Six of them (E7, E9,
E11, E12, E17, E18) make the suite crash or hang (stopped by the run's timeout)
rather than fail an assertion, which still fails the run. The first run
found five gaps in the fuzz harness and one in the suite. The suite gap was the
passes' last-pair scan (M4b): the oracle encodes that used to catch it are
short, so they now take the rank path, and a test past `ENC_MAX` whose only hit
is the last pair now covers it. The fuzz gaps were closed this way:
- long probes now sometimes land exactly at the `ENC_MAX` boundary (E11);
- some long probes are the corpus repeated, so its merges fire through the
  passes at u8 (M6c);
- P7 gained two kinds of hostile state: a base that shrinks but stays in range,
  which produces tables that are not well-founded (E3), and an overwritten
  `g_char2id` entry (E13);
- every round whose vocab passes 257 ids encodes the corpus past `ENC_MAX`
  under the table cut to exactly 257 ids. Its last merge mints id 256, which is
  the u8 boundary on the passes (M6b).
