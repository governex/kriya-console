# Post-quantum dual-signature receipts — assessor one-pager (A5)

> For a security/compliance assessor asking the CNSA-2.0 procurement question: *"is your long-retention audit
> evidence signed with a quantum-resistant algorithm?"* This states, precisely and without inflation, **what
> kriya adds next to the Ed25519 signature, what it does and does not prove, and where the claim stops.**
>
> The receipts below are **real, Rust-generated fixtures** — `aws-lc-rs` (the production PQ signer) actually
> produced these bytes, byte-for-byte identical to the committed
> `crates/kriya/tests/fixtures/pq-{dual-signed,checkpoint}-ledger.jsonl` (runtime repo) and
> `src-tauri/crates/kriya-verify/fixtures/pq-{dual-signed,checkpoint}-ledger.jsonl` (this repo), both verified
> byte-identically by the Rust lane's own test suite. The long `pq_public_key`/`pq_sig` hex fields are truncated
> below for readability (full lengths noted) — the fixture files carry the complete values.

## The one-line answer

When built with the opt-in **`pq-crypto`** feature, kriya adds an **ML-DSA-87 (FIPS 204) countersignature**
*next to* the existing Ed25519 signature — **by default one countersignature per chain checkpoint** (sealing a
run of receipts through the quantum-resistant SHA-256 hash chain), with **per-receipt dual-sig available as an
opt-in** for low-volume, high-value receipts. The claim is **post-quantum-ready (CNSA 2.0-aligned, ML-DSA-87)** —
never "FIPS-validated PQ" (ML-DSA is behind `aws-lc-rs`'s `unstable` feature and **outside cert #5298's approved
boundary** — ML-KEM is in it, ML-DSA is not) and never "quantum-proof."

## Sample 1 — a per-receipt dual-signed receipt (opt-in mode)

The `pq_*` fields are **additive top-level wire siblings** of `public_key`/`signature` — outside the Ed25519
signed bytes (`serde_json::to_vec(&receipt)`), so the receipt still verifies **byte-for-byte** under any pre-A5
verifier. Real fixture line (`pq-dual-signed-ledger.jsonl`, line 2):

```json
{
  "step_id": "pq-fx-2",
  "action_id": "kriya.test.dual_signed_action",
  "params": { "note": "per-receipt ML-DSA-87 dual signature" },
  "success": true,
  "ts_ms": 1700200000001,
  "actor": { "agent": "claude-code", "user": "platform-eng" },
  "prev_hash": "8a4c28d182ed94d8434d484475323b9a09a4e3b9179d7d862d7d7382fd424b19",
  "public_key": "0e677548decd6508d4b47cae5b4814641754f006511b4aa295f295b742c13d79",
  "signature": "d44ac83ee9cca819013adda1ade789cbe23f1544b8713df17db5d9f8ede53b997a9826c7d6e58ad6c9007ab6572cf31110782378c2e1001be981c6e6e3babe09",
  "pq_alg": "ML-DSA-87",
  "pq_public_key": "ea8413e7b4f2229ba65da233bc8023bce30b86ff52c9fbaf75e7a04d977737d…(5184 hex chars total, 2592 bytes)",
  "pq_sig": "303fb4fb48505e52a66e4d899377d48e9425ae5087079641bab4abe7421a98c5…(9254 hex chars total, 4627 bytes)",
  "pq_key_id": "80d5337fba46deb2"
}
```

- **Require-if-present:** if any `pq_*` sibling is present, a verifier requires the complete, valid set
  (`pq_alg == "ML-DSA-87"`, well-formed `pq_public_key`, a `pq_sig` that verifies, `pq_key_id` matching
  `pq_public_key`). A partial/mismatched set fails with a distinct reason.
- `pq_sig` covers the **identical** bytes the Ed25519 `signature` covers — both signatures attest the same
  who/what/when.
- `pq_key_id` = `80d5337fba46deb2` = the first 8 bytes of SHA-256(`pq_public_key`), lowercase hex.

## Sample 2 — a `kriya.crypto.pq_checkpoint` receipt (default mode)

One ML-DSA-87 signature over the SHA-256 **chain head** post-quantum-anchors every receipt in the sealed
prefix (through the quantum-resistant hash chain). Real fixture line (`pq-checkpoint-ledger.jsonl`, line 7 —
sealing the 5 preceding receipts, `from_seq: 1` through `count: 5`):

```json
{
  "step_id": "pq-fx-cp-checkpoint",
  "action_id": "kriya.crypto.pq_checkpoint",
  "params": {
    "component": "kriya-gateway",
    "count": 5,
    "from_seq": 1,
    "pq_alg": "ML-DSA-87",
    "to_head_hash": "973fbf4658266350f64cb4133584db54e523bebc7750850fbc93775571ee2497"
  },
  "public_key": "0e677548decd6508d4b47cae5b4814641754f006511b4aa295f295b742c13d79",
  "signature": "…128-hex Ed25519 over this receipt's own canonical bytes…",
  "prev_hash": "973fbf4658266350f64cb4133584db54e523bebc7750850fbc93775571ee2497",
  "pq_alg": "ML-DSA-87",
  "pq_public_key": "…5184-hex ML-DSA-87 public key (2592 bytes)…",
  "pq_sig": "…9254-hex ML-DSA-87 signature over this checkpoint's own canonical bytes, which include to_head_hash (4627 bytes)…",
  "pq_key_id": "…16-hex…"
}
```

The **pq_alg-authority rule** (design §4.1): `params.pq_alg` (signed) and the unsigned top-level `pq_alg`
sibling MUST agree; a mismatch fails with `"incomplete or inconsistent PQ signature (pq_alg)"`. The signed
`params.pq_alg` is authoritative.

The PQ public key is bound to the host's existing Ed25519 identity by a separate **Ed25519-signed**
`kriya.crypto.pq_key` attestation (the checkpoint fixture's first line), so an auditor who already pins the
Ed25519 key transitively trusts the PQ key.

## Measured size note (acceptance #4)

Measured directly from the committed fixtures (real `aws-lc-rs`-signed bytes, not an estimate):

| Line type | Measured size | vs. a plain receipt line (~450–470 B) |
|---|---|---|
| Plain Ed25519-only receipt | ~440–472 bytes | baseline |
| Per-receipt dual-signed (opt-in) | ~15,040 bytes | **+~14.6 KiB/line** — matches the design's ~14.5 KiB estimate |
| `pq_checkpoint` (default mode) | ~15,170 bytes | **+~14.7 KiB, but ONE per N receipts** (default N = 256) |

Projected onto the design's 10,000-receipt-day baseline (~5 MB/day at ~500 B/line):

- **Per-receipt dual-sign, every line:** ≈ +146 MB/day ⇒ **≈ 30×** blow-up — confirmed untenable as a default,
  matching the design's 19–30× estimate.
- **Checkpoint every 256 receipts:** ≈ 39 checkpoints/day × ~14.7 KiB ≈ +0.57 MB/day ⇒ **≈ 1.13×** — confirmed
  the checkpoint-default choice (design estimate: ≈1.12×).
- **Checkpoint every 1000 receipts:** ≈ 10 checkpoints/day × ~14.7 KiB ≈ +0.15 MB/day ⇒ **≈ 1.03×** — matches
  the design estimate exactly.

## Scope (honest ceiling — read this)

- **Post-quantum-ready, not FIPS-validated.** ML-DSA-87 here is `aws-lc-rs`'s `unstable` implementation, outside
  cert #5298's approved boundary. The claim is algorithm-conformance + crypto-agility, never validation, never
  "quantum-proof."
- **The checkpoint proves integrity of the sequence, not per-receipt PQ non-repudiation.** One ML-DSA-87
  signature over the SHA-256 chain head gives post-quantum tamper-evidence of the whole sealed prefix
  (collision-resistant hash + one PQ signature); per-receipt PQ non-repudiation is the opt-in per-receipt mode.
- **The browser/TS lane does not independently verify the PQ signature in this build.** The compiled Rust lane
  (`kriya-verify`) is the PQ-verifying authority; the TS lane verifies Ed25519 + message parity + PQ structure —
  a PQ-tampered receipt still shows "Verified (Ed25519)" here with an explicit "PQ signature not checked in
  this lane" note, never a false green PQ checkmark.
- **Receipts after the most-recent checkpoint are Ed25519-only** until the next checkpoint / export seal /
  retention checkpoint seals them — the residual tail-gap, stated not glossed.
- **`pq-crypto` and `fips-crypto` cannot be enabled together** in this build (a real `aws-lc-rs` upstream
  constraint discovered during implementation, not a kriya restriction — see `crypto.rs`'s deviation note in
  both repos and `docs/design/a5-pq-dual-sig.md`'s build-report addendum).

_This one-pager describes an opt-in build. It is evidence toward a control, not a certification of compliance.
See `docs/design/a5-pq-dual-sig.md` for the full design and `docs/TRUST.md` for the post-quantum-readiness
section._
