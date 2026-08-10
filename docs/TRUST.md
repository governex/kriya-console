# Trust & tamper-evidence — what kriya Console lets you prove

> For security, compliance, and procurement reviewers. This explains, in plain terms, **what an
> auditor can independently verify** about agent activity through kriya Console, how the
> tamper-evidence works, and — honestly — the boundaries of that guarantee. The underlying
> cryptography lives in the **open** kriya runtime (MIT) and is documented at
> [`docs/SECURITY.md`](https://github.com/governex/kriya/blob/main/docs/SECURITY.md); this
> document is the buyer-facing companion and does not contradict it.

## The one-sentence claim

Every action an AI agent actually performed in a kriya-governed app is recorded as a
**cryptographically signed receipt**, and kriya Console lets you (and your auditor) **re-verify
every one of those signatures locally — on your own machine, with no network and no trust in the
vendor** — so altered or forged records are detected, not assumed.

## What you can prove

| Question a regulator / auditor asks | How the Console answers it |
|---|---|
| *"Show me everything the agent did."* | The audit view aggregates the signed receipts from every kriya app into one table — action, parameters, who, when, success — and verifies each signature on-device, inside the Console app. |
| *"How do I know this log wasn't edited?"* | Each receipt is Ed25519-signed by the host. The Console re-derives the signed bytes and checks the signature; **any edit to a retained receipt fails verification and is flagged in red.** |
| *"Who authorized the risky ones?"* | Guarded actions (e.g. `delete_transaction`, `close_account`) were held for a human; the approval — with the operator's identity and a recorded reason — is part of the trail (R8 `actor`). |
| *"What is the agent even allowed to do?"* | The policy view shows the exact allow / require-approval / deny rules the runtime enforces, and produces the `agent-policy.yaml` the host loads — so the control is provable, not aspirational. |
| *"Give me evidence for our EU AI Act / SOC 2 / ISO 42001 control."* | The compliance view maps the verified trail to specific controls (EU AI Act Art. 12 record-keeping, Art. 14 human oversight, SOC 2 CC7.2, ISO 42001 A.9) and exports a Markdown report + JSON bundle, with **gaps shown honestly**, not hidden. |
| *"Give me evidence toward our ISO/IEC 42001 AI-management-system controls / CSA STAR-for-AI submission."* | A second, dedicated "ISO 42001 / CSA AICM" pack (D3, doc 27) maps the trail to ISO 42001 Annex A **A.6.2.8** (event logs), **A.6.2.6** (operation & monitoring), and **A.9.4** (intended use), plus CSA AI Controls Matrix (AICM) v1.0 **LOG / A&A / GRC** domains (directly/supportingly) and **IAM / DSP** (context-only) — every row capped at **◐ partial**, never ✓, and every NOT-COVERED control (risk/impact assessment, technical documentation, data quality, and 13 more) is listed **explicitly**, not hidden. A companion AI-CAIQ v1.0.2 **self-assessment support sheet** points each domain at its supporting receipt evidence — kriya never answers, scores, or attests any item, and issues no certification: STAR-for-AI's third-party element is the accredited ISO 42001 certificate from a certification body, never kriya. See [`docs/design/d3-iso42001-pack.md`](design/d3-iso42001-pack.md) and the golden sample at [`docs/samples/iso42001-sample/`](samples/iso42001-sample/). |

## How the tamper-evidence works (in brief)

1. **The host signs, the agent can't.** The kriya runtime holds an Ed25519 signing key in the host
   process; the agent never sees it. After an action clears the policy/approval/budget gates, the
   host signs a receipt over the action id, parameters, who did it, when, and the success flag.
2. **Receipts are append-only and self-describing.** Each is one line in a JSONL log carrying the
   signature and the signer's public key.
3. **Verification is independent and offline.** The Console re-computes the exact bytes that were
   signed and checks the signature on-device, inside the Console app's compiled backend. The
   verification is proven **byte-identical** to the host's signing in the test suite — if it
   drifted by a single byte, real receipts wouldn't verify. **Nothing is sent anywhere; you are
   not trusting kriya's word, you are checking the math.**

Because the signature covers *who/what/when*, you cannot quietly change the amount on a transaction,
flip a failure to a success, or reassign an action to a different operator without invalidating that
receipt's signature.

## Cryptographic module (the FIPS signing lane)

> For CMMC / NIST 800-171 reviewers asking *"is the cryptography FIPS-validated?"* — see doc 27 A4.

**The load-bearing fact, read this first:** Ed25519 is deterministic (RFC 8032) — the 64 signature
bytes produced over a given message by a given 32-byte seed are identical no matter which compliant
implementation computed them. That means **there is no way to tell from a receipt's signature bytes
alone which cryptographic module signed it** — a feature, not a gap: it's what lets a FIPS-signed
receipt verify under this document's own default-build verifier, and vice versa, with zero special
handling. It also means the record of *which module signed* is a **host self-attestation**, not a
cryptographic property of the signature itself — exactly as trustworthy as the host is (this
document already scopes out a fully compromised host).

kriya ships **three** signing lanes, each with its own honest wording — never upgraded past what the
lane actually is:

| Build / OS | Lane | Wording |
|---|---|---|
| Default (any OS) | `ed25519-dalek` | The default build signs with `ed25519-dalek` — a widely-audited pure-Rust implementation that is **not** a FIPS-validated module. |
| **Linux**, `fips-crypto` feature on | `aws-lc-rs` `fips` | FIPS 140-3 validated module (cert #5298). |
| **macOS**, `fips-crypto` feature on | `aws-lc-rs` `fips` | same cryptographic module code as cert #5298, running outside a CMVP-tested operational environment. |

The `fips-crypto` lane is **opt-in and OFF by default everywhere** — the free tier, the notarized
macOS `.dmg`, and every default `cargo build`/`tauri build` are byte-unchanged by its existence. When
a signer runs under it, it self-attests with a new, additive `kriya.crypto.module` receipt (chained
and signed exactly like any other receipt — the receipt envelope itself never changes) recording the
backend, the validated module name, the CMVP certificate, a **live** `try_fips_mode()` check (a
runtime fact, never a build flag), the operational environment, and how the signing key itself was
minted (`module-drbg` when generated under the FIPS lane, `external-rng` under the default lane, or
`imported-seed` when a pre-existing key was loaded — disclosed honestly rather than laundered).

**The operational-environment refinement.** Cert #5298's CMVP-*tested* operational environment is
Amazon Linux 2023 only, with no vendor-affirmed OEs. Strictly, only AL2023 is the *validated* OE;
other Linux distributions run the identical validated module code outside the tested OE — the
attestation's `operational_environment` field records this finer truth even though the surface
wording above stays at the coarser, honest "Linux" grain. macOS never gets upgraded to the Linux
wording, no matter how the module behaves — the CMVP validation is OE-specific, not a general "this
binary is FIPS" claim.

**Signing ≠ verifying ≠ key generation — three separate statements, never conflated.** A4's claim is
about the **signer**: the runtime process that produces receipts. The compiled-Rust verification lane
in this Console (the one described in "How the tamper-evidence works" above) inherits whichever lane
it was built with. The **TS/WebCrypto verification lane** (`src/lib/verify.ts`, and the self-verifying
HTML artifact) holds no signing keys, signs nothing, and makes **no FIPS claim of any kind** — it
independently re-verifies a receipt regardless of which module produced it, precisely because of the
load-bearing determinism fact above. Key *generation* is a separate statement too: under the FIPS
lane, a freshly minted signing key is seeded from the module's own approved DRBG, not the host's
general-purpose CSPRNG — `key_provenance` in the attestation says so explicitly, and discloses when a
key predates the lane switch.

**The honest ceiling.** This is a signed, host-attested checkbox for a specific procurement question —
it is **not** an own-module CMVP validation (kriya is not itself a certified cryptographic module; the
underlying `aws-lc-rs`/AWS-LC-FIPS module is), and it is never described as "FIPS-certified product,"
"FIPS compliant," or any phrase implying whole-product certification. See
[`docs/samples/fips-module-boundary.md`](samples/fips-module-boundary.md) for the assessor-facing
one-pager version of this section.

## Post-quantum readiness (the ML-DSA-87 dual-signature lane)

> For a security/compliance assessor asking *"is your long-retention audit evidence signed with a
> quantum-resistant algorithm?"* — see doc 27 A5.

kriya ships an **opt-in** `pq-crypto` feature that adds an **ML-DSA-87 (FIPS 204) countersignature**
*next to* the existing Ed25519 signature — never replacing it. The approved claim is
**post-quantum-ready (CNSA 2.0-aligned, ML-DSA-87)** — algorithm-conformance and crypto-agility, never
a validation claim: **ML-DSA-87 (FIPS 204) — not a FIPS-validated module (ML-DSA is outside cert #5298's approved boundary; ML-KEM is in it, ML-DSA is not).**
Never "FIPS-validated PQ," never "quantum-proof," never "quantum-safe" (we say "quantum-resistant" or
"post-quantum-ready" only).

**Two PQ modes, one default.** By default, kriya seals a run of receipts with **one ML-DSA-87
countersignature per chain checkpoint** (`kriya.crypto.pq_checkpoint`, every N receipts — default 256,
a policy dial) over the SHA-256 chain head — because the hash chain is already **quantum-resistant**
(SHA-256 under Grover is 128-bit), one PQ signature over the chain head transitively anchors the whole
sealed prefix at a **measured ≈1.03–1.13× storage cost** (see
[`docs/samples/pq-dual-signed-receipt.md`](samples/pq-dual-signed-receipt.md) for the real numbers).
A **per-receipt dual-sig** opt-in exists for low-volume, high-value receipts (policy bundles, org-key
signatures, retention checkpoints, evidence-export seals) at a measured **≈14.6 KiB/line** cost — a
**≈30× blow-up** if applied to every receipt on a busy host, which is exactly why it is opt-in, not
the default.

**The honest distinction.** The checkpoint gives post-quantum **tamper-evidence / integrity of the
log** (the sealed sequence cannot be altered without a SHA-256 collision, anchored by one PQ
signature) — it does **not** put a PQ signature on every receipt. Per-receipt dual-sig additionally
gives post-quantum **per-receipt authenticity / non-repudiation**. Most compliance drivers need the
former; kriya never implies the checkpoint gives the latter.

**Additive, frozen-schema.** The PQ material (`pq_alg`, `pq_public_key`, `pq_sig`, `pq_key_id`) rides
as **top-level wire siblings** of `public_key`/`signature` — outside the Ed25519-signed bytes, exactly
like `public_key`/`signature` already are. Every existing receipt verifies **byte-for-byte unchanged**
under a `pq-crypto` build; a dual-signed receipt verifies as a normal receipt to a pre-A5 verifier
(old verifiers ignore unknown fields — a property this design depends on, not assumes).

**Key binding + rotation.** The ML-DSA-87 keypair is a separate key, seed-deterministic (FIPS 204),
stored beside the existing Ed25519 identity — but **bound to it** by an Ed25519-signed
`kriya.crypto.pq_key` attestation: the pinned Ed25519 trust anchor vouches for the PQ public key, so
an auditor who already pins the Ed25519 key transitively trusts the PQ key without a second
out-of-band pinning step. Rotation is a fresh attestation; older checkpoints stay self-verifying under
their own inline PQ public key — no history re-signing.

**The browser/TS lane does not independently verify the PQ signature in this build.** The compiled
Rust lane (`kriya-verify`, shared by the Console/kriyad/CLI) is the PQ-verifying authority — it checks
both Ed25519 and ML-DSA-87. The TS/browser lane verifies Ed25519 + message-byte parity + PQ
**structure** (the algorithm tag, hex lengths, `pq_key_id` binding) but does **not** run ML-DSA
cryptographic verification — it holds no PQ keys and makes no PQ claim beyond structure. A
PQ-tampered-but-structurally-valid receipt shows "Verified (Ed25519)" with an explicit note that the
PQ signature was not checked in this lane — never a false green PQ checkmark. A genuinely independent
browser-side PQ re-verification (`@noble/post-quantum`) is a disclosed, conscious follow-on, not a
current claim.

**The tail-gap (an honest limit, stated not glossed).** Receipts emitted after the most-recent
checkpoint are Ed25519-only (quantum-forgeable) until the **next checkpoint / evidence-export seal /
retention checkpoint** seals them. For the harvest-now-decrypt-later threat model this residual is
minor — historical, retained, and exported evidence is exactly what subsequent checkpoints and the
export seal cover — but it is not zero, and the checkpoint cadence (`N`) bounds how large the unsealed
tail can grow.

**Whole-line PQ downgrade is an honest, disclosed limit too.** Stripping ALL `pq_*` siblings from one
line is undetectable per-line (the siblings aren't inside the Ed25519-signed bytes) — a partial/
mismatched set fails loudly (require-if-present), but a clean strip of the whole set does not, per
line. The real defence is the checkpoint: its SHA-256 head-seal detects any line alteration
post-quantum regardless of per-line siblings, and a chain that was PQ-sealed and later carries no
checkpoint is a signal a policy-aware verifier can flag.

**Build posture.** `pq-crypto` is opt-in, off by default, Linux-first like the FIPS lane — the
default build and the notarized `.dmg` are byte-unchanged by its existence (RustCrypto's `ml-dsa`
crate, used only for cross-implementation test parity, is a dev-dependency and never ships).
**`pq-crypto` and `fips-crypto` cannot be enabled together** in this build — a real upstream
constraint of the `aws-lc-rs` crate (its `unstable` PQ module is compiled out whenever its `fips`
feature is active), not a kriya restriction; see `crypto.rs` in both repos for the technical note.
See [`docs/samples/pq-dual-signed-receipt.md`](samples/pq-dual-signed-receipt.md) for the
assessor-facing one-pager with real, Rust-generated sample receipts and measured size numbers.

## Honest boundaries (read this)

A trustworthy vendor states the limits of its guarantee. Tamper-*evidence* is not the same as
tamper-*proofing*:

- **Pin your signer.** Verification proves a receipt wasn't altered *under the key that signed it*.
  A meaningful audit also confirms that key is **your** host's key. The Console surfaces every
  distinct signer across your logs precisely so an unexpected key stands out — make pinning the
  expected key part of your review.
- **Whole-record deletion is detected — receipts are hash-chained.** Each receipt carries a
  `prev_hash` (the SHA-256 of the previous log line) *inside the signed bytes*, so the log is a
  tamper-evident chain, not just a set of independent signatures. Removing, truncating, or
  reordering entire records breaks the chain: re-verification flags a `CHAIN-BREAK` at the gap.
  Signatures prove *no retained record was altered*; the chain extends that to a *completeness*
  guarantee against whole-receipt deletion. (Anchoring the chain head to an external timestamp is
  a further hardening option.)
- **A fully compromised host is out of scope.** The guarantee is against the *agent* and against
  *after-the-fact editing by anything without the key* — not against arbitrary code running inside
  the trusted host process.
- **Signing key lifecycle.** A persisted, stable signing identity has shipped (R20): the pinned
  public key stays the same run-to-run, so the audit trail is verifiable over months, not just within
  one session. A deployment only shows multiple signer keys if it runs with the ephemeral
  per-process key instead of a persisted one — the Console shows you exactly how many. Hardware-backed
  (Secure Enclave) anchoring of that identity is the remaining hardening.
- **Policy enforcement now actually reaches every install path (B0, fixed).** Until this fix, the
  Policy view authored a policy that was never written to any file the runtime could load, and
  every "Install hook" / "Govern everything" / manual-connection action installed `kriya-hook`,
  `kriya-hermes-hook`, and `kriya-gateway` with **no `--policy` flag at all** — every enforcement
  point silently ran the permissive built-in default, regardless of what the Policy view showed.
  A deny rule the operator saved was never actually enforced. Fixed: the authored policy now
  persists to `~/.kriya/agent-policy.yaml` on every edit, and every install path wires `--policy`
  at that file. Stated here retroactively, in the same spirit as the rest of this document — an
  honest account includes bugs that shipped, not just the ones caught before release.
- **The approval tier is a `tty`/GUI-dialog prompt, not a live Console popup.** `RequiresApproval`
  actions are decided by the hook/gateway process itself (a terminal prompt or a macOS dialog),
  self-bounded at 300s. We deliberately do **not** use Claude Code's native
  `permissionDecision:"ask"` — it has documented, reproducible reliability gaps (unreliable in
  headless `claude -p` mode, and has been observed silently overridden by a broad
  `permissions.allow` rule elsewhere in a user's settings, letting the tool run with no prompt at
  all). The **Approvals** view in this Console is a separate, manual/historical decision queue —
  load a JSONL file of pending requests, decide with a reason, and the decision is recorded in the
  local trail — it is not wired to unblock a paused hook process live. A true remote,
  Console-mediated approval flow is a possible future addition, not something this build claims.
- **A hook that times out, crashes, or emits malformed output on Claude Code's own side fails
  open — kriya cannot change that from this side of the seam.** Claude Code's hook timeout
  (600s default) is documented to let the tool proceed if a hook doesn't answer in time; the same
  is true of a hook that crashes or produces output Claude Code can't parse. `kriya-hook`
  mitigates this everywhere it can control the outcome — the approval gates self-bound well under
  that ceiling, and the hook's own internal errors (bad payload, unreadable policy) always fail
  closed — but a cooperative seam is still cooperative: whoever controls `settings.json`, or
  whatever kills the hook process externally, has the last word. See `kriya`'s
  [`docs/SECURITY.md`](https://github.com/governex/kriya/blob/main/docs/SECURITY.md) for
  the full detail.

These boundaries are shared with the open runtime's threat model and are not unique to the paid
tier; we publish them rather than paper over them.

## Coverage — what is (and isn't) being recorded, as a signed metric

Signatures prove a *retained* receipt is authentic; they cannot prove an event **no source
observed** ever produced a receipt. The Console makes that boundary a first-class, verifiable
surface instead of a footnote — the **Coverage Map**:

- **Six lanes, three states.** Claude Code tools · remote/attached MCP · local stdio MCP ·
  desktop apps · raw file & exec · raw egress — each classified **GREEN** (configured, with
  receipts or a live watcher heartbeat inside the window), **AMBER** (configured but silent), or
  **GREY** (uncovered: events there leave *no* receipt). The window is stated on the map itself.
- **The map is itself evidence.** On every lane-state change (and at least daily) the Console
  signs a `kriya.coverage.snapshot` receipt into its own hash chain
  (`~/.kriya/audit/coverage.jsonl`), verifiable by the same offline verifiers as any receipt.
  So "we were covered all quarter" is a *checkable chain of signed statements*, and a silenced
  Console, a stopped watcher, or a deleted stretch of history is **visible by absence** — a gap in
  the heartbeat chain, not a quiet nothing.
- **What a GREEN lane does NOT claim.** GREEN means the configured source was alive and recording
  in the window — not that every event in that lane was captured (a watcher can be stopped *before*
  an action and restarted after; the heartbeat bounds the gap in time but cannot manufacture the
  missing event), and never that payload content was read (recording is metadata: action, actor,
  time, outcome — no TLS payloads). A GREY lane is the honest statement that nothing would have
  been recorded there at all.

## The data-boundary promise — per lane, not a comforting average

The Console now spans four distinct postures — free single-machine use, an enrolled device that
reports over one of **two transport lanes**, and the cockpit that reads the fleet back — and each
carries a different, precisely scoped promise about what leaves the machine and who, if anyone,
briefly handles it in transit. Stating them side by side, rather than letting "nothing leaves the
device" quietly become the answer for all four, is the same honesty discipline as the Coverage Map
above: say exactly what is and isn't true, per lane, not a comforting average — and, because one of
these lanes now touches infrastructure kriya operates, say *that* plainly instead of preserving a
blanket "never kriya" that a hosted connector would make false.

**Two transport lanes exist for an enrolled device**, and the buyer chooses which (or runs both
across a fleet):

- **L1 · direct** — the device pushes to a hub the **customer runs**: the Console itself acting as
  an in-process hub on an always-on machine, a self-hosted `kriyad`, or the server-TLS-plus-bearer-
  token serve mode pointed at the org's own hostname. **No kriya infrastructure is on the path.**
- **L2 · connector** — the device pushes to a per-tenant **mailbox kriya operates**
  (`<company-code>.console.kriyanative.com`); the cockpit pulls from that same mailbox, re-verifies
  every byte locally, and **acks**, after which the mailbox **deletes**. It is a buffer that holds
  the undelivered tail, not a store — see "The connector cloud (L2)" below for exactly what it can
  and cannot see.

| Posture | Promise | What that means concretely |
|---|---|---|
| **Free / un-enrolled device** | **Machine-level: nothing leaves the device, full stop.** | No fleet connection is configured, so no socket to any server — kriyad or otherwise — is ever opened for audit/evidence purposes. This is unchanged, byte-for-byte, by anything below: the free tier's claim on this page has not been weakened or reinterpreted. |
| **Enrolled device — L1 direct** (paid `control-plane`) | **Boundary-level: minimized, signed envelopes and the device-info beacon go to a hub the customer runs — never anywhere else, no kriya infra on the path.** | The device signs and sends redaction-minimized evidence envelopes plus the periodic `DeviceInfo` inventory beacon (§7 fields only, see below) to one server the customer stands up — their Console-as-hub, their `kriyad`, their VM/k8s/air-gapped enclave — over mTLS or server-TLS-with-token. Raw receipts and raw payload values are never included — see "Honest boundaries" above for what recording even means. This traffic never reaches kriya's infrastructure or any third party; it terminates at infrastructure the customer alone controls. |
| **Enrolled device — L2 connector** (paid `control-plane`) | **Transit-level: the same minimized, signed envelopes pass through a kriya-operated mailbox that verifies, buffers the undelivered tail, and deletes on the operator's ack — kriya can read no content and can never forge.** | For orgs that want connectivity without standing up their own reachable hub, the device pushes the *same* minimized envelopes to a per-tenant mailbox at `<code>.console.kriyanative.com` over public-CA TLS with a device-bound bearer token. The mailbox re-verifies each Ed25519 envelope, holds only the tail the cockpit has not yet pulled, and **deletes each envelope once the cockpit acks it**. An envelope carries no raw receipts and no payload values (identical minimization to L1), so there is no content for the mailbox to read; the Ed25519 signature is the integrity root, so the mailbox — or anyone who steals its bearer token — can connect but can never forge or alter a byte the cockpit will accept. What the mailbox unavoidably handles is disclosed below. |
| **Operator / cockpit** (paid `fleet-console`) | **Boundary-level: the cockpit reads coverage/evidence from — and publishes org-key-signed policy to — the customer's own hub (L1) or the connector mailbox (L2), re-verifying every byte it pulls; the mailbox holds nothing kriya can read.** | The fleet cockpit is the same on-device Console binary in "operator mode," not a hosted dashboard. Against an L1 hub it talks only to the customer's own server. Against an L2 mailbox it pulls signed envelopes over an operator bearer token, re-verifies each one locally with the same offline verifier this whole document describes, and acks — which is what triggers the mailbox to delete. Policy it publishes is org-key-signed and re-verified device-side before applying; kriya never authors, signs, or can alter it. |

**The through-line, restated honestly per lane — because L2 changes it, and we say so rather than
keep a promise a hosted connector would break:**

- **Free:** nothing leaves the device. Byte-for-byte unchanged; the free tier's claim on this page
  has not been weakened or reinterpreted.
- **L1 direct:** data leaves the device only to a server **you** run. No kriya-operated server is on
  the path — the pre-connector promise, intact for any org that runs its own hub.
- **L2 connector:** data passes through a kriya-operated **connector** that verifies, buffers the
  undelivered tail, and deletes on ack. kriya **can read no content** (a minimized envelope has
  none) and **can never forge** (Ed25519 is the root and the cockpit re-verifies every byte). This
  is the one lane on which anything touches kriya-operated infrastructure at all — so we state
  exactly what it is (a buffer, not a store) and exactly what it sees (next).
- **The org picks the trade:** L1 = zero kriya infrastructure on the path, at the cost of running a
  reachable hub; L2 = zero customer infrastructure on the path, at the cost of the undelivered tail
  transiting a kriya-operated buffer that holds it briefly and reads none of it.

### The connector cloud (L2) — what the mailbox does, does not, and cannot see

The per-tenant mailbox (`console.kriyanative.com`) is *"just a connector — it helps the two pieces
connect."* That is a verifiable claim, not marketing, because of how little the mailbox is allowed
to be:

- **Buffer, not store — structurally, on the hosted lane only.** The mailbox deletes every envelope
  a short grace window after the cockpit acks it; the archive of record is the device's own
  append-only outbox, which is never truncated. The full history lives on the customer's devices;
  the mailbox holds only the tail the cockpit has not yet collected. On a customer's own L1 hub this
  retention sweep is **off** — that hub is the permanent, queryable store — so "kriya's cloud is a
  buffer" is a property enforced in code on the hosted lane specifically, not a promise about
  operator behavior.
- **No content to read.** An envelope is a minimized, signed rollup: allowlisted action ids and
  counts, chain hashes, the device's public key. No raw receipts, no payload values, no request/
  response bodies ever enter it (the minimizer forecloses this structurally at every verbosity — see
  the redaction floor above). The mailbox verifies signatures and buffers bytes; there is no
  plaintext for it to inspect even if it tried.
- **Cannot forge.** Every envelope is Ed25519-signed by the device; the cockpit re-verifies each one
  after pulling it. A mailbox — or an attacker who steals a device's bearer token — can connect and
  push, but a forged or altered envelope fails verification at the cockpit and is flagged, exactly
  like a tampered receipt. The transport credential (the bearer token) is deliberately **not** the
  evidence key: a stolen token can connect, it can never sign.
- **What it unavoidably handles, disclosed:** any TCP connection reveals the device's **source IP**
  to the server — the mailbox, like `kriyad`, does **not** persist it to the store (verified in code,
  the same as the L1 hub); connection **timing** and **volume** are visible to any relay by nature;
  and **tenant identity** is visible by design (the company-code subdomain is how the wire is
  routed). None of these is content; all of them are the honest cost of a hosted connector, stated
  rather than glossed.
- **Not yet operating.** No hosted tenant is live as of this writing — the connector lane is built
  and tested, but the production route and the first tenant are gated on the remaining rollout steps
  (a rehearsed backup+restore, this per-lane disclosure's founder sign-off, and the DNS/edge route).
  This section describes a capability that ships **before** the first tenant, not one already
  carrying traffic.

**Why this doesn't undercut the ops story** — and why it makes the L2 claim *stronger*, not weaker.
Raw receipts and raw payload values stay device-local on every lane — not merely as a courtesy, but
because it keeps the customer's own hub non-sensitive (a backup is one small SQLite file, not a
honeypot of raw agent payloads), because it's the posture that survives an employee-privacy review
(a regulated buyer must be able to show they did *not* centralize keystroke-level activity), and —
now — because it is exactly what lets the L2 connector be a buffer that reads nothing: there is no
content in an envelope for a transiting mailbox to see. Envelope verbosity beyond the minimized
default is the customer's own policy dial on their own hub — never something this Console decides
for them, and never something the connector lane can widen.

### What the device-info beacon does — and does not — collect

The new `POST /v1/device-info` beacon (used by the enrolled-device tier above, and read back by
the cockpit's fleet table and per-device drill-in) is schema-constrained to an explicit allowlist
of device-scoped, technical fields, enforced in code, not just by convention:

| Collected (device-scoped, technical) | Excluded — never collected, never transmitted | Excluded — unavoidably seen in transport, never persisted |
|---|---|---|
| Console / runtime / verifier / agent / adapter versions | OS **username** | Source **IP** — any TCP connection reveals it to the server; kriyad must not write it to the store |
| Coarse OS platform, version, and architecture | **Hostname** — never auto-derived; the only device name shown is an optional, enterprise-assigned asset tag from the customer's own MDM, and the fleet cockpit falls back to a short public-key fragment (never a locally-known OS identity string) when that tag is absent | |
| Per-agent wired/unwired status, applied policy version | Timezone, locale, MAC address, hardware serial numbers | |
| Outbox depth (a health signal), enrollment time | | |

One scope sentence that must accompany this table wherever it is shown: **on a single-user device,
device-scoped records are still personal data under GDPR** — `device_pub` plus an MDM asset tag is
indirectly identifiable (pseudonymization is not anonymization). This table is *minimization within
scope*, not an exemption from it; the customer is the controller of what their kriyad receives.

This is the same field list documented as canonical in the runtime's `DeviceInfo` schema (see the
open kriya repo's `kriya-verify` crate), which ships with an adversarial test proving the exclusion
structurally: a probe that deliberately offers a username, a hostname, a source IP, a timezone, a
locale, a MAC address, and a serial number is fed through the real constructor, and none of those
seven values — or their field names — can appear anywhere in the signed bytes actually placed on
the wire. The guarantee here isn't "we chose not to send it today"; it's that the schema has no
field to put it in.

## Egress governance (E1) — honest limits

The Console can allowlist, budget, and receipt an agent's outbound calls through the governed
connector lanes (MCP-over-HTTP, gateway-proxied tool calls, and the hook-observed WebFetch/WebSearch
lane) — a standalone signed receipt in the `kriya.io.<direction>.<kind>.<decision>` vocabulary per
governed call, correlated to the underlying action receipt. Read this section before treating any of
it as a network boundary control, because it isn't one:

- **Governed lanes only, not the host.** *"When a stdio MCP server routed through kriya calls an
  external API, kriya sees — and signs — only the tool call and result that crossed its stdio pipe;
  the server's own outbound network traffic (which hosts it contacted, what it sent) is invisible to
  kriya and appears in no receipt. Host-level observation of that traffic is a separate, later
  capability."* A spawned subprocess or a stdio MCP server's own sockets bypass this lane entirely —
  the Coverage Map's grey **raw egress** lane names that gap on purpose; a green chip on a governed
  lane never colors it.
- **Two different byte-hash definitions, never conflated.** Every `kriya.io.*` receipt's `params`
  names `hash_scheme`: `"wire-bytes"` (the gateway/broker lane, where kriya is the TLS client and
  hashes the exact bytes on the wire) or `"canonical-json"` (the hook lane, which hashes the
  canonical key-sorted serialization of `tool_input`/`tool_response` — a different, less precise
  commitment). Byte counts are labeled *observed payload bytes*, never a wire/TLS-level accounting —
  connection reuse, headers, and keep-alive are invisible either way, and an SSE reply's `bytes_in`
  is flagged partial (a lower bound, not an exact count).
- **A denied call is receipted at the decision point.** A `kriya.io.*.deny` receipt is written
  before/instead of execution — the call never reaches the destination — so `deny` rows exist even
  though nothing crossed the wire. The one exception: fail-closed mode (below) can itself deny an
  egress because the receipt couldn't be written; that block carries no receipt by construction (the
  precondition failed), which is the whole point of the mode.
- **Fail-closed is opt-in, and inverts the honest default.** "No receipt, no egress" is a policy
  flag, off by default — the documented default is fail-**open** (a receipt-write failure doesn't
  block the call). Turning it on makes the signed receipt itself a precondition of the egress, which
  is unusually strong evidence, but it is not the out-of-the-box behavior.
- **The egress chip on the Coverage Map is a window observation, not a configuration attestation.**
  "ON" means at least one `kriya.io.egress.*` receipt was observed on that lane within the window;
  "OFF" means the lane is otherwise covered but none appeared. Neither state proves the egress tier
  was configured for the *entire* window — that requires a signed toggle/policy-version receipt this
  build does not yet emit. The compliance export's governed-surface posture statement says so
  explicitly (see "not monitored" vs "zero observed" below) rather than overclaiming a bound it can't
  prove.
- **What the chip, and the ledger, do NOT claim.** Not a firewall, not "DLP" (that word never appears
  in an E1-gated export), and never "every byte" — the honest label is "governed connector traffic."
  Enforcement rides a cooperative hook or gateway process that can be disabled at the host, same as
  every other governance seam in this document.

**The governed-surface posture statement** a compliance export prints is deliberately three-valued,
never a bare "zero egress":
- *"NOT MONITORED"* — zero governed-lane receipts of any kind were observed in the window, so no
  statement about egress can honestly be made either way (absent-by-configuration, not a finding).
- *"zero observed"* — the governed surface was active (other governed-lane receipts exist) and
  produced no egress, but this does **not** prove the ledger was continuously enabled for the full
  window (no toggle receipt bounds it yet).
- *"NOT zero"* — at least one `kriya.io.egress.*` receipt was observed and verified.

A physical air gap or network isolation, if one exists, is the organization's own attested posture —
kriya cannot verify it, and does not claim to.

## Fleet kill switch and the A2A seam (EG-F) — honest limits

The signed `PolicyBundle` an org publishes can carry `kill_switch: true` — a device
applying that bundle replaces its policy with a fixed, maximally-restrictive deny-all fallback
instead of the bundle's own `policy`/`budgets`. A device also engages the SAME fallback on its own,
without any bundle saying so, the moment its currently-applied bundle's `expires_ms` passes — "the
stale-policy kill switch." Read this before treating either as a remote-shutdown button:

- **Device-pulled, never device-pushed.** kriyad cannot force a kill switch onto a device — the
  device fetches the bundle on its own heartbeat cadence (or an operator carries it across an
  air gap). A kill switch takes effect only as fast as the next pull, not instantly, and a device
  that cannot reach kriyad at all keeps running its LAST applied policy until that bundle itself
  goes stale.
- **"kriyad authors nothing" applies here too**. The explicit switch is
  operator-authored and org-key-signed like every other bundle field; the staleness trigger is the
  device's own clock, not a kriyad decision — kriyad may be the very thing that's unreachable when
  it fires.
- **The fallback is fixed, not derived.** It never reads the bundle's own `policy`/`budgets` — a
  malformed or compromised bundle cannot shape its way out of the fallback by manipulating those
  fields.
- **Recovery is a fresh, superseding bundle.** There is no separate "un-kill" action; applying any
  later bundle (with `kill_switch` absent or `false`) overwrites the fallback like any other apply.

**The A2A (agent-to-agent) governance seam is unbuilt — code only, no live enforcement.** The
runtime (`kriya` crate) has a `Policy.a2a` field and an `evaluate_a2a_target` decision function that
reuse the exact egress-allowlist engine, but nothing in this codebase calls it: there is no
agent-to-agent transport today, so this is a roadmap-depth scaffold for a future lane, not a
shipped control. Do not cite it in any compliance export.

## Fleet destination patterns (pattern-echo, EG-4) — honest limits

A `PolicyBundle` can set `io_verbosity: "pattern-echo"` (default `"off"`) — once applied, a device
starts including a NEW field, `io_destinations`, in its signed envelopes: which operator-authored
egress destination *pattern* (never a raw host) each governed call matched, plus a count and a
denied count. This is new per-device metadata that did not exist before EG-4, and it deserves the
same scrutiny E1's employee-behavioral-data floor got (see "Employee privacy" below) — read this
section, and get your works-council/GDPR review done, before turning it on for a real fleet.

- **Pattern-echo, never raw-host.** A device NEVER puts the host it actually contacted into a
  signed envelope. It matches the observed host against the org's OWN authored `policy.egress.
  rules[].host` patterns and echoes back the PATTERN STRING that matched — a string that already
  exists, verbatim, in the operator-signed bundle. A host matching no authored pattern collapses to
  a fixed `"unlisted"` sentinel, never itself. This is enforced structurally by a sealed minimizer
  (`kriya_verify::minimize_io`, `#[non_exhaustive]`, sole constructor) — the same technique
  `minimize_window` uses for the drop-by-default redaction floor — not a policy choice a future
  change could quietly widen.
- **Raw-host mode does not exist in this product.** If a customer ever needs literal destination
  hostnames centralized, that is its own named, explicit, non-default product decision — it
  is not a dial anywhere in the shipped code today, and turning pattern-echo on never approaches it.
- **New metadata, even in pattern form.** Per-device destination-pattern activity, correlated over
  time, is new information about what one person's agent did — that is exactly the concern the EG-3
  privacy artifact pack (`docs/privacy/`) exists to walk through BEFORE deployment, and it is why
  this feature is gated on that pack existing, not merely on demand.
- **Privacy mitigations that ship WITH the feature, not after:**
  - Envelope timestamps are DAY-BUCKETED (UTC) whenever pattern-echo is active, coarser than the
    Compiler's normal window cadence — correlating a destination pattern against an hour-level
    timestamp is a stronger re-identification signal than correlating it against a day.
  - A pattern seen on fewer than a handful of devices fleet-wide is flagged (⚠) in the cockpit — a
    single person's own domain showing up as "a pattern" is a surveillance-shaped signal worth
    a human noticing, not hiding it and not auto-blocking it either.
  - A device's unlisted-destination count below a small threshold is WITHHELD from the fleet-wide
    org evidence REPORT by default — that specific export artifact does not carry the number, not
    just a UI mask over it.
- **The withholding is a property of the fleet-wide report artifact, not of the underlying signed
  data.** `io_destinations` is an ordinary field on the envelope itself — a `fleet-console`-licensed
  operator's own per-device drill-in (P4) already shows raw envelope internals (chain hashes, seq,
  `policy_state`) for verification purposes, and now shows the exact `io_destinations` counts the
  same way, without a threshold. This is consistent with the existing trust model (kriyad and the
  cockpit are inside the operator's own trust boundary; the fleet report is the artifact that leaves
  it, e.g. to an assessor) rather than a gap in it — but it means the k-threshold suppression is a
  courtesy on the widely-shared export, not a hard boundary inside the cockpit itself. Don't describe
  it as the latter in a customer conversation.
- **The surveillance is itself audited.** Revealing a withheld count in the fleet-wide REPORT is a
  deliberate cockpit action,
  and that action signs its own receipt — `kriya.console.drilldown` (`{device_pub, operator_pseudonym,
  scope, ts}`) — into the Console's own local hash-chain, tailed into evidence exactly like every
  other control-plane-internal event. "Who looked at device X's low-count detail, and when" always
  has an answer. The operator identity in that receipt is an HMAC pseudonym computed device-side
  from the local OS user, never a caller-supplied string — an operator cannot claim to be someone
  else in their own audit trail.
- **A `purpose_statement` travels with the bundle** and is echoed into every export once
  pattern-echo is on — an explicit, operator-authored answer to "why is this being collected,"
  printed alongside the data itself rather than left to a separate policy document.

## Credential brokering (B13) — a new trust posture

Every control above is kriya acting as a **witness**. Credential brokering is kriya acting as a
**custodian** — it briefly holds a real secret in process memory so it can inject it into a governed
outbound call the agent never sees the value of. That is a different kind of claim, with its own
document: **read [`THREAT-MODEL-brokering.md`](THREAT-MODEL-brokering.md) before treating this as
"kriya is now a secrets manager,"** because it isn't one (that document's explicit non-goal).

The custody + receipt summary:

- **Custody is OS Keychain, never a file.** `agent-policy.yaml` carries only a Keychain
  service+account *reference* per alias — the schema cannot express a plaintext secret. The runtime
  reads the real value at substitution time only, via the system `security` CLI with the
  service/account passed as separate process arguments (not shell-interpolated).
- **The agent composes a placeholder, `{{kriya:<alias>}}`, never the credential.** Substitution
  happens in exactly two places — the governed HTTP transport and the Claude Code hook lane's
  documented `updatedInput` mechanism — both as late as possible, after everything that gets hashed
  or recorded has already committed to the placeholder form.
- **No receipt ever carries the value.** A brokered call's `kriya.io.*` receipt is additively flagged
  (`"b13-brokered:<alias>"`) — the alias name, never the secret. The hook lane additionally treats
  Claude Code's own (undocumented) `PostToolUse` echo behavior as adversarial and actively redacts
  each configured alias's real value back to its placeholder before recording anything, regardless of
  which form was actually echoed.
- **Scope is per-alias, independent of the general egress tier.** A credential is only ever
  substituted into a call bound for that ONE alias's own destination allowlist — a misrouted call is
  denied, not brokered, before any Keychain read happens.
- **What it does not defend against:** a compromised kriya process itself (Keychain access control is
  the remaining barrier, same as for any macOS app with a Keychain reference), or a genuinely
  compromised alias-allowed destination. The full threat model, including what a compromised host can
  and cannot do, lives in the linked document.

## Employee privacy — E1

An identity-bound, timestamped record of which destinations an agent reached is employee-behavioral
data the moment it exists — this is architectural, not a policy choice, and it holds regardless of
how the feature is used:

- **What is recorded per user:** the destination host, observed payload byte counts, a content hash,
  the policy decision, and the acting agent + operator identity (the same `actor` field every kriya
  receipt already carries) — never the request/response content itself.
- **Purpose limitation, stated once, meant everywhere it's echoed:** compliance and security
  evidence — never individual performance evaluation, productivity scoring, or general monitoring of
  an operator's work. This sentence is the one to check any downstream export or fleet purpose-field
  against.
- **Who can read it:** whoever has filesystem access to the device (device-local, the default), or
  operator/console access to the customer's own hub (their `kriyad` / Console-as-hub). On the L2
  connector lane a kriya-operated mailbox **briefly buffers** the minimized envelope in transit —
  but it can read none of it (there is no content in an envelope), never persists the source IP, and
  deletes it on the operator's ack; only the customer's own hub and cockpit can actually read the
  ledger. See "The data-boundary promise — per lane" above for the full per-lane statement.
- **Retention default:** unset (indefinite) until the operator configures one — see "Retention and
  the chain" below.
- **Per-device deny counts already reach an enrolled fleet's `kriyad`.** The minimized envelope's
  allowlisted action ids include the `kriya.io.*` facets, so an "attempted-policy-violation" tally
  per device is visible centrally even though the destination itself is not — disclosed here, not
  discovered later. Raw params (`dest_host`, `content_sha256`, byte counts) are structurally
  unreachable by the minimizer at any verbosity — see [redaction](#) below — so only the *count* of
  each `kriya.io.*` id travels, never what it names.
- **Ingress digests are OFF by default even when egress is ON**, and are keyed (HMAC under a
  device-local salt) rather than a plain hash when enabled — an unsalted hash of guessable content
  (a filename, a common phrase) is itself a content-disclosure risk ("did this agent read
  salary.xlsx?"), which a keyed hash forecloses for anyone without the salt.
- **The privacy artifact pack.** Deploying the egress/ingress ledger to employee devices is a real
  co-determination and data-protection question in many jurisdictions, not a formality — see
  [`docs/privacy/`](privacy/README.md) for a DPIA template (Art. 35), an employee-notice template
  (Arts. 13/14), and a model works-agreement clause for co-determination jurisdictions (DE/AT/NL/FR
  and similar). Treat the works-council step as a precondition to deployment there, not paperwork
  that follows it.

## Retention and the chain

Compliant deletion and tamper-evidence pull in opposite directions by default: pruning old receipts
to honor a retention limit (or a GDPR Art. 17 erasure request) normally looks exactly like an
attacker truncating the log. The design that resolves this:

- **A signed epoch-checkpoint receipt** (`kriya.retention.checkpoint`) seals a pruned prefix: its
  params attest *"receipts before T pruned per policy P; prior chain head was H"*, and its own
  `prev_hash` equals H — so it sits at the exact point the pruned lines used to be.
- **Every verifier accepts the checkpoint as a legitimate chain boundary**, not a break — the offline
  CLI, `kriya-verify`, and the TS verifier all recognize `kriya.retention.checkpoint` and treat the
  seal as sealed, the same way they already treat a genesis receipt's absent `prev_hash`.
- **Retained receipts re-chain onto the checkpoint**, re-signed by the same signing key (a prune can
  never re-attribute a receipt to a different key — that's a hard error, not a silent skip).
- **`kriya.io.*` gets a shorter default retention class than policy/approval receipts** — the
  egress/ingress ledger is the most granular, least evidence-durable data on the trail, so it is the
  first candidate for a shorter window when an operator sets one. Neither class has a retention limit
  by default; an unset `retention:` policy means indefinite retention, unchanged from before this
  feature existed.
- **The organization decides the actual retention period.** kriya provides the mechanism (the
  checkpoint design + the `retention:` policy field); it does not impose or default to a specific
  number of days. See [`docs/privacy/README.md`](privacy/README.md#retention-defaults) for suggested
  starting points tied to the compliance drivers this ledger supports.

## Verifiable Telemetry (F3) — observed vs. proven, and a flagged network question

kriya can export its own signed receipts as OTel GenAI-conformant spans (`invoke_agent`/
`execute_tool`) so your existing Grafana/Datadog/collector can hold telemetry a stranger can
independently re-verify — and it can import third-party OTel spans (e.g. Claude Code's own
telemetry) as a governed-visibility lane. Both directions honor the SAME observed-vs-proven line
the rest of this document draws:

- **Every exported span carries the receipt's own signature, not a summary of it.** Alongside
  human-readable attributes, each `execute_tool` span embeds `kriya.receipt.raw` (the exact signed
  JSONL line, base64) plus `kriya.chain.head`/`kriya.chain.prev_hash` — so `kriya-audit
  --verify-otel <file>` re-checks the real Ed25519 signature and chain position **offline**, with
  the same compiled verifier this whole document describes. A collector holding these spans holds
  something checkable, not just plausible-looking JSON.
- **Ingested spans are OBSERVATIONS, never receipts — a structurally separate store.** A
  third-party OTel export lands in its own file, never the receipt log, and the Sessions view
  renders it visibly stratified below receipt-backed rows (a dashed border, a hollow status dot, an
  explicit "observed-unsigned" badge) with a "governance gap" count for spans no receipt can vouch
  for. Nothing in this feature calls an observed span "verified" — kriya did not sign it and cannot
  prove it wasn't altered before or after it reached the collector.
- **Matching an observed span to a receipt is either a real correlation or an honest guess, never
  blended.** A `kriya.otel.traceparent` stamped into a receipt's params (when the emitting seam
  supplies one) is a genuine correlation; absent that, a time+tool-name heuristic is used and
  labeled `"heuristic"` in the UI — distinct from `"traceparent"` and from `"none"` (the gap).
- **What this build does NOT do, and why — read before assuming a live collector connection
  exists.** The shipped export/ingest path is **file-based only**: exporting writes an OTLP/HTTP
  JSON document to disk; importing reads one back in. Neither opens a network connection. A live
  push to an operator-configured collector endpoint, and a live localhost OTLP listener, are the
  two pieces of the wider F3 design (internal frontier design tracker §3) this build does
  **not** include — building either as specified (an ungated, always-compiled runtime toggle) would
  give the default free Console binary its first network-capable dependency, contradicting the
  claim two sections up ("no socket to any server ... is ever opened") and this repo's own
  `README.md` ("the free tier ... opens no network connection ... that's dormancy-tested, not a
  promise"). That is a founder/architect-level call, not a build detail, so it is named here rather
  than resolved quietly — see `src-tauri/src/otel/exporter.rs`'s doc comment for the full argument.

## Fleet telemetry (F3-fleet) — envelope-granularity spans, to YOUR collector, inside YOUR boundary

F3 (above) exports one device's own signed receipts as OTel spans. This is the fleet-wide sibling:
the `fleet-console` cockpit can export the org's aggregated evidence — pulled from the customer's own
`kriyad` — as signed OTel spans too, plus a read-only Prometheus-style metrics endpoint on `kriyad`
itself. Same observed-vs-proven discipline, applied at fleet scope:

- **Envelope granularity, not per-receipt — this is structural, not a policy choice.** `kriyad` only
  ever holds signed envelope **rollups** (see "The data-boundary promise — per lane" above): raw
  receipts and per-receipt signatures never leave the originating device. So the fleet export is ONE
  SPAN PER ACCEPTED ENVELOPE — keyed on `kriya.envelope.device_pub` + `kriya.envelope.seq`, signed by
  `kriya.envelope.sig`, chained via `kriya.chain.head`/`prev_envelope_hash` — never a per-receipt
  signature span. A collector holding a fleet export sees "device X's window of activity, rolled up
  and signed" — not the individual actions inside that window.
- **Still independently re-verifiable, offline.** `kriya-audit --verify-otel` re-verifies an
  envelope-mode span exactly like a receipt-mode one: it decodes the embedded `kriya.envelope.raw`,
  re-checks the Ed25519 signature via the SAME `kriya_verify::verify_envelope` `kriyad`'s own ingest
  path uses, and cross-checks chain continuity. A tampered span fails with the real reason; a genuine
  chained sequence re-verifies clean.
- **To YOUR collector, inside YOUR boundary — never kriya's, and never via the connector.** The
  export endpoint is entirely operator-configured (OFF by default); enabling it signs a fleet-scoped
  `kriya.otel.export.enabled` receipt (the same non-egress dial F3 uses, at fleet scope). kriya never
  hosts, proxies, or sees a copy of the exported spans — this telemetry lane targets the customer's
  own collector directly, distinct from the L2 evidence-transport connector above (which never
  carries OTel spans); the **L1 boundary-level** promise ("go to a hub the customer runs — never
  anywhere else, no kriya infra on the path") applies here without modification. Unlike F3's own free-tier build (which
  deliberately ships file-export-only, for the reason explained in the section above), this fleet lane
  DOES support a live push: it is license-gated behind `control-plane`/`fleet-console` already (the
  free tier links none of this code), so a live network call here does not touch the free-tier
  dormancy claim F3's own section is protecting.
- **`GET /metrics` on `kriyad`: aggregate governance KPIs only, never content.** Receipts/day,
  governed-connector-lane allow/deny/approve counts, per-device-id coverage completeness (`current`/
  `behind`/`silent`, labeled by device id ONLY — never a hostname or username), and the fleet's
  policy-bundle version spread. Every field is documented in its own `# HELP` line at the endpoint
  itself. `kriyad` structurally cannot leak more than this — it never held per-action rows to begin
  with. `docs/samples/grafana-kriya-fleet.json` is an importable dashboard over exactly these metrics:
  point your own Prometheus at your own `kriyad`, inside your own boundary — kriya remains the only
  signer in the chain.
- **Honest ceiling.** Grafana (or any collector) rendering this data is reading the OUTPUT of
  verification that already happened at `kriyad`'s ingest boundary and, again, independently, in
  `kriya-audit --verify-otel` — it does not itself re-verify a signature. And "no content, no
  per-action rows" is a floor, not a promise about what an operator's OWN collector retention policy
  might later do with metric label cardinality over time (device ids are stable identifiers; an
  operator correlating them against an out-of-band device inventory is an operational choice outside
  kriya's control, same as any pseudonym-correlation caveat elsewhere in this document).

## Attested Local Inference (F1) — what the proxy proves, and its honest ceiling

`kriya-llm-proxy` (in the open runtime, `../experiment1`) is a local reverse proxy you point an
OpenAI-compatible or Ollama client at instead of your model server's own port. It signs a receipt for
model identity, the policy verdict, and usage/performance on every completion it forwards.

- **Bypass honesty — read this before claiming coverage.** The proxy governs ONLY clients explicitly
  configured to route through it (`OPENAI_BASE_URL`, `OLLAMA_HOST`, or a client's own base-url
  setting). A process that connects straight to the upstream inference server — e.g.
  `http://localhost:11434` directly — is completely invisible to it: no receipt, no gate, nothing.
  This is the exact same channel-not-host-wide limitation `kriya-gateway run --`'s egress containment
  discloses (see the egress section above); closing the direct-connection gap is a watcher-ladder
  item, not something this proxy attempts. **Never say "governs all local inference" — say "governs
  clients pointed at it."**
- **Model identity is resolved, never assumed.** A digest comes from an Ollama registry manifest (the
  digest Ollama itself computed when it pulled the model) or an operator-precomputed file-hash cache
  keyed by `(path, size, mtime)` — the proxy **never hashes a model weight file on the request path**.
  When neither source has an answer, the receipt says `digest_source: "unresolved"` — honestly, not a
  fabricated or guessed digest.
- **The policy gate is fail-OPEN by default, and says so on every completion.** An unrecognized
  model's default action is `warn`: the completion still proceeds, but an explicit, signed
  `kriya.model.gate` receipt records that no rule matched. This is an observation lane first (doc 24
  §11.5 — enforcement verbs only where enforcement is real). Setting `unknown_model: require-approval`
  or `deny` in the policy's `model:` section makes the gate real: a denied or approval-tier digest
  is refused (HTTP 403), not silently allowed. There is no synchronous approval channel in v1 — an
  approval-tier model is refused outright, the same honest limitation `mcp::contain`'s CONNECT tunnel
  already discloses for network egress.
- **No content ever.** Prompts, messages, and model output are represented in receipts ONLY as sha256
  hashes (`prompt_hash`, `output_hash`) — never as text. Sampler parameters (temperature, seed, …) and
  token counts are recorded; the words the model saw or produced are not.
- **v1 scope, per doc 28 §F1 — read before assuming more.** This build ships observe-and-gate-on-
  identity only. Deterministic re-execution (byte-identical replay of a completion) is doc 28's phase
  3 and is NOT built or claimed here. OMS/Sigstore model-manifest verification is a clearly-marked
  seam in the runtime code, also not built. GPU-backend determinism claims are out of scope entirely.

## Attested pipeline (F6-T1) — measurement, not attestation

kriya can sign a `kriya.attest.pipeline` receipt describing the enforcing stack itself — a sha256
self-hash of the Console binary, hashes of the wired runtime hook/gateway binaries, the active
policy bundle's hash, and a macOS `codesign --verify` check — plus a `kriya.attest.sandbox` receipt
whenever the Seatbelt containment lane (EG-C) launches a run, mirroring the profile hash and the
config knobs in effect. Read the discipline below before treating either as proof of anything beyond
what it literally says.

- **This is Tier 1 ONLY: measurement, not attestation, and the difference is load-bearing.**
  "Attestation" in the security literature means a claim rooted in something the claimant cannot
  forge — typically a hardware root of trust (a TPM quote, a Secure Enclave, an SEV-SNP report) that
  signs *before* the OS can lie about itself. Nothing here is rooted in hardware. A
  `kriya.attest.pipeline` receipt is this same running Console process reading its own binary off
  disk, hashing it, and signing what it found with an on-device key — the identical "the app signs
  its own governance-internal events" idiom the Coverage Map (`kriya.coverage.snapshot`) and the F3
  OTel export toggle (`kriya.otel.export.enabled`) already use. **A host that is already
  compromised can lie about every field: patch the binary after hashing it, feed a stale or forged
  hash, or simply not run this code path at all.** The receipt vocabulary keeps the `kriya.attest.*`
  namespace name (a deliberate design decision — the family name is stable across tiers), but every
  UI and docs surface for this Tier-1 slice says **"measurement,"** never "attestation" — checked by
  a manual honesty grep at build time (doc 28 §F6's gate), not yet an automated CI check.
- **What each field does and does not prove.**
  - *Binary self-hash* — proves the file on disk hashed to X at the moment this process looked. It
    does not prove the process now running IS that file (a sufficiently privileged attacker can swap
    the binary after this check, or patch the running process's memory directly) and it does not
    prove X is a hash any particular reader should trust — that comparison is the reader's job (e.g.
    against a known-good release hash you obtained out-of-band).
  - *Hook/gateway binary hashes* — resolved via the SAME connector-wiring lookup
    (`onboarding::resolve_hook`/`resolve_gateway`/`resolve_hermes_hook`) the Console itself uses to
    decide what to wire. A binary that resolution can't find is recorded as the literal string
    `"UNMEASURED"` — never a guessed path, never a fabricated hash. `"UNMEASURED"` is not a failure
    state to hide; it is the honest answer when there is nothing to measure yet.
  - *Policy-bundle hash* — the sha256 of `~/.kriya/agent-policy.yaml` if present, `"UNMEASURED"`
    otherwise. Proves which bytes were on disk at measurement time; says nothing about whether the
    runtime actually loaded that exact file for the actions in question (that binding is what B0's
    `--policy` wiring already provides, receipted separately).
  - *The codesign check* — a real, non-mocked invocation of macOS `codesign --verify` plus a parsed
    `Authority=` chain from `codesign -dv`. `verified: true` with an **empty** authority chain means
    "codesigned" in the trivial sense every arm64 binary must be to execute at all (macOS ad-hoc
    signs unsigned binaries automatically) — it is NOT evidence of a trusted publisher. Only a
    **non-empty** authority chain naming a real identity (a Developer ID, or an org's own signing
    identity) is meaningful, and even then it proves the binary was signed by that identity at build
    time, not that it wasn't tampered with afterward by something with write access to the file (the
    self-hash catches *that* half, with the caveat above). `applicable: false` off macOS is an
    honest non-check, never a fabricated pass.
  - *Sandbox incarnation receipts* — `kriya.attest.sandbox` mirrors the Seatbelt containment lane's
    own `kriya.io.run.start` bookend (already signed by the runtime — see
    `crates/kriya/src/mcp/contain.rs` in the open kriya repo) into the Console's attest chain: the
    profile hash and the config knobs actually in effect (proxy port), plus the launched program's
    NAME and argument COUNT — **never its full argv**, which can carry paths, flags, or
    secret-looking values (the same "hashes/counts/names only" discipline the egress ledger keeps).
    This is an observation of a receipt the runtime already signed, re-stated under the Console's own
    signature for one place to look — it adds no trust the runtime's receipt didn't already have, and
    inherits EG-C's own honest ceiling (network-only containment, scoped to the launched process, no
    file/exec visibility — see this document's egress-governance section above).
- **Drift is surfaced, never enforced.** The Integrity card (Settings → Integrity) compares the two
  most recent pipeline measurements' binary hash and highlights a change — an *observation* for a
  human to judge (expected after an upgrade; worth investigating otherwise), never an automatic
  block. Fewer than two measurements yields an honest "not enough history yet," never a fabricated
  "no drift."
- **Tier 2/3 are named, not built.** Hardware-rooted attestation — a Linux TPM2 quote against an IMA
  log (Tier 2, Keylime-based) and an SEV-SNP TEE report for `kriyad` (Tier 3) — is the upgrade path
  that would make "the host wasn't already compromised" an actual claim this build can make. Neither
  is scaffolded here; they are demand-gated future work (doc 28 §F6), and nothing in this Tier-1 slice
  should be read as a step toward claiming their guarantee early.

## Signed artifacts (F12a) — self-managed anchor, and what "unknown issuer" means

When a governed session writes a file, Sessions can export a C2PA **sidecar** manifest (`<file>.c2pa`
— never embedded into the file itself; text/code embedding via Unicode is fragile, so embedding is a
later phase) bound to this device's receipt chain: the manifest carries `kriya.chain.head` at
emission time, and the `kriya.artifact.provenance` receipt it also emits carries the manifest's own
hash plus the target file's hash and path back — two independently-checkable directions, not one.

- **Provenance for those who keep it, not a watermark.** A sidecar file is ordinary, removable
  metadata. Deleting or replacing the `.c2pa` file removes the provenance claim entirely — nothing
  about kriya's signature survives inside the artifact itself, and nothing here claims otherwise.
- **A self-managed, self-signed certificate is the trust anchor — not a public one.** The artifact
  keypair and its certificate are generated once on-device (Settings → Artifact signing) and never
  submitted to any public C2PA/CAI trust list. Concretely: `kriya-audit --verify-artifact <file>
  <sidecar>` — the offline "stranger" check, with no access to this Console's Settings — reports the
  manifest as structurally and cryptographically **valid** (the hard binding to the file's bytes is
  checked; an edited file fails), but with exactly one honest, non-fatal issue:
  `signingCredential.untrusted`. A public C2PA validator (e.g. Adobe's or Truepic's verify tools)
  will report the same "unknown issuer" — this is expected and correct, not a bug: kriya is not, and
  does not claim to be, a member of any public C2PA conformance/trust program. Inside the Console
  itself, the SAME device's own persisted certificate is used as the trust anchor for the in-app
  check, so a manifest this device signed shows as fully trusted there — "this device recognizes its
  own key," never a public PKI claim.
- **Deployers, not providers — Art. 50 is out of scope.** The EU AI Act's Art. 50 machine-readable
  AI-content marking duty falls on *providers* (of the generative system), not on kriya's deployer
  customers (doc 28 §3) — this feature is never sold, marketed, or documented as Art. 50 compliance,
  and none of its assertions make a "this content is AI-generated for regulatory disclosure" claim.
  It attests the SESSION RECORD (who/what produced the file, under which policy verdicts, as of which
  chain position) — never code quality, never content correctness.
- **Content stays out of the manifest.** The `kriya.artifact.session` assertion carries counts
  (policy-verdict allow/deny/approve totals), hashes, and identifiers only — never file content,
  matching the C1 discipline every `kriya.*` receipt vocabulary follows.

## Deterministic Execution (F4-wasm) — what re-execution covers, and its honest ceiling

**Naming law, stated once here so it never drifts:** "Deterministic Execution" is **re-EXECUTION** —
a disputed action is actually **re-run** under a pinned Wasmtime configuration and its output is
hash-compared to what was originally produced. This is a completely different claim from **"Verified
Replay"** (B1), which **re-DERIVES** a session timeline purely from already-signed receipts, offline,
without re-running anything. The two never share wording anywhere in this document, the app, or the
CLIs — if you see "replay" applied to a WASM re-execution result, or "deterministic execution" applied
to a receipt-derived timeline, that is a labeling bug, not a feature.

**What it covers.** An opt-in lane (`kriya-run-wasm` in the runtime; policy's `exec:` section on the
Console side): a governed tool call executes as a **WASI-p2 component** under Wasmtime with fuel
metering on, NaN canonicalization on, threads and relaxed-SIMD off, and **no ambient clocks or
randomness** — wall/monotonic time and both WASI random interfaces are virtualized functions of a
recorded seed and epoch, not the host's real clock or OS entropy. Every run is captured as a
`kriya-exec-bundle/1` file: the module's sha256, the exact Wasmtime version, a digest of every
deterministic knob in force, the args/env/stdin **as bytes** (re-runnable, not just hashed — size-capped
with an honest refusal above the cap, never a silent truncation), the fuel consumed, and stdout/stderr
hashes. A signed `kriya.exec.deterministic` receipt binds only the bundle's own hash plus a handful of
summary fields (module hash, fuel, output hashes) — never the input/output content itself, the same
hashes-and-counts-only discipline every `kriya.*` vocabulary follows.

**The claim, precisely.** Re-running the **same bundle** (same module bytes, same Wasmtime version,
same recorded config digest, same args/env/stdin, same seed/epoch) on a machine with this lane
installed reproduces **byte-identical stdout/stderr and identical fuel consumption** — checked, not
asserted, in both repos' test suites, and cross-verified independently: the runtime's own
`kriya-run-wasm --verify` and the Console's `kriya-audit --verify-exec` are two SEPARATE
implementations of the identical check (a deliberate consequence of the one-way open/private
dependency boundary — see CLAUDE.md), and both report the identical verdict, including the identical
fuel-consumption number, on the same real bundle.

**The honest ceiling — read this before citing this feature in any procurement conversation.**
- **No cross-Wasmtime-version guarantee.** The bundle records the exact Wasmtime version and a
  digest of the deterministic configuration; `--verify`/`verify-exec` report a version or config
  mismatch as an honest, explicit failure rather than silently comparing across versions. A future
  Wasmtime release legitimately changing codegen for the same wasm bytes is expected, not a bug — this
  lane never claims otherwise.
- **Scoped to tools that actually ran through this lane.** A tool call executed the normal way (no
  registered WASM variant, or the "prefer deterministic lane" policy tier off) carries the SAME
  tamper-evidence every other kriya receipt does — signed, hash-chained — but makes NO re-execution
  claim. This lane never implies broader coverage than the specific calls it recorded.
- **Never a general shell-command determinism claim.** This is a WASI-p2 sandbox rung, not a
  statement about arbitrary subprocess/shell execution — those stay outside this lane's scope entirely
  (doc 28 §6: no rr/syscall recording in this phase, no macOS-native syscall replay — that path is
  explicitly rejected as a research program with no viable rr/PMU/SIP route on macOS; this WASM lane
  is the honest cross-platform answer instead of a stopgap for one).
- **A fully compromised host is still out of scope**, exactly as stated in "Honest boundaries" above —
  a compromised Wasmtime/OS could in principle fake a re-execution's result; the guarantee is against
  tampering with a bundle or module *after* an honest run, not against a host that lies about
  everything from the start.

## Spend evidence (C1) — honest limits

The Console reconstructs and **signs** LLM token spend for **governed Claude Code sessions** from the
transcript files Claude Code writes on this device — per-session, per-model token counts, priced by a
**bundled, dated price sheet**, as offline-verifiable `kriya.spend.*` receipts in their own hash
chain. Read this before treating a number here as a bill:

- **Governed Claude Code sessions, not all spend.** This reads local Claude Code transcripts. It does
  **not** see MCP broker-lane / tool-provider spend (a brokered tool's own model bill is not in the
  transcript), direct API usage outside Claude Code, local-inference spend (that is the separate
  local-model lane), or any vendor-cloud / web Claude usage. The honest label is *"governed
  Claude Code session spend evidence,"* never *"complete"* or *"all"* spend.
- **A priced estimate, not an invoice.** Costs are computed from a **named, versioned** sheet the
  receipt records by id and hash — not the amount Anthropic billed you. A model the active sheet
  doesn't recognize is recorded **unpriced** (its tokens are counted; its cost is withheld, shown as
  such), never guessed. The words *"invoice," "chargeback," "the amount you were charged"* are not
  used.
- **Settle time vs activity time.** Each receipt's signing timestamp is the device clock when the
  session was settled; the session's own activity window is recorded separately from the transcript's
  timestamps. The two are never conflated.
- **Content never enters a receipt.** Only token counts, model names, priced costs, a hashed project
  identifier, the session UUID, and timestamps are recorded — never prompts, tool inputs/outputs,
  file paths, or the working directory string. Extraction runs entirely on-device in the signer; the
  transcript text never leaves it.

Single-user GDPR scope note (same discipline as the DeviceInfo table): a `session_id` UUID plus a
hashed project id on a single-user device is still indirectly identifiable — pseudonymization, not
anonymization.

## Spend budget gate (C2) — honest limits: a trailing-state gate

Building on C1's spend evidence, the Console lets you author **USD spend budgets** (session /
rolling-day / user scope) that the runtime's `kriya-hook` **pre-execution gate** enforces before the
**next** governed action runs — blocking or routing it to approval once observed spend crosses your
threshold. Read this before treating a budget figure as precise or absolute:

- **A trailing-state budget gate.** The hook is a fresh process with no in-process rate state; it
  can only consult a spend figure written to disk moments earlier. It **blocks the next gated action
  after** observed spend crosses the threshold, gated on state as of a timestamp every enforcement
  receipt records. It does **not**, and structurally cannot, pre-count the cost of the turn it is
  gating — a turn's cost isn't known until the turn produces its own usage record. A single
  expensive turn between extractor passes lands **after** the gate check.
- **The under-count window is quantified, not hidden.** With only the always-on primary (a live
  snapshot the Console rewrites roughly every 5 minutes), the maximum under-count is about one tick,
  plus the in-flight turn, plus sub-second flush lag. With the optional statusline accelerator
  installed, the window collapses to **≈ one assistant turn** for session-scope budgets — the
  irreducible floor, since a turn's cost cannot be known before it completes. Every
  `kriya.spend.gate.*` receipt records `state_source` and `state_as_of_ms`, so a reviewer can see
  exactly how stale the enforcing figure was for that decision.
- **Budgets only ever tighten.** On the hook `pre` lane, the action-tier policy check runs first; the
  budget consult runs only after an `Allow` or a granted approval, and only ever **escalates** to
  approval/deny — it can never loosen a prior deny or an ungranted approval back to allow. Across
  multiple breaching rules, the strictest wins (`deny` > `require-approval` > `warn`).
- **Fail-closed by default on missing/stale state.** A `deny` budget defaults to blocking when its
  state is missing or older than the configured staleness window (e.g. the Console app is closed); a
  `require-approval` budget defaults to routing to a human; a `warn` budget is observe-only by
  definition and never blocks. An operator may explicitly relax a `deny` budget to fail-open — that
  loosening is itself receipted (`on_missing_state` on the enforcement receipt), never silent.
- **Rolling-day / user totals are honest-but-coarse, never exact.** Each session's whole priced total
  attributes to the UTC day of its **own** `window_end_ms` — a session spanning midnight is counted
  whole against the day it ended, never split. This is a deliberately coarse boundary, disclosed here,
  not presented as a precise per-day figure.
- **The runtime never prices anything.** Every USD figure an enforcement receipt carries — the
  observed spend, the pricing sheet id and hash — is copied straight through from the Console-written
  state file; the open runtime crate gains no pricing dependency at all.

The permitted description of this feature is **"trailing-state budget gate — blocks the next gated
action after observed spend crosses the threshold, gated on state as of T."** Every stronger-sounding
alternative — implying a fixed ceiling, an absolute promise, or that spend is structurally bounded —
is deliberately absent from this description and from the app's own copy (PolicyView, SpendView),
verified by a wording check over exactly those surfaces.

## Verified Replay (B1) — re-derivation, not re-execution

Verified Replay reconstructs a governed session's timeline **from its signed receipts alone**, and
lets anyone re-derive that same timeline, byte-for-byte, on their own machine — proving the record was
not hand-authored or reconstructed by a language model. "Replay" here means **playing back the signed
recording**, the way a flight recorder is replayed — **not** re-running anything.

**What "Verified" applies to:** the *derivation* (an offline stranger gets the identical bytes) and its
*inputs* (every receipt's signature and the hash-chain are re-checked). It does **not** claim the
agent's actions were correct, nor that the session was complete beyond what receipts captured.

**The explicit non-claims — read before conflating this with re-execution:**
- **No tool is re-run.** No shell command, no MCP call, no file write is performed again.
- **No LLM call is re-issued**, and no model output is regenerated.
- **No network request is re-sent**, and no side effect is reproduced.
- **No counterfactual is simulated** (that is I3's policy replay harness, a different feature).
- The sibling capability **F4 (WASM re-execution)** is re-*execution* — a deterministic re-run in a
  sandbox. Verified Replay is re-*derivation* of a recording. **The two are different claims and are
  never described in the same words.**

This replay contains only what receipts recorded — ungoverned lanes produce no receipt and therefore
appear in no step, a visible gap rather than a blank the derivation fills in. If any receipt in a
replay's scope fails signature verification, or its source's hash-chain breaks, the derivation
**refuses to produce a replay at all** rather than silently rendering a plausible-looking remainder —
the halt-don't-fabricate discipline (doc 26 I2) applied to this feature.

## Memory-write receipts (D1) — governed-lane memory, hash-only, observe-not-block

When an agent writes to a **governed persistent-memory surface**, kriya signs an additive,
hash-only `kriya.memory.write` / `.update` / `.delete` receipt, and the Console's Memory ledger
shows *what entered persistent memory, when, from which session and action, and under what
ingress-trust conditions*. Read the scope before treating it as memory-integrity assurance:

- **Four governed surfaces, nothing else.** On the Claude Code hook lane: `CLAUDE.md` /
  `CLAUDE.local.md` edits, `~/.claude/**/memory/**` writes, and `.claude/settings*.json`
  standing-config writes. On the MCP broker lane: memory tools the operator has **explicitly
  registered** in policy. A tool is receipted as "memory" only on this evidence — a tool merely
  *named* like memory is shown as an unconfirmed candidate, never minted as a memory receipt.
- **Explicitly out of scope:** in-context / working memory (the model's context window — kriya
  never observes it), vendor-cloud memory (off-device, no lane), and any write on a lane kriya
  does not govern — a raw shell redirection, a subprocess, or an ungoverned MCP server's own store.
  These are the GREY lanes the Coverage Map already names; memory adds no new lane, it is a
  cross-cut over the hook and MCP lanes. We say "governed-lane memory writes," never "all memory."
- **Hashes, never content.** A memory receipt carries the SHA-256 and byte-size of the written
  content, a class, a keyed path reference, and correlation to the writing session/action — never
  the content itself. (The underlying generic file-write receipt on the Claude Code hook lane does
  record tool arguments verbatim, unchanged by this feature; memory receipts add no content.)
- **Provenance is derived and honest.** The ledger flags a memory write whose session had ingress
  from a `scan`- or `block`-classed MCP server (EG-P/B12) as *untrusted-ingress-present* — a
  provenance signal for an IR team, not a poisoning verdict. A session with no governed ingress is
  shown as *"no governed ingress observed,"* never as "trusted." kriya does not claim to detect
  semantic memory poisoning; it makes the write, and the trust conditions around it, evidence.
- **Observe and evidence, not block.** kriya records memory writes; it does not prevent them in
  this build. A policy tier that gates memory writes is possible later and is not claimed here.
- **The provenance signal has a precondition.** The ingress digests this join reads
  (`kriya.io.ingress.*`) are recorded only when the operator opts in — `egress.record_ingress` on
  the hook lane, or an installed ingress/trust-class control on the broker lane — and that switch
  is **default OFF**. Until it's enabled, every memory write's provenance badge reads *"no governed
  ingress observed"* out of the box, which is the honest absence of a signal, not a clean bill of
  health. Enable ingress recording (or classify a server) to see the badge escalate.

## Behavioural baselines (B3) — what "shifted" means, and what it does not

The Console can form a **baseline** of how a governed agent has behaved on this device — which tools it
calls and in what proportions, how often its actions come back unsuccessful, the mix of egress classes
it produces, and the shape of its per-session spend — and then tell you when a stream has **shifted
versus its own baseline**. Everything below is the limit of that statement.

- **It compares a stream to its own past, not to anything good.** A shift is **not** an anomaly, an
  attack, a failure, a regression, or a quality signal. A team adopting a new tool, a refactor week, or
  a change of task shifts these distributions exactly as much as anything unwelcome would. kriya says
  *what moved and by how much*; the judgement is yours.
- **It only sees what the governed lanes recorded.** Baselines are built from signed receipts. An
  agent action no lane observed is not in the baseline and cannot shift it. This is the same
  governed-lane boundary as every other kriya claim. The spend-shape baseline is narrower still: a
  session containing any model the price sheet does not know is **left out of it entirely** rather than
  counted at an understated cost, and the card shows how many sessions that was.
- **No verdict below the minimum sample.** Until a stream has enough observations, it reads
  **BASELINE FORMING** and produces no state and no receipt. There are no verdicts on ten sessions.
  The minimum is deliberately large — 150 recorded actions for the tool, outcome and egress-class
  baselines — because a smaller one produces a number we could not stand behind. Many devices will sit
  at BASELINE FORMING for a long time, and some will never leave it.
- **The false-alarm number is a pre-registered threshold, not a measured rate you should expect.** The
  alarm threshold is set so that, *under the assumption that a stream is a sequence of independent draws
  from one unchanging distribution*, the chance it ever raises an alarm during one baseline epoch is at
  most α (default 1 %). On simulated streams where that assumption holds exactly, we measure **0.2 % to
  0.6 %** — under the stated level. Four things qualify it, and the fourth is the one that matters
  most in practice:
  1. The bound is **per baseline epoch**, not per lifetime. Epochs end and restart; the number does not
     accumulate into a guarantee about a month of use.
  2. It is **per stream and per dimension**. With S streams × D dimensions armed, the chance that
     *something somewhere* alarms in an epoch is about S·D·α, not α. The card shows how many baselines
     are armed so you can do that arithmetic yourself; kriya does not correct for it, because correcting
     for it on a single device would remove what little sensitivity the test has.
  3. The guarantee is **exact for the mixture of distributions the test averages over**, and only
     approximate for any one fixed distribution. Where a stream's true mix is close to uniform across
     many tools — the hardest case — we measure up to **0.6 %** at the shipped minimum sample, and
     **1.7 – 2.3 %** if that minimum were set as low as earlier drafts proposed. That is why the minimum
     is what it is.
  4. **Real agent activity is bursty, and burstiness breaks the assumption harder than everything else
     here combined.** On a simulated stream that merely repeats its previous tool with probability 0.3
     — which on a realistic tool mix produces a measured mean run length of 1.7, not an extreme
     setting — the measured false-alarm rate is **21 %, about twenty times the stated level**, rising
     to **29 %** when the tool mix is even across many tools. Raise the repeat probability to 0.5 and
     it is **73 % to 83 %**: a stream that has not changed at all alarms more often than not. The
     other two dimensions are hit too — at the same
     burst level we measure **4 – 5 %** for unsuccessful outcomes and **11 – 12 %** for spend shape,
     and **33 %** for a tool mix wide enough that the baseline had to bucket the long tail. Treat α as a
     floor set in advance, never as a prediction of how often this will alarm on your machine. If a
     stream alarms, the honest reading is *"the recorded mix moved"* — read the evidence on the card,
     which is why **the card states α as a threshold setting and never as a rate or a percentage.**
- **It is far more likely to miss a shift than to invent one.** The test is deliberately conservative,
  and it is slow. In simulation it catches a large shift (a tool going from 30 % of activity to 10 %)
  between about **90 % and 99 %** of the time, but a shift half that size only between about **4 % and
  22 %** of the time — the range is that wide because how visible a shift is depends on *which* tools
  absorb the moved mass, not only on how big it is, and the honest thing to report is the spread rather
  than the one recipe that flatters it. It is also slower
  the later a change arrives inside an epoch: a change immediately after a baseline is armed is
  typically flagged within about 40 further observations, while the same change arriving 200
  observations later typically takes about 300. **A quiet card is not evidence that nothing changed.**
- **The most common trigger is a tool it has never seen.** Categories outside the frozen baseline
  alphabet all land in one bucket, and that bucket started at zero — so the first two or three calls to
  a genuinely new tool are enough to cross the threshold on their own. That is arithmetically correct
  and it is usually uninteresting. The card names the category responsible, so *"this is a new tool"*
  and *"the old tools changed proportion"* are never confused for each other.
- **It never enforces anything.** A shift emits an advisory receipt and lights a card. It blocks
  nothing, routes nothing, changes no policy, and sends nothing anywhere. Clearing a shifted baseline
  is a deliberate operator action, and it is itself recorded as a signed receipt.
- **Nothing about content is recorded.** A behavioural receipt carries counts, integers, the agent's
  name, and the tool/facet identifiers that already appear in the receipts being counted — never
  commands, paths, hosts, prompts, outputs, dollar amounts, or session identifiers.
- **This is not the "drift" on the fleet screens.** There, drift means *a device is behind on its
  policy bundle*. This is behaviour, on one device, over time. The two are never described in the same
  words.

Permitted claim: *"tells you when a governed agent's recorded behaviour has shifted versus its own
earlier behaviour on this device, with a pre-registered false-alarm level and the evidence on the face
of a signed receipt."* Forbidden: *"detects anomalies"*, *"detects compromise/poisoning/misbehaviour"*,
*"catches attacks"*, *"monitors model quality"*, *"1 % false-alarm rate"* as a description of what will
happen on a real machine, and any unqualified *"exact"*, *"non-asymptotic"*, *"guaranteed"* or
*"provable"* attached to the false-alarm number. Those words are true only of the mathematical
statement under its stated null; on a real, bursty stream the measured rate is about **twenty times**
α, and the paragraph above says so with a number. **α may not be quoted as a rate or as a percentage on
any affirmative surface at all** — not "1 %", not "0.01", not "about one in a hundred", not with a
conditional attached. On a bursty stream at one dimension it is 20 – 29 × wrong, and multiplied out
over the streams and dimensions a working device has armed, "something alarmed this epoch" is not a
1-in-100 event, it is close to routine. What α honestly is, and all it may be called, is **the
pre-registered threshold setting recorded in the receipt** — a threshold, with no frequency attached to
it.

## Temporal policy conditions (B4) — honest limits

A policy may attach a small, closed set of preconditions to a rule — *"deny this action unless a
matching action succeeded earlier in this same session"* — evaluated pre-execution at the hook
gate over this session's own verified receipts, and signed into a `kriya.policy.cond.*` receipt
that records exactly which condition evaluated to what. The canonical rule: deny `git push` unless
an `npm test` run succeeded this session. Read the scope before treating it as more than that:

- **Governed-lane only.** *"A temporal condition is evidenced only by governed-lane receipts. An
  action the lanes did not observe — a test run in a raw terminal, a tool outside the hook —
  produces no receipt, so kriya treats it as **not having happened**. A temporal gate reasons over
  what the lanes recorded, never over the machine as a whole."*
- **Ordering, not code-coverage.** *"`succeeded(test)` attests that a governed test-run **succeeded
  earlier this session** — it does **not** attest that the code being pushed is the code that was
  tested, nor that the tests cover it. B4 is a **workflow-ordering** gate, not a proof of test
  adequacy or of code freshness since the last green run."*
- **One lane at a time.** *"A temporal condition is evaluated over the receipts of **the lane the
  gated action ran on** — the hook lane's own signed log. A prior action recorded on a different
  governed lane (a gateway front, the Hermes hook, the WASM runner) does **not** satisfy it, even
  when the two share a run id. The gate reasons over the chain it can actually read."* This is a v1
  ceiling, not a claim of completeness; cross-lane conditions are a v2 lever.
- **Tighten-only.** A temporal rule can only escalate an already-resolved Allow/granted-approval to
  approval/deny — it can never re-open a base policy Deny or an ungranted approval. Authoring a
  `tier: allow` temporal rule is rejected as a no-op and linted in the Console, because it can never
  have any effect.
- **Permitted claim:** *"blocks a governed `git push` when no governed `npm test` success receipt
  exists this session."* Every stronger-sounding alternative — implying the underlying code was
  actually exercised, that tests are structurally required before a push can happen at all, or that
  the precondition covers activity outside the governed lanes — is deliberately absent from this
  description and from the app's own copy (`PolicyView`, `ApprovalsView`), verified by a wording
  check over exactly those surfaces. A developer can always run the precondition in an ungoverned
  shell; the gate has no receipt for that and the bypass is disclosed, not hidden.
- **A narrower evidence set than the whole receipt corpus, on purpose.** Every `kriya.*` governance
  vocabulary (kriya's own bookkeeping about spend, memory classification, crypto attestation, and
  so on) is excluded from a temporal fold before it ever reaches an evaluator — a wildcard or
  `kriya.*`-prefixed condition selector counts only what the **agent** did, never what **kriya**
  recorded about the agent. This exclusion set is closed and named, not inferred from a receipt's
  name, and is proved identical between the live gate and the offline replay below by a shared,
  committed test fixture.
- **I3 replays temporal rules too.** The Policy CI "test before apply" simulation extends to a
  candidate policy's `temporal:` section: *"this temporal rule would have fired N× in the last
  week."* Its governed-corpus filter is deliberately **wider** than the base action-tier replay's
  own (for the reason above), so a `temporal_fired` count and the base tier's `total_replayed` count
  must never be assumed to range over the same receipts — the simulation report says so explicitly
  whenever a temporal section is present.

## Payment governance (F-4) — a decision-and-report chain, never proof that money moved

When a governed call matches a `payment`-class action gate, the hook signs the standard
`kriya.gate.payment.*` receipts **plus** a three-link purchase chain — `kriya.pay.intent` (what the
agent asked) → `kriya.pay.decision` (what the gate decided) → `kriya.pay.outcome` (what the tool
reported) — sharing one `pay_id` derived deterministically from the call itself (session id + tool
name + canonical input), so the separate pre- and post-execution hook processes chain without any
shared state. Read the boundaries before treating a purchase row as a bank record:

- **Governed channels only.** A payment the agent makes outside a governed channel — a raw shell
  command, an ungoverned MCP server's own network calls — produces **no receipt and no chain**. This
  is the same governed-lane boundary as every other claim in this document; the Coverage Map's grey
  lanes name it.
- **The honest-amount rule: an integer or nothing.** An amount enters a receipt only in the one
  unambiguous shape — a non-negative **integer** `amount` in minor units plus a non-empty `currency`
  string (the PaymentIntent-style shape). A float, a missing or empty currency, a negative, or an
  absent amount yields `amount_known: false` and **no number at all** — the Console renders
  "unknown," never a scaled guess. Extraction is best-effort **from the call's own shape**; a
  payment tool with an unrecognized parameter shape is honestly unpriced, not creatively parsed.
- **The chain proves the gate and the report — never settlement.** `intent → decision → outcome`
  proves what the agent asked, what the policy decided (`approved | held | denied`, the matched rule,
  and the human approver when one granted it), and what the governed lane's tool response reported
  back (`executed | denied | timed_out`, plus a status field only when the tool surfaced one —
  absent otherwise, never invented). It does **not** prove money actually moved: the outcome's
  source is the tool's own response, not a processor or bank statement. Reconciling against the
  processor's records is the reader's job; this chain is the evidence to reconcile *against*.
- **Denied and held are honest shapes, not gaps.** A denied payment's outcome is written at the
  decision point — the call never reaches the destination. A held payment awaiting a human is an
  intent + decision with no outcome yet, surfaced as a 2-of-3-link chain, never padded to look
  complete. A chain is marked verified only when every present link passes the same offline
  verification as every other receipt.
- **No PAN, no card data, ever.** `merchant` is a processor/host string only — the MCP server name,
  or the parsed host of a URL-bearing tool call; it is never extracted from a shell command. Custody
  of real payment credentials stays in the EG-B credential-brokering lane (its own section above,
  its own threat model); the payment chain is a witness record and holds no secret at any point.

## Annotation receipts (O-7) — who labeled what, never whether the label is right

An operator can attach a signed label to a past governed action: the Console signs a
`kriya.annotation.set` receipt into its own hash chain (`annotations.jsonl`, its own signing key),
verified by the same spine as every other receipt. Read what an annotation is before citing one:

- **It proves who labeled what, and when — never that the label is *correct*.** An annotation is a
  recorded human judgement over evidence, not a quality verdict about the action, and nothing in
  this build scores, aggregates, or adjudicates the labels.
- **The label set is closed.** `correct | incorrect | needs-review | unsafe` — four values, no free
  text anywhere in the receipt (a notes field would carry content). The optional `rubric_id` is a
  rubric *name*, never rubric content. An unknown label is rejected at the signer, never minted as
  new vocabulary.
- **Only evidence that verifies is annotatable.** Before signing, the backend re-verifies that the
  target `step_id` exists as a **verified** receipt in the audit dir — an absent or tampered target
  is rejected with an error, never silently annotated. And labels do not label labels: the receipt
  drawer never offers the Annotate row on an annotation receipt itself.
- **Latest wins; history stays.** Re-labeling appends a new receipt; the resolution reads the latest
  verified label per target, and the full labeling history is preserved by the append-only chain —
  never rewritten. An annotation that itself fails verification attributes nothing.
- **The annotator is the local operator identity, not an authenticated one.** The receipt's actor is
  the device's OS username at signing time — the same self-reported operator identity every receipt
  in this document carries, not an SSO/OIDC-authenticated principal; whoever operates this device's
  Console signs as that user. Signing is desktop-only, in the Console's compiled backend — the
  browser/TS lane holds no annotation key and signs nothing.

## Analytics (O-1 · O-4 · O-5 · O-12) — re-derivable counts over verified receipts, never a score

Monitor › Analytics (Reliability, SLOs, Topology, Posture) renders **only** numbers you can
re-derive yourself from the verified receipts in the selected range — counts, rates, and latencies.
Read the derivation rules before quoting a number from these views:

- **No composite risk score exists — anywhere, by design.** The posture panel is week-over-week
  threshold *crossings*, each a plain count with its own one-line note, under a fixed caption the
  tests assert verbatim: *"Evidence posture — counts from verified receipts. Not a risk score."*
  Nothing weights, scales, or blends the rows into an index.
- **Unverifiable rows are never data points.** Every Analytics fold counts only rows that pass
  verification. A line that fails stays visible in the Audit view (the tamper story) and is itself
  surfaced honestly — as the SLO verification pass-rate's `unverifiable` count and the posture
  panel's "Unverifiable receipts" row — excluded from the aggregates, never hidden.
- **"Did not complete" is not "blocked."** A receipt with `success: false` is rendered *"did not
  complete,"* never *"blocked"* — a policy deny and a runtime failure are indistinguishable on the
  plain action receipt. Enforcement verbs are reserved for explicit deny/hold receipts, classified
  in exactly one shared place, so a failure count is never dressed up as an enforcement count.
- **Approval latency pairs by `corr_step`, or not at all.** A held→decision latency sample exists
  only when the two receipts share a `corr_step`; a hold with no linkage is skipped, never joined by
  guesswork. An open hold counts as **pending** — a queue depth, never a latency sample (an
  undecided hold has no latency yet) — and the view labels the pending column as exactly that.
- **Governed-channel evidence only.** The topology map folds governed receipts into agents, tools,
  destinations, and models, under its own fixed caption: *"This map shows governed lanes only; see
  Coverage for what isn't recorded."* An event no lane observed is not an edge, not a count, and not
  a trend — Analytics measures the governed record, never everything the process or the agent did.

## Evidence MCP (O-10) — a read-only reader that answers only from verified receipts

`kriya-mcp --evidence` serves a local, stdio-only MCP server whose five read-only tools
(`receipts_search`, `receipt_get`, `chain_verify`, `session_tree`, `spend_summary`) let a connected
agent query its own signed history — "what did I deploy yesterday," "is the chain intact." Read the
posture before wiring it in:

- **The honesty axiom.** Every answer is computed only from receipts that pass verification against
  their own embedded public key. A line that fails is **counted and reported** (`unverifiable`),
  never surfaced as if proven; fetching a tampered line by `step_id` returns
  `found: true, verified: false` with a reason — never its contents as fact. The store is re-read
  and re-verified fresh on every tool call, so no stale or tampered-after-load view is ever cached.
- **Read-only, and not an enforcement path.** There is no executor, no policy engine, no approval
  gate, no write path, and no network socket in this mode. It is the inverse of the governed
  gateway: it can tell an agent what its record says; it cannot gate, block, or approve anything.
- **The reader is itself in evidence.** Before serving, it signs one `kriya.evidence.mcp.start`
  receipt (scope `read-only` plus the exact tool list) into a dedicated evidence log — the boot is
  queryable through the very tools it exposes. If its persistent signing key cannot be loaded it
  falls back to an ephemeral key, disclosed on stderr rather than refusing to start — a read-only
  reader's availability trade, stated not hidden.
- **It exposes receipt data to whatever model you connect.** Receipts are content-free by
  construction (hashes, counts, names — this server adds nothing they don't carry), but their
  params **are** visible to the connected LLM: action ids, actor identities, timestamps, destination
  hosts, merchant strings, spend amounts. Connecting a vendor-cloud model places that governance
  metadata in that model's context — a deliberate operator choice, named here, not a default.

## Governed launcher (F-5) — a launch attestation, not a second enforcement path

`kriya-run --pack <p> [--lane …] [--shift] -- <agent-cmd>` signs one `kriya.run.launched` receipt —
the launch attestation — and then execs the agent command. Read what that one receipt claims:

- **It proves the composition at launch, nothing after.** The receipt records `{agent, pack, lanes,
  shift}` — signed by the same runtime Signer, byte-identical to every other receipt — attesting
  *this agent was started under this pack with these lanes recorded*. It is content-free by
  construction: never the agent command's argv and never the working directory, both of which can
  carry paths, flags, or secret-looking values (asserted by test, not convention).
- **Not a second enforcement path.** Per-call governance still flows through the agent's own hook or
  gateway, exactly as elsewhere in this document. The `--lane` choices are **recorded intent** in
  this version — kriya-run does not itself intercept the launched agent's tool calls, and never
  claims to.
- **The pack on the receipt is a launch fact, not an enforcement binding.** kriya-run passes the
  pack and audit-log path to the child through its environment so downstream wiring can read them;
  whether the runtime hook actually enforced that pack's policy on the calls that followed is B0's
  `--policy` wiring, receipted separately. The launch receipt never substitutes for it.
- **Copy-first v1.** The Console composes the exact command; no process is spawned from the Console,
  and the user pastes it into their own terminal. A user who edits the composed command after copy
  launches whatever they edited — the attestation covers what `kriya-run` was actually invoked
  with, not what the Console displayed.
- **Absence proves nothing.** An agent started without kriya-run produces no launch attestation —
  same governed-channel boundary as every lane above. And the operator identity on the receipt
  (`--user`, defaulting to the OS username) is self-reported by the invoker, like every actor field
  in this document — not an authenticated principal.

## Why on-device matters here

For local and regulated apps, the audit cannot live in a cloud gateway — the data and the human are
on the device, and so the proof must be too. kriya Console verifies and aggregates **on your
machine**: the receipts, the policy, and the evidence export never leave it. That is the posture
EU AI Act record-keeping and SOC 2 monitoring expect when an agent touches real data, in exactly the
place a cloud MCP gateway structurally cannot reach.

*Questions for a security review:* **Sandeepshekhar26@gmail.com**.
