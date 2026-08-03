# Fleet Grafana sample (F3-fleet, doc 28 §F3-fleet)

**Your Grafana reads kriya; kriya remains the only signer in the chain.**

`kriyad` (the customer-controlled, single-tenant aggregator) exposes a read-only, Prometheus-style
`GET /metrics` endpoint — aggregate governance KPIs only (receipts/day, governed-connector-lane
allow/deny/approve counts, per-device-id coverage completeness, policy-bundle version spread). No
content, no per-action rows: kriyad structurally cannot leak either, since it only ever stores signed
envelope **rollups** pulled from devices (doc 22 §2/§11-B1), never raw receipts.

This is OFF by default (the `/metrics` route only exists once `kriyad` is running at all, which itself
requires a valid `control-plane` license) and every field is documented in its own `# HELP` line at the
endpoint itself — `curl https://your-kriyad:8443/metrics` (or `http://localhost:8443/metrics` in
plain-HTTP/local dev mode) to see the full, current list.

## Import the dashboard

[`grafana-kriya-fleet.json`](grafana-kriya-fleet.json) is a standard, importable Grafana dashboard
(schema version 39) over that endpoint:

1. Point **your own Prometheus** at **your own `kriyad`'s** `/metrics` — inside **your own boundary**.
   kriya never hosts, proxies, or sees a copy of this data; the export goes **to YOUR collector, inside
   YOUR boundary**, exactly like the fleet OTel span exporter (the sibling artifact this dashboard is
   named after) is scoped.
2. In Grafana: **Dashboards → New → Import**, upload `grafana-kriya-fleet.json`.
3. When prompted, select your Prometheus data source for the `DS_PROMETHEUS` input.
4. The dashboard renders: fleet overview stats (receipts total, devices current/behind/silent),
   receipts-per-day, governed-lane allow/deny/approve counts, a per-device coverage table (device id
   only — never a hostname or username), and the policy-bundle version spread.

## What this proves, and what it doesn't

- Every number Grafana renders here was **already locally re-verified** before it ever reached
  `kriyad`'s store — `kriyad` re-verifies every envelope's Ed25519 signature and chain continuity on
  ingest, offline, against the device's own evidence key. Grafana is reading the OUTPUT of that
  verification, not re-deriving trust itself.
- `kriyad` **authors nothing** (doc 22 §3): it verifies, stores append-only, and serves. A compromised
  `kriyad` can delay or withhold metrics — it cannot forge a receipt count that was never actually
  ingested (a forged envelope still fails Ed25519 verification at ingest, before it ever touches a
  counter this endpoint sums).
- This dashboard is an **operational/GTM convenience**, not a substitute for the offline,
  cryptographically re-verifiable evidence exports (`fleet_org_evidence`, the org-wide compliance
  report) or `kriya-audit --verify-otel` on a real fleet envelope-span export. Grafana shows you trends
  over already-trusted aggregates; it does not itself re-verify a single signature.
