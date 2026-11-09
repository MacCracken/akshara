# akshara — Roadmap

> Milestone plan. State lives in [`state.md`](state.md); this file is the
> sequencing — what shipped, what is next, against what gates.

## v1.0 criteria — met at 1.0.0 (2026-07-05), evidence completed in 1.1.2

- [x] Public API frozen — every exported symbol documented ([`docs/api.md`](../api.md); `cyrius doc --check` clean since 1.1.2) and tested (the 1.1.2 suite covers every public function)
- [x] Test coverage adequate for the surface area — golden digests, the 1.1.1 oracle, contract tests, a property fuzzer (1.1.2)
- [x] Benchmarks captured in [`docs/benchmarks.md`](../benchmarks.md) (1.1.2)
- [x] At least one downstream consumer green — attn11 (origin), tarka, tentib, prajna
- [x] CHANGELOG complete from v0.1.0 onward
- [x] Security audit pass — [`docs/audit/2026-11-09-audit.md`](../audit/2026-11-09-audit.md) (1.1.2)

## Milestones

### M0 — Scaffold (v0.1.0) — shipped 2026-06-22

### M1 — Extraction from attn11 (v0.1.0) — shipped 2026-06-22

The text → token-id layer carved out of attn11 byte-identically; attn11
re-pointed to `[deps.akshara]`.

### M2 — v1.0 freeze — shipped 2026-07-05

### 1.x — toolchain realigns and the P(-1) sweep

1.0.1 – 1.1.1 follow the cyrius pin (6.5.27 → 6.7.18) with no source change.
1.1.2 is the first P(-1) hardening, optimization and repair sweep: bit-identical
outputs, 5–10× faster learn and encode, validated tables and arguments.

## Next (unscheduled)

- Faster BPE learn on large corpora: the per-merge rewrite pass is O(n), so a
  64 MiB corpus at K = 512 still takes tens of seconds (audit, *not changed*).
- Consumers onto the current 1.x tag (all four were on 1.0.2 at 1.1.2).

## Out of scope (for 1.x)

- Any change to merge selection, counting, or encode semantics — a 2.0 change
  ([ADR 0001](../adr/0001-optimizations-preserve-tokenizer-output.md)).
- Pre-tokenization (word splitting), Unicode normalization, special tokens.
