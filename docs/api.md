# akshara — Public API (1.x FROZEN)

> **Frozen at v1.0.0 (2026-07-05).** Additions land as 1.x minors; a removal
> or a change to a valid call's result is a 2.0 change. `_`-prefixed symbols
> are internal. Since 1.1.2, invalid arguments get documented error returns
> instead of undefined behaviour; every valid call returns what it did before,
> bit for bit ([ADR 0001](adr/0001-optimizations-preserve-tokenizer-output.md)).

अक्षर — the text → token-id data layer: adaptive byte vocab, opt-in BPE,
width-generic packed token store, streaming read path. Pure i64; the pure
path needs only rosnet (`t_alloc`/`t_zero`) + stdlib.

## Corpus + vocab

| fn | contract |
|----|----------|
| `corpus_set(text, len)` → V | Build the byte vocab (first-occurrence order) and the packed store at `g_data_w` over a caller buffer, which is retained, not copied, as `g_text`. Resets BPE state. Returns the vocab size, or **-5** (`len < 1`, or the store allocation refused) or **-6** (`g_data_w` not 1, 2 or 8). A refusal leaves the loaded corpus untouched. The pure path (tarka, tentib, prajna) — no I/O. |
| `corpus_build(buf)` → len | Write the embedded 193-byte demo corpus (NUL-terminated) into `buf` (≥ 256 bytes). |
| `build_corpus()` → V | `corpus_set` over the embedded corpus; **-4** if its 1 KiB buffer is refused. |
| `corpus_load_file(path)` → V | Load through the consumer's `secure_read_file`. Errors: the hook's own (passed through), **-2** over `MAX_CORPUS_BYTES` (from the hook, or a hook count past the cap), **-3** empty, **-4** buffer refused, `corpus_set`'s -5 / -6. |
| `corpus_load_stdin()` → V | Load through the consumer's `read_stdin`, reading cap + 1 so an oversize input is **-2**, never truncated. Other returns as the file path. |
| `MAX_CORPUS_BYTES()` | 64 MiB loader cap. |

| global | contract |
|----|----------|
| `g_vocab` / `g_char2id` | id → byte value (256 slots) / byte value → id (−1 absent) |
| `g_corpus_vlen` | byte vocab size of the loaded corpus (the BPE base). Not the consumer's model vocab — akshara has no `g_V`; that is attn11's. |
| `g_text` / `g_textlen` | the raw corpus bytes retained by `corpus_set` (the checkpoint loader re-encodes them) |

## Packed token store

| fn / global | contract |
|----|----------|
| `gd_ld(i)` / `gd_st(i, v)` | load / store token `i` of the store at `g_data_w` (u8 / u16 / i64). `gd_ld` reads through the shard when `g_stream != 0`. **Unchecked**: `i` in `[0, g_datalen)` is the caller's contract (the sampler's hot path). |
| `g_data` / `g_datalen` / `g_data_w` | the store, its token count, its width (1, 2 or 8; set before `corpus_set`, which refuses others; `bpe_learn` widens 1 to 2) |
| `stream_tok(i)` | one token from a streamed shard through the single chunk cache. **Fails loud** (stderr, exit 1) on `i` outside `[0, g_datalen)`, an unset cache, a failed seek, a short read, or (since 1.3.1) a chunk holding an id outside the installed vocabulary: `[0, g_bpe_base + g_bpe_nmerges)` under BPE, `[0, g_corpus_vlen)` at byte level. Each chunk is checked once, as it is read, so no id of a bad chunk is returned. Retries a read interrupted by a signal. |
| `g_stream`, `g_stream_fd`, `g_stream_doff`, `g_stream_buf`, `g_stream_cap`, `g_stream_lo`, `g_stream_clen` | the streaming state. The **consumer** opens and validates the shard, installs its tokenizer, and sets every field (attn11: `src/stream.cyr` `shard_open`). Since 1.3.1 `stream_tok` checks every id against that tokenizer, so install it before the first read. `corpus_set` does not touch them: clear `g_stream` before loading an in-memory corpus. |
| `STREAM_CHUNK()` | 65,536 tokens per chunk read. |

## BPE (opt-in; byte level is the default)

| fn | contract |
|----|----------|
| `bpe_learn(K)` → V | Learn up to K merges over the byte-level store (Sennrich 2016; pure i64; overlap counting; argmax tie-break highest count → smallest left id → smallest right id; stops below `BPE_MIN_FREQ`; skips a pair whose span would pass `BPE_MAX_TOKLEN`). Returns `B + merges`, or **-1** K outside [1, 512]; **-2** no byte-level corpus (none loaded, < 2 tokens, or a base outside [1, 256]); **-3** an allocation refused; **-4** already BPE — learn again only after a fresh `corpus_set`; **-5** `g_data_w` not 1, 2 or 8. A refusal changes nothing. |
| `bpe_alloc_once()` | the one-time, cap-sized BPE tables (`g_merges`, `g_tok_len`, `g_tok_bytes`, `g_enc_buf`); exits (stderr) if refused. Public because consumers install a checkpoint's table staged. |
| `bpe_build_spans()` → 0 / -1 | Rebuild every span from `g_bpe_base` / `g_bpe_nmerges` / `g_merges`. **Validates the whole table first** — base in [1, 256], count in [0, 512], every merge well-founded (`l, r < B + m`), every span ≤ `BPE_MAX_TOKLEN` — and returns **-1** with every span unchanged on a bad one. Call it after installing any table. |
| `BPE_MAX_MERGES()` / `BPE_VMAX()` / `BPE_MAX_TOKLEN()` / `BPE_MIN_FREQ()` / `ENC_MAX()` | 512 / 768 / 64 / 2 / 8192 |

| global | contract |
|----|----------|
| `g_bpe_on` / `g_bpe_base` / `g_bpe_nmerges` / `g_merges` | BPE state — serialized into consumer checkpoints (attn11 checkpoint v6, token shards) |
| `g_tok_len` / `g_tok_bytes` | each id's span length / bytes (stride `BPE_MAX_TOKLEN`). Only ids below `B + K`, and only the first `tok_len` bytes of a slot, belong to the table. |
| `g_enc_buf` | `ENC_MAX` i64 prompt-encode scratch (attn11 encodes prompts into it at ow = 8) |
| `g_paircnt` | `bpe_learn`'s pair-count table (internal state; exact counts of the final store after a learn) |

## Encode / decode

| fn | contract |
|----|----------|
| `tok_encode(text, len, out, ow)` → n | Encode `len` bytes into `out` (≥ `len * ow` bytes) at width `ow`: unseen bytes → id 0, then every merge in order. Encoding the corpus bytes reproduces the store bit for bit. `n ≤ len`. Returns **-1** when `ow` is not 1, 2 or 8, no tokenizer is loaded, or `ow = 1` and a merge minting an id past 255 applies to the text (a u8 cannot hold it; 1.1.1 truncated it). At `ow = 1` any text whose encoding stays below 256 ids encodes as before. Since 1.3.0 an input of 2 to `ENC_MAX` bytes runs an O(n + K) rank-ordered encoder with the same output ([architecture 003](architecture/003-rank-ordered-encoder.md)); longer inputs, and tables it cannot reproduce, run the per-merge passes. |
| `tok_emit(id)` → 0 / -1 | Emit one id's bytes: byte level through the consumer's `_putc`, BPE through `print`. **-1** (nothing emitted) for `id < 0`, `id ≥ 256` at byte level, or `id ≥ g_bpe_base + g_bpe_nmerges` under BPE. |
| `decode_char(id)` | base id → byte value. **Unchecked**: `id` in `[0, 256)` (and `< g_bpe_base` under BPE). |

## Contract notes (frozen semantics)

- **Caller-buffer, caller-guarantees**: the lib never owns I/O — file/stdin
  loading, shard seeks and byte-level emit go through four consumer-provided
  hooks (`secure_read_file`, `read_stdin`, `file_seek`, `_putc`), left
  unresolved in `dist/akshara.cyr`. Consumers on the pure path stub them.
  (Through 1.1.1 the bundle also needed `puts`; fatal messages now go to
  stderr through the stdlib.)
- **Process-global tokenizer state** (the `g_*` tables above) — one corpus at
  a time, matching the single-threaded execution model; the globals are part
  of the frozen surface because consumer checkpoints serialize them.
- **Bit-identical across versions and targets**: every output of 1.3.0 equals
  1.1.1's on x86_64 and aarch64 (tests/golden.cyr digests 15 fixed cases; CI
  checks them on both).
