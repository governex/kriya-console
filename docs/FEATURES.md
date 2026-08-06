# kriya Console — every feature, in plain words

kriya governs the AI agents that act on your machine: every action passes a **policy check**, an
optional **human approval**, and a **budget cap**, and comes out the other side as a **signed,
tamper-evident receipt** you can re-verify yourself, offline. Everything runs on your device or
your own server — there is no vendor cloud to trust.

Status labels, used honestly:

- ✅ **Shipped** — in the current notarized DMG (v0.5.0).
- 🟢 **Merged** — built, tested, and merged on `main`; ships in the next DMG.
- 🧭 **Roadmap** — not built; we don't sell it.

Every ✅/🟢 claim traces to a test, a signed sample, or a public release —
[`FEATURE-PROOF.md`](FEATURE-PROOF.md) is the claim→proof ledger.

---

## 1 · See everything your agents do — ✅ free

- **Today** ⭐ *(new in v0.3.4)* — the default landing view: what your agents did in the last 24 h
  as human-readable headlines — *"Deployed kriya-api to Fly.io" · "Published a package to npm" ·
  "Blocked by the self-mod gate"* — with routine actions rolled up, the approvals that need you as
  cards, and the day's posture (agents active, receipt-chain state, pack in force, spend) in one
  strip. Every headline links to its full signed receipt; the headline never replaces the evidence.
- **Monitor** — a live feed of every action every governed agent takes on this machine. Each row is
  re-verified against its cryptographic signature as it arrives: you watch checked facts, not a
  trusted log file.
- **Audit log** — the full searchable history. An altered, forged, or wrong-key receipt is flagged
  red on sight; deleting or reordering records breaks a hash chain and is flagged too. *(New in
  v0.3.4)*: a **severity lens** — Headline / Notable / Denied / per-class chips filter the ledger
  and roll routine actions into per-hour rows; the classic full ledger stays the default. "Blocked"
  appears only where the runtime actually blocked and receipted it.
- **Receipt drawer** *(new in v0.3.4)* — one detail surface everywhere (Today, Audit, Approvals):
  the claim, the signature verdict, a live receipt-chain re-check, the full payload, and raw
  signed-line export for independent re-verification.
- **Coverage Map** — the honest view of what *isn't* recorded. Six lanes (Claude Code, remote MCP,
  local MCP, desktop apps, file & exec, network egress) each show GREEN / AMBER / GREY, and every
  change to the map is itself a signed receipt — nobody can quietly claim coverage they didn't have.
- **Sessions** — every governed run reconstructed as a tree from the signed receipts alone: *which
  session → which sub-agent → which action, in order*, with blocked attempts marked. Computed from
  verified receipts only; where a seam exposes no parent pointer (Claude Code's hook), none is
  invented — sub-agents group by their real `agent_id`. Run ids never leave the device: they live in
  receipt params, which the fleet-envelope minimizer structurally cannot read.
- **Verified Replay** ⭐ *(new in v0.3.2)* — step through a past session **re-derived from its signed
  receipts alone**. Re-derivation, never re-execution: no tool call, no model call, no network
  request is re-run anywhere in this feature. If any contributing receipt fails signature
  verification or a hash chain breaks, the whole derivation **refuses** — citing the exact source and
  line — rather than render a plausible-looking partial. Every field carries a FULL/ABSENT fidelity
  class, so a gap is something you see, never something the derivation invents.

- **Analytics** ⭐ *(new in v0.4.0)* — the aggregate read of your own verified receipts, three
  tabs: **Reliability** (actions, success rate, deny/hold split, denies/day + failures/day charts,
  top-failing tools, per-agent trends — everything deep-links into the Audit log), **SLOs**
  (approval latency p50/p95 per gate class, verification pass rate, deny rate, budget headroom,
  heartbeat gaps), and **Posture** — week-over-week threshold crossings, captioned verbatim:
  *"Evidence posture — counts from verified receipts. Not a risk score."* No composite score
  exists anywhere. A `success:false` action reads "did not complete", never "blocked". Charts are
  hand-rolled SVG, zero new dependencies; Spend gains a 30-day **Trend** and Local Models gain
  per-model **p50/p95 latency** the same release. *(New in v0.5.0)*: a fourth tab, **Topology** —
  agents → servers/tools → destinations & models folded from verified receipts, denies marked
  typographically, with the boundary stated on the map: *"governed lanes only — see Coverage for
  what isn't recorded."*
- **Sessions Timeline** ⭐ *(new in v0.5.0)* — each run's actions on a time axis with **real
  measured durations**: the hook lane times a call across its pre/post processes, the
  gateway/model lanes measure in-process, and the receipt carries two additive optional params.
  A bar renders only where a receipt recorded a duration; everything else is an honest instant
  marker — absent is shown, never invented.
- **Annotation receipts** ⭐ *(new in v0.5.0)* — attach a **signed human label** to any past
  verified action: Correct / Incorrect / Needs review / Unsafe — a closed set with **no free
  text anywhere** (a note would carry content). The latest label wins; every prior label stays
  signed in its own chain. The Audit log gains an **Annotated** lens; annotating a receipt that
  doesn't verify is refused, never signed.
- **Config-change evidence** ⭐ *(new in v0.5.0)* — Memory shows, per config surface (CLAUDE.md,
  settings, registered MCP memory), whether the device's **behavior baseline shifted within 48 h
  after each change** — worded as it is: *"shifted N h after this change (advisory)"* —
  correlation, never causation.
- **Evidence MCP server** ⭐ *(new in v0.5.0)* — `kriya-mcp --evidence`: your agent queries its
  own verified history (searches, chain checks, session trees, spend) through five read-only
  tools. Every answer comes **only from receipts that pass verification**; a tampered line is
  excluded and counted, never surfaced. The reader's own start is a signed receipt.
- **Verify it yourself, three ways** ⭐ *(new in v0.5.0)* — a per-lane **self-verifying HTML
  export** (opens in any browser, airplane mode), the `kriya-audit` CLI one-liners, and a raw
  3-command **OpenSSL recipe** needing only the public key — documented in
  [`VERIFY-OFFLINE.md`](VERIFY-OFFLINE.md) and executed in CI against a real signed fixture.
  Plus a **Grafana panel plugin** that re-verifies kriya-exported OTel spans in the dashboard's
  own browser: VERIFIED / BAD-SIG / NOT-A-KRIYA-SPAN.

## 2 · Control what agents may do — ✅ free

- **Policy** — ordered *allow / require-approval / deny* rules with live preview and linting. The
  Console authors the exact policy file the open runtime enforces. Fails closed on errors.
- **Action gates** ⭐ *(new in v0.3.4)* — the highest-stakes action classes, each behind one dial:
  **deploys** (fly/vercel/netlify/railway/kubectl/gcloud/aws + PaaS MCP deploy tools, prod vs
  preview split) · **destructive git** (force-push, `reset --hard`, branch delete, history rewrite —
  protected refs deny) · **publishes** (npm/PyPI/crates/RubyGems; dependency installs are receipted
  and aggregated, never blocked) · **production DB** (prod migrations and destructive SQL; branch
  DBs pass) · **infra** (terraform/pulumi apply & destroy, IAM/key mutations) · **outbound sends**
  (agent-sent email/messages, internal/external split) · **self-modification** (the agent editing
  its own hooks, settings, or `mcp.json` — **Deny by default**, the prompt-injection vector).
  Tiers: Allow / Receipt-only / Approve / Deny. **Enforced pre-execution in the Claude Code hook
  lane** as a tighten-only escalation — a gate can never loosen a rule your policy already denies;
  receipts from other lanes are classified and displayed, not blocked, and the UI says so. Every
  decision is a signed `kriya.gate.*` receipt. **Payments** joined as an eighth, fully-enforced
  class in v0.4.0 — see *Agent payment governance* below.
- **Policy packs** ⭐ *(new in v0.3.4)* — **Developer** (build freely; blast-radius steps need a
  human), **Analyst** (no side effects leave the machine; egress locked), **Planner** (draft
  everything, execute nothing). A pack is a named, versioned preset over the gates applied to the
  device in one click — and the application itself is a signed receipt. Duplicate-then-edit for
  custom packs; per-identity assignment is recorded for the launcher and fleet rollout to consume
  (one policy file per device today, and the UI says exactly that).
- **Agent payment governance** ⭐ *(new in v0.4.0)* — the `payment` gate class is a primary
  enforced dial: a payment-shaped call on the governed Claude Code hook lane produces a signed,
  **content-free** `kriya.pay.{intent,decision,outcome}` chain — the merchant host and a
  best-effort amount (an unreadable amount reads **"unknown", never a guess**), the decision
  against your per-txn cap and day spend, and the real outcome. A denied payment closes all three
  links synchronously; a held one stays the honest 2-of-3 shape until a human decides. A card
  number **never enters a receipt** — custody stays with credential brokering. Spend gains a
  **Purchases** tab; payment approvals show the amount against your cap. Enforced on the governed
  hook lane only — never sold as PCI or DLP.
- **Shift reports** ⭐ *(new in v0.4.0)* — declare an unattended window (default 22:00–07:00) and
  the Console composes a signed, chain-linked report of what the governed agent did across it —
  with a signed `kriya.attest.shift.gap` receipt for every heartbeat gap, **visible by absence,
  never smoothed over**. Arm the shift and the hook lane runs a **fail-closed clamp**: a missed
  beat inside the window tightens the action tier, each clamp its own signed receipt.
  Tighten-only; inert when disarmed. Honest scope: measurement of the governed record, not a
  promise nothing happened off-lane.
- **Test before apply** — replay a policy edit over this device's own re-verified receipts before
  applying it: *"this change would have changed N of last week's M actions — here they are."* Works
  on the single device and as the fleet pre-publish gate. Scope stated in the UI: the action-tier
  gate only (not budget/egress heuristics); the simulation is itself a signed receipt.
- **Approvals** — one queue across all agents, ranked by risk. Destructive and financial actions
  wait for a human; the decision, the person, and the reason become part of the signed record.
- **Budgets & rate caps** — a runaway agent stops at the cap, not at your data.
- **Identity** — who ran what, per human operator and per agent, computed only from verified
  receipts. Console roles: admin / approver / operator / viewer.
- **Local model governance** *(new in v0.3.2)* — `kriya-llm-proxy` sits in front of your local model
  server (Ollama, llama.cpp, vLLM, LM Studio) and receipts every completion: which model digest
  served it, token usage, prompt and output hashes. Policy gains a `model:` dimension so an
  unapproved model identity can be gated. Honest ceiling: this governs **clients you point at the
  proxy** — a client connecting straight to the model port is invisible, and the Coverage Map says so.

## 3 · Wire it up without pain — ✅ free

- **Govern All** — one button: detect the agents on this Mac (Claude Code, Hermes, **Cursor, Cline,
  GitHub Copilot, Gemini CLI**) and wire hooks + gateway + policy for all of them, one click each.
  Idempotent and reversible — un-govern restores every config byte-for-byte.
- **Connections** — add governed MCP servers without hand-editing JSON; walks you through macOS
  permissions.
- **Governed run launcher** ⭐ *(new in v0.4.0)* — Start › **New governed run** composes an agent +
  policy pack + lanes into one `kriya-run …` command; the open runtime's bin signs a single
  `kriya.run.launched` attestation (content-free — never argv or cwd), then starts the agent.
  Copy-first: a launcher, not a second enforcement path; per-call governance stays in the agent's
  own hook. Sessions run cards gain a `pack:` chip.
- **Broad coverage** — Claude Code (native hook: every tool call, including subagents and headless
  runs), Hermes, the VS-Code-family and CLI MCP clients above via the gateway, any MCP server via
  the zero-change gateway, and no-API desktop apps via computer-use. Vendor-neutral by design.
  Honest ceiling where it's shown: for broker-wired clients the *MCP lane* is governed — their
  native built-in tools bypass MCP unless launched under containment, and cloud-executed agents
  (Cursor background, Copilot's cloud coding agent) are out of scope.
- **In-process agents & CI** (open runtime) — SDK middleware wraps a framework's tool callback
  (LangGraph, OpenAI Agents SDK, CrewAI, Claude Agent SDK — TypeScript + Python) so in-process tool
  calls are gated and receipted with no MCP hop; **`kriya-ci`** runs an agent step in CI under a
  repo-committed policy, fails the build on a policy block, and leaves the signed receipts as a
  build artifact anyone re-verifies offline. Both are cooperative lanes and say so.

## 4 · Prove it to an auditor — ✅ paid (CLI free)

- **Evidence export** — controls across 6 frameworks (NIST 800-171 / CMMC L2, SOC 2, ISO 42001,
  EU AI Act, data-residency, and **ISO/IEC 42001 + CSA AICM** new in v0.3.2). Every control's status
  is **computed from re-verified receipts**, never typed in. Gaps are shown, not hidden.
- **ISO 42001 / CSA AICM pack** *(new in v0.3.2)* — maps the verified trail to ISO/IEC 42001 Annex A
  (A.6.2.8, A.6.2.6, A.9.4) and CSA AICM domains, plus an **AI-CAIQ support sheet**. Deliberately
  conservative: every Annex A row is capped at ◐ *partial*, eight controls are listed by name as
  NOT COVERED, and a wording lint bans "compliant" / "certified" / "guarantees" from the output.
- **Auditor CLI** (`kriya-audit`, free) — *don't trust us: check.* A standalone offline tool any
  assessor can run to re-prove receipts, fleet envelopes, and a live server read-back. Exit 0 or 1.
  v0.3.2 adds five verifiers: `--verify-replay`, `--verify-otel`, `--verify-artifact`,
  `--verify-bundle`, `--verify-exec`.
- **Machine posture** — one number per Mac: verified vs failed, signers seen, share of actions
  under policy.

## 5 · Run a whole fleet from your own server — ✅ paid

- **`kriyad` aggregator** — one static binary on **your** infrastructure: box, Kubernetes, or fully
  air-gapped. It verifies everything it ingests, stores only signed bytes, and holds no signing
  keys — a compromised server can delay evidence, never forge it.
- **Fleet cockpit** — every enrolled device on one screen: alive or silent, versions, which agents
  are wired. Drill into any device's signed evidence chain.
- **Central policy, signed** — author once, sign with an org key only you hold; every device
  verifies, applies, and emits a signed "applied" receipt. Anti-rollback built in.
- **Drift view** — "is the fleet actually on policy v13?" answered from each device's own signed
  statements, not the server's word.
- **Org-wide evidence** — the export a CMMC assessor asks the organization for, computed across
  every machine; silent devices are named honestly as red cells.
- **Privacy by structure** — what leaves each device is a minimized, allowlisted summary. Raw
  parameters and operator names **cannot** leave: the schema has no field to put them in.

## 6 · Control what leaves the machine — ✅ shipped in v0.2.4

The egress pack closes the loop from *"what did the agent do"* to *"what did it send, and to
whom."*

- **Egress allowlist** — deny-by-default outbound rules: agents talk only to hosts you listed.
- **"No receipt, no egress"** ⭐ — the inversion nobody else has: the signed receipt is a
  **precondition** of the network call. If the tamper-evident record can't be written, the byte
  doesn't leave. Others log what their firewall did; kriya makes the proof the gate.
- **Byte budgets** — per-destination caps that catch slow-drip exfiltration, not just big leaks.
- **Ask-before-send** — unknown destination? The call parks until a human decides, in the same
  approval queue as everything else.
- **Secret & PII scanning** — outbound bodies scanned for credentials and personal data; redact or
  deny per policy. Only a hash and a match-type are stored — never the secret.
- **Exfiltration detection** — DNS-tunnel patterns, anomalous destinations, and canary tokens that
  trip a signed alarm the moment they leave.
- **SSRF & rebinding guard** — private-IP, cloud-metadata, and DNS-rebinding attempts blocked on
  governed lanes.
- **Connector registry** — a new MCP server or tool is disabled until a human approves it; tool
  descriptions are scanned for drift and poisoning.
- **Operation rails** — allow or deny specific outbound operations, down to the HTTP verb, path,
  or GraphQL mutation.
- **Credential brokering** — the agent holds a placeholder; the real secret (kept in the OS
  keychain) is injected only at the moment of egress. The agent never sees your keys. Own threat
  model: [`THREAT-MODEL-brokering.md`](THREAT-MODEL-brokering.md).
- **OS containment** — launch an agent with `kriya-gateway run -- <agent>` and a generated macOS
  Seatbelt profile forces its traffic through the governed lane — turning the controls above from
  *observed* into *enforced* for everything kriya launches.
- **Fleet egress + kill switch** — the allowlist, budgets, kill-switch, and egress evidence,
  distributed and rolled up across the fleet under the same signed policy bundle.
- **Fleet destination visibility** — a privacy-minimized pattern-echo of destinations in the
  fleet envelopes, so the cockpit can answer "which hosts is the fleet talking to" without raw
  parameters ever leaving a device.

**Honest scope, stated first:** these controls cover the governed lanes — what routes through
kriya's hook, gateway, and broker, plus anything launched under containment. A determined agent
spawning raw processes outside a contained session can bypass a lane; the Coverage Map shows that
honestly instead of papering over it. kriya does not claim host-wide DLP or network-boundary
enforcement. Full limits: [`TRUST.md`](TRUST.md).

## 7 · Know what your agents cost — ✅ shipped in v0.3.2, free

Every other tool that reports AI cost reads an invoice or sits on a gateway. Neither can see a
**subscription seat** — the flat-fee Claude Code plan your engineers actually work on. kriya
reconstructs it from the transcript already on the device, and signs the result.

- **Spend receipts** — what each governed Claude Code session actually cost, reconstructed on-device
  from the local transcript JSONL, priced by a bundled offline sheet, and signed into its own hash
  chain (`kriya.spend.session`, plus a `kriya.spend.rollup` daily checkpoint that cross-checks the
  total and can never inflate it). A **priced estimate**, never an invoice or a chargeback record.
- **Zero content, by construction** — only token usage, model id, message/request/session ids,
  timestamp, and the sidechain flag are ever read. Never message content, tool inputs or outputs,
  file paths, or the raw project path (the project is identified by a hash of the directory *name*).
  Extraction runs entirely in Rust, so transcript bytes never cross into the UI layer at all — the
  strongest structural form of the rule, not a promise to behave.
- **Dollar budget gates** — budgets per session, per rolling day, and per user, consulted **before
  the next action runs**: a breach denies it, routes it to human approval, or warns, per policy. The
  decision is itself a signed receipt (`kriya.spend.gate.*`) that records **how stale** the spend
  figure was at decision time.
- **Deliberately not a hard cap** — this is a trailing-state gate: spend is known from settled
  transcripts, so a burst inside the settle window can exceed a budget before the gate sees it. The
  copy is lint-enforced never to say "hard cap", "guarantee", "prevent", or "cannot exceed".
- **Unpriced stays unpriced** — a model the bundled sheet doesn't recognize is reported as unpriced,
  never guessed and never silently counted as zero.

## 8 · Hand the proof to a stranger — ✅ shipped in v0.3.2, free

Everything above is verifiable *inside* the Console. This section is evidence that leaves the
Console and stays checkable — by someone who has no kriya install, no trust in you, and no network.

- **Replay bundle** — one file (`<run>.kriya-replay.json`) holding the manifest, the verbatim raw
  signed logs, the derived timeline, and the origin's own verdict. `kriya-audit --verify-replay`
  re-derives from the bundle's **own** logs and hash-compares; it never trusts the bundle's
  self-reported claims. Cross-language parity is CI-enforced against a checked-in golden fixture.
- **OpenTelemetry bridge** — every receipt maps to a conformant OTel **GenAI** span carrying its own
  signature and chain position, so your observability stack can hold kriya's telemetry and a stranger
  can still verify it (`kriya-audit --verify-otel`). Third-party GenAI spans import back as clearly
  marked **unsigned observations** with a governance-gap count, so an unsigned span can never be
  mistaken for a governed one. **Scope: file export and import. There is no live collector push and
  no listener** — that half is not built.
- **C2PA artifact provenance** — files written by a governed session get a `.c2pa` sidecar manifest
  naming the session, agent, and policy verdicts, cryptographically bound to the receipt chain and
  checkable with `kriya-audit --verify-artifact`. Self-managed anchor: public C2PA validators will
  correctly report `signingCredential.untrusted` — expected and documented, not a bug. Provenance
  for those who keep it; **not** a watermark, and not EU AI Act Article 50 compliance.
- **Diode bundle** — a chunked, signed, non-executable bundle (`kriya-diode-bundle/1`) built to cross
  a real data diode or sneakernet. The receiver verifies everything and detects **gaps or forks**
  between successive bundles with no channel back to the sender (`kriya-audit --verify-bundle`).
  Spec: [`DIODE-BUNDLE.md`](DIODE-BUNDLE.md). A missing chunk is reported by index — erasure-code
  reconstruction is v2 and is not built.
- **Deterministic execution lane** — an opt-in lane where a tool runs as a WASI-p2 component under
  pinned Wasmtime with fuel metering on, NaN canonicalization on, threads off, and **no ambient
  clocks or randomness** (both are virtualized functions of a recorded seed). The run is captured as
  a `kriya-exec-bundle/1` so anyone can **re-run it** and hash-compare bit-for-bit
  (`kriya-audit --verify-exec`). Scoped strictly to tools that ran in this lane — this is
  re-*execution*, a different and narrower claim than Verified Replay's re-*derivation*.

## 9 · Crypto and integrity for procurement — ✅ shipped in v0.3.2, free

Not moats — procurement checkboxes, converted before someone raises them. **Both crypto lanes are
opt-in builds, off by default; the notarized DMG is byte-unchanged by them.**

- **FIPS signing lane** — an optional build where receipt signing and verification run through
  `aws-lc-rs`'s FIPS module (AWS-LC-FIPS 3.x, CMVP cert **#5298**, FIPS 140-3 Level 1, Ed25519 inside
  the approved boundary). Wording is exact by policy: on a CMVP-tested operational environment it is
  a *FIPS 140-3 validated module (cert #5298)*; on macOS it is **the same cryptographic module code
  as cert #5298, running outside a CMVP-tested operational environment**. Never "a FIPS-certified
  product." Assessor one-pager: [`samples/fips-module-boundary.md`](samples/fips-module-boundary.md).
- **Post-quantum dual-signature** — an optional second, quantum-resistant signature (**ML-DSA-87**,
  CNSA 2.0-aligned) alongside Ed25519, so evidence retained for years survives the procurement
  question. Default checkpoint mode seals a chain checkpoint every 256 receipts (~1.03–1.13×
  storage); a per-receipt mode exists for high-value receipts. Post-quantum-**ready**, never
  "quantum-proof" — and mutually exclusive with the FIPS lane in this build (an upstream constraint,
  asserted by a CI job).
- **Pipeline integrity** — kriya records a signed **measurement** of itself: its own binary hash, the
  wired hook and gateway binaries, the active policy hash, and the macOS code-signature check, plus
  the sandbox profile each contained run used. Settings → Integrity shows drift against the previous
  measurement. **Measurement, not attestation** — there is no hardware root of trust here, and a
  fully compromised host can lie about its own measurement. TPM and SEV-SNP tiers are not built.

---

## 10 · Watch what agents remember, and notice when they change — ✅ shipped in v0.3.3, free

Sections 1–9 record what an agent *did*. This one covers the two things that outlast a single
action: what the agent wrote into its **own memory**, and whether it still **behaves** the way it did.

- **Memory-write receipts** — every write to a governed memory surface (`CLAUDE.md`,
  `CLAUDE.local.md`, the Claude memory dir, `.claude/settings*.json`, and MCP memory tools you have
  explicitly **registered**) produces a signed receipt carrying a **hash of the bytes, never the
  content**. `write` vs `update` is derived from tool shape plus a bounded scan for an earlier write
  to the same path, and every receipt discloses which basis was used rather than implying more
  certainty than a stateless hook has. An MCP tool that merely *looks* like a memory tool but was
  never registered mints nothing — the honest default.
- **The Memory view** — per-target timelines with verification ticks, plus a provenance badge
  (`untrusted ingress present` / `trusted only` / `none observed`) that joins the run's ingress
  receipts against your own MCP trust-class policy. **The badge is inert until you turn on ingress
  recording or install a trust-class control — both default off**, and the view says so.
  **Observe and evidence, never block.** kriya makes **no claim to detect semantic memory poisoning**.
- **Temporal policy conditions** — a policy rule may require a precondition met *earlier in the same
  session*: the canonical case is *"deny a governed `git push` unless a governed `npm test` succeeded
  this session."* Conditions **tighten only** — they can escalate an allow to approval or deny, and
  can never loosen an existing deny. Scoped to one session and one lane, evaluated over that lane's
  own signed receipts, and **fails closed** when the session log can't be read.
- **Behavior baselines** — for each agent, kriya freezes a baseline from that agent's own first
  signed receipts across four dimensions (tool mix, unsuccessful outcomes, egress class mix, spend
  shape), then runs an anytime-valid sequential test over what follows and signs an advisory receipt
  when the evidence against "same distribution" crosses a pre-registered threshold. **Advisory only —
  there is no enforcement lever, by design**, and a flagged shift means "this stopped resembling the
  baseline," never "this is bad." The surface reports a **threshold crossing and an effect size,
  never a probability**: real agent traffic is bursty, which pushes the false-alarm level well above
  the nominal one, so quoting a rate would be dishonest. Not to be confused with the fleet **Drift
  view** (§5), which answers whether machines are on the current signed policy bundle.

---

## Free vs paid

| | Free (no account, no license, no network) | Paid (offline license) |
|---|---|---|
| Monitor, Audit, Coverage Map, Sessions | ✅ | ✅ |
| Policy, Approvals, Budgets, Identity | ✅ | ✅ |
| Govern All, Connections, egress controls | ✅ | ✅ |
| Spend receipts + dollar budget gates | ✅ | ✅ |
| Verified Replay + replay-bundle export | ✅ | ✅ |
| OTel export · C2PA provenance · diode bundles | ✅ | ✅ |
| Local model governance (`kriya-llm-proxy`) | ✅ | ✅ |
| Memory-write receipts + Memory view | ✅ | ✅ |
| Temporal policy conditions | ✅ | ✅ |
| Behavior baselines | ✅ | ✅ |
| FIPS + post-quantum build lanes | ✅ | ✅ |
| Auditor CLI (all six verifiers) | ✅ | ✅ |
| Evidence export (6 frameworks) | — | ✅ |
| Fleet cockpit + `kriyad` control plane | — | ✅ |
| Fleet OTel export + `/metrics` + Grafana | — | ✅ |

The license is an Ed25519-signed offline token — no phone-home, no accounts.
Releases: [`CHANGELOG.md`](../CHANGELOG.md) ·
Setup: [kriyanative.com/docs/console-setup](https://kriyanative.com/docs/console-setup/) ·
Site: [kriyanative.com](https://kriyanative.com)
