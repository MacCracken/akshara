# Security Policy

## Reporting

Report vulnerabilities to **cyriusmaccken@gmail.com**. Include reproduction
steps and the akshara version from `VERSION`. Expect an initial response within
one week. Coordinated disclosure is appreciated — do not open a public GitHub
issue with exploit details.

## Threat model

akshara is a single-process, CPU-only **tokenizer library**. It has **no
networking**, opens **no files**, and runs **no processes**. It is consumed
in-process (attn11, tarka, tentib, prajna); it does not run as a standalone
service. Its untrusted inputs are:

- the **corpus bytes** a caller hands it (`corpus_set`, `corpus_load_file`,
  `corpus_load_stdin`, or a streamed token shard),
- a **merge table** a consumer installs from a checkpoint or shard
  (`g_bpe_base` / `g_bpe_nmerges` / `g_merges`, then `bpe_build_spans`),
- the **arguments** of the public functions (`K`, `len`, `ow`, `id`, and the
  `g_data_w` width).

A realistic attacker is assumed able to place a hostile corpus file, shard or
checkpoint where a consumer opens it, pipe hostile bytes to a consumer's stdin
path, and choose the BPE hyperparameter `K`. akshara *does not* defend against
an attacker with code execution in the host process, network attacks (there is
no networking), or side channels. Corpus *content* is untrusted by design —
akshara faithfully tokenizes whatever bytes it is given.

## Attack surfaces & mitigations

| Surface | Mitigation |
|---|---|
| **Corpus bytes** | The byte-level vocab accepts any of the 256 byte values into a fixed table; there is no parse step to overflow. Loaded corpora are capped at `MAX_CORPUS_BYTES` (64 MiB) **before** use — the stdin path reads `cap + 1` and rejects (-2) rather than truncating, and since 1.1.2 the file path also rejects a hook that reports more than the cap. `corpus_set` validates its arguments **before** touching any state: `len` in [1, 2^60 − 1] (so `len * width` cannot wrap) and `g_data_w` in {1, 2, 8} (an invalid width used to size the store at `len * w` and then write 8-byte tokens into it). A refusal leaves the loaded corpus intact. |
| **BPE learning** (`bpe_learn`) | `K` is range-checked to [1, 512]. Merges are minted **in order**, so every merge references only ids that already exist — a well-founded DAG by construction, which lets decode run as a flat span table. A token's expansion is bounded by `BPE_MAX_TOKLEN` (64): a pair that would exceed it is skipped, so there is no exponential-expansion path. Since 1.1.2 a second learn over an already-merged store is refused (-4) — it used to mint ids that collided with the store's existing merged ids, corrupting the table silently. |
| **Installed merge tables** (`bpe_build_spans`) | Since 1.1.2 the whole table is validated before any span is written — base in [1, 256], count in [0, 512], every merge `l, r < B + m`, every span ≤ 64 bytes — and a bad table returns -1 with every span unchanged. Consumers validate tables they load (attn11 checkpoint -33 … -38, shard -60 … -68); this makes akshara memory-safe even for a consumer that does not. |
| **Encode / emit** | `tok_encode` refuses an output width other than 1 / 2 / 8, a u8 encode that would mint an id past 255 (which used to be truncated silently), and a call with no tokenizer loaded. `tok_emit` refuses an id outside the table (which used to print adjacent heap bytes). Since 1.3.0, inputs of 2 to `ENC_MAX` bytes run a rank-ordered encoder over a fixed one-time scratch block sized for `ENC_MAX` nodes; it validates the installed table (every merge well-founded, base in [1, 256]) and the `g_char2id` ids (below 1024) on every call and hands anything else to the per-merge passes, so a hostile table cannot index past the scratch or change an output. |
| **Streaming token shard** (`stream_tok`) | Chunk reads are clamped to `g_datalen` and the chunk cap; an index outside the corpus, an unset cache, a failed seek or a short read fails loud (stderr, exit 1) instead of returning a stale token; a read interrupted by a signal is retried. Shard size vs token count is the consumer's open-time gate (akshara holds the fd it is handed). The token payload is too large to scan at open, so since 1.3.1 every chunk is checked as it is read: a chunk holding an id outside the installed vocabulary fails loud before any of its ids is returned. Through 1.3.0 such an id reached the consumer, where it indexes the embedding table; in attn11 ≤ 1.15.1 that was an out-of-bounds gradient write ([issue](docs/development/issues/2026-11-09-stream-tok-unchecked-ids.md)). |
| **File / stdin I/O** | akshara does **not** open files — the loader, streaming and emit paths call consumer-supplied hooks (`secure_read_file`, `read_stdin`, `file_seek`, `_putc`). Path-level hardening (`O_NOFOLLOW`, `fstat` size caps, symlink / TOCTOU handling) lives in the consumer's tree (attn11: `fileio.cyr`). |

Deliberately unchecked, as documented in `docs/api.md`: `gd_ld` / `gd_st`
(index in `[0, g_datalen)`) and `decode_char` (id in `[0, 256)`) — the hot
read paths of a training loop, whose callers hold the bounds.

## Audits

- [`docs/audit/2026-11-09-audit.md`](docs/audit/2026-11-09-audit.md) — the 1.1.2
  P(-1) hardening, optimization and repair sweep: findings, fixes, and what was
  deliberately left.

Before 1.1.2 akshara had no standalone audit; its guarantees were inherited
from the attn11 code it was carved from.
