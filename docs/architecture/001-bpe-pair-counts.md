# 001 — bpe_learn's pair counts are incremental, and only the live region is meaningful

Since 1.1.2, `bpe_learn` counts the adjacent pairs of the byte-level store
**once**, then keeps `g_paircnt` exact by updating it during each merge pass
(`_bpe_merge16`) instead of zeroing and recounting it per merge as 1.1.1 did.
Three facts follow that are not visible from any single function:

1. **Only `[0, V)²` is meaningful, with stride `BPE_VMAX`.** `g_paircnt` is a
   768 × 768 table allocated once and reused across learns. At the start of a
   learn only the `B × B` block is zeroed and counted; each newly minted id's
   row and column are zeroed just before the pass that fills them. Cells
   outside the live region hold whatever an earlier learn left. The argmax
   reads only the live region, so stale cells never reach a merge choice — but
   anything else reading `g_paircnt` must respect the same bound. After a learn
   the live region holds the exact pair counts of the final store (the suite's
   `_paircnt_exact` checks this).

2. **A hit's left pair is shared with the previous hit when they are
   adjacent.** For a hit at old positions `(rd, rd+1)` the pass removes
   `(old[rd-1], ba)` — unless the previous hit ended at `rd-1`, because then
   that pair, `(bb, ba)`, was already removed as the previous hit's right pair.
   Dropping that condition double-counts on any stream like `abab`
   (mutation M1 in the 1.1.2 audit).

3. **Reading `old[rd-1]` from the buffer being rewritten is safe.** The pass
   writes nothing before the first hit (the prefix is unchanged), and after the
   first hit `wr < rd`, so position `rd-1` has not been rewritten yet.

A further consequence of reusing fixed buffers, true since 1.0: span slots in
`g_tok_bytes` are written only up to each id's `g_tok_len`, and only for ids
below `g_bpe_base + g_bpe_nmerges`; the bytes past a span and the slots past the
table keep an earlier table's contents. They are not part of the table.
