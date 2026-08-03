# `kriya-diode-bundle/1` — a real, worked sample (F10)

> For a CDS/diode-filter vendor or an auditor's own independent implementation. Every file in this
> directory is **real, generated output** (`cargo run --example gen_diode_sample` from
> `src-tauri/`), not hand-typed — the hashes below are exactly what a fresh regeneration produces,
> and exactly what `kriya-audit --verify-bundle` independently re-derives. See
> [`../../DIODE-BUNDLE.md`](../../DIODE-BUNDLE.md) for the full wire-format spec this sample follows.

## Contents

```text
diode-bundle-sample/
  manifest.json      — signed, schema kriya-diode-bundle/1
  chunks/00000.bin   — 2 receipt lines, 873 bytes
```

## Re-verify this exact sample right now

```
cargo run -p kriya-audit-cli --bin kriya-audit -- --verify-bundle docs/samples/diode-bundle-sample
# -> manifest signature OK, schema OK
# -> 2 receipt(s) declared, 2 verified, 0 failed
# -> whole-bundle Merkle root OK
# -> OK
```

## The manifest, in full

```json
{
  "manifest": {
    "schema": "kriya-diode-bundle/1",
    "device_pub": "4ed32f63bf35f0eeefcb25f28a2e1fbdc873ae2835671b0c9460f5f12e4556a8",
    "range": {
      "kind": "full"
    },
    "created_ms": 1753400002000,
    "chunk_size_bytes": 8388608,
    "total_receipts": 2,
    "merkle_root": "006dad386302e030e64ec2c13dee81489971c18985ba4d03696593ff277321ca",
    "chunks": [
      {
        "index": 0,
        "filename": "00000.bin",
        "sha256": "f32593360db0a39894dba47d57dde3bdc895a86e9d37afc18d2e56ce8c29647f",
        "receipt_count": 2,
        "byte_len": 873
      }
    ],
    "chain_head": "4a25c173f8334949ac3058d75a592bf7bbb57eef37519a5a21fd45b1740f047f"
  },
  "public_key": "4ed32f63bf35f0eeefcb25f28a2e1fbdc873ae2835671b0c9460f5f12e4556a8",
  "signature": "a209aeb7afce1f95136ec1de58267a63cdb5c77959695802cb145871028083398eabecf6e7c72443fad07809d84e7ae0d71390a231ce0d1d559431b43ebd1f0a"
}
```

## Hand-checking the hashes yourself

The single chunk file (`chunks/00000.bin`) is nothing but the two receipt lines joined by
`\n` with a trailing `\n` — `cat chunks/00000.bin` shows them directly, and
`shasum -a 256 chunks/00000.bin` reproduces `f32593360db0a39894dba47d57dde3bdc895a86e9d37afc18d2e56ce8c29647f` with a stock Unix tool, no kriya
code involved.

The Merkle root (`006dad386302e030e64ec2c13dee81489971c18985ba4d03696593ff277321ca`) is `SHA-256(0x01 ‖ leaf(line1) ‖ leaf(line2))` where
`leaf(x) = SHA-256(0x00 ‖ x)` — domain-separated (leaf tag `0x00`, node tag `0x01`) so an internal
node can never be presented as a leaf. See [`../../DIODE-BUNDLE.md`](../../DIODE-BUNDLE.md#merkle-root-construction)
for the full construction.

This is the FIRST export this demo device ever took (`previous_chain_head` absent) — a second export
would cite `chain_head` (`4a25c173f8334949ac3058d75a592bf7bbb57eef37519a5a21fd45b1740f047f`) as its own `previous_chain_head`, the continuity link a
zero-backchannel receiver checks between successive bundles.

