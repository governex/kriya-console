# kriya — agent governance for the enterprise

Your teams run AI agents on their machines — Claude Code, Claude Desktop, Cursor, MCP tools. kriya
controls what those agents may do, pauses the dangerous parts for a human, and turns everything
they did into **cryptographically signed receipts you can re-verify yourself, offline**.

Built on the open-source [kriya runtime](https://github.com/governex/kriya) (MIT). The engine is
open; the apps ship signed + notarized. Downloads:
[Releases](https://github.com/governex/kriya-console/releases) · [kriyanative.com](https://kriyanative.com).

## Two apps, one wire

- **K-Apter** — a ~15 MB menu-bar agent for every device. It wires itself into each detected agent
  (and re-wires new ones within a minute), enforces your policy locally — allow / deny / hold for
  approval — shows Approve/Deny right in the menu bar, signs a receipt for every action, and pushes
  sealed envelopes to your hub. Installs free; pairing takes two fields and two minutes.
- **kriya Console** — the cockpit, one per organization. Author the org policy (published to the
  fleet, enforced on-device), watch org-wide activity, manage devices and seats, and export
  auditor-grade evidence. The Console **is** the hub — no separate server to run.

Connect them your way — same build, same signed wire:

| Lane | How | For |
|---|---|---|
| Same machine | nothing leaves the device | solo / evaluation — free forever |
| Direct | devices → your Console or headless hub, on your network | orgs that want zero vendor infra on the path |
| Connector | devices → `console.kriyanative.com/t/<your-code>` → your Console | different networks, zero infra — the relay **verifies evidence, buffers it, and deletes on ack**, and can forge nothing. It also holds your current org policy so a device that was offline can fetch it on return, and it can read that policy — see [TRUST](docs/TRUST.md) |

## Why it holds up in front of an auditor

Every governed action becomes an **Ed25519-signed receipt**; receipts are sealed into
hash-chained envelopes, re-verified at every hop, and finally re-verified **on the cockpit,
offline**. Tamper-evident, honestly scoped — the limits of the guarantee are written down in
[docs/TRUST.md](docs/TRUST.md), and an auditor can independently re-check the evidence with the
free CLI verifier. Compliance exports map the verified trail to EU AI Act, SOC 2, ISO 42001,
NIST 800-171 controls — gaps shown, never hidden.

## Ten-minute start (one machine, free)

1. Install K-Apter and the Console (one `.pkg`).
2. Open the Console → it finds your agents governed and receipts flowing.
3. Write one rule — *hold `git push` for approval* — run it, and answer the prompt in your menu bar.
4. Open Compliance → export the evidence bundle, and re-verify it with the CLI.

Fleet setup (two machines → N): [docs/INSTALL.md](docs/INSTALL.md).

## Documentation

| Doc | What |
|---|---|
| [docs/DESIGN.md](docs/DESIGN.md) | The architecture: two binaries, three lanes, licensing, invariants |
| [docs/INSTALL.md](docs/INSTALL.md) | Enterprise install + the 8-step acceptance test |
| [docs/TRUST.md](docs/TRUST.md) | What you can prove, and the honest limits |
| [docs/PRICING.md](docs/PRICING.md) | Tier structure (numbers land with pilot partners) |
| [docs/privacy/](docs/privacy/) | DPIA template, employee notice, works-council clause |

Licence: free tier is free forever, no account, no network. Fleet features unlock with an org
licence carrying device seats, enforced at your hub — devices never phone home. See [LICENSE](LICENSE).
