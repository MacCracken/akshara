# Security Policy

## Reporting

Report vulnerabilities to **cyriusmaccken@gmail.com**. Include reproduction
steps and the akshara version from `VERSION` (currently **0.1.0**). Expect an
initial response within one week. Coordinated disclosure is appreciated — do not
open a public GitHub issue with exploit details.

## Threat model

akshara is a single-process, CPU-only **tokenizer library**. It has **no
networking** and no privileged operations. It is consumed in-process by attn11
and tarka; it does not run as a standalone service. Its only untrusted inputs
are the **corpus bytes** a caller hands it (`corpus_set`, `corpus_load_file`,
`corpus_load_stdin`, or a streaming token shard) and the **merge count** `K`
passed to `bpe_learn`.

A realistic attacker is assumed able to:

- supply build inputs (source / embedded corpus) — equivalent to code execution,
  which akshara does not try to defend against, and/or
- place a hostile corpus file / shard at a path the consumer opens, or pipe
  hostile bytes to a consumer's stdin path, and/or
- choose the BPE hyperparameter `K`.

akshara *does not* defend against an attacker with arbitrary code execution in
the host process (they own the address space), remote/network attacks (there is
no networking), or side channels. The *meaning* of corpus content is untrusted
by design — akshara faithfully tokenizes whatever bytes it is given.

## Attack surfaces & mitigations

| Surface | Mitigation |
|---|---|
| **Corpus bytes** (`corpus_set` / `corpus_load_file` / `corpus_load_stdin`) | The byte-level vocab accepts any of the 256 byte values into a fixed table; there is no parse step to overflow. Loaded corpora are size-capped to `MAX_CORPUS_BYTES` (64 MB) **before** use — the stdin path reads `cap + 1` and rejects (`0-2`) rather than silently truncating; empty / failed reads return distinct negative errors, never a crash. The packed token store (u8 / u16 / i64) is a single cap-sized allocation; indices are derived from the validated corpus length. |
| **BPE merge learning** (`bpe_learn`) | `K` is range-checked to `[1, BPE_MAX_MERGES]` (512). Merges are minted **in order**, so every merge references only ids that already exist (`< base + m`) — a well-founded DAG by construction (no forward or cyclic references possible), which is what lets decode (`bpe_build_spans`) run as a flat span table rather than recursing. A token's byte expansion is bounded to `BPE_MAX_TOKLEN` (64): any merge that would exceed it is skipped, so there is no exponential-expansion ("decompression-bomb") path. All BPE buffers are allocated once at cap size; the learn/encode loops allocate nothing. |
| **Streaming token shard** (`stream_tok`) | Chunk reads are clamped to `g_datalen` and the chunk cap, so a read never passes EOF; the shard size vs token-count consistency is the consumer's open-time gate (akshara holds the fd it is handed). |
| **File / stdin I/O** | akshara does **not** open files itself — the loader, streaming, and emit paths call consumer-supplied symbols (`secure_read_file` / `read_stdin` / `file_seek` / `file_read`). Path-level hardening (`O_NOFOLLOW`, `fstat` size caps, symlink/TOCTOU handling) lives in the consumer's tree (attn11: `fileio.cyr`), **not** in this library. akshara's contribution is the in-memory byte cap and the structural merge-table guarantees above. |

## Maturity

akshara is **0.1.0** — extracted from attn11 and validated transitively by
attn11's full suite (1060/1060 green at extraction; the byte-identity gate). It
has **not** had a dedicated, standalone security audit; the structural
guarantees above are inherited from the attn11 code they were carved from, not
from an akshara-specific review. A formal audit (`docs/audit/YYYY-MM-DD-audit.md`)
is a v1.0 criterion (see [`docs/development/roadmap.md`](docs/development/roadmap.md)).
Treat the merge-table and corpus-cap guarantees as load-bearing; treat the
consumer-supplied I/O paths as out of akshara's trust boundary.
