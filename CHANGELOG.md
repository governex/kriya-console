# Changelog — kriya Console

All notable changes to the Console and the `kriyad` control plane. Dates are release dates of the
signed, notarized macOS DMG unless noted.

## v0.6.9 — 2026-08-12 — One hub per org (K-Apter 0.1.8)

**The rethink (founder, 2026-08-11).** The kriya-hosted tenant stops being a transit mailbox bolted
next to the org's real hub and **becomes the org's hub** — the same binary, hosted per tenant as
**Diplo** (`DIPLO_HOSTED_HUB=1`): permanent per-tenant store on its own volume, activity records
served, policy publish, enrollment. One hub per org; the only question is who hosts it. The Console
is now a pure interface over whichever hub the org points it at — which is what clears the path to a
webapp Console. The stricter transit-only mailbox posture survives as a per-tenant opt-in, unchanged,
for orgs that want kriya holding nothing at rest. Disclosed in TRUST.md; invariants re-scoped in
DESIGN.md §5.

**Added**

- **Diplo hosted-hub serve mode** (`DIPLO_HOSTED_HUB=1`) — full hub semantics behind the tenant
  edge; refuses to start without a token store; banner states the store posture plainly. `DIPLO_*`
  env names now work everywhere (`KRIYAD_*` still accepted; the binary also ships as `diplo`).
- **Two-lane pairing in K-Apter** — *Our own server* (hostname/IP inside the org's network; zero
  kriya in the path) or *Kriya-hosted* (company code + enrollment code). The form says which lane
  needs what.
- **Open enrollment** (`DIPLO_OPEN_ENROLL=1`, own-hub lanes only) — a device joins by hostname
  alone, no code; the licence seat cap is the limit; the roster records the join path
  (`enrolled_via: "open"`). Default off. Kriya-hosted lanes always require codes.
- **Mint enrollment codes from the Console, against the connected hub** — new operator-gated
  `POST /v1/enroll-codes`; the cockpit mints remotely first and falls back to its embedded hub,
  labelling which hub issued the code.
- **Detail records over the connector, per-tenant opt-in** (`DIPLO_RECORDS_RELAY=1`, mailbox
  posture only — a hub serves records inherently): fixes fleet Audit/Sessions/Memory/Spend staying
  empty for connector-only orgs. Agent parameters remain structurally absent (stripped on-device
  before signing).

**Cockpit — the fleet dimension actually renders (2026-08-12)**

- **Every record-shaped view works for any device in the fleet, not just the host.** Pick another
  machine in the device picker and Audit (with its full filter bar), Sessions, Shift, Spend and
  Memory render their REAL bodies over that device's records — the projections are standard signed
  receipts, so they flow through the same verify-and-parse path as local receipts, re-verified row
  by row in the cockpit. Previously a fleet device fell back to one generic activity list, so the
  audit filters simply weren't there. Coverage additionally shows that device's reported facts
  (status · agents wired · policy version · last seen). Agent parameters stay absent by
  construction — that is the product's claim, and the columns say so rather than inventing content.
- **The roster moved to the Devices page** — seats, enrollment codes, mint and revoke. It was
  previously rendered only inside "Fleet correlation", a view with no sidebar entry, which made
  "mint a code for a new device" unfindable in practice.
- **The Devices page's nested Policy tab is gone** (Policy is a left-nav pillar; the tab duplicated
  it). The Evidence tab stays.

**K-Apter 0.1.8 (UX pass)**

- Menu-bar glyph rendered at its full slot (~20 pt) and a larger app-icon mark; rounded popover
  corners with a real shadow (native layer mask — no private API); the panel folds when you click
  anywhere else, Esc closes it; approvals promoted to the top; humanized action names and relative
  times; identity demoted to a reference block; Return submits pairing.

## v0.6.8 — 2026-08-10 — Two binaries, one signed wire (K-Apter 0.1.7)

The first public cut of the **two-binary** model, and the first release since v0.5.0 — the whole
0.6.x line (0.6.0 → 0.6.7) was developed and tested privately and lands here in one release.

**The shape.** Governance now runs as **K-Apter**, a standalone macOS menu-bar agent installed on
every device: it wires each detected AI agent, enforces the org policy locally (allow / deny /
hold-for-approval), shows Approve/Deny in the menu bar, signs a receipt for every action, and pushes
sealed envelopes. The **Console** is the per-org cockpit *and* the hub — it embeds the aggregator
in-process, so there is no separate server to stand up. Connect them on the same machine, directly to
a hub you host, or through the per-tenant connector mailbox at `console.kriyanative.com`
(verify → buffer → delete on ack). Licensing is an **org licence carrying device seats**, enforced at
the hub roster; devices carry no licence file and never phone home.

**Added**

- **K-Apter (0.1.7)** — its own signed, notarized `.pkg`; menu-bar approvals with the real command
  text, live status (you · org · hub · policy version), last-N receipts, agents-on-this-Mac with a
  *Govern now* button, pair / un-pair, start-at-login. New agents installed *after* setup are
  detected and governed within 60 s, and announced.
- **One-paste pairing** — the admin shares a single `kriya1.…` code that carries the hub address and
  the enrollment code together (checksummed, so a truncated paste is refused, not mis-dialled).
  Typing a URL and a code separately still works.
- **Fleet cockpit** — Overview (is my fleet ok?), Devices with roster and seats, org-wide Activity
  with per-device lenses, one org Policy surface published to the fleet, whole-org Evidence reports.
  A device scope picker runs through every view.
- **The connector lane** — device push over bearer tokens, `POST /v1/enroll` with argon2id enrollment
  codes and seat caps, mailbox ack + delete-after-ack retention, and a TLS+token serve mode for a
  direct hub without a certificate ceremony.
- **Org policy distributes over the connector.** The cockpit publishes an org-key-signed bundle to
  its tenant mailbox and devices poll it back, so "author once, enforced fleet-wide" now holds on the
  connector lane and not only on-device or against a hub you run. The device enforces the bundle's
  scope itself (it was previously applied only by whoever served it), the mailbox keeps your current
  policy rather than your policy history, and a bundle published to the wrong tenant is refused
  rather than stored. Disclosure: on this lane kriya can read the policy you publish — it can never
  author, alter or sign one, and reads no agent activity. See `docs/TRUST.md`.
- **The cockpit machine pairs itself** — it enrolls its own K-Apter against the hub it already uses;
  no code to type on the machine that mints them.

**Fixed**

- A connector tenant that pinned no org policy key handed devices no trust anchor at enrollment, so
  they never fetched policy and fell back to the permissive built-in default — enrolled, reported as
  current, and effectively ungoverned. Enrollment now carries the anchor, and the install guide's
  acceptance test checks for it before any device is enrolled.
- An already-wired config never picked up a new approval mode, so a machine set up before K-Apter
  existed kept drawing the `osascript` window forever. It now self-heals on the watch tick, and
  touches only kriya's own wiring.
- The policy anti-rollback watermark was global rather than per-org, so a device that had seen a
  higher version from one org silently refused a new org's policy.
- A stale `kriyad` could survive a machine wipe, hold the port, and 401 the device with an orphaned
  token DB. The hub script preflights the port and the reset script stops it.
- K-Apter hid pending approvals when un-enrolled — enforcement is local, so a machine holding an
  action now always has UI to answer it.
- Policy authoring reports its org-wide publish result instead of saying nothing either way.

**Security** — four findings from an enterprise-surface review, each with a regression test:

- The detail-records lane decided which parameters to ship using the envelope *name* allowlist, which
  admits app actions whose parameters **are** user content (note bodies, transaction detail). Records
  now carry parameters only for the audited `kriya.*` governance vocabulary; app content stays on the
  device, as documented.
- Token authentication ran a memory-hard argon2 verify before any rate limit, and the well-known
  operator handle exists on every hub — so unauthenticated floods drove unbounded CPU. A
  pre-authentication throttle now bounds it.
- `device_label` at enrollment was unvalidated free text stored and shown in the operator roster; it
  is now length-bounded and stripped of control/markup characters.
- The device evidence key, pepper and enrollment token were written world-readable and chmod'd
  afterwards. They are now created `0600` atomically, inside a `0700` directory.

**Removed — the desktop-reach integration (D1, 2026-08-07).** The reach-in / computer-use / router
desktop fronts are gone from the runtime (the library refocused on govern/audit), and every Console
surface that drove them goes with them:

- **Connections** loses the "Govern a desktop app" connector and its macOS-permission steps; the
  Govern All surface no longer lists a desktop-apps target (and with it the `needs-permission`
  state, which nothing else produced).
- **Settings** loses the Permissions pane (Accessibility / Screen Recording existed only for the
  desktop fronts); **Get Started** drops the permission step — setup is now two required steps.
- **Coverage Map**: the desktop-apps lane is NOT deleted — it stays on the map as an honest,
  permanent gap: *"Desktop apps — not governed. The desktop-reach lanes were removed 2026-08-07;
  agent desktop activity outside governed channels is not recorded."* Always grey, never green.
  Legacy desktop-reach chain files from older installs remain valid signed history in Audit, but
  no longer light any coverage lane.
- The bundled `kriya-gateway` sidecar is built without the `reach-in`/`computer-use`/`router`
  features (proxy/broker only).

Last shipped in v0.5.0. Existing receipts verify byte-unchanged.

## v0.5.0 — 2026-08-06 — The observability wave: see more, verify more

See more of what your agents do — and let anyone verify more of it — without a single new trust
assumption.

**New in the app:**

- **Sessions Timeline** — per-action durations captured in both lanes (the hook's identity-keyed
  pre→post marker; the gateway/llm-proxy measured in-process) as two **additive optional** params
  (`kriya.dur.ms`, `kriya.dur.basis`); each run renders as a waterfall — a bar only where a
  receipt carries a real measured duration, an instant marker where it doesn't. **Absent is
  shown, never invented.** Exported OTel spans gain real timing the same way.
- **Analytics › Topology** — agents → servers/tools → destinations & models, folded from verified
  receipts in range; edge weight = count, denies marked **typographically** (never color); node
  click → counts, top actions, recent receipts, view-in-Audit. The map states its own boundary:
  *"governed lanes only — see Coverage for what isn't recorded."*
- **Annotation receipts** — `kriya.annotation.set`: attach a signed human label to any past
  verified action. The label set is a **closed enum** (correct · incorrect · needs-review ·
  unsafe) with **no free text anywhere**; the latest label wins and every prior label stays
  signed in its own hash chain. Audit gains an **Annotated** lens; the receipt drawer gains an
  **Annotate** row. Speaks the pre-stable OTel GenAI evaluation vocabulary
  (`gen_ai.evaluation.*`, evaluator = human).
- **Config-change evidence** — Memory joins config-class writes (CLAUDE.md, settings, registered
  MCP memory) with behavior-baseline shifts recorded within a disclosed 48h window after:
  *"behavior baseline shifted N h after this change (advisory)"* — correlation, **never
  causation**, and "device-wide" is spelled out when a shift can't be tied to the writing agent.
- **Evidence MCP server** — `kriya-mcp --evidence`: five read-only tools over the local audit
  store (`receipts_search` · `receipt_get` · `chain_verify` · `session_tree` · `spend_summary`).
  Every answer is computed **only from receipts that pass verification** — a tampered line is
  counted and excluded, never surfaced. The reader's own boot is a signed receipt. Connections
  gains the card + a copy-paste `mcp.json` wiring it through the governed lane.
- **The verify-it-yourself ladder** — `docs/VERIFY-OFFLINE.md`: a self-verifying HTML page (any
  browser, airplane mode), `kriya-audit` one-liners, and a raw 3-command OpenSSL recipe from the
  public key alone (the recipe is executed in CI against a real signed fixture). Evidence gains a
  per-lane **self-verifying HTML export** (complete-chain-gated — a scoped subset would show a
  false chain break, so scoped exports route to the CLI ladder). Plus a **Grafana panel plugin**
  (`tools/grafana-kriya-verify/`) that re-verifies kriya-exported OTel spans in the dashboard's
  own browser — VERIFIED / BAD-SIG / NOT-A-KRIYA-SPAN, three-way parity-tested against the CLI
  and the app verifier.
- **Remote approvals, step 1** — the pager decision core: which tiers page, the redacted payload
  format, Settings › **Notifications**, and the Approvals delivery row. The outbound transport
  deliberately did **not** ship: an egress client in the free build would break the
  dormancy-tested *"opens no network connection"* guarantee — the send lane is parked on a
  recorded decision, and the UI says so honestly.

All additive: no schema change, no new trust assumption — old receipts verify byte-unchanged.

## v0.4.0 — 2026-08-06 — Payments, shift reports, the governed launcher, and Analytics

The v0.4.0 wave: govern the money, govern the night, start governed — and read your own reliability
from your own verified receipts.

**New in the app:**

- **Agent payment governance** — the `payment` gate class is promoted from view-only preview to a
  **primary enforced dial** (Allow / Receipt-only / Approve / Deny). A payment-shaped call on the
  governed Claude Code hook lane produces a signed, **content-free**
  `kriya.pay.{intent,decision,outcome}` chain — the intent (merchant host + a best-effort amount),
  the policy decision against your per-txn cap and day spend, and the real outcome. A denied
  payment closes the full 3-link chain synchronously; a held one is the honest 2/3 shape until a
  human decides. Amounts are honest: an unparseable amount reads **"unknown", never a guess**. A
  card number **never enters a receipt** — custody stays with credential brokering. New Spend ›
  **Purchases** tab (merchant · amount · decision · I→D→O ✓-steps → receipt drawer); payment
  approvals show the amount against your cap; Today's spend card counts purchases and holds.
- **Shift reports** — sign a report of what a governed agent did across a declared **unattended
  window** (default 22:00–07:00), plus a signed `kriya.attest.shift.gap` receipt for every
  heartbeat gap — **visible by absence, never smoothed over**. Arm a shift and the hook lane runs
  a **fail-closed clamp**: a missed heartbeat inside the window tightens the action tier
  (approval-hold or deny), each clamp its own signed receipt. Tighten-only; inert when disarmed.
  New Compliance › **Shift reports** view + the Today Overnight-shift card.
- **Governed run launcher** — Start › **New governed run** composes an agent (Claude Code, Cursor
  CLI, headless SDK, cron) + a policy pack + the lanes you want into a single `kriya-run …`
  command; the runtime bin signs one `kriya.run.launched` attestation (content-free: agent, pack,
  lanes, shift — never argv or cwd), then execs the agent. Copy-first: it records how the run was
  launched; per-call governance still flows through the agent's own hook. Sessions run cards gain
  a `pack:` chip when a launch started the run.
- **Analytics** — a new Monitor view over your own verified receipts, three tabs:
  **Reliability** (actions, success rate, deny/hold split, denies/day + failures/day charts,
  top-failing tools, per-agent trends, error facets — every element deep-links into the Audit
  log), **SLOs** (approval latency p50/p95 per gate class paired from held→decision receipts,
  verification pass rate, deny rate, budget headroom, heartbeat gaps), and **Posture**
  (week-over-week threshold crossings, captioned verbatim: *"Evidence posture — counts from
  verified receipts. Not a risk score."*). **No composite score exists anywhere** — that's the
  point.
- **Charts** — the Console's first chart layer: hand-rolled SVG, zero new dependencies. Spend
  gains a 30-day **Trend** (priced spend/day + tokens/day); Local Models gain per-model
  **p50/p95 latency** ("no latency data" when the receipts carry none — never a fabricated 0 ms).

**Honest scope, as always:**

- Payment enforcement runs **where the pre-execution hook runs** (the Claude Code lane); other
  lanes are observed. Never sold as PCI or DLP.
- The shift report is **measurement of the governed record, not a promise nothing happened
  off-lane** — and it says so in the UI.
- `kriya-run` is a launcher, **not a second enforcement path**.
- Analytics failure counts render "did not complete", never "blocked" — deny wording stays
  reserved for explicit enforcement receipts.
- Every new receipt vocabulary is additive + optional on the frozen envelope: **old receipts
  verify byte-unchanged**, and TS↔Rust parity fixtures lock each format.

## v0.3.4 — 2026-07-30 — Today, action gates, and packs

The R30 wave: open the app and see what your agents did — and gate the actions that matter.

**New in the app:**

- **Today** — the new default landing view: what your agents did in the last 24 h as human-readable
  **headlines** (deploys, publishes, releases, denies) with routine actions rolled up, the approvals
  that need you, and the receipt-chain / coverage / spend posture in one strip. A **severity lens**
  on the Audit log (Headline · Notable · Denied · per-class chips) rolls routine actions into
  per-hour rows; the classic ledger stays byte-identical under the default "All" chip. One shared
  **receipt drawer** everywhere: claim, signature verdict, live chain re-check, payload, raw-line
  export.
- **Action gates** — seven high-stakes action classes (deploys · destructive git · publishes+deps ·
  production DB · infra · outbound sends · agent self-modification), each set to Allow /
  Receipt-only / Approve / Deny. **Enforced pre-execution in the Claude Code hook lane** as a
  tighten-only escalation over your existing rules; every decision is a signed
  `kriya.gate.<class>.{evaluated,held,approved,denied}` receipt, with the blocked attempt's own
  action receipt chain-linked beside it. Self-modification (the agent editing its own hooks,
  settings, or `mcp.json`) ships **Deny by default**. The payments card renders as a disabled
  preview — it arrives with the payment-governance release.
- **Policy packs** — Developer / Analyst / Planner: named, versioned presets over the gates
  (+ an egress-lock lane choice), applied to the device in one click and **receipted**
  (`kriya.policy.pack.applied/changed`, signed on the Console's own chain). Duplicate-then-edit for
  custom packs; per-identity assignment recorded for the launcher and fleet distribution to consume.

**Honest scope, as always:**

- Gates enforce **only where the pre-execution hook runs** (the Claude Code lane today). Receipts
  arriving from other lanes are classified and displayed — the UI says "observed lanes", never
  "blocked".
- "Blocked"/"Held" wording appears **only** on explicit runtime deny/hold receipts. A plain
  `success:false` action receipt reads "did not complete" — a policy deny and a runtime failure are
  indistinguishable on that receipt alone.
- Dependency installs are **counted and aggregated, never judged** — no "unknown package" claim
  ships until a first-seen ledger exists.
- One policy file per device: a pack **applies to the device**; per-identity assignments are
  bookkeeping until per-identity enforcement exists, and the UI labels them that way.
- The severity classifier and the gate matchers compile from **one shared table**, parity-locked
  across the TS and Rust implementations by committed cross-repo test vectors — the same discipline
  as the signed-receipt byte parity.

Dogfood: this release was cut under its own governance — the DMG build, codesign, notarization,
and publish appear as release-class headlines in the Today feed of the capture dataset.

## v0.3.3 — 2026-07-28 — memory, time, and behaviour

Four items from the hard-to-copy build programme, three of which add a new way to *notice* something
the previous releases could record but not reason about.

**New in the app:** a **Memory** view — signed, hash-only receipts for every write to a governed
persistent-memory surface (`CLAUDE.md`, the Claude memory dir, `.claude/settings*.json`, and
operator-registered MCP memory tools), with a provenance badge that escalates when untrusted ingress
is present in the same run. **Policy gains temporal conditions**: a rule may now require that
something already happened in this session before it allows an action — *"don't allow a governed
`git push` unless a governed `npm test` succeeded earlier in this session."* And **Monitor gains
Behaviour baselines**: per-agent, per-dimension baselines formed from a stream's own signed receipts,
flagging when later behaviour stops looking like the baseline.

**New for fleet operators (control-plane licensed):** kriyad speaks OTel at the fleet boundary — one
span per accepted signed envelope to *your own* collector, plus aggregate governance KPIs on
`GET /metrics` and an importable Grafana dashboard.

Honest scope, as always: the memory lane **observes and evidences, never blocks**, and does not claim
to detect semantic memory poisoning. Temporal conditions are **session-scoped and per-lane** — they
tighten, never loosen, an existing decision. Behaviour baselines are **advisory only**, carry no
enforcement lever, and their false-alarm level rises materially on bursty traffic — the card
deliberately never quotes it as a rate. Details in each section below.

### B3: local drift sentinel (doc 27 §4)

- **On-device, non-egress behavioural baselines over a device's own signed receipts.** For each
  `(dimension, agent-stream)` pair — tool mix, unsuccessful outcomes, egress-class mix, and
  per-session spend shape — the Console forms a baseline from that stream's own first N verified
  receipts, then runs an anytime-valid sequential test over what follows, and signs an advisory
  `kriya.drift.observation` when the evidence against "same distribution" crosses a pre-registered
  threshold. Entirely on-device, from receipts only: no content, no enforcement, no claim that a
  shift is bad. See [`docs/TRUST.md`](docs/TRUST.md)'s new "Behavioural baselines (B3)" section for
  the full honest-scope statement, including the measured false-alarm rate on real (bursty) traffic.
- **The statistic (D6/D7): a pooled conditional predictive sequential Bayes factor** — `E_n =
  KT(b)·KT(x)/KT(b⊕x)`, nothing estimated, `E[E_n|b] = 1` proved exactly by exhaustive
  rational-arithmetic enumeration (no sampling). Parity-safe: integer operands, exactly two IEEE
  double operations per step in a fixed order, no `ln`/`exp`/`pow`; every receipted number is an
  integer (`e_milli`/`tv_milli`/`threshold_milli`/`alpha_milli`), with `e_milli` capped at `2^31-1` —
  confirmed load-bearing (not dead code) in a real signed run on this machine, where the degenerate
  `m_min`-suppressed path drove `e_milli` to exactly `2147483647`.
- **NEW `src-tauri/crates/kriya-verify/src/drift.rs`** — the canonical, pure detector: corpus
  folding (D2/D3/D4), alphabet freezing (D5), the e-process (D6/D7), and the no-unsigned-state epoch
  replay (D8) that recomputes every stream's current status from the corpus plus its own prior
  `kriya.drift.baseline` receipts. **`src/lib/drift.ts`** is the load-bearing TS mirror — what the
  Monitor card actually renders, not a test-only shadow — kept honest by shared JSON fixtures
  (`test/fixtures/drift/`), never shared source.
- **`src-tauri/src/policy_sim.rs`** — `is_behaviour_corpus_excluded`, a structural superset of B4's
  `is_governance_internal_b4` plus two narrowing arms of its own (`kriya.watch.*`,
  `kriya.model.serve` — excluded here for a proportion question even though B4 counts them for its
  presence question). The whole predicate collapses to a bare `kriya.` namespace check for every id,
  which is why B3's own `kriya.drift.*` vocabulary is excluded from its own corpus automatically.
- **`src-tauri/src/drift/mod.rs`** — the Console wiring: reads the audit dir, folds every dimension,
  signs transitions only (`COLD→STABLE`/roll/manual-rebaseline baselines, one `SHIFTED` observation,
  never per-tick) into `~/.kriya/audit/drift.jsonl`, idempotent by `(epoch_id, state)` tail-scan.
  `drift_status`/`drift_rebaseline` Tauri commands; `drift.jsonl` joins the Coverage Map's own-chain
  exclusion list (`coverage.rs`) alongside `coverage.jsonl`/`attest.jsonl`/`spend.jsonl`.
- **`MonitorView`'s new "Behavior baselines" card** — one row per armed stream×dimension, the
  armed-baseline count (the multiplicity disclosure), the receipt-backed status alongside a live
  recomputation with a loud mismatch flag, and a Rebaseline button on `SHIFTED` rows. The word
  "drift" never appears on this surface (it already means policy-**bundle** drift on the fleet
  screens, `policyDrift.ts` — a different object); the card may never quote the false-alarm level as
  a rate or percentage, enforced by a widened wording lint (`test/b3-wording-lint.test.ts`) that
  scans for any numeric percent literal, not just two hard-coded strings.
- **NEW `scripts/drift-sim.mts`** — the committed measurement harness (`npm run drift:sweep`),
  importing the real `src/lib/drift.ts` detector: a checked-in deterministic SplitMix64 PRNG (never
  `Math.random`), the exhaustive-rational-arithmetic exactness check, false-alarm and detection-lag
  sweeps, and the ONE burstiness generator (a stream's baseline is its own first B observations —
  not an i.i.d. baseline spliced onto a bursty monitor). `test/drift-sim.test.ts` runs a seeded,
  cheap subset of the same functions inside `npm test`.
- **Runtime (`../experiment1`, public): no source change.** `kriya.drift.observation`/`.baseline`
  added under `internal` to the mirrored `governance-filter-ids.json` fixture in both repos — B4's
  own namespace default-exclude already covers them with zero predicate change.
- **OT-1 registration** — `kriya.drift.observation`/`.baseline`, both `no_equivalent`, in all three
  surfaces (`registry.ts`, `semconv.ts`, `kriya-verify/src/otel.rs`).

### B4: temporal policy conditions (doc 27 §4)

- **A policy may attach a small, closed set of session-scoped, cross-event preconditions to a
  rule** — *"deny this action unless a matching action succeeded earlier in this same session"* —
  evaluated pre-execution at the hook gate, over this session's own verified receipts, and signed
  into a `kriya.policy.cond.*` receipt recording exactly which condition evaluated to what. The
  canonical rule: deny `git push` unless an `npm test` run succeeded this session. See
  [`docs/TRUST.md`](docs/TRUST.md)'s new "Temporal policy conditions (B4)" section for the full
  honest scope statement.
- **Runtime (`../experiment1`, public): `crates/kriya/src/permissions.rs`** — additive
  `Policy.temporal: Option<TemporalPolicy>` (BC-3, byte-identical when unauthored); four closed
  predicate forms (`happened`/`succeeded`/`count`/`since_minutes`) over a
  `Selector{action, command, outcome}`; tighten-only (a matched `tier: allow` rule is always a no-op
  `Pass` — it can never re-open a base policy deny). **`Policy::check` and every existing `Rule`
  literal are untouched.**
- **NEW `crates/kriya/src/session_cond.rs`** — the session-scoped fold + cache feeding the
  evaluator. The B4 governed-corpus filter, `is_governance_internal_b4`, is a STRUCTURAL SUPERSET of
  the shipped `is_governance_internal` (never modified — the founder's "older build should not
  broke" ruling): every `kriya.*` id is governance-internal by default unless on a small, named
  allowlist (`kriya.watch.{proc,file,net,dns}.*` — real observed machine activity — and
  `kriya.model.serve` — the agent's own forwarded model call); a future `kriya.*` family is excluded
  automatically. The cache rides C2's own `state_dir_for_audit_log` seam (no new CLI flag,
  no rival store), tail-appending on a cache hit and fully rebuilding on any log/cache mismatch.
- **Hook-lane wiring** — `kriya-hook.rs`'s `pre` gate, inserted between the resolved action-tier
  `Decision` and the C2 budget gate (temporal is a cheaper workflow-ordering precondition, checked
  before any spend/keychain work runs). `on_unavailable` (`fail-closed`/`fail-open`/
  `require-approval`, defaulting per-tier) covers the case where the session index itself can't be
  computed (a genuine I/O error — never a merely-empty session, which is a valid state).
- **Console (this repo): `src-tauri/crates/kriya-verify/src/simulate.rs`** — the I3-replay twin,
  `SimTemporalPolicy`/`simulate_temporal`, hand-ported and kept honest by shared fixtures (the
  established one-way-dependency discipline). **`src-tauri/src/policy_sim.rs`** — the Policy CI
  extension: a candidate policy's `temporal:` section is replayed over windowed verified receipts
  grouped by `(source log, run_id)` — never across lanes — reporting `temporal_fired` +
  `temporal_by_rule` ("this temporal rule would have fired N× last week") and an explicit
  `temporal_corpus_note` whenever the temporal fold's (deliberately wider) governed corpus diverges
  from the base tier replay's own. Additive `temporal_fired` field on the signed
  `kriya.policy.sim.result` receipt.
- **`src/lib/policy.ts`** — the full TS mirror (`TemporalPolicy`/`Selector`/`Condition`/
  `evaluateTemporal`), BC-3 YAML round-trip, and two authoring lints: a `tier: allow` rule (can only
  restrict, never re-open) and a command-less Bash `count`/`since_minutes` selector (matches every
  Bash call, not a specific command).
- **`PolicyView`** — a new "Temporal conditions" form-based editor (never free-text — the closed
  predicate set stays closed by construction), plus a `temporal_fired`/corpus-note readout in the
  "Test before apply" simulator. **`ApprovalsView`** — a condition badge on a routed
  `kriya.policy.cond.approval`, rendering each `when:` condition as `predicate(selector) →
  observed`, e.g. `succeeded(bash contains "npm test") → false`.
- **Three-twin parity, proved by mirrored, committed fixtures** — `TemporalPolicy::evaluate`
  (runtime) ↔ `simulate_temporal` (kriya-verify) ↔ `evaluateTemporal` (TS), over **F-A** (the fold:
  two real, Ed25519-signed agent bash receipts survive a mix with `kriya.spend.gate.warn`/
  `kriya.memory.write`/`kriya.artifact.provenance` sharing the same session id), **F-B** (the
  evaluators, a wildcard `count()` rule), and **F-C** (the governance predicate + its superset
  property against the shipped, unmodified base predicate).
- **Closed a pre-existing OT-1 registry gap, landed as its own self-contained commit**:
  `kriya.exec.deterministic`, `kriya.memory.{write,update,delete}`, and
  `kriya.retention.checkpoint` were shipped vocabularies with no entry in the OTel semconv
  registry — found while auditing every `kriya.*` family for B4's governed-corpus filter, fixed
  independently of (and before) B4's own `kriya.policy.cond.*` registration.
- **`.warn` reserved, not emitted in v1** — `TemporalDecision` has no `Warn` variant; the observe-only
  path is a documented v2 lever, so the "a temporal `tier: allow` rule is a rejected no-op" invariant
  has no carve-out.
- **Real, live proof on this machine**: a `kriya-hook`-signed `git push` was denied (exit 2) with no
  prior test this session, the signed `kriya.policy.cond.deny` receipt showed the exact failed
  condition (`succeeded(bash contains "npm test")`, `observed: false`, `match_count: 0`), and the
  SAME push was allowed after a real signed `npm test` success landed on the same session — all
  through the compiled `kriya-hook` binary, spawned as a real subprocess, never a mocked evaluator.
  I3's `simulate_policy` replayed the same scenario over real signed receipts and reported
  `temporal_fired: 1`, attributed to the right rule id. See
  [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md) (§A19).

### D1: memory-write receipts (doc 27 §4)

- **Signed, hash-only receipts for writes to a governed persistent-memory surface** — `CLAUDE.md`/
  `CLAUDE.local.md`, `~/.claude/**/memory/**`, `.claude/settings*.json` on the Claude Code/Hermes hook
  lane, and an operator-**registered** MCP tool on the broker lane. Observe and evidence, never a
  block; kriya does not claim to detect semantic memory poisoning. See
  [`docs/TRUST.md`](docs/TRUST.md)'s new "Memory-write receipts (D1)" section for the full honest
  scope statement.
- **Runtime (`../experiment1`, public): `crates/kriya/src/memwrite.rs`** (new, shared) — the
  four-class path classifier (`claude-md`/`claude-memory-dir`/`claude-settings`/`mcp-registered`) and
  the hash-only `emit_memory_receipt` both `kriya-hook` (Claude Code) and `kriya-hermes-hook` (Hermes)
  call from `post`, unconditionally. New additive `action_id`s `kriya.memory.write`/`.update`/
  `.delete` (`audit.rs`); all fields ride under a reserved `params["kriya.memory"]` sub-key (mirrors
  `kriya.corr`'s placement). `write`-vs-`update` comes from tool shape, upgraded to `update` only when
  a bounded in-run tail scan (≤500 lines) finds an earlier write to the same path — the `verb_basis`
  field always discloses which, never implying more certainty than a stateless hook actually has.
- **Broker `memory:` registry** (`permissions.rs` `MemoryRegistryEntry`, BC-3 unauthored round-trip)
  — `Governor::dispatch` emits `kriya.memory.<verb>` for a REGISTERED tool only; an unregistered tool
  whose name merely *looks* like memory (`mcp__*memor*__*`, `*__save`, …) mints nothing — the honest
  default the design calls for, never a bare name heuristic.
- **Console (this repo): `src/lib/memory.ts`** (new) — a pure, VERIFIED-only view-model (mirrors
  `spend.ts`): per-target (`path_hmac` for files, `server+tool` for MCP tools) write/update/delete
  timelines, a derived provenance badge (`untrusted-ingress-present` / `trusted-only` /
  `none-observed`) joining the run's `kriya.io.ingress.*` receipts via the operator-authored MCP
  trust-class policy, best-effort derived reads, and advisory candidates (unregistered heuristic MCP
  tools + unconfirmable relative memory-dir paths — surfaced, never minted as a signed receipt).
- **`src/lib/policy.ts`** — the `memory:` registry mirror (`MemoryRegistryEntry`, BC-3 parse/serialize
  round-trip), alongside the existing `TrustClass`/`McpResponsePolicy` mirrors.
- **New "Memory" nav entry** (`src/views/MemoryView.tsx`, after "Spend") — per-target write timeline
  with verify ticks, the ⚠ untrusted-ingress banner naming the offending server + trust class, an
  advisory-candidates section, drill-in to Sessions.
- **Honest disclosure (design review F3):** the provenance badge is inert (always
  `none-observed`) until the operator enables ingress recording (`egress.record_ingress`, hook lane)
  or installs a trust-class control (broker lane) — both default OFF. Documented in `TRUST.md` and
  `MemoryView`'s help copy.
- **TS↔Rust parity fixture** — `d1-memory-receipts-ledger.jsonl`, a real Ed25519-signed chain covering
  all four classes, committed byte-identical in both repos, verified by `src/lib/verify.ts`.
- **Manual proof on this machine**: a real `kriya-hook`-signed `CLAUDE.md` write and memory-dir update
  landed in the actual on-device audit log and were re-verified through the production code path
  (`loadAuditLog` → `summarizeMemory`, the exact view-model `MemoryView` renders) — both targets
  shown, 0 verification failures across the loaded chain; a second, sandboxed run with
  `egress.record_ingress: true` and a `scan`-classed MCP server showed the provenance badge correctly
  escalate to "Untrusted ingress present," naming the server and its trust class. **Limitation,
  honestly disclosed:** the build session had no Tauri display to launch the actual GUI, so the
  proof is the redacted receipt + the real view-model's output, not a `MemoryView` screenshot — see
  the D1 PR description for the redacted receipt + ledger-state dump. See
  [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md) (§A18).

### F3-fleet: kriyad speaks OTel — the fleet telemetry boundary (doc 28 §F3-fleet, control-plane-licensed)

- **Envelope granularity, not per-receipt (doc 22 §2/§11-B1).** kriyad only ever holds signed envelope
  ROLLUPS, never individual receipts — so the fleet export lane is ONE SPAN PER ACCEPTED ENVELOPE,
  keyed on `kriya.envelope.device_pub` + `kriya.envelope.seq`, the signature is `kriya.envelope.sig`,
  and `kriya.chain.head` is the hash of the canonical envelope bytes chained via
  `prev_envelope_hash` — a materially different scope from F3's own per-receipt export. (The originally
  scoped per-receipt-signature design was rejected by a build-agent STOP-on-contradiction and
  corrected in place before this build; see the internal frontier design tracker's amendment note.)
- **`kriya_verify::otel::map_envelope`/`verify_envelope_span`** (`crates/kriya-verify/src/otel.rs`) —
  the envelope-mode Rust twin, reusing (never forking) F3's `attr_str`/`attr_int`/`base64_encode`/
  `SCHEMA_VERSION` primitives; `kriya-audit --verify-otel` gained an envelope-mode branch, dispatched
  per-span on `kriya.envelope.raw` vs `kriya.receipt.raw` — a mixed export is supported.
- **`fleet_otel_export`** (`src-tauri/src/control_plane/fleet_otel.rs`, new) — pulls the fleet's
  envelope evidence via the existing P0/P5 windowed mTLS client (re-verified LOCALLY), builds the
  OTLP/HTTP JSON document, and either pushes it live **to the operator's OWN configured collector**
  ("to YOUR collector, inside YOUR boundary" — kriya never hosts or sees a copy) or writes it to a
  file (network-free path, always available). OFF by default; a false→true transition on
  `fleet_otel_settings_set` signs a fleet-scoped `kriya.otel.export.enabled` receipt (OT-3/OT-7).
  **Architecture note:** this lane lives in the Console's `fleet-console` role, not in
  `kriya-aggregator`/kriyad — kriyad's own crate doc states "no outbound calls" as an architectural
  invariant, and this module reuses the ALREADY `control-plane`-feature-gated `reqwest` dependency
  rather than opening kriyad's first outbound call. Every symbol is dormancy-gated exactly like every
  other `control_plane/` module (BC-1).
- **`GET /metrics` on kriyad extended** with AGGREGATE governance KPIs — `kriyad_receipts_total` /
  `kriyad_receipts_by_day` (per-UTC-day, capped 30 days), `kriyad_egress_allow/deny/approve_total`,
  `kriyad_device_coverage{device_pub,status}` (device id only — never a hostname or username, doc 22
  §7 minimization), `kriyad_policy_bundle_applied_devices{version}` /
  `kriyad_policy_bundle_latest_version` — every metric documented in its own `# HELP` line. The route
  now requires an operator peer (a device cert is 403'd, matching `GET /v1/coverage`'s existing P6
  access class — a deliberate widening once per-device-id rows were added).
- **`docs/samples/grafana-kriya-fleet.json`** — an importable Grafana dashboard over those metrics
  (fleet overview, receipts/day, governed-lane decisions, per-device coverage, policy-bundle version
  spread) + `docs/samples/grafana-kriya-fleet.md`'s README snippet: "your Grafana reads kriya; kriya
  remains the only signer in the chain."
- **Manual proof:** a real 2-envelope, genuinely Ed25519-signed, chained sequence POSTed to a live
  local `kriyad`; `curl /metrics` reflects the real ingested counts; `kriya-audit --verify-otel` on
  the real `build_fleet_export` output re-verifies both spans clean (`2 re-verified (2 OK, 0 FAIL)`).
- Non-goals (explicit): Tempo/Jaeger trace-query APIs, storing spans in kriyad beyond existing
  evidence, any push to endpoints not operator-configured, per-user/per-hostname labels, any free-tier
  surface change.
- See [`docs/TRUST.md`](docs/TRUST.md)'s new "Fleet telemetry (F3-fleet)" section and
  [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md)'s §A17 for the full claim↔proof ledger.

## v0.3.2 — 2026-07-25 — the moat harvest: spend, replay, telemetry, provenance, air-gap

The largest release since v0.1.0 — twelve items from the hard-to-copy build programme land together.
Four of them add a whole new thing you can *hand to someone else and have them check without kriya
installed*: a replay bundle, an OTel span export, a C2PA artifact manifest, and a diode bundle.

**New in the app:** a **Spend** view (what your governed Claude Code sessions actually cost, signed,
computed on-device from local transcripts) with dollar **budget gates**; **Verified Replay** in
Sessions; a **Local Models** view governing local inference; and new Settings panes for **Telemetry**,
**Integrity**, **Artifact signing**, and **Air-gap**.

**New offline verifiers** on the free `kriya-audit` CLI: `--verify-replay`, `--verify-otel`,
`--verify-artifact`, `--verify-bundle`, `--verify-exec`. **New opt-in build lanes:** FIPS (A4) and
post-quantum dual-signature (A5). **New evidence framework:** ISO/IEC 42001 + CSA AICM.

Honest scope, as always: the OTel bridge is **file export/import** — there is no live collector push.
The budget gate is a **trailing-state** gate, not a hard cap. Pipeline integrity is **measurement**,
not hardware attestation. Details in each section below.

### F4-wasm: deterministic execution lane, WASM rung (doc 28 §F4)

- **Naming law: "Deterministic Execution" = re-EXECUTION, never "replay".** B1's "Verified Replay"
  re-DERIVES a session timeline from already-signed receipts, offline; this item actually **re-runs**
  a WASI-p2 component and hash-compares — a different claim, scoped only to tools that ran through
  this lane. See [`docs/TRUST.md`](docs/TRUST.md)'s new "Deterministic Execution (F4-wasm)" section
  for the full honest-ceiling wording.
- **Runtime (`../experiment1`, public): `kriya-run-wasm`** (`crates/kriya/src/wasmexec/`, off-by-
  default `wasm-exec` feature) — executes a WASI-p2 component under pinned Wasmtime (`=41.0.4`, the
  newest release line whose MSRV still matches this repo's rustc-1.90 toolchain pin) configured
  deterministically: fuel metering on, NaN canonicalization on, threads and relaxed-SIMD off, and
  **no ambient clocks or randomness** — `wasi:clocks/wall-clock`/`monotonic-clock` and BOTH
  `wasi:random/*` interfaces are virtualized functions of a recorded seed + epoch. Every run is
  captured as a `kriya-exec-bundle/1` JSON file (module sha256, exact Wasmtime version, a digest of
  every deterministic knob, args/env/stdin as re-runnable bytes AND hashes — 4 MiB size-capped with
  an honest refusal above the cap, never a silent truncation — fuel consumed, stdout/stderr hashes)
  and a signed `kriya.exec.deterministic` receipt binding only the bundle's own hash plus summary
  fields (module hash, fuel, output hashes) — never raw content, the usual `kriya.*` discipline.
  `kriya-run-wasm --verify <bundle>` re-executes and hash-compares.
- **Two example governed WASI-p2 tools** under `examples/f4-wasm-tools/`: `text-transform`
  (upper/lower/reverse, plus `--show-clock`/`--show-hash` that deliberately exercise the virtualized
  clock/RNG so the demo proves reach beyond plain stdin→stdout transforms) and `json-filter`
  (field/value array filter) — the rail is the product, not tool coverage.
- **Policy tier `exec.prefer_deterministic_lane` + `exec.wasm_variants`** (`permissions.rs`
  `ExecPolicy`) — an allowed action with a registered WASM variant routes through a composing
  `WasmRoutingExecutor` instead of its normal executor; either path signs a receipt. Off by default,
  BC-safe like every other optional policy dimension.
- **Console (this repo): `kriya_verify::exec`** (new opt-in `exec` Cargo feature on `kriya-verify`,
  alongside `artifact`) — an INDEPENDENT Rust re-implementation of the identical bundle-verification
  logic (the one-way open/private dependency law forbids a path dependency between the repos; this is
  the same "two copies until the shared `kriya-verify` crate ships" convention `crypto.rs` already
  established for A4/A5). `kriya-audit --verify-exec <bundle> [--module <path>]` and a Tauri bridge
  command (`verify_exec_bundle`, `src/lib/tauri.ts`'s `verifyExecBundle`) both call straight into it
  — TypeScript renders the verdict, never re-derives one.
- **Cross-repo manual proof**: a real `kriya-run-wasm run` on this machine, independently re-verified
  by BOTH `kriya-run-wasm --verify` (runtime) and `kriya-audit --verify-exec` (console) — identical
  verdict, identical fuel-consumption number, from two separately-written Rust implementations; a
  one-byte-flipped bundle fails both with the matching hash-mismatch reason.
- Non-goals (doc 28 §F4/§6, not built): rr/syscall recording (gated, Linux-later), general shell-
  command determinism claims, running untrusted third-party modules without the existing containment
  posture, native macOS syscall replay (rejected outright — no rr/PMU/SIP path on macOS; this WASM
  lane is the honest cross-platform answer). See [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md)
  (§A16).

### B1: Verified Replay (doc 27 §B1)

- **Deterministic re-derivation of a governed session's timeline from its signed receipt chain
  alone** (`src-tauri/crates/kriya-verify/src/replay.rs`, canonical; `src/lib/replay.ts`, the TS
  byte-parity mirror). **Re-derivation, never re-execution** — no tool call, no LLM call, no network
  request is re-run anywhere in this feature; see [`docs/TRUST.md`](docs/TRUST.md)'s "Verified Replay
  (B1)" section for the full honest-naming statement.
- `derive_replay(sources, run_id)` sorts steps by `(ts_ms, source BYTEWISE, line_no)` — an explicit
  UTF-8-byte comparator on both sides, never a bare `Array.sort()`/`localeCompare` (a latent
  cross-platform hash split for any non-ASCII source filename, now caught by a dedicated fixture).
  `category`/`decision` are derived by parsing `action_id`/`params` facets only — **never** inferred
  from the signed `success` boolean (the same trap `policy_sim.rs` already documents). Every field
  renders an honest **FULL**/**ABSENT** fidelity class — a gap you can see, never a value the
  derivation invented.
- **Halt-don't-fabricate.** If any receipt in a contributing source fails signature verification, or
  that source's hash-chain breaks, the WHOLE derivation refuses — citing the exact source, line, and
  signature-vs-chain reason — rather than silently rendering a plausible-looking partial replay.
- **The one-file portable bundle**, `<run>.kriya-replay.json` — a manifest, the verbatim raw JSONL of
  every contributing source, the derived replay, and the origin's own re-computable verifier verdict.
  Two independent offline "stranger" verifiers ship: `kriya-audit --verify-replay <bundle>` (the
  Console's own `kriya-audit-cli`, alongside its existing `--verify-artifact`/`--verify-otel`/
  `--verify-bundle` modes) and the TS mirror (`verifyReplayBundle`) — both re-derive from the
  bundle's OWN `logs` and hash-compare, never trusting its self-reported claims.
- **Cross-language parity is CI-enforced, not just asserted**: a checked-in golden fixture
  (`replay-golden.jsonl`, byte-identical in both `src-tauri/crates/kriya-verify/fixtures/` and
  `test/fixtures/`) derives the SAME hardcoded `canonical_replay_sha256` in two independently-written
  implementations.
- `SessionsView` gains a **Replay** stepper (prev/next/scrubber) inside each run card, distinct from
  the existing lineage tree — the scope + non-claim banner fixed at the top, per-step fidelity chips,
  a "same ms" marker on tied timestamps, and an "Export replay bundle" button.
- New free-tier Tauri commands `derive_session_replay`/`export_replay_bundle`; the export emits the
  additive `kriya.replay.export` self-attestation receipt (own dedicated on-device key + chain),
  registered `no_equivalent` in the OT-1 registry on both languages.
- A dedicated wording-lint test (`test/b1-wording-lint.test.ts`) enforces the naming law across the
  shipped surfaces: forbids `re-run`/`re-execute`/`reproduce the result`/`simulate`/`replays the
  actions`, requires "re-derivation, not re-execution" — the sibling item F4-wasm (deterministic
  re-EXECUTION) is a different claim and never shares this wording.
- See [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md) (§A15) for the full claim → proof table.

### F10: diode pack (doc 28 §F10)

- **A new one-way evidence transport profile, `kriya-diode-bundle/1`** (`src-tauri/crates/kriya-verify/src/diode.rs`,
  `src-tauri/src/diode/`): a plain directory bundle — a signed, fixed-schema `manifest.json` (no
  active content, statically inspectable) plus `chunks/NNNNN.bin` raw JSONL receipt batches (default
  8 MiB/chunk, configurable). Every chunk's sha256 is declared in the manifest alongside a whole-
  bundle RFC-6962-style Merkle root (the same primitive the P0 envelope format already uses) over
  every included receipt line. Built for a real data diode / CDS filter / plain sneakernet — files on
  media only, no network transport anywhere in this module.
- **Continuity across successive exports without a backchannel.** Every manifest carries this
  export's `chain_head` (the tail of what it just sent) AND the PREVIOUS export's `previous_chain_head`
  — so a receiver holding nothing but two manifests, days apart, with zero ability to query the
  sender, can still detect a gap or a fork.
- **Console action "Export diode bundle"** — range `since-last-export` (a per-source-file cursor
  persisted in `~/.kriya/console/diode-state.json`, so repeated exports never resend already-sent
  receipts) | `date-range` | `full`. `full`/`date-range` are idempotent: the same range re-run over
  an unchanged audit dir produces byte-identical chunks, hashes, and Merkle root — only `created_ms`
  (and therefore the signature) varies, documented exactly in `docs/DIODE-BUNDLE.md`.
- **`kriya-audit --verify-bundle <dir> [<prior-manifest.json>]`** — the offline "stranger" check:
  manifest signature, every chunk's hash, every receipt's own Ed25519 signature, the whole-bundle
  Merkle root, and (with a prior manifest) the continuity link. The Console's own
  `diode_verify_bundle`/`diode_import_bundle` commands delegate to the exact same
  `kriya_verify::diode::verify_bundle_dir` so a stranger and the Console agree byte-for-byte. Import
  is idempotent (a re-imported bundle is a detected no-op) and partial-bundle tolerant — a missing
  chunk is reported by index, never silently dropped or reconstructed; everything that DID arrive
  still verifies and imports. Real FEC/Reed-Solomon reconstruction is explicitly **v2, not built**.
- **`kriya.diode.export`/`kriya.diode.import`** — new additive OT-1 vocabulary entries (own dedicated
  on-device key, own dedicated chain `kriya-diode-events.jsonl`), both `no_equivalent` with
  `kriya.diode.*` param extraction on both the TS (`src/lib/otel/semconv.ts`) and Rust
  (`src-tauri/crates/kriya-verify/src/otel.rs`) sides.
- **`docs/DIODE-BUNDLE.md`** — the wire-format spec written for CDS/diode-filter vendors (Owl/Fend/
  Waterfall world), plus a worked golden sample under `docs/samples/diode-bundle-sample/`.
- Settings → **Air-gap → Diode bundle** — export (range picker) and verify+import (bundle directory
  + optional prior-manifest path) in one pane; see [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md)
  (§A13).

### C1: signed spend receipts (doc 27)

- **Two new additive receipts, own chain (`~/.kriya/audit/spend.jsonl`, own key).**
  `kriya.spend.session` — one per Claude Code transcript `sessionId`, once the session has been
  quiet for 30 minutes (idempotent by a `settle_fingerprint`; a session that resumes and grows after
  a premature seal emits a **superseding** receipt, never a duplicate). `kriya.spend.rollup` — a
  non-additive, signed daily checkpoint that re-states the sum of the `session` receipts it
  references; the panel's authoritative total always comes from `session` receipts alone, so the
  rollup can never inflate it.
- **Reconstructed entirely on-device, in Rust, from local Claude Code transcripts**
  (`~/.claude/projects/*/*.jsonl`) — the SAME "extraction stays out of the JS layer" discipline
  `coverage.rs` already uses. Only `message.usage`/`message.model`/`message.id`/`requestId`/
  `sessionId`/`timestamp`/`isSidechain` are ever read; transcript **content never enters a
  receipt** — a dedicated adversarial test feeds a fixture whose message content carries a fake
  secret and a fake file path and asserts neither ever appears in the emitted, signed receipt line.
- **A bundled, versioned, offline pricing sheet** (`src-tauri/resources/pricing/claude-2026-07-01.json`,
  compiled in via `include_str!` — fetches nothing) prices each model's tokens using Anthropic's own
  cache-tier formula (5-minute/1-hour cache-write multipliers, cache-read discount). Every receipt
  records the sheet's id **and** its sha256 hash. A model the sheet doesn't recognize is
  **unpriced** — its tokens are still counted, its cost recorded as `null`, never guessed or zeroed.
- **Dedup by `(message.id, requestId)`** — ccusage's exact discipline — with the dropped-duplicate
  count recorded (`messages_deduped`) as an honesty/test hook. Subagent (sidechain) turns roll up
  into the parent session's own totals, broken out under a `subagent` sub-object for visibility —
  **never** a separate per-subagent receipt (that would double-count).
- **Console: a new "Spend" view**, distinct from the existing byte/rate "Budgets & rate" surface (a
  deliberate vocabulary/UI split — "budget" stays bytes/rate-caps, "spend" is dollars). Renders
  ONLY from the verified `kriya.spend.*` rows the audit pipeline already loads: per-model and
  per-session cost, the active pricing-sheet id, unpriced-model callouts, and an honest rollup
  cross-check (flagged on mismatch, never silently trusted).
- **`kriya.spend.session`/`kriya.spend.rollup` are the FIRST real `mapped` (non-`no_equivalent`)
  vocabulary decided up front in the OT-1 registry** — `gen_ai.usage.input_tokens`/`output_tokens`
  map verbatim; `gen_ai.request.model` is surfaced ONLY when a session priced exactly one model
  (never asserted for a multi-model session or a period-spanning rollup); cost, cache-tier
  breakdown, and pricing-sheet provenance stay `kriya.spend.*`-namespaced.
- **Honest scope, stated up front** (new `docs/TRUST.md` section): this is **governed Claude Code
  session spend evidence**, not complete spend (MCP broker-lane spend, direct API usage outside
  Claude Code, local-inference spend, and vendor-cloud/web Claude usage are all out of scope, named
  as such) — and a **priced estimate**, never an invoice. See
  [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md) (§A11) and [`docs/TRUST.md`](docs/TRUST.md).

### A4: FIPS signing lane (doc 27)

- **An opt-in `fips-crypto` build lane** routes Ed25519 receipt sign/verify through `aws-lc-rs`'s
  `fips` feature (AWS-LC-FIPS 3.x, CMVP cert #5298) instead of `ed25519-dalek` — a signed, honest
  answer to the CMMC SC.L2-3.13.11 / NIST 800-171 "is the cryptography FIPS-validated?" question.
  **Off by default everywhere:** the free tier, the notarized macOS `.dmg`, and every default `cargo
  build`/`tauri build` are byte-unchanged — no new dependency, no new supply-chain surface, unless the
  feature is explicitly enabled. Per-OS honest wording, never upgraded past what's actually true:
  **Linux (validated OE)** — "FIPS 140-3 validated module (cert #5298)"; **other Linux / macOS** —
  "same cryptographic module code as cert #5298, running outside a CMVP-tested operational
  environment"; **default build** — "not a FIPS-validated module." See the new "Cryptographic module"
  section in [`docs/TRUST.md`](docs/TRUST.md) and the assessor one-pager
  [`docs/samples/fips-module-boundary.md`](docs/samples/fips-module-boundary.md).
- **A new signed, hash-chained `kriya.crypto.module` self-attestation** records which lane a signer
  ran under (backend, validated-module name, CMVP cert, a **live** `try_fips_mode()` check, the
  attested operational environment, and how the signing key itself was minted). Purely additive — the
  receipt envelope itself never changes, so every existing receipt and every existing verifier keeps
  working byte-for-byte, in both directions (a FIPS-signed receipt verifies under a default-build
  verifier and vice versa; proven against the real `runtime-egress-ledger.jsonl` fixture). This is a
  **host self-attestation, not a cryptographic proof** — Ed25519 is deterministic (RFC 8032), so a
  signature can never itself reveal which module produced it; `docs/TRUST.md` and the one-pager say so
  explicitly.
- **Settings → Advanced** now shows the running binary's crypto lane; the compliance evidence export
  gains a three-valued "Cryptographic module" line, read from the signed attestation(s) actually
  present in the verified log — never an assumed value (absence prints "not attested for this log").
- **In the open runtime** (same release train): the same `crypto` facade lands in `crates/kriya/src/
  crypto.rs`, wired into the signer (`audit.rs`), the offline `verify-receipts` CLI, and the
  `kriya-gateway`/`kriya-hook`/`kriya-hermes-hook` binaries' new `kriya.crypto.module` emission.

### A5: PQ dual-signature receipts (doc 27)

- **An opt-in `pq-crypto` build lane** adds an **ML-DSA-87 (FIPS 204) countersignature** next to the
  existing Ed25519 signature — a signed, honest answer to the CNSA 2.0 procurement question ("is your
  long-retention audit evidence signed with a quantum-resistant algorithm?"). **Off by default
  everywhere:** the free tier, the notarized macOS `.dmg`, and every default `cargo build`/`tauri
  build` are byte-unchanged. The claim is **"post-quantum-ready (CNSA 2.0-aligned, ML-DSA-87)"** —
  never "FIPS-validated PQ" (ML-DSA-87 is behind `aws-lc-rs`'s `unstable` feature and outside CMVP
  cert #5298's approved boundary — ML-KEM is in it, ML-DSA is not) and never "quantum-proof." See the
  new "Post-quantum readiness" section in [`docs/TRUST.md`](docs/TRUST.md) and the assessor one-pager
  [`docs/samples/pq-dual-signed-receipt.md`](docs/samples/pq-dual-signed-receipt.md) (real,
  Rust-generated sample receipts + measured size numbers).
- **Two PQ modes, one default.** By default, one ML-DSA-87 countersignature seals a **chain
  checkpoint** every N receipts (`kriya.crypto.pq_checkpoint`, N=256 default — a policy dial),
  anchoring the whole sealed prefix through the already-quantum-resistant SHA-256 hash chain at a
  **measured ≈1.03–1.13× storage cost**. A **per-receipt dual-sig** opt-in exists for low-volume,
  high-value receipts (policy bundles, org-key signatures, retention checkpoints, evidence-export
  seals) at a measured **≈14.6 KiB/line** cost (≈30× if applied to every line on a busy host — why
  it's opt-in, not the default). Both modes are **purely additive top-level wire siblings**
  (`pq_alg`/`pq_public_key`/`pq_sig`/`pq_key_id`) — outside the Ed25519-signed bytes, so every
  existing receipt and every existing verifier keeps working byte-for-byte unchanged; require-if-present
  validation gives a distinct failure reason for a partial/mismatched/unsupported PQ set.
- **A new signed `kriya.crypto.pq_key` attestation** binds the ML-DSA-87 public key to the host's
  pinned Ed25519 identity — an auditor who already pins the Ed25519 key transitively trusts the PQ
  key without a second out-of-band pinning step. Rotation is a fresh attestation; older checkpoints
  stay self-verifying under their own inline PQ public key.
- **The TS/browser lane verifies Ed25519 + message parity + PQ structure, never ML-DSA-87 crypto** —
  the compiled Rust lane (`kriya-verify`) is the sole PQ-verifying authority. A PQ-tampered-but-
  structurally-valid receipt shows "Verified (Ed25519)" with an explicit "PQ signature not checked in
  this lane" note, never a false green PQ checkmark — the same honest-scope discipline A4's FIPS claim
  uses for the TS lane.
- **Settings → Advanced** now shows the running binary's PQ lane; the compliance evidence export gains
  a three-valued "Post-quantum seal" line (read from the verified log's `pq_key`/`pq_checkpoint`
  evidence, never a build flag) and an additive `integrity.pqSeal` field for a fresh export-time PQ
  seal when a live PQ signer is available.
- **Cross-implementation parity, both directions:** `aws-lc-rs` (the production PQ signer) signs and
  RustCrypto's independent `ml-dsa` crate (test-only, unaudited — never a signing/verification
  dependency of any production path) verifies, and vice versa — proven in both repos' test suites.
- **Deviation from the original design (documented, not improvised):** `pq-crypto` and `fips-crypto`
  are **mutually exclusive** in this build. This is a real `aws-lc-rs` upstream constraint discovered
  during implementation — its `unstable` module (where the PQ types live) is compiled out whenever its
  own `fips` feature is active — not a kriya restriction. A `compile_error!` in both repos'
  `crypto.rs` turns an accidental double-enable into a clear build-time message instead of a
  confusing downstream compile error. No other part of the design was affected.
- **In the open runtime** (same release train): the PQ facade extends `crates/kriya/src/crypto.rs`;
  `Signer::pq_checkpoint`/`attest_pq_key`/`with_pq` land in `audit.rs`; the offline `verify-receipts`
  CLI gains full require-if-present PQ verification + cross-impl parity tests; the
  `kriya-gateway`/`kriya-hook` binaries attest their PQ key at startup (paired with the existing A4
  crypto-module attestation) when a persisted Ed25519 identity + PQ seed are available.

### F6-T1: pipeline measurement + sandbox incarnation receipts (doc 28 §F6, Tier 1)

- **`kriya.attest.pipeline`** — a sha256 self-hash of the Console binary, hashes of the connector-
  wired runtime hook/hermes-hook/gateway binaries, the active `agent-policy.yaml` hash, and a real
  macOS `codesign --verify` + parsed authority-chain check, signed into a new on-device chain
  (`~/.kriya/audit/attest.jsonl`, own key `~/.kriya/keys/attest.key`) at app start and on demand
  (Settings → Integrity → "Measure now"). Anything the connector wiring can't resolve is recorded as
  the literal sentinel `"UNMEASURED"` — never a guessed path or a fabricated hash.
- **`kriya.attest.sandbox`** — whenever the Seatbelt containment lane (EG-C) launches a
  `kriya-gateway run --` session, a background watcher mirrors the runtime's own `kriya.io.run.start`
  bookend into the attest chain under the Console's own signature: the profile hash, the proxy port,
  and the launched program's NAME + argument COUNT — **never its full command line**. Idempotent per
  `scope_token`, including across app restarts.
- **Settings → Integrity** — the new card: latest measurement, per-field measured/UNMEASURED status,
  drift vs. the previous measurement (binary-hash change highlighted as an observation, never
  auto-blocked), and history.
- **This is measurement, not attestation, and every surface says so.** No hardware root of trust
  (no TPM, no Secure Enclave) backs any of this — a compromised host can lie about every field. See
  [`docs/TRUST.md`](docs/TRUST.md)'s new "Attested pipeline (F6-T1)" section for the full honesty
  discipline; Tier 2 (Linux TPM2/Keylime) and Tier 3 (`kriyad` SEV-SNP) remain gated, not built.
- **`kriya.attest.*` registered in the OT-1 vocabulary** (`src/lib/otel/registry.ts` +
  `src-tauri/crates/kriya-verify/src/otel.rs`) as `no_equivalent` — not a GenAI operation.

### F12a: C2PA signed artifacts bound to the receipt chain (doc 28 §F12)

- **Sidecar C2PA manifests for governed file writes.** SessionsView can now export a `.c2pa` sidecar
  manifest (never embedded — text/code embedding via Unicode is fragile, so this is a later phase)
  for any file a governed session wrote: producing session id, agent identity, policy-verdict COUNTS
  (never content), and `kriya.chain.head` at emission time. Signed with a dedicated Ed25519 keypair +
  a self-signed certificate, generated once and persisted beside the Console's other on-device keys
  (Settings → Artifact signing shows the fingerprint).
- **The binding loop, both directions.** The manifest embeds `kriya.chain.head`; the new
  `kriya.artifact.provenance` receipt (its own free-tier self-attestation chain) embeds the
  manifest's own hash + the target file's hash + its path — so the chain and the artifact each point
  at the other, independently checkable.
- **`kriya-audit --verify-artifact <file> <sidecar>`** — the offline auditor re-prover extended with
  a c2pa validation mode: the hard file-binding (an edited file fails), fully offline, exit 0/1.
- **Honest by design: self-managed anchor, not a public trust list.** The artifact certificate is
  never enrolled in any public C2PA/CAI trust program — a stranger's `kriya-audit --verify-artifact`
  or a public C2PA validator correctly reports `signingCredential.untrusted` (structurally valid,
  tamper-evident, but an unrecognized issuer); inside the Console, the SAME device's own cert is used
  as the trust anchor, so a manifest this device signed shows fully trusted in-app. Provenance "for
  those who keep it" — a removable sidecar, never a watermark — and never sold or documented as EU AI
  Act Art. 50 compliance (a provider obligation, not a deployer one). See
  [`docs/TRUST.md`](docs/TRUST.md)'s "Signed artifacts" section.

### F3: verifiable telemetry (OTel GenAI bridge, doc 28 §F3)

- **Every receipt maps to an OTel GenAI-conformant span** (`invoke_agent`/`execute_tool`, `gen_ai.*`
  attributes where a semconv equivalent exists, `kriya.*` otherwise) via a new mapping module in both
  languages (`src/lib/otel/semconv.ts`, `src-tauri/crates/kriya-verify/src/otel.rs`), enforced by a
  vocabulary-registry test that fails the build if a future `kriya.*` receipt vocabulary ships without
  a decided OTel mapping. Export writes the OTLP/HTTP JSON document to a **file** (Settings →
  Telemetry) — every span carries its receipt's Ed25519 signature and chain position, so the new
  `kriya-audit --verify-otel <file>` command re-verifies it fully offline, signature-by-signature,
  with the SAME compiled verifier the Console uses.
- **Observed-span ingest (never a receipt).** Settings → Telemetry can also import a third-party
  OTLP/HTTP JSON export (e.g. Claude Code's own OTel telemetry) into a separate observations store;
  Sessions renders them visually stratified below receipt-backed rows, with an honest "governance gap"
  count for spans no receipt can vouch for.
- **Scope note.** This build ships the network-free half of doc 28's F3 spec only — a live push to a
  collector endpoint and a live localhost listener are NOT included; see
  [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md) (§A7)
  and [`docs/TRUST.md`](docs/TRUST.md) for why (the free-tier non-egress dormancy claim) and what a
  file export already proves.

### F1: attested local inference (doc 28 §F1)

- **New runtime binary `kriya-llm-proxy`** (`../experiment1`, `llm-proxy` feature, off by default):
  a governed HTTP reverse proxy in front of a local OpenAI-compatible / Ollama-native inference
  server (Ollama, llama.cpp's server, vLLM's OpenAI-compat endpoint, LM Studio, …). Streaming
  passthrough (SSE / Ollama NDJSON) is real — bytes relay to the client as they arrive from upstream;
  only a running hash and a small rolling "last line" tail are kept in memory, never a full body.
- **Three new receipts**, signed via the same Ed25519 signer path every other lane uses:
  `kriya.model.identity` (first sight of a served model — name, resolved digest, digest source),
  `kriya.model.gate` (the policy verdict, every completion), `kriya.model.serve` (usage/perf per
  OT-1: `gen_ai.request.model`, `gen_ai.usage.input_tokens`/`output_tokens`, sampler params, prompt/
  output as sha256 hashes — **never text**).
- **Model-identity resolution reads a digest from where it already lives** — an Ollama registry
  manifest, or an operator-precomputed file-hash cache keyed by `(path, size, mtime)` — and never
  hashes a model weight file on the request path; an unresolvable model is honestly `unresolved`.
- **A new `model:` policy dimension** (`ModelPolicy`, in the SAME `agent-policy.yaml` every governed
  binary already loads): an approved-digest allowlist per tier, plus an `unknown_model` default
  (`warn` by default — fail-OPEN and disclosed, never silently blocked; `require-approval`/`deny` are
  real enforcement once the operator opts in).
- **Console: a new "Local Models" view** — every observed model, digest, approval state, and serve
  count, built entirely from verified `kriya.model.*` receipts already in the audit trail; one-click
  "Approve this digest" writes an `allow`-tier rule into `agent-policy.yaml`.
- **`kriya.model.serve` is the first `kriya.*` vocabulary in OT-1's `mapped` bucket** — F3's own doc
  comment anticipated this ("a future vocabulary that genuinely has GenAI-semconv attributes to use,
  e.g. a model-serving receipt once F1 lands"); `kriya.model.identity`/`kriya.model.gate` are
  `no_equivalent` governance markers.
- **Bypass honesty.** The proxy governs ONLY clients pointed at it — a direct connection to the
  upstream inference server's own port is invisible to it. See
  [`docs/FEATURE-PROOF.md`](docs/FEATURE-PROOF.md) (§A8) and [`docs/TRUST.md`](docs/TRUST.md).
- **v1 scope, per doc 28 §F1.** Observe + gate on model identity only — deterministic re-execution /
  strong-mode determinism is doc 28 phase 3, not built here; OMS/Sigstore manifest verification is a
  clearly-marked seam (phase 2), also not built.

### D3: ISO/IEC 42001 + CSA AICM evidence pack (doc 27)

- **A new "ISO 42001 / CSA AICM" export**, a second selectable framework alongside the existing
  narrower "ISO42001" (A.9) card. Maps the same verified receipt trail to ISO/IEC 42001:2023 Annex A
  **A.6.2.8** (event logs), **A.6.2.6** (operation & monitoring), and **A.9.4** (intended use), plus
  CSA AI Controls Matrix (AICM) v1.0 domains — **LOG / A&A / GRC** directly/supportingly, **IAM / DSP**
  as context-only signal — cited at the domain grain only (no fabricated per-objective IDs; AICM's
  per-objective cells and AI-CAIQ's per-question text are registration-gated and never reproduced).
- **Every Annex A row is capped at ◐ partial — never ✓ satisfied.** These are AI-management-system
  process controls kriya cannot itself complete (risk criteria, competence, review cadence, intended-
  use determination); the honest ceiling is evidence *toward* the operational sub-claim, never the
  control as a whole. **8 Annex A controls + 3 AICM domain groups are explicit NOT-COVERED (✗)** rows —
  risk & impact assessment, technical documentation, data quality/provenance, deployment, resourcing,
  and more — listed by name, because that honesty is the credibility feature, not a gap to paper over.
- **A third artifact, the AI-CAIQ v1.0.2 support sheet** (`ai-caiq-support.md`) — explicitly a
  **self-assessment context** document, never an attestation: a fixed disclaimer, no answer/score
  column (structurally absent), organized by AICM domain with kriya's supporting evidence + sample
  pointers filled in and the question text left as `UNVERIFIED — from your licensed AI-CAIQ v1.0.2`
  placeholders the org fills from its own copy.
- **Same signing/verification story as the AU-family pack:** generated on-device, re-verifiable
  offline via `kriya-audit`, TS↔Rust parity on the scope block and disclaimer text. A wording-lint
  bans nine claim-inflation phrases (`compliant`, `certified`, `guarantees`, `STAR certified`,
  `third-party assess*`, `attestation of conformity`, `renders … MET`, and more) across every
  generated artifact, with a narrow, explicit carve-out for the noun "certification" in the standard
  footer and the AI-CAIQ disclaimer's "certification body."
- **New golden sample pack** at
  [`docs/samples/iso42001-sample/`](docs/samples/iso42001-sample/) — real, generated data (not
  hand-typed), extended with a freshly-signed `kriya.crypto.module` attestation and one deliberately
  tampered receipt, proving the verify gate catches forgery (`kriya-audit` genuinely reports 2 FAIL
  lines, exit 1 — the intended, honest verdict).
- **Fleet rollup** (customer-hosted `kriyad` aggregator): three boundary-scoped ISO 42001 rows added
  to the org-wide evidence export, computed from the same re-verified envelope rollups the AU-family
  fleet rows already use — no new envelope field, no raw receipts leaving the customer boundary. The
  AI-CAIQ sheet is explicitly an org-level, once-produced artifact — never rolled up per device.
- See [`docs/design/d3-iso42001-pack.md`](docs/design/d3-iso42001-pack.md) for the full mapping table
  and the red-team pass, and the extended ISO 42001 row in [`docs/TRUST.md`](docs/TRUST.md).

### C2: endpoint budget gate (doc 27 §4)

- **A new additive `budgets:` policy dimension** (session / rolling-day / user scope, a USD
  threshold, and a `deny`/`require-approval`/`warn` breach action, with an optional per-rule
  `on_missing_state` override) that the runtime's `kriya-hook` **pre-execution gate** enforces on the
  Claude Code hook `pre` lane — blocking or routing the **next** gated action to approval once
  observed spend, built on C1's signed spend evidence, crosses the authored threshold. Additive and
  nullable exactly like `egress`/`detection`/`secrets`/`model` — a pre-C2 policy round-trips
  byte-identically with no `budgets:` key at all.
- **A trailing-state gate — the honest ceiling is the feature.** The hook is a
  fresh process with no in-process rate state, so it can only consult a spend figure the Console
  wrote moments earlier: an always-on `spend-live.json` live-tick snapshot (≤ ~5 min freshness) plus
  an OPTIONAL `kriya-spend-statusline` accelerator that collapses a session budget's window to
  ≈ one assistant turn. A single expensive turn between extractor passes always lands **after** the
  gate check; every `kriya.spend.gate.*` receipt records `state_source`/`state_as_of_ms`/
  `state_stale` so a reviewer can see exactly how stale the enforcing figure was.
- **The B0-critical placement fix (F-B1):** the budget consult sits immediately after the action-tier
  decision and strictly BEFORE the credential-brokering block, so a budget `deny`/ungranted-approval
  always short-circuits before any `{{kriya:*}}` placeholder is substituted or any Keychain secret is
  read — proven at the real process boundary by a dedicated `kriya_hook_smoke.rs` regression row (a
  `$0.01` deny budget + a configured `secrets:` alias + a brokered placeholder ⇒ exit 2, a signed
  `kriya.spend.gate.deny` receipt, and no `updatedInput` ever printed).
- **Fail-closed by default on missing/stale state, extending B0** — `deny` defaults to blocking,
  `require-approval` defaults to routing to a human, `warn` is observe-only by definition and never
  blocks; an operator may explicitly relax a `deny` budget to fail-open, and that loosening is itself
  receipted (`on_missing_state`), never silent.
- **Additive `kriya.spend.gate.{deny,approval,warn}` receipt vocabulary**, signed on the existing
  hook `Signer`'s own chain — `no_equivalent` in the OT-1 registry (a governance enforcement
  decision, not a GenAI usage measurement, unlike C1's `mapped` spend receipts), TS↔Rust twin kept in
  sync by the OT-1 vocabulary-registry test.
- **Rolling-day/user totals attribute each session's whole priced total to the UTC day of its own
  `window_end_ms`** — a session spanning midnight is counted whole against the day it ended, never
  split; documented as honest-but-coarse, never presented as an exact per-day figure.
- **Console UI:** a "Spend budgets" authoring section in `PolicyView` (naming-disciplined copy, a
  lint warning on a `deny` budget explicitly set to fail-open, and the optional statusline
  accelerator install control with a replace-warning when a different status line command is already
  configured), a "Budget status" panel in `SpendView` (observed vs. threshold, freshness, recent
  breaches), and a budget badge on routed `kriya.spend.gate.approval` approvals in `ApprovalsView`.
- **The runtime never prices anything** — every USD figure on an enforcement receipt is copied
  through from the Console-written state file; the open `kriya` crate gains no pricing dependency.
- See [`docs/design/c2-budget-gate.md`](docs/design/c2-budget-gate.md) for the full design + the
  adversarial re-review, and the new "Spend budget gate (C2)" section in
  [`docs/TRUST.md`](docs/TRUST.md) for the quantified staleness window.

## v0.3.1 — 2026-07-23

- **Connections: clearer guidance for the MCP-only agents.** When nothing is detected, the "Govern
  everything" empty state now names all seven supported clients and explains the key distinction:
  Claude Code and Hermes are governed whole-lane the moment their CLI is on your PATH (via a hook),
  while **Cursor, Cline, GitHub Copilot, and Gemini CLI have no hook** — kriya governs their *MCP
  lane* via the gateway, so each appears here only once it has a **local (stdio) MCP server** in its
  config. Previously the empty state mentioned only Claude Code / Claude Desktop / Hermes, so there
  was no way to discover that the VS-Code-family and CLI clients were supported. No behavior change —
  detection and wiring were already correct; this is the missing signpost.

## v0.3.0 — 2026-07-22 — sessions, test-before-apply, more agents

- **Sessions — run correlation.** A new **Sessions** view reconstructs every governed run as a tree
  — *which session → which sub-agent → which action, in order* — from the signed receipts alone.
  Governed lanes now stamp an optional `kriya.corr` block into each receipt (`run_id`, and where the
  seam really exposes them, `parent_step_id` / `agent_id`); the bundled `kriya-hook` and
  `kriya-gateway` sidecars emit it, and the SDK middleware threads explicit nested-call lineage.
  Honest by construction: the tree is computed from **verified receipts only**, Claude Code's hook
  payload has no parent pointer so none is invented (sub-agents group by `agent_id`), and run ids
  live in receipt `params` — structurally unreachable by the fleet envelope minimizer, so they never
  leave the device. The compliance export gains a session-correlation appendix **only** when
  correlated receipts exist; a zero-correlation export is byte-identical to v0.2.6's.
- **Policy — "test before apply."** Replay a candidate policy over this device's own re-verified
  receipts and see which past actions would land on a different tier ("this edit would have changed
  N of last week's M actions") — in the Policy view and as the fleet pre-publish gate. Scope stated
  in the UI: the action-tier gate only; the simulation itself is a signed, chained
  `kriya.policy.sim.result` receipt.
- **Govern All: Cursor · Cline · GitHub Copilot · Gemini CLI.** One-click detection + routing of
  each client's stdio MCP servers through the governed gateway — idempotent, non-clobbering, fully
  reversible. Ceiling stated where it's shown: the MCP lane is governed; each agent's native
  built-in tools bypass MCP unless launched under containment; cloud-executed agents are out of
  scope.
- **In the open runtime** (same release train): `kriya-govern` (per-call govern + sign over stdio),
  SDK middleware for **LangGraph · OpenAI Agents SDK · CrewAI · Claude Agent SDK** (TypeScript +
  Python, no crypto in the wrappers), and **`kriya-ci`** — the governed CI lane (run an agent step
  in CI under a repo-committed policy; the build fails on a policy block and the signed receipts are
  the build artifact, re-verifiable offline).

## v0.2.6 — 2026-07-16

- **Audit log: date-range filter + sort by time.** The Audit log now has a From/To date filter
  (UTC, matching the "When" column) and a Newest/Oldest sort, defaulting to newest-first — so a
  receipt is findable by *when* it happened, not only by text/status/source.

## v0.2.5 — 2026-07-15

- **Stale-hook detection.** After an in-place upgrade, Claude Code can keep calling an *older*
  `kriya-hook` (a leftover `cargo install`, or a pre-`--policy` wiring) that predates egress
  capture — so WebSearch/WebFetch egress silently never records and the network-egress lane stays
  grey. The Coverage view now compares the wired hook against the binary this build ships and, when
  they differ, shows a warning with a one-click **Re-run Govern All** to re-point it. Previously the
  app treated any `kriya-hook` string as healthy.

## v0.2.4 — 2026-07-15 — the egress pack

- **Egress governance core** — per-destination allowlists (deny-by-default), byte budgets,
  fail-closed *"no receipt, no egress"* (the signed receipt is a precondition of the network call),
  and ask-before-send approvals for unknown destinations.
- **Detection pack** — secret & PII scanning on outbound bodies (redact/deny; only hashes stored),
  DNS-exfiltration and subdomain-entropy detection, SSRF / private-IP / cloud-metadata /
  DNS-rebinding guard, canary tokens, operation rails (verb / path / GraphQL mutation),
  connector registry (new MCP servers disabled until approved) with tool-description drift
  scanning, per-connector read-only presets, MCP-response trust classes.
- **Credential brokering** — agents hold placeholders; real secrets live in the OS keychain and
  are injected only at egress. New public threat model: `docs/THREAT-MODEL-brokering.md`.
- **OS containment (macOS)** — `kriya-gateway run -- <agent>` launches an agent inside a generated
  Seatbelt profile with a recording CONNECT proxy, forcing traffic through the governed lane;
  contained sessions light up the raw-egress Coverage lane.
- **Fleet egress** — egress policy, budgets, and a kill switch distributed in the org-signed
  PolicyBundle; fleet egress-receipt report; agent-to-agent lane governance.
- **Evidence & privacy** — egress control rows in the compliance export (scoped honestly to
  governed lanes), redaction manifest for egress receipts, and a customer privacy pack
  (`docs/privacy/`): DPIA template, employee notice, works-agreement clause.
- **Fleet destination visibility** — privacy-minimized pattern-echo of destinations in fleet
  envelopes (additive `io_destinations` field, sealed minimizer, per-bundle `io_verbosity`).

## v0.2.3 — 2026-07-10

The control-plane cockpit comes together — central governance, fleet drift, org-wide evidence.

- **Central policy authoring + signed downlink** — author once, sign with your org key, publish to
  your on-prem `kriyad`; each device pulls, re-verifies, and applies (anti-rollback included), and
  the applied policy becomes part of that device's own signed evidence trail.
- **Fleet drift & governance view** — per device: in-sync / behind / silent-behind, every verdict
  re-verified locally from the device's own signed envelopes, never the server's word.
- **Org-wide assessor-ready evidence export** — coverage-completeness + AU-family + CM controls
  across the fleet, computed from re-verified envelopes.
- **mTLS cert-role separation** — a device cert can't read the fleet; an operator cert can't post
  device evidence.
- `kriyad` ship skins: static binary, distroless image, systemd box install, cosign-signed
  air-gap bundle, release CI gated on the trust-spine tests.

## v0.2.2 / v0.2.1 — 2026-07-08

- **Hermes native-tool governance** via the new `kriya-hermes-hook` — terminal, file edits,
  computer-use, browser automation, plus every MCP server it's attached to; one-click install
  from Govern All. (v0.2.1 fixed Hermes detection: `mcp_servers` vs `mcpServers`.)

## v0.2.0 — 2026-07-08

- **Govern All** — one button detects every governable agent on the machine (Claude Code, Claude
  Desktop, Hermes, desktop apps) and wires each through its seam: preview, apply, revert. Idempotent.
- **Bundled `kriya-hook`** — the Console ships the Claude Code hooks adapter itself; no separate
  install to govern native tools.
- **Multi-agent Coverage Map** — lanes grouped per agent, with an honest "cloud, out of scope"
  line for surfaces that execute off-device.
- Compliance export names the distinct governed agents and cites the signed
  coverage-completeness chain (NIST 800-171 3.3.1 / 3.3.4).

## v0.1.2 — 2026-07-07

- **NIST SP 800-171 / CMMC L2 AU-family mapping** (3.3.1–3.3.9, with 800-53 crosswalk) in the
  evidence export — every status derived from re-verified receipts, never hard-coded.
- Notarized universal (Intel + Apple Silicon) DMG.

## v0.1.1 — 2026-07-03

- **The Coverage Map** — six lanes, three states, signed into its own hash chain so a stopped
  watcher is visible by absence, not a quiet nothing.
- `kriya-hook` shipped in the public runtime; the gateway's remote-MCP broker (hosted MCP servers
  over HTTP/SSE).

## v0.1.0 — 2026-07-01

- First public release: the live governance Monitor, offline receipt verification, Connections
  manager, guided first-run setup — signed with our Apple Developer ID and notarized by Apple.
- The free **`kriya-audit` CLI** published alongside — verify any signed receipt log offline,
  independent of the Console.
- The trust spine: byte-for-byte parity between the TypeScript verifier and the Rust signer,
  enforced by `npm test`.

Every tagged release (notarized DMG + SHA-256) lives on
[GitHub Releases](https://github.com/governex/kriya-console/releases), tagged `vX.Y.Z`
(releases through v0.2.4 were published on the runtime repo, tagged `console-vX.Y.Z`).
