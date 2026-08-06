# kriya Console

The control cockpit built on the open-source [kriya](https://github.com/governex/kriya)
runtime (MIT). **The engine is open; the cockpit ships as a signed, notarized app — free tier
free forever, paid features unlock with an offline license.** This repository is the Console's
public home: downloads under [Releases](https://github.com/governex/kriya-console/releases),
documentation, and verifiable samples. The Console's source is not published — what you *can*
verify, offline and without trusting us, is every receipt it produces ([license](LICENSE)).

> **Your AI agents act on your machine. kriya controls what they can do — and gives you signed
> proof of what they did.** Everything runs on your device or your own server. Nothing goes to
> our cloud, because we don't have one.

**Current release: v0.4.0** — signed + notarized macOS app: **agent payment governance** (the
`payment` gate is a primary enforced dial; every governed payment a signed, content-free
**intent → approval → transaction** chain — honest amounts, never a card number), **shift
reports** (signed unattended-window evidence + heartbeat gaps, with an armed fail-closed clamp on
the hook lane), a **governed run launcher** (`kriya-run` composes a pack + lanes + agent and signs
one launch attestation before the agent starts), and a new **Analytics** view (reliability,
governance-native SLOs, and a posture panel that is deliberately *not* a risk score) — on top of
v0.3.4's **Today** landing view, **action gates**, and **policy packs**.
[Download the latest DMG](https://github.com/governex/kriya-console/releases/tag/v0.4.0) ·
[kriyanative.com](https://kriyanative.com)

---

## The problem

You let an AI agent loose on your laptop — Claude Code, Hermes, an MCP server, a desktop app.
It edits files, calls APIs, spends money, talks to the internet. Then someone asks a simple
question:

> *"What exactly did the agent do — and can you prove it?"*

A chat transcript is not proof. A plain log file can be edited after the fact. And "we trust the
vendor's dashboard" doesn't work when your rules say agent activity can't leave the building at
all (defense, government, healthcare, banks, air-gapped sites).

kriya answers that question with cryptography instead of trust: **every agent action passes
through a policy check, an optional human approval, and a budget cap — and comes out the other
side as a signed, tamper-evident receipt you can re-verify yourself, offline.**

## What kriya does, in one minute

- **See** — open the app to **Today**: what your agents did in the last 24 h as verified
  headlines — *deployed to prod, published a package, cut a release, got blocked* — with routine
  actions rolled up, plus a live signature-checked view of every action. Edited or faked entries
  show up red.
- **Control** — you write the rules: *allow this, ask a human for that, never allow those.* The
  open runtime enforces them. **Action gates** put the highest-stakes classes (deploys,
  destructive git, publishes, prod DB, infra, sends, self-modification, and — new in v0.4.0 —
  **payments**) each behind Allow / Receipt / Approve / Deny; **packs** apply a whole persona's
  posture in one receipted click. Runaway agents hit rate and budget caps.
- **Approve** — dangerous actions (deleting things, moving money) pause until a person says yes.
  Who approved, and why, becomes part of the permanent record.
- **Prove** — one click turns the verified record into compliance evidence (CMMC/NIST 800-171,
  SOC 2, ISO 42001 + CSA AICM, EU AI Act) that an auditor can independently re-check with a free
  CLI tool.
- **Cost** — what each governed session actually spent, reconstructed on-device from local
  transcripts and signed. Dollar budgets gate the next action before it runs.
- **Hand off** — export a replay bundle, an OpenTelemetry span set, a C2PA artifact manifest, or a
  diode bundle. A stranger re-checks any of them offline, with no kriya installed.
- **Scale** — the same story across a whole fleet of machines, aggregated on **your** server
  (`kriyad`), on-prem or fully air-gapped. Never our cloud.

## Quick start

**Use it (macOS):**

1. [Download the DMG](https://github.com/governex/kriya-console/releases/latest) and open **kriya Console**.
2. Click **Govern All**. The Console detects the agents on your Mac (Claude Code, Hermes, Cursor,
   Cline, GitHub Copilot, Gemini CLI) and wires them into governance in one click — reversible, no
   config files to edit.
3. Use your agents as normal. Watch the **Monitor** view fill with live, signed, verified receipts.

The free tier needs no account, no license, and opens no network connection — that's
[dormancy-tested](docs/TRUST.md), not a promise.

Every feature in plain words: [`docs/FEATURES.md`](docs/FEATURES.md). Every claim mapped to its
proof: [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md). Release history:
[`CHANGELOG.md`](CHANGELOG.md). Setup with screenshots:
[kriyanative.com/docs/console-setup](https://kriyanative.com/docs/console-setup/).

## What it does

### 1. See everything your agents do

| Feature | In plain terms |
|---|---|
| **Monitor** (home view) | Opens on a live feed of agent activity on this machine. Every entry is re-verified against its signature as it arrives — you're watching checked facts, not trusted logs. |
| **Audit log** | The full searchable history. Any receipt that was altered, forged, or signed by an unexpected key is flagged red on sight. Deleting or reordering records breaks a hash chain and gets flagged too. |
| **Coverage Map** | The honest view of what *isn't* recorded. Six lanes (Claude Code, remote MCP, local MCP, desktop apps, file & exec, network egress) each show GREEN / AMBER / GREY — and every change to that map is itself a signed receipt, so nobody can quietly claim coverage they didn't have. |
| **Sessions** | Agent work grouped into run trees — a session, its subagents, and every tool call underneath, with the lineage drawn from signed receipts rather than guessed from timestamps. |
| **Verified Replay** ⭐ *(new in v0.3.2)* | Step through a past session **re-derived from its signed receipts alone** — no tool call, no model call, nothing re-run. If a receipt fails verification or a chain breaks, the whole derivation refuses rather than render a plausible-looking partial. Missing fields show as visible gaps, never invented values. |
| **Memory** ⭐ *(new in v0.3.3)* | What your agents wrote into their own persistent memory — `CLAUDE.md`, the Claude memory dir, `.claude/settings*.json`, registered MCP memory tools — as signed, **hash-only** receipts. Flags when a run that wrote to memory also read from a source you've classed as untrusted. |
| **Analytics** ⭐ *(new in v0.4.0)* | The aggregate read of your own verified receipts — reliability (actions, success rate, deny/hold split, top-failing tools), governance-native **SLOs** (approval-latency p50/p95, verification pass rate, heartbeat gaps, budget headroom), and a week-over-week **posture** panel. Every number is a count or a threshold; there is **no composite risk score** — the panel says so verbatim. A `success:false` action reads *"did not complete,"* never *"blocked."* View-side only: nothing new is signed. |

### 2. Control what agents are allowed to do

| Feature | In plain terms |
|---|---|
| **Policy** | Write ordered rules — *allow*, *require approval*, *deny* — with live preview and linting. The Console authors the exact policy file the open runtime enforces. Fails closed on errors. |
| **Approvals** | One queue, across all agents, ranked by risk. Destructive and financial actions wait for a human; the decision, the person, and the reason are recorded and signed. |
| **Budgets & rate caps** | A runaway agent stops at the cap, not at your data. Per-app, per-agent, per-operator usage against limits, visible at all times. |
| **Identity & access** | Who ran what — per human operator and per agent — computed only from verified receipts. Console roles: admin / approver / operator / viewer. |
| **Local model governance** *(new in v0.3.2)* | Point Ollama / llama.cpp / vLLM / LM Studio clients at `kriya-llm-proxy` and every completion gets a receipt — which model digest served it, token counts, prompt and output hashes — and policy can gate on approved model identity. Governs the clients you point at the proxy; a direct connection to the model port stays invisible, and the Coverage Map says so. |
| **Payment governance** ⭐ *(new in v0.4.0)* | The `payment` gate is now a **primary enforced dial** (Allow / Receipt / Approve / Deny): a payment-shaped call on the governed Claude Code hook lane produces a signed, content-free **intent → approval → transaction** chain — merchant host and a best-effort amount (honest *unknown* when it can't be read cleanly, never a guess), **no card number ever in a receipt** (custody stays with credential brokering). A new Spend › **Purchases** tab lists each governed payment; approvals show the amount against your cap. Enforced on the hook lane only — never sold as PCI or DLP. |
| **Shift reports** ⭐ *(new in v0.4.0)* | Sign a report of what a governed agent did across an **unattended window** — plus honest **gaps** wherever the heartbeat went quiet. Arm a shift and the hook lane runs a **fail-closed clamp**: a missed heartbeat inside the window tightens the tier (an otherwise-allowed action holds or is denied), each decision itself a signed receipt. Tighten-only, and inert when disarmed. |

### 3. Wire it up without pain

| Feature | In plain terms |
|---|---|
| **Govern All** | One button: detect the agents on this Mac and wire hooks + gateway + policy for all of them. Reversible. |
| **Connections** | Add and manage governed MCP servers without hand-editing JSON config files; walks you through macOS permissions. |
| **Broad agent coverage** | Claude Code (native hook — every tool call, including subagents and headless runs), Hermes, any MCP server via the zero-change gateway, and no-API desktop apps via computer-use. Vendor-neutral by design. |
| **Governed run launcher** ⭐ *(new in v0.4.0)* | Start › **New governed run** composes an agent + a policy pack + the lanes you want into a single `kriya-run …` command and signs one launch attestation before the agent starts — so a run's pack is on the record from the first byte. Copy-first: it records how the run was launched; per-call governance still flows through the agent's hook. |

### 4. Prove it to an auditor *(paid tier)*

| Feature | In plain terms |
|---|---|
| **Evidence export** | Controls across 6 frameworks — NIST 800-171 / CMMC L2 (AU 3.3.1–3.3.9), SOC 2, **ISO/IEC 42001 + CSA AICM** *(new in v0.3.2)*, EU AI Act, data-residency. Every control's status is **computed from re-verified receipts**, never typed in. Gaps are shown, not hidden — the ISO 42001 pack caps every Annex A row at *partial* and names eight controls kriya cannot evidence at all. Markdown + JSON, plus an AI-CAIQ support sheet. |
| **Auditor CLI** (`kriya-audit`, free) | *Don't trust us — check.* A standalone offline tool any assessor can run to re-prove the receipts, the fleet envelopes, and a server read-back. Exit 0 or exit 1. v0.3.2 adds five more verifiers: `--verify-replay`, `--verify-otel`, `--verify-artifact`, `--verify-bundle`, `--verify-exec`. |
| **Fleet correlation (this machine)** | One posture number per Mac: verified vs failed, signers seen, share of actions under policy. |

### 5. Run a whole fleet from your own server *(paid tier)*

| Feature | In plain terms |
|---|---|
| **`kriyad` aggregator** | One static binary on **your** infrastructure — box, Kubernetes, or fully air-gapped. It verifies everything it ingests and stores only signed bytes. It holds no signing keys, so it **can author nothing**: a compromised server can delay evidence, never forge it. |
| **Fleet cockpit** | Every enrolled device on one screen: alive or silent, versions, which agents are wired (and which aren't). Drill into any device's signed evidence chain. |
| **Central policy, signed** | Author policy once, sign it with an org key **only you hold**, and the fleet converges on it — every device verifies the signature, applies locally, and emits a signed "applied" receipt. Anti-rollback built in. An admin never walks 500 laptops. |
| **Drift view** | "Is the fleet actually on policy v13?" answered from each device's own signed statements — not the server's word. Disagreements get a loud mismatch badge. |
| **Org-wide evidence** | The export a CMMC assessor asks the *organization* for: fleet coverage (silent devices named honestly as red cells), AU-family + CM-family, computed across every machine. |
| **Privacy by structure** | What leaves each device is a minimized, allowlisted summary — raw parameters and operator names **cannot** leave; the schema has no field to put them in. Operators become pseudonyms. Survives a works-council review. |
| **Your Grafana reads kriya** ✅ *(new in v0.3.3)* | `kriyad` speaks OpenTelemetry at the fleet boundary: one signed span per accepted evidence envelope, exported **to your own collector, inside your own boundary** — we never host or see a copy. Plus aggregate governance KPIs on `GET /metrics` and an importable Grafana dashboard. Off by default; turning it on is itself a signed receipt. Metrics carry a **device id only** — never a hostname or a username. |

### 6. Control what leaves the machine — egress governance ✅ *(shipped in v0.2.4)*

The egress pack closes the loop from *"what did the agent do"* to *"what did the agent send, and
to whom"* — at full feature parity with the strongest egress tools, plus one thing nobody else
has.

| Feature | In plain terms |
|---|---|
| **Egress allowlist** | Deny-by-default outbound rules by destination: agents talk only to the hosts you listed. |
| **"No receipt, no egress"** ⭐ | The kriya-native inversion: the signed receipt is a **precondition** of the network call. If the tamper-evident record can't be written, the byte doesn't leave. Competitors log what their firewall did; kriya makes the proof the gate. |
| **Byte budgets** | Per-destination caps that catch slow-drip exfiltration, not just single big leaks. |
| **Ask-before-send** | Unknown destination? The call parks and a human decides — same approval flow as everything else. |
| **Secret / PII scanning** | Outbound bodies scanned for credentials and personal data: redact or deny per policy. Only a hash and a match-type are ever stored — never the secret itself. |
| **Exfiltration detection** | DNS-tunnel patterns, anomalous destinations, canary tokens that trip a loud alarm the moment they leave. |
| **SSRF & rebinding guard** | Private-IP, cloud-metadata, and DNS-rebinding attempts blocked on governed lanes. |
| **Connector registry** | A new MCP server or tool is **disabled until a human approves it**, and tool descriptions are scanned for drift and poisoning. |
| **Operation rails** | Allow or deny specific outbound API operations — down to the HTTP verb, path, or GraphQL mutation. |
| **Credential brokering** | The agent holds a placeholder; the real secret is injected only at the moment of egress. The agent never sees your keys. |
| **OS containment** | `kriya-gateway run -- <agent>` launches an agent inside a macOS Seatbelt sandbox that *forces* its traffic through the governed lane — turning all of the above from "observed" into "enforced" for everything kriya launches. |
| **Fleet egress + kill switch** | The allowlist, budgets, kill-switch, and egress evidence, distributed and rolled up across the fleet in the org-signed policy bundle. |

Honest scope, stated up front: these controls cover the **governed lanes** (what routes through
kriya's hook, gateway, and broker, plus anything launched under containment) — a determined agent
spawning raw processes outside a contained session can bypass a lane, and the Coverage Map shows
that honestly. Limits: [`docs/TRUST.md`](docs/TRUST.md).

### 7. Know what your agents cost — governed spend ✅ *(new in v0.3.2)*

Every other tool that tells you what AI cost you reads an invoice or sits on a gateway. Neither can
see a **subscription seat** — the flat-fee Claude Code plan your engineers actually run on. kriya
reconstructs it from the transcript already on the device, and signs the result.

| Feature | In plain terms |
|---|---|
| **Spend view** | What each governed Claude Code session actually cost — reconstructed on-device from the local transcript, priced by a bundled offline sheet, and signed into its own hash chain. A **priced estimate**, never an invoice. |
| **Zero-content by construction** | Only token counts, model id, and timestamps are ever read — never message content, tool inputs/outputs, file paths, or the project path string. Extraction runs entirely in Rust, so transcript bytes never reach the UI layer at all. |
| **Dollar budget gates** | Budgets per session, per rolling day, per user, checked **before the next action runs** — breach denies it, routes it to a human, or warns. The decision is itself a signed receipt that records how stale the figure was. A trailing-state gate, deliberately *not* described as a hard cap. |
| **Unpriced is unpriced** | A model the bundled sheet doesn't recognize is reported as unpriced — never guessed, never silently zero. |

### 8. Hand the proof to a stranger — portable evidence ✅ *(new in v0.3.2)*

Everything above is verifiable *in the Console*. This section is about evidence that leaves the
Console and stays checkable — by someone who doesn't have kriya, doesn't trust you, and is offline.

| Feature | In plain terms |
|---|---|
| **Replay bundle** | One file (`<run>.kriya-replay.json`) holding the manifest, the raw signed logs, the derived timeline, and the origin's own verdict. `kriya-audit --verify-replay` re-derives from the bundle's *own* logs and hash-compares — it never trusts what the bundle claims about itself. |
| **OpenTelemetry bridge** | Every receipt maps to a conformant OTel **GenAI** span carrying its own signature and chain position, so your observability stack can hold kriya's telemetry and a stranger can still verify it. Third-party GenAI spans import back as clearly-marked *unsigned observations* with a governance-gap count. **File export and import — there is no live collector push.** |
| **C2PA artifact provenance** | Files written by a governed session get a C2PA sidecar manifest naming the session, agent, and policy verdicts, bound to the receipt chain. Self-managed anchor: public C2PA validators correctly report an untrusted signing credential — that's expected, and [documented](docs/TRUST.md). |
| **Diode bundle** | A chunked, signed, non-executable bundle built to cross a real data diode or sneakernet. The receiver verifies everything and detects gaps or forks between successive bundles with **no channel back**. A missing chunk is reported by index — reconstruction is not built. |
| **Deterministic execution lane** | An opt-in lane where a tool runs as a WASI component under pinned Wasmtime with fuel metering, canonicalized NaNs, and virtualized clocks and randomness — then anyone can **re-run it** and hash-compare bit-for-bit. Scoped strictly to tools that ran in this lane. |

### 9. Crypto and integrity for procurement ✅ *(new in v0.3.2)*

Not moats — checkboxes, converted before someone raises them. Both crypto lanes are **opt-in builds,
off by default**; the notarized DMG above is byte-unchanged by them.

| Feature | In plain terms |
|---|---|
| **FIPS signing lane** | An optional build where signing and verification run through the AWS-LC-FIPS module (CMVP cert **#5298**). On a validated operational environment that is a *FIPS 140-3 validated module*; on macOS it is honestly described as **the same cryptographic module code as cert #5298, running outside a CMVP-tested operational environment**. Never "a FIPS-certified product." |
| **Post-quantum dual-signature** | An optional second, quantum-resistant signature (**ML-DSA-87**, CNSA 2.0-aligned) alongside Ed25519, so evidence retained for years survives the procurement question. Checkpoint mode seals every 256 receipts at ~1.1× storage. Mutually exclusive with the FIPS lane in this build. |
| **Pipeline integrity** | kriya records a signed **measurement** of itself — its own binary hash, the wired hook and gateway binaries, the active policy hash, the macOS code-signature check — plus the sandbox profile each contained run used, so drift is visible. **Measurement, not attestation**: there is no hardware root of trust, and a fully compromised host can lie. |

### 10. Watch what agents remember — and notice when they change ✅ *(new in v0.3.3)*

Everything above records what an agent *did*. This release is about the two things that outlast a
single action: what the agent **wrote into its own memory**, and whether its behaviour is still what
it used to be.

| Feature | In plain terms |
|---|---|
| **Memory-write receipts** | Every write to a governed memory surface — `CLAUDE.md`, `CLAUDE.local.md`, the Claude memory dir, `.claude/settings*.json`, and operator-**registered** MCP memory tools — gets a signed receipt. **Hashes only, never memory content.** A new **Memory** view shows the per-target timeline, and a provenance badge escalates to *"untrusted ingress present"* when the same run also pulled from a server you've classed as untrusted. This **observes and evidences — it never blocks**, and it does not claim to detect semantic memory poisoning. |
| **Temporal policy conditions** | A rule can now require that something already happened *earlier in this session* before it allows an action — the canonical one being *"don't allow a governed `git push` unless a governed `npm test` succeeded in this session."* Conditions **tighten only**: a temporal rule can escalate an allow to approval or deny, never loosen a deny. Session-scoped and per-lane, and it **fails closed** if the session log can't be read. |
| **Behavior baselines** | For each agent, kriya forms a baseline from that agent's own first signed receipts — which tools it reaches for, how often things fail, its egress mix, its spend shape — then flags when later behaviour stops resembling it. **Advisory only: there is no enforcement lever, by design.** The card reports a *threshold crossing*, never a probability, because on bursty real-world traffic the false-alarm level rises well above the nominal one — so we don't quote one. |

*(Not to be confused with the fleet **Drift view** above, which answers a different question: whether
your machines are on the current signed policy bundle.)*

## How it works

```
your agents                 the open kriya runtime (MIT)                 kriya Console (this app)
Claude Code · Hermes        every action:                                re-verifies every receipt
MCP servers · desktop  ──▶  policy → approval → budget → Ed25519-  ──▶   on-device · authors policy
apps                        signed, hash-chained receipt                 approvals · evidence export
```

And for a fleet:

```
each device (Console)                your kriyad (on-prem / air-gap)          operator (Console cockpit)
verified receipts                    mTLS · verifies all ingest               fleet · drift · org evidence
 └→ minimized signed envelopes ────▶ append-only, signed bytes only ◀──────── author policy → org-key sign
 ◀── org-key-signed policy (pull on heartbeat · verify · anti-rollback · apply · signed receipt)
```

Three design laws hold everywhere:

1. **Verify, don't trust.** Every claim on every screen traces to a signed artifact re-verified
   locally. `npm test` proves the TypeScript verifier and the Rust signer agree byte-for-byte.
2. **The server authors nothing.** Evidence is signed by devices, policy by the operator's org
   key. `kriyad` holds neither key — it can withhold, never invent, and withholding is caught.
3. **Your data stays yours.** Free tier: nothing leaves the machine, ever. Fleet tier: minimized
   evidence moves only to *your* server. Air-gap: signed files on approved media, same verifier.

## What it does *not* do (read this)

We publish our limits instead of papering over them — [`docs/TRUST.md`](docs/TRUST.md) is
canonical:

- **Tamper-evidence, not tamper-proofing.** Altered, forged, or deleted receipts are *detected*.
  A fully compromised host that holds the signing key is out of scope — pin your signer.
- **kriya sees governed lanes.** Actions that route through the hook, gateway, or SDK are
  recorded; the Coverage Map shows honestly what doesn't. GREEN means "the watcher was up," not
  "physics guarantees capture."
- **Seams belong to their owners.** Claude Code's own hook timeout fails open on *its* side;
  whoever controls its settings file has the last word there. kriya fails closed on its own errors.
- **Evidence, not certification.** Every export says so in the footer. Controls kriya can't
  earn (e.g. OS-level audit-role separation, 3.3.9) are shown as permanent, visible gaps.
- **Egress claims stay scoped.** The egress controls govern kriya's lanes (hook, gateway, broker,
  contained sessions) — kriya does not claim host-wide DLP or network-boundary enforcement, and
  enforcement verbs stay scoped to the lanes that actually enforce.
- **Memory receipts observe; they don't defend.** kriya records *that* a governed memory surface was
  written and *what the bytes hashed to* — it does not read the content and makes **no claim to detect
  semantic memory poisoning**. The untrusted-ingress badge stays inert until you enable ingress
  recording or install a trust-class control; both default off.
- **Behavior baselines are advisory, and noisy on bursty traffic.** They flag "this stopped looking
  like the baseline," never "this is bad," and carry no enforcement lever. Real agent traffic arrives
  in bursts, which pushes the false-alarm level well above the nominal threshold — which is exactly
  why the surface reports a crossing and never quotes a rate.

## Free vs paid

| | Free (no account, no license) | Paid (offline license) |
|---|---|---|
| Monitor, Audit, Coverage Map, Sessions | ✅ | ✅ |
| Policy, Approvals, Budgets, Identity | ✅ | ✅ |
| Govern All, Connections, egress controls | ✅ | ✅ |
| Spend receipts + dollar budget gates | ✅ | ✅ |
| Verified Replay + replay-bundle export | ✅ | ✅ |
| OTel export, C2PA provenance, diode bundles | ✅ | ✅ |
| Local model governance (`kriya-llm-proxy`) | ✅ | ✅ |
| Memory-write receipts + Memory view | ✅ | ✅ |
| Temporal policy conditions | ✅ | ✅ |
| Behavior baselines | ✅ | ✅ |
| Auditor CLI (all six verifiers) | ✅ | ✅ |
| Evidence export (6 frameworks) | — | ✅ |
| Fleet cockpit + `kriyad` control plane | — | ✅ (`fleet-console`) |
| Fleet OTel export + `/metrics` + Grafana | — | ✅ (`fleet-console`) |

The license is an Ed25519-signed offline token — no phone-home, no accounts. Licensed via
design-partner engagements today (not self-serve yet) — pricing and contact at
[kriyanative.com](https://kriyanative.com).

## How it compares

| | Vendor dashboards / lab logs | Cloud GRC (Vanta, Drata) | Egress proxies / firewalls | **kriya** |
|---|---|---|---|---|
| Works where data can't leave the building | ❌ | ❌ | some | ✅ built for it |
| Governs a *competitor's* agent too | ❌ | partial | ✅ | ✅ vendor-neutral |
| Record is independently re-verifiable | ❌ trust us | ❌ trust us | some (signed logs) | ✅ offline, free CLI |
| Sees agent *decisions* (hook seam), not just packets | ❌ | ❌ | ❌ | ✅ |
| Maps evidence to CMMC / SOC 2 / ISO 42001 / EU AI Act | ❌ | ✅ cloud-resident | ❌ | ✅ on-device |
| Proof is a precondition of the action | ❌ | ❌ | ❌ | ⭐ "no receipt, no egress" |

## Who it's for

Teams where *"an agent did something"* is not an acceptable answer — they must **prove what it
did and constrain what it can do**. Sharpest fit: organizations that legally can't ship agent
activity to a cloud dashboard (defense/CMMC, sovereign, air-gapped) — kriya installs where a
cloud governance product structurally can't.

## Relationship to the open runtime

Dependency is **one-way**: the Console consumes the open `kriya` runtime's signed audit + policy
formats and never the reverse. The runtime — the part that enforces policy and signs receipts —
is MIT and fully auditable at
[github.com/governex/kriya](https://github.com/governex/kriya), and the free
auditor CLI re-proves the Console's evidence offline. *Don't trust the cockpit — check its
receipts.*

---

Enterprise & regulated deployments → [kriyanative.com](https://kriyanative.com) ·
**Sandeepshekhar26@gmail.com**
