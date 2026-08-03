# Compliance evidence — Sample contractor — illustrative data

_Generated 2026-07-06T02:00:00.000Z by kriya Console. Evidence derived from cryptographically signed audit receipts, verified locally._

**Period:** 2026-06-17T12:40:00.000Z → 2026-07-06T01:00:00.000Z

## Audit integrity

- Receipts: **47**
- Verified: **45**
- Failed / tampered: **2**
- Distinct signer keys: **6**
- Cryptographic module: The default build signs with `ed25519-dalek` — a widely-audited pure-Rust implementation that is **not** a FIPS-validated module.
- Post-quantum seal: not attested for this log — no `kriya.crypto.pq_key` or `kriya.crypto.pq_checkpoint` receipt was found in this window, so no statement about post-quantum readiness can be made either way.

## Attribution (who acted)

- Coverage: **100%** of verified receipts carry an actor
- Agents: claude-code, claude-desktop, cursor, kriya-console, langgraph
- Operators: data-eng, fin-ops, platform-eng, sales-ops

## Human oversight & on-device posture

- Deny-by-default policy: **yes**
- Approval-gated actions observed: restart_service, scale_service, deploy
- Budget cap: 60/min
- On-device attestations: **0**

## Egress/ingress ledger (governed lanes)

Governed-lane egress: NOT zero — 5 kriya.io.egress.* receipt(s) observed and verified in this window.

- Signed `kriya.io.*` receipts verified in this window: **6** (4 allow, 1 deny, 1 approve)

> Scope: this artifact covers only agent traffic proxied through the kriya gateway (MCP-over-HTTP connectors, gateway-proxied tool calls) and the hook-observed tool lane. Agent processes can generate network traffic outside these lanes — spawned subprocesses, and the outbound connections of stdio MCP servers — which kriya does not observe, control, or record. Enforcement rides a cooperative hook that can be disabled at the host (see TRUST.md). This artifact is supporting evidence toward the identified assessment objectives for the agent-connector lane only; it does not by itself render any control MET, and coverage of non-governed agent egress must be documented in the organization's SSP under its own boundary and flow controls.

## Action inventory

| Action | Count | Policy tier | Destructive |
| --- | ---: | --- | --- |
| `categorize_transaction` | 6 | allow |  |
| `update_contact` | 3 | allow |  |
| `get_logs` | 3 | allow |  |
| `create_note` | 3 | allow |  |
| `list_transactions` | 2 | allow |  |
| `list_contacts` | 2 | allow |  |
| `list_services` | 2 | allow |  |
| `get_balance` | 2 | allow |  |
| `claude-code__read` | 2 | deny |  |
| `claude-code__task` | 2 | deny |  |
| `restart_service` | 1 | approval |  |
| `list_deals` | 1 | allow |  |
| `scale_service` | 1 | approval |  |
| `deploy` | 1 | approval |  |
| `web_search` | 1 | deny |  |
| `fetch_url` | 1 | deny |  |
| `summarize` | 1 | deny |  |
| `write_report` | 1 | deny |  |
| `claude-code__grep` | 1 | deny |  |
| `claude-code__bash` | 1 | deny |  |
| `claude-code__edit` | 1 | deny |  |

## Control mapping

| Framework | Control | Status | Evidence |
| --- | --- | --- | --- |
| EU AI Act | Art. 12 — Record-keeping | ◐ partial | 45 signed receipt(s) verified, 2 failed/tampered; 6 signer key(s). |
| EU AI Act | Art. 14 — Human oversight | ✓ satisfied | 3 action(s) gated behind human approval: restart_service, scale_service, deploy. Deny-by-default: yes. |
| EU AI Act | Art. 12(2) — Traceability | ✓ satisfied | 100% of verified receipts attributed to an agent + operator (agents: claude-code, claude-desktop, cursor, kriya-console, langgraph). |
| EU AI Act | Art. 26(6) — Deployer log retention | ◐ partial | 47 receipt(s) retained locally as JSONL under the deployer's own control; kriya does not enforce or verify a specific retention schedule (e.g. the six-month minimum) — that is the deployer's responsibility. |
| SOC 2 | CC7.2 — Monitoring | ◐ partial | 2 receipt(s) failed verification — tampering or corruption detected. |
| SOC 2 | CC7.3 — Security event evaluation | ◐ partial | Per-receipt verification failures and hash-chain-break flags surface the security-event signal; the evaluation and response process itself is organizational, outside kriya. |
| SOC 2 | CC8.1 — Change management | ✓ satisfied | 3 agent-driven change action(s) require human approval before execution: restart_service, scale_service, deploy. Deny-by-default: yes. |
| ISO 42001 | A.9 — Operation controls | ✓ satisfied | Deny-by-default policy with 21 action(s) observed; budget cap 60/min. |
| ISO 42001 | A.6.2.6 — Operation and monitoring | ◐ partial | The signed receipt stream is the operation/monitoring log (45 verified of 47), surfaced live in the Console Monitor. |
| Data residency | On-device processing | ✗ gap | No on-device attestations in this trail. |
| NIST 800-171 | 3.3.1 (AU.L2-3.3.1 · AU-2/3/12) — Audit record creation & retention | ◐ partial | 47 signed receipt(s) retained across 1 app(s) and 5 governed agent(s) as a hash-chained local JSONL log; 2 failed verification; each record carries action id, parameters, timestamp, outcome, and signer. Completeness is itself attested: 14 signed coverage snapshot(s) (chain intact) record which lanes were governed over the window — what was and wasn't logged is provable, not asserted. |
| NIST 800-171 | 3.3.2 (AU.L2-3.3.2 · AU-3) — Individual accountability | ✓ satisfied | 100% of verified receipts carry a signed agent + individual-operator identity (operators: data-eng, fin-ops, platform-eng, sales-ops). |
| NIST 800-171 | 3.3.3 (AU.L2-3.3.3 · AU-2) — Review & update logged events | ◐ partial | 21 distinct action type(s) captured across policy tiers (allow/approval/deny); the periodic review and update of which events to log is an organizational process outside kriya. |
| NIST 800-171 | 3.3.4 (AU.L2-3.3.4 · AU-5) — Audit logging process failure alerting | ◐ partial | Per-receipt verification failures and hash-chain breaks surface live in the Console, and the Coverage Map flags silent lanes; no external paging/alerting integration exists. The signed coverage chain makes a stopped or silenced logging process visible by absence — a gap in the heartbeat chain, not a quiet nothing. |
| NIST 800-171 | 3.3.5 (AU.L2-3.3.5 · AU-6(3)) — Correlate audit review & analysis | ◐ partial | Cross-app correlation on this machine (Audit view filtering across 1 app(s)) plus tamper flags support investigation; this is single-machine correlation, not cross-machine SIEM aggregation. |
| NIST 800-171 | 3.3.6 (AU.L2-3.3.6 · AU-7) — Audit record reduction & report generation | ✓ satisfied | This Markdown + JSON evidence bundle is itself the reduction/report artifact, generated on-demand from the signed trail and independently re-verifiable offline via kriya-audit. |
| NIST 800-171 | 3.3.7 (AU.L2-3.3.7 · AU-8) — Clock synchronization for time stamps | ◐ partial | Every receipt carries a host timestamp (ts_ms); clock synchronization against an authoritative source is OS-provided (NTP), outside kriya's control — this control is capped at partial regardless of trail size. |
| NIST 800-171 | 3.3.8 (AU.L2-3.3.8 · AU-9) — Protect audit information & tools | ◐ partial | 2 receipt(s) failed verification — tampering detected; the detection control is functioning as intended, investigate the flagged record(s). |
| NIST 800-171 | 3.3.9 (AU.L2-3.3.9 · AU-9(4)) — Limit audit-logging management to privileged users | ✗ gap | kriya's audit tooling runs under the operator's own OS account, and in-app roles are self-asserted (see docs/TRUST.md) — kriya enforces no privileged-user restriction on who can manage audit logging; this must be enforced by an OS-level or organizational access control. |
| NIST 800-171 | 3.1.3 (AC) — CUI flow enforcement | ◐ partial | Egress allow/deny/approve by destination for governed connector lanes (6 signed kriya.io.* receipt(s) verified: 4 allow, 1 deny, 1 approve), signed per-decision. Governed lanes only — a spawned subprocess or a stdio MCP server's own outbound traffic is not observed. Scope: this artifact covers only agent traffic proxied through the kriya gateway (MCP-over-HTTP connectors, gateway-proxied tool calls) and the hook-observed tool lane. Agent processes can generate network traffic outside these lanes — spawned subprocesses, and the outbound connections of stdio MCP servers — which kriya does not observe, control, or record. Enforcement rides a cooperative hook that can be disabled at the host (see TRUST.md). This artifact is supporting evidence toward the identified assessment objectives for the agent-connector lane only; it does not by itself render any control MET, and coverage of non-governed agent egress must be documented in the organization's SSP under its own boundary and flow controls. |
| NIST 800-171 | 3.4.2 (CM) — Enforce configuration settings | ◐ partial | The egress allowlist is an enforced, receipted setting — 6 governed-lane decision(s) signed against it this window. Product-scoped: this is one enforced setting on one control-plane app, never a system-wide configuration-management claim. |
| NIST 800-171 | 3.14.6/3.14.7 (SI-4) — Monitor / identify unauthorized use | ◐ partial | Unapproved-endpoint and anomalous-egress detection on governed lanes. 1 denial(s) against the allowlist observed in this window (unapproved-endpoint / anomalous-destination detection on governed lanes). |
| NIST 800-53 | AC-4 — Information flow enforcement | ◐ partial | A signed, per-decision enforcement point on governed connector lanes (6 kriya.io.* receipt(s) verified). Nothing at this layer stands in the way of a flow that avoids it entirely — a spawned subprocess bypasses it (see the E2 host-observation roadmap and TRUST.md). |
| NIST 800-53 | SI-4 — System monitoring | ◐ partial | The governed-lane egress ledger FEEDS an organization's SI-4 monitoring program as one signed source among others — it is a contributing signal, never claimed to BE the organization's system monitoring. 6 receipt(s) verified in this window. |
| SOC 2 | CC6.1 — Logical access boundaries | ◐ partial | The gateway is a managed access point for governed connector lanes — 6 signed access decision(s) this window (4 allow, 1 deny, 1 approve). |
| SOC 2 | CC6.7 — Restrict transmission and movement | ◐ partial | A transmission-restriction control for governed agent lanes: destination-based allow/deny/approve, signed per decision (6 receipt(s) verified this window). 1 denial(s) against the allowlist observed in this window (unapproved-endpoint / anomalous-destination detection on governed lanes). |
| SOC 2 | CC7.2 — Anomaly monitoring (governed-lane egress) | ◐ partial | Detection tooling and logging of unusual egress activity on governed lanes. 1 denial(s) against the allowlist observed in this window (unapproved-endpoint / anomalous-destination detection on governed lanes). |
| EU AI Act | Art. 12 — Record-keeping (governed-lane egress) | ◐ partial | Readiness-framed: 6 governed-lane egress/ingress event(s) signed and verified this window; Annex III high-risk obligations are deferred to Dec 2, 2027 pending the Digital Omnibus. If this agent system is not classified high-risk, this row is INAPPLICABLE, not partial — that classification is the deploying organization's own determination, not derived from this trail. |
| DORA | Art. 28(3) — Register reconciliation | ◐ partial | A signed, actual-usage enumeration of governed-lane destinations feeds register reconciliation against the organization's Art. 28(3) ICT third-party register — 6 receipt(s) verified this window. This is one input to that register, never a substitute for the organization's own maintained register. |
| DORA | Art. 10(2) / Art. 17 — Detection & incident management | ◐ partial | A lane-scoped detection/incident-timeline layer for governed agent egress — one of the "multiple layers of control" DORA expects, not the organization's full ICT risk framework. 1 denial(s) against the allowlist observed in this window (unapproved-endpoint / anomalous-destination detection on governed lanes). |
| ISO 42001 / CSA AICM | A.6.2.8 — AI system recording of event logs | ◐ partial | DIRECT-EVIDENCE: the signed action-receipt stream (Ed25519 + hash-chain, prev_hash inside the signed bytes — 45 verified this window) is the tamper-evident event-log recording; the kriya.coverage.snapshot chain (GA-3) attests which lanes were logging. Honest ceiling: the recording exists and re-verifies offline, but which events to log and the retention period are org decisions kriya does not make; governed-lane scope only. |
| ISO 42001 / CSA AICM | A.6.2.6 — AI system operation and monitoring | ◐ partial | SUPPORTING-EVIDENCE: the live Console Monitor over the receipt stream, the signed kriya.coverage.snapshot chain (six lanes / three states), and the kriya.io.* egress/ingress ledger together evidence one input to operation & monitoring. Honest ceiling: kriya monitors the governed agent surface, not the whole AI system's operation — model performance, output drift, and infrastructure health are out of scope. |
| ISO 42001 / CSA AICM | A.9.4 — Intended use of the AI system | ◐ partial | SUPPORTING-EVIDENCE: the operator-authored agent-policy (allow / require-approval / deny) is the enforced statement of permitted use; per-action tier verdicts and signed approval decisions are evidence the system operated within those bounds (deny-by-default: yes). Honest ceiling: kriya evidences enforcement of operator-authored use-bounds; it does not determine intended use, assess its appropriateness, or validate that the bounds match documented intended use. |
| ISO 42001 / CSA AICM | A.9.2 — Processes for responsible use of AI systems | ◐ partial | CONTEXT-ONLY: approval-gated actions (human-in-the-loop before high-risk actions) and a deny-by-default policy are signed evidence a responsible-use process is operating. Honest ceiling: the process's design and documentation are organizational; kriya shows the runtime footprint of one, not that the process itself meets any particular standard. |
| ISO 42001 / CSA AICM | A.6.2.6+ — Cryptographic-module posture (operation sub-control) | ◐ partial | CONTEXT-ONLY: Cryptographic module: The default build signs with `ed25519-dalek` — a widely-audited pure-Rust implementation that is **not** a FIPS-validated module. Honest ceiling: this is a host self-attestation, not independently verified FIPS validation; most deployments run the default ed25519-dalek lane, not a validated module. |
| ISO 42001 / CSA AICM | A.9.3 — Objectives for responsible use of AI systems | ✗ gap | NOT-COVERED: objective-setting is a documentary/governance activity; kriya emits no evidence of it. |
| ISO 42001 / CSA AICM | A.6.2.7 — AI system technical documentation | ✗ gap | NOT-COVERED: kriya produces operational evidence, not system technical documentation; it does not measure the AI system's design documentation. |
| ISO 42001 / CSA AICM | A.6.2.5 — AI system deployment | ✗ gap | NOT-COVERED: deployment process and gates are org-owned; kriya observes post-deployment agent actions, not the deployment control itself. |
| ISO 42001 / CSA AICM | A.5.x — AI risk & impact assessment (A.5.2 / A.5.4) | ✗ gap | NOT-COVERED: kriya performs no risk or impact assessment and holds no evidence of one — this is the org's AIMS core. |
| ISO 42001 / CSA AICM | A.7.x — Data for AI systems (A.7.2–A.7.6) | ✗ gap | NOT-COVERED: kriya does not observe training/inference data quality, provenance, or preparation. The kriya.io.* ledger records that a governed call happened plus a content hash, never data content or quality. |
| ISO 42001 / CSA AICM | A.8.x — Information for interested parties (A.8.2–A.8.4) | ✗ gap | NOT-COVERED: transparency-to-users is documentary. Explicit non-claim: artifact-provenance receipts attest a session record and are scoped OUT of EU AI Act Art. 50 content-marking in docs/TRUST.md — they are not A.8 transparency evidence. |
| ISO 42001 / CSA AICM | A.10.x — Third-party & customer relationships | ✗ gap | NOT-COVERED (ISO grain): supplier/customer governance is org-owned. The kriya.io.* egress ledger enumerates governed-lane destinations and can feed a separate DORA Art. 28(3) register (already carried by the AU/egress pack) — that is not ISO 42001 A.10 evidence and is not mapped here. |
| ISO 42001 / CSA AICM | A.4.x / A.3.x — Resources for / policies & internal organization of AI systems | ✗ gap | NOT-COVERED: resourcing, competence, roles, and the AI policy itself are AIMS governance; kriya emits none of it. |
| ISO 42001 / CSA AICM | AICM LOG — Logging and Monitoring | ◐ partial | DIRECT/SUPPORTING-EVIDENCE: the signed receipt trail, the coverage-snapshot chain, and the kriya.io.* ledger are the log/monitoring evidence this domain asks for — kriya's primary D3 target. Objective-level detail is UNVERIFIED — cite your licensed AICM v1.0 spreadsheet. |
| ISO 42001 / CSA AICM | AICM A&A — Audit and Assurance | ◐ partial | SUPPORTING-EVIDENCE: the offline-re-verifiable Markdown+JSON evidence bundle (this pack) supports independent audit without trusting the vendor. Objective-level detail is UNVERIFIED — cite your licensed AICM v1.0 spreadsheet. |
| ISO 42001 / CSA AICM | AICM GRC — Governance, Risk & Compliance | ◐ partial | SUPPORTING-EVIDENCE: policy authoring plus signed enforcement verdicts are evidence a governance control operated; the pack itself is a compliance-evidence artifact. Objective-level detail is UNVERIFIED — cite your licensed AICM v1.0 spreadsheet. |
| ISO 42001 / CSA AICM | AICM IAM — Identity and Access Management | ◐ partial | CONTEXT-ONLY: the actor (agent + operator) attributed on each receipt is an accountability signal only. Objective-level detail is UNVERIFIED — cite your licensed AICM v1.0 spreadsheet. |
| ISO 42001 / CSA AICM | AICM DSP — Data Security and Privacy | ◐ partial | CONTEXT-ONLY: on-device / non-egress posture, the redaction floor (envelope minimizer, doc 22), and content-hash-never-content discipline are a weak context signal. Objective-level detail is UNVERIFIED — cite your licensed AICM v1.0 spreadsheet. |
| ISO 42001 / CSA AICM | AICM MDS — Model Security | ✗ gap | NOT-COVERED: the model-gate control is unbuilt with no sample in this pack; do not cite it as MDS evidence. |
| ISO 42001 / CSA AICM | AICM SEF — Security Event and Failure | ✗ gap | NOT-COVERED: per-receipt verification failure is a tamper signal, not an incident-management program; no sample beyond the AU tamper row is cited here. |
| ISO 42001 / CSA AICM | AICM CEK / CCC / AIS / BCR / DCS / HRS / I&S / IPY / STA / TVM / UEM — 11 domains | ✗ gap | NOT-COVERED: kriya emits no evidence for cryptography-KMS lifecycle, change-config, app/interface security, continuity, data-center, HR, infra/virtualization, interoperability, supply-chain agreements, threat/vuln management, or endpoint management. |

## NOT-COVERED (kriya emits no evidence for these)

AICM coverage: of AICM v1.0's 18 domains, kriya provides evidence toward 3 directly/supportingly (LOG, A&A, GRC) and 2 as context-only (IAM, DSP); 13 are NOT-COVERED.

> Scope: kriya provides operational evidence toward AI-management-system (AIMS) controls for the governed agent surface only. It is not an AI management system: it performs no risk or impact assessment, does not determine or validate an AI system's intended use, and does not render any Annex A control MET. This artifact issues no certification and makes no attestation of any kind — an organization's status is never established on kriya's say-so alone. See the mapping table below for the specific operational sub-claim each row supports, and the NOT-COVERED section for what kriya emits no evidence for at all.

| Control | Reason |
| --- | --- |
| A.9.3 — Objectives for responsible use of AI systems | NOT-COVERED: objective-setting is a documentary/governance activity; kriya emits no evidence of it. |
| A.6.2.7 — AI system technical documentation | NOT-COVERED: kriya produces operational evidence, not system technical documentation; it does not measure the AI system's design documentation. |
| A.6.2.5 — AI system deployment | NOT-COVERED: deployment process and gates are org-owned; kriya observes post-deployment agent actions, not the deployment control itself. |
| A.5.x — AI risk & impact assessment (A.5.2 / A.5.4) | NOT-COVERED: kriya performs no risk or impact assessment and holds no evidence of one — this is the org's AIMS core. |
| A.7.x — Data for AI systems (A.7.2–A.7.6) | NOT-COVERED: kriya does not observe training/inference data quality, provenance, or preparation. The kriya.io.* ledger records that a governed call happened plus a content hash, never data content or quality. |
| A.8.x — Information for interested parties (A.8.2–A.8.4) | NOT-COVERED: transparency-to-users is documentary. Explicit non-claim: artifact-provenance receipts attest a session record and are scoped OUT of EU AI Act Art. 50 content-marking in docs/TRUST.md — they are not A.8 transparency evidence. |
| A.10.x — Third-party & customer relationships | NOT-COVERED (ISO grain): supplier/customer governance is org-owned. The kriya.io.* egress ledger enumerates governed-lane destinations and can feed a separate DORA Art. 28(3) register (already carried by the AU/egress pack) — that is not ISO 42001 A.10 evidence and is not mapped here. |
| A.4.x / A.3.x — Resources for / policies & internal organization of AI systems | NOT-COVERED: resourcing, competence, roles, and the AI policy itself are AIMS governance; kriya emits none of it. |
| AICM MDS — Model Security | NOT-COVERED: the model-gate control is unbuilt with no sample in this pack; do not cite it as MDS evidence. |
| AICM SEF — Security Event and Failure | NOT-COVERED: per-receipt verification failure is a tamper signal, not an incident-management program; no sample beyond the AU tamper row is cited here. |
| AICM CEK / CCC / AIS / BCR / DCS / HRS / I&S / IPY / STA / TVM / UEM — 11 domains | NOT-COVERED: kriya emits no evidence for cryptography-KMS lifecycle, change-config, app/interface security, continuity, data-center, HR, infra/virtualization, interoperability, supply-chain agreements, threat/vuln management, or endpoint management. |

## Session correlation (appendix)

Computed from verified `kriya.corr` receipts: **2** run(s) across **11** correlated action(s); **2** sub-agent(s) observed; **2** subagent-spawn action(s); **1** blocked/failed attempt(s).

_Run correlation groups a session's actions from the signed receipts; approval decisions are recorded separately in the approvals queue._

_Status: ✓ satisfied · ◐ partial · ✗ gap. This report is evidence, not a certification._
