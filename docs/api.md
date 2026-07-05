# akshara — Public API (1.x FROZEN)

> **Frozen at v1.0.0 (2026-07-05).** A clean freeze — no behavior change from
> 0.1.0; this documents what the shipping consumers (attn11 — the origin —
> and tarka; anukūlana wires it when its data path needs text) have exercised
> unchanged since the extraction. Additions land as 1.x minors. `_`-prefixed
> symbols are internal.

अक्षर — the text → token-id data layer: adaptive byte vocab, opt-in BPE,
width-generic packed token store, streaming read path. Pure i64; the pure
path needs only rosnet (`t_alloc`/`t_zero`) + stdlib.

## Corpus + vocab

| fn / global | contract |
|----|----------|
| `corpus_set(text, len)` → V | build the byte vocab + packed token store over a caller buffer; returns vocab size. The pure path (tarka, tests) — no I/O. |
| `corpus_build(buf)` / `build_corpus()` | the attn11 loader-path builders (consumer-provided `secure_read_file`/`read_stdin` feed them) |
| `corpus_load_file(path)` / `corpus_load_stdin()` | load + build via the consumer I/O hooks; unresolved in the dist bundle by design (rosnet→tyche pattern) |
| `g_V` | vocab size (base bytes + learned BPE merges) |
| `g_vocab` / `g_char2id` | id → byte value / byte value → id (−1 absent) |
| `g_text` | the raw corpus bytes retained by `corpus_set` |

## Packed token store

| fn / global | contract |
|----|----------|
| `gd_ld(i)` / `gd_st(i, v)` | width-generic load/store into the packed corpus (u8/u16/i64 per `g_data_w`) |
| `g_data` / `g_datalen` / `g_data_w` | the packed store, its token count, its width |
| `stream_tok(i)` | streaming read path (file-backed corpora larger than RAM) |

## BPE (opt-in; byte-level is the default per ADR 0002)

| fn / global | contract |
|----|----------|
| `bpe_learn(K)` | learn K merges over the loaded corpus (Sennrich 2016; pure i64) |
| `bpe_alloc_once()` / `bpe_build_spans()` | the learn-path setup pieces (public because the origin calls them staged) |
| `g_bpe_on` / `g_bpe_base` / `g_bpe_nmerges` / `g_merges` | BPE state — serialized into consumer checkpoints (attn11 checkpoint v6) |

## Encode / decode

| fn | contract |
|----|----------|
| `tok_encode(text, len, out, ow)` | encode a prompt buffer into ids at width `ow` |
| `tok_emit(id)` | emit one id's bytes via the consumer `_putc` hook |
| `decode_char(id)` | id → base byte value |

## Contract notes (frozen semantics)

- **Caller-buffer, caller-guarantees**: the lib never owns I/O — file/stdin
  loading goes through consumer-provided hooks (`secure_read_file`,
  `read_stdin`, `file_seek`, `puts`/`_putc`), left unresolved in
  `dist/akshara.cyr`. Consumers on the pure path stub them.
- **Process-global tokenizer state** (the `g_*` table above) — one corpus at
  a time, matching the single-threaded execution model; the globals are part
  of the frozen surface because consumer checkpoints serialize them.
- Byte-identical to the attn11-proven code at extraction; the freeze is the
  multi-consumer soak (attn11 1060-suite + tarka suites green since
  2026-06-22).
