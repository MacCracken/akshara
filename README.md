# akshara

> अक्षर — *the indivisible unit of text and sound* (lit. "imperishable"); the
> atom-of-writing in Indian linguistics. A sovereign, dependency-light
> **tokenizer** library in [Cyrius](https://github.com/MacCracken/cyrius): raw
> bytes → token ids and back.

akshara is the **text → token-id data layer** extracted from
[attn11](https://github.com/MacCracken/attn11) (the third extraction after
[rosnet](https://github.com/MacCracken/rosnet) and
[tyche](https://github.com/MacCracken/tyche)). It is shared by attn11 (the
SFT/diffusion training reference) and [tarka](https://github.com/MacCracken/tarka)
(the RL/reasoning reference) — one tokenizer, two consumers.

## What it owns

- **Adaptive byte-level vocab** — only the byte values that actually occur
- **Opt-in BPE** (Sennrich et al. 2016) — learn `K` merges, encode, decode; pure
  i64 arithmetic, bit-reproducible across x86_64 / aarch64
- **Packed token store** — width-generic (u8 / u16 / i64) accessors + a streaming
  read path for RAM-independent large corpora
- Checkpoint-facing tokenizer state (vocab / merges / spans) the consumer
  serializes into its own checkpoint blob

## Consuming it

`cyrius distlib` bundles `[lib].modules` into `dist/akshara.cyr`. A consumer:

```toml
[deps.akshara]
git = "https://github.com/MacCracken/akshara"
tag = "0.1.0"
modules = ["dist/akshara.cyr"]
```

then `include "lib/akshara.cyr"`. The bundle leaves a few **consumer-supplied**
symbols unresolved (the rosnet→tyche pattern): `t_alloc` / `t_zero` come from
[`rosnet`](https://github.com/MacCracken/rosnet); the loader / streaming / emit
paths reference `puts` / `_putc` / `secure_read_file` / `read_stdin` / `file_seek`,
which the consumer's own tree provides. The **pure** tokenizer path
(`corpus_set` / `bpe_learn` / `tok_encode` / `decode_char`) needs only rosnet + stdlib.

## Build

```sh
cyrius deps                              # resolve stdlib + rosnet
cyrius distlib                           # regenerate dist/akshara.cyr
cyrius build src/main.cyr build/akshara  # pure-path smoke (vocab / BPE / encode)
./build/akshara
```

## License

GPL-3.0-only
