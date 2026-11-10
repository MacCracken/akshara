# `stream_tok` returns out-of-vocabulary ids unchecked, so a corrupt shard reaches the model as an embedding index

**Status:** ✅ **RESOLVED** in akshara 1.3.1 (see *Resolution* at the end).
**Placement:** 1.3.1.
**Discovered:** 2026-11-09, by attn11's 1.15.2 P(-1) audit (three of its four independent reviewers
found the attn11 side; this is the upstream half).
**Severity:** Medium for akshara as a library: defense in depth. In attn11 ≤ 1.15.1 it was
**HIGH**: an id ≥ V from a `--stream-corpus` shard reached `embed_bwd` as an **out-of-bounds
gradient write**. attn11 1.15.2 closes that downstream with a range check at every consumer.
Any other consumer that streams shards and indexes by id is still exposed.
**Affects:** akshara 1.1.3 (attn11's pin) through 1.2.0 and the current working tree. The
`stream_tok` body is identical across all three. Verified with the repro below at cyrius 6.7.18.

## Summary

The consumer opens and validates a shard and installs its tokenizer (attn11: `src/stream.cyr`
`shard_open`, as `docs/api.md` describes). That validation covers the header and the
tokenizer blob, but not the token **payload**: shards are meant to be GB-scale, so scanning them
at open would cost a full read. `stream_tok` then returns whatever u8/u16 the shard stores:

```
return _tw_ld(g_stream_buf, i - g_stream_lo, g_data_w);
```

Nothing on the read path compares the id with the installed vocabulary. With width 1 any id up
to 255 passes, and with width 2 any id up to 65,535. A corrupt or hostile shard therefore hands
the consumer an index past its embedding table. In attn11 1.15.1 a byte-level shard with id 255
(V = 41) trained to completion with exit 0 and produced garbage samples. A width-2 shard with
id 0xFFFF writes ~16.7 MB past the gradient buffer.

`stream_tok` is the single function every streamed id passes through, and it already **fails
loud** on a bad index, an unset cache, a failed seek and a short read. A vocabulary check fits
its existing contract.

## Reproduction

[`repros/2026-11-09-stream-tok-unchecked-ids.cyr`](repros/2026-11-09-stream-tok-unchecked-ids.cyr)
installs a byte-level tokenizer over `"hello"` (V = 4) and streams a 7-id memfd shard holding
`0 1 2 3 200 3 255`. Its exit code is the number of out-of-vocabulary ids `stream_tok` returned.

```sh
cyrius build docs/development/issues/repros/2026-11-09-stream-tok-unchecked-ids.cyr /tmp/aktok
/tmp/aktok; echo "exit=$?"
```

On 1.2.0 (this tree, cyrius 6.7.18):

```
stream_tok(4) = 200  >= V = 4  (out of vocabulary)
stream_tok(6) = 255  >= V = 4  (out of vocabulary)
out-of-vocabulary ids returned: 2
exit=2
```

Once fixed, the run should end in the akshara fail-loud message (exit 1 from `_ak_fatal`)
before any bad id is returned.

## Ask

After each chunk read in `stream_tok` (the `g_stream_lo` / `g_stream_clen` refill), scan the
freshly cached ids. Any id ≥ V, with V = `g_corpus_vlen + g_bpe_nmerges` (K is 0 at byte
level), should end in `_ak_fatal("akshara: shard token id out of vocabulary -- corrupt shard\n")`.

- **Cost:** one compare per token, once per refill. That is O(chunk), next to a `read` of the
  same bytes, and the per-token hot path (a cache hit) is untouched.
- **Contract:** it keeps today's fail-loud contract, so it is independent of the roadmap item
  "`stream_tok` returns an error instead of exiting".
- **Where V comes from:** it relies on the documented rule that the consumer installs the
  shard's tokenizer before streaming, which attn11's `shard_open` does. If that coupling is
  unwanted, a consumer-set `g_stream_vmax` (0 = unchecked) would do the same job explicitly.
- **Tests:** a suite case modelled on `test_stream` with a poisoned id, and a fuzz mode that
  flips payload bytes, which today's harness never reaches.

attn11 will keep its own check (`tok_ld`) either way: it also guards the in-memory paths and the
eval readers.

## Resolution (akshara 1.3.1)

Done as asked. After each chunk read, `stream_tok` checks the whole chunk
(`_tw_all_below`) against `_ak_vocab()`: `g_bpe_base + g_bpe_nmerges` under BPE
and `g_corpus_vlen` at byte level. That is the same bound `tok_emit` enforces,
and for attn11's `shard_open`, which sets both bases to `Vb` and K = 0 at byte
level, it equals the issue's `g_corpus_vlen + g_bpe_nmerges`. A failing chunk
ends in `_ak_fatal("akshara: shard token id out of vocabulary -- corrupt shard\n")`
before any of its ids is returned, including ids of that chunk that are in
range. At width 8 a negative id fails too. The cache-hit path is unchanged.

The repro now ends in that message with exit 1, on x86_64 and on aarch64
(qemu). Tests:
- the suite's `stream_tok: ids outside the vocabulary fail loud` group, which
  runs each failing read in a forked child and checks its exit code and stderr;
- fuzz property P9, which streams the learned store from a shard and then a
  copy with one id overwritten;
- 11 injected bugs in the check, each caught by both.

Consumers must install the shard's tokenizer before the first read, as
`shard_open` already does. With no tokenizer installed every chunk fails.
