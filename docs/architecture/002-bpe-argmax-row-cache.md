# 002 — bpe_learn's argmax reads a row cache that is revalidated after each pass

Since 1.2.0, `bpe_learn` no longer scans the live V × V block of `g_paircnt`
before every merge. For each live row `a` it caches the row's best **eligible**
count, `_bpe_rmax[a]`, and the smallest column holding it, `_bpe_rarg[a]` (0 and
−1 when no eligible cell is positive). A cell is eligible when the merged span
would fit `BPE_MAX_TOKLEN`. Four facts follow that no single function shows:

1. **The two-level pick is the 1.1.1 scan.** 1.1.1 scanned rows ascending and
   columns ascending, kept a cell only on a strict `>`, and tested the span
   inside that branch. That picks the eligible cell with the highest count, then
   the smallest row, then the smallest column. Taking each row's maximum at its
   smallest column (`_bpe_rowscan`), then the row maxima in ascending row order
   with a strict `>`, picks the same cell. Both levels must stay strict: `>=` at
   either one moves ties to the largest column or row (mutations R3, R4).

2. **A merge pass only lowers old cells and only raises new ones.** In the pass
   that mints `nid`, every decrement lands in an old column: the pairs it
   removes, `(old[rd-1], ba)`, `(ba, bb)` and `(bb, old[rd+2])`, hold only ids
   below `nid`. Every increment lands in row `nid` or column `nid`.
   `_bpe_rowfix` relies on exactly this:
   - an old row whose cached cell still holds the cached count keeps that
     maximum over the old columns, and keeps its column: every cell left of it
     was below the maximum and could only fall;
   - so only the new cell `(r, nid)` can beat it, and a tie there loses,
     because `nid` is the largest column (R2);
   - an old row whose cached cell fell is rescanned over `[0, nid]` (R1, R6),
     and row `nid` is always scanned (R5).

   A change that lets a pass raise an old cell (a different counting rule, say)
   would leave rows stale without any error. Such a change must rescan instead.

3. **The new id's `g_tok_len` is set before `_bpe_rowfix`.** The span test for
   column `nid` reads it (R11).

4. **The cache is exact after every learn,** over rows and columns
   `[0, B + merges)`. `tests/props.cyr` `_rowcache_exact` checks it against an
   independent right-to-left scan after each of the suite's 432 oracle learns
   and after every fuzz round's learn. As with `g_paircnt`
   ([001](001-bpe-pair-counts.md)), slots past the live region keep an earlier
   learn's values.

**Why revalidate instead of maintaining the cache inside the pass.** The first
1.2.0 design updated the cache at every count change in `_bpe_merge16`. Those
per-hit calls cost more than the smaller argmax saved at small K: the K = 64,
64 KiB learn got about 5% slower. Revalidating once per pass costs one O(V)
sweep plus the rescans and leaves the merge pass byte-identical to 1.1.3. On the
K = 512, 64 KiB benchmark corpus that is 3.2 rescanned rows a merge besides the
new one: about 1.1 M cell reads over the whole learn, where 1.1.3 read 52 M.

## Mutation coverage (1.2.0)

Each mutant was injected into `src/tok.cyr` alone, and the suite and the fuzz
harness were rebuilt and run against it. Every one is killed by both.

| Mutant | Bug injected |
|---|---|
| R1 | a row whose maximum cell fell is never rescanned (a stale row) |
| R2 | a tie at the new column takes the row over (a wrong column on a tie) |
| R3 | `_bpe_rowscan` breaks ties toward the largest column |
| R4 | the row pick breaks ties toward the largest row |
| R5 | the new row is never scanned |
| R6 | a fallen row is rescanned without the new column |
| R7 | the span test is dropped for the new column |
| R8 | the span test is dropped in `_bpe_rowscan` |
| R9 | `_bpe_rowfix` skips the last old row |
| R10 | the initial scan misses the last base column |
| R11 | `_bpe_rowfix` runs before the new id's span length is set |
| R12 | revalidation compares the wrong cell |

R11 first survived the fuzz harness. The 1.1.1 oracle learn ran first and
left the right span lengths in the shared `g_tok_len`, so the library learn
read correct values that it had not written. `tests/reference.cyr`
`_ref_poison` now overwrites them before the library learn runs.
