# Signed spend receipts — a real sample (C1)

> For a compliance/finance reviewer asking "can I trust this dollar figure?" — this states, precisely
> and without inflation, **what a `kriya.spend.*` receipt proves and what it doesn't.**
>
> The three receipts in [`receipts.jsonl`](receipts.jsonl) are **real, genuinely signed output** —
> settled by this Console build from THIS machine's own real `~/.claude/projects` transcripts (two
> real sessions + the rollup covering them), Ed25519-signed with a throwaway demo key, and
> independently re-verifiable right now:
>
> ```
> cargo run -p kriya-audit-cli -- docs/samples/spend-receipt-demo/receipts.jsonl
> # -> receipts.jsonl: 3 receipt(s), 3 signature(s) verified, hash-chain intact — OK
> ```
>
> Nothing here is fabricated or hand-edited — `session_id`/`project_hash` are already opaque
> (a UUID and a SHA-256 hash), so nothing needed redacting to publish this.

## The one-line answer

**Priced by sheet `kriya-pricing-2026-07-01`; unpriced models this period: `claude-fable-5`.** Every
number below traces to one of the three receipts — never a live, unreceipted figure.

## Sample 1 — a priced session, with a subagent breakout

```json
{
  "step_id": "bc602e6ee230ec52c1890ff60d2714e0",
  "action_id": "kriya.spend.session",
  "params": {
    "v": 1,
    "session_id": "273468f8-d30d-49d5-a40b-fff5173a0657",
    "agent": "claude-code",
    "project_hash": "63291fe066c375cf4bbf357cbcc25f1e7be5013d8a6519715868bcd314a238e3",
    "window_start_ms": 1782742661914,
    "window_end_ms": 1782791632328,
    "sealed": true,
    "messages_counted": 839,
    "messages_deduped": 1043,
    "by_model": {
      "claude-opus-4-8": {
        "input_tokens": 525217, "output_tokens": 552003,
        "cache_creation_5m": 2797878, "cache_creation_1h": 1452979, "cache_read": 143855936,
        "cost_usd": 361.1119665, "priced": true, "reason": null
      }
    },
    "subagent": { "input_tokens": 482660, "output_tokens": 13957, "cost_usd": 101.155596 },
    "totals": {
      "input_tokens": 525217, "output_tokens": 552003,
      "cache_creation": 4250857, "cache_read": 143855936,
      "cost_usd": 361.1119665, "unpriced_models": []
    },
    "pricing_sheet": "kriya-pricing-2026-07-01",
    "pricing_sheet_hash": "dfce536ca1f40502399e5d687a285a3c55e9846d268ca371cd8ef5c5c5ffb78a",
    "settle_fingerprint": "b901f0966510a4f2a8d9c5e7da4a5d3bfb17a5b9c85b5dd3b098d9b9370b063b"
  },
  "success": true,
  "ts_ms": 1784557517578,
  "actor": { "agent": "kriya-console", "user": "skumar" },
  "public_key": "6231fb09ee134e72b10cf93547e75448f737bc37d7256e2cc8bc8ce36a73abc0",
  "signature": "de69c8d2418b1c0709b7656aa6f21eeb05e1264f1fe42e41c3d2a83c15aad7a9157785ab72f2d9a9774b7be51df1da13f5892973395d8d5eb69e119ca1cddc06"
}
```

Read this line: this real 839-message session (1043 duplicate JSONL lines dropped by the
`(message.id, requestId)` dedup — a real, heavily-resumed session) ran entirely on
`claude-opus-4-8`, cost **$361.11** priced by sheet `kriya-pricing-2026-07-01`. Of that,
subagent (Task-tool) turns contributed 482,660 of the 525,217 input tokens and **$101.16** of the
total — a visible breakout, not a second receipt (no double count). `ts_ms` (1784557517578, the
device clock when this Console **settled** the session) is well after `window_end_ms`
(1782791632328, the session's own last-activity time) — settle time and activity time, never
conflated (D4).

## Sample 2 — the honest unpriced path

```json
{
  "step_id": "740b4e6f1ec350f79eb1553dc212a7a5",
  "action_id": "kriya.spend.session",
  "params": {
    "session_id": "37eb388c-d75d-458a-b149-28f23783f590",
    "by_model": {
      "claude-fable-5": {
        "input_tokens": 440, "output_tokens": 37803,
        "cache_creation_5m": 0, "cache_creation_1h": 91057, "cache_read": 1204422,
        "cost_usd": null, "priced": false,
        "reason": "model not in sheet kriya-pricing-2026-07-01"
      }
    },
    "totals": {
      "input_tokens": 440, "output_tokens": 37803,
      "cache_creation": 91057, "cache_read": 1204422,
      "cost_usd": 0.0, "unpriced_models": ["claude-fable-5"]
    }
  },
  "success": true, "ts_ms": 1784557517578
}
```

(Trimmed to the fields that matter here — the committed JSONL line carries the full shape.) A
model the active sheet doesn't recognize is priced `null`, never `0` as a guess — its 37,803 output
tokens are still honestly counted, and `unpriced_models` names it so the panel shows an "unpriced"
line instead of a silently-wrong cheap total.

## Sample 3 — the rollup: a non-additive checkpoint, never summed into the total

```json
{
  "action_id": "kriya.spend.rollup",
  "params": {
    "period": "2026-07-20",
    "sessions": ["740b4e6f1ec350f79eb1553dc212a7a5", "bc602e6ee230ec52c1890ff60d2714e0"],
    "session_count": 2,
    "totals": {
      "input_tokens": 525657, "output_tokens": 589806,
      "cost_usd": 361.1119665, "unpriced_models": ["claude-fable-5"]
    }
  }
}
```

The rollup's `sessions` array lists the two `session` receipts' own `step_id`s (verifiable
references, not a re-derivation) — anyone can re-sum `totals.cost_usd` across those two receipts
and get the identical `$361.1119665` this rollup states. Note the rollup's `cost_usd` equals
Sample 1's alone — Sample 2's unpriced session contributes tokens to the rollup's totals but **zero
dollars**, exactly like the panel's own math. The rollup is a cheap, signed period-total checkpoint
for the compliance-export story ("this quarter's governed spend, in one verifiable number") — the
panel's authoritative total is always computed from `session` receipts directly, never from this.

## What this is not

- **Not complete spend.** This reads local Claude Code transcripts only — MCP broker-lane spend,
  direct API usage outside Claude Code, local-inference spend, and vendor-cloud/web Claude usage are
  all out of scope. "Governed Claude Code session spend evidence," never "all spend."
- **Not an invoice.** `$361.1119665` is priced from a named, dated, bundled sheet this receipt
  states by id and hash — not what Anthropic actually billed. A different, newer sheet would price
  the SAME tokens differently, and that receipt would say so honestly (a new `pricing_sheet` id).
- **Not proof the host wasn't compromised.** Same tamper-*evidence*-not-tamper-*proof* ceiling as
  every other kriya receipt — see [`docs/TRUST.md`](../../TRUST.md).

See [`docs/TRUST.md`](../../TRUST.md)'s "Spend evidence (C1)" section for the full honesty model.
