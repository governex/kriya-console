# FIPS cryptographic module boundary — assessor one-pager (A4)

> For a security/compliance assessor evaluating kriya against **CMMC SC.L2-3.13.11** / **NIST SP 800-171
> 3.13.11** / **FIPS 140-3**. This states, precisely and without inflation, **what module signs kriya's audit
> receipts, on which operational environment the validation holds, and where it does not.** It is a boundary
> statement, not a certification. Illustrative values below are synthetic.

## The one-line answer

When built with the opt-in **`fips-crypto`** feature, kriya's Ed25519 receipt **signing** (and the Console's
Rust verification and its own signing) run through **`aws-lc-rs` with the `fips` feature** — the **AWS-LC-FIPS
3.x** cryptographic module, **CMVP certificate #5298**, FIPS 140-3, Level 1, Active. Ed25519/EdDSA (FIPS 186-5)
is within the approved boundary of the 3.x certificate. The default build uses `ed25519-dalek`, a
widely-audited pure-Rust implementation that is **not** a FIPS-validated module.

## The per-operational-environment matrix (this is the whole point)

| Where kriya runs | What is true — the exact claim we make |
|---|---|
| **Linux, CMVP-tested OE (Amazon Linux 2023)** | **FIPS 140-3 validated module (cert #5298).** The tested operational environment for the certificate. |
| **Linux, other (Ubuntu/RHEL/…)** | The **same validated module code as cert #5298**, running in an operational environment CMVP did not test. |
| **macOS** | **Same cryptographic module code as cert #5298, running outside a CMVP-tested operational environment.** |
| **Default build (any OS)** | Not FIPS. Signs with `ed25519-dalek` — audited, not validated. |

We never describe kriya as a "FIPS-certified product." kriya is not itself CMVP-validated; it *uses* a
validated module. Own-module validation is a separate, multi-year undertaking and is not claimed here.

## How to verify the claim on a given deployment

Each signer emits a signed, hash-chained **`kriya.crypto.module`** attestation receipt at startup, verifiable
offline with the same tools as any receipt. Illustrative payload (Linux, FIPS lane):

```json
{
  "action_id": "kriya.crypto.module",
  "params": {
    "backend": "aws-lc-rs",
    "fips_module": "AWS-LC-FIPS 3.x",
    "cmvp_cert": "5298",
    "fips_mode_active": true,
    "operational_environment": "validated-oe",
    "key_provenance": "module-drbg",
    "os": "linux", "arch": "x86_64",
    "component": "kriya-gateway"
  },
  "public_key": "…", "signature": "…", "prev_hash": "…"
}
```

- `fips_mode_active` is the **runtime** result of the module's own `try_fips_mode()` check at process start —
  not merely that a build flag was set.
- `operational_environment` degrades honestly: `validated-oe` (Amazon Linux 2023) → `validated-module-untested-oe`
  (other Linux) → `outside-cmvp-oe` (macOS) → `not-fips` (default build).
- `key_provenance` discloses how the signing key was generated: `module-drbg` (minted in the module boundary),
  or `imported-seed` (a key that predates the FIPS lane — stated, not laundered).

Corroborate out-of-band with the build's dependency manifest (`aws-lc-fips-sys` present, pinned version) and
the running OS.

## Scope (honest ceiling — read this)

- **A signature does not, and cannot, reveal which module produced it.** Ed25519 is deterministic (RFC 8032):
  the same key and message yield the same 64 bytes under any conformant implementation. The
  `kriya.crypto.module` receipt is therefore the **host attesting its own crypto configuration** — trustworthy
  to the degree the host is. A fully compromised host is outside kriya's threat model (see `docs/TRUST.md`).
  This attestation is build/runtime **provenance**, not cryptographic proof that FIPS crypto produced the
  neighboring signatures.
- **"Validated" is bounded to the tested operational environment** (Amazon Linux 2023 for cert #5298).
  Anywhere else runs the *same module code* outside a *tested* OE — a distinction the matrix and the
  attestation both preserve.
- **This covers the signing lane.** The Console's browser-based self-verifier uses WebCrypto and makes **no**
  FIPS claim; the compiled-Rust desktop verifier can run the same validated module, as a separate statement.
- **Interoperability is preserved:** a FIPS-signed receipt verifies in a non-FIPS build and vice-versa (same
  algorithm, same bytes), so enabling the lane never breaks an existing audit trail or re-pins a key.

_This one-pager describes an opt-in build. It is evidence toward a control, not a certification of compliance.
Values shown are illustrative/synthetic. See `docs/design/a4-fips-lane.md` for the full design and
`docs/TRUST.md` for the cryptographic-module section._
