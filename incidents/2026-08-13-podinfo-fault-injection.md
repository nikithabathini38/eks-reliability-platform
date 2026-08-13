# Incident Review: Podinfo Availability Degradation (Simulated)

**Date**: 2026-08-13
**Severity**: Simulated (chaos engineering exercise)
**Status**: Resolved
**Author**: Nikitha Bathini

## Summary

Two failure scenarios were deliberately simulated against the podinfo service to
validate the platform's resilience and observability: (1) a pod deletion, and
(2) Istio-level fault injection (delayed and aborted requests). Both were
detected and handled as designed.

## Timeline

- **T+0:00** — Baseline confirmed: 2 podinfo replicas healthy, SLO dashboard at 100%
- **T+0:01** — Deleted one podinfo pod manually (`kubectl delete pod`)
- **T+0:11** — Kubernetes scheduled and started a replacement pod automatically;
  no manual intervention required
- **T+0:15** — Applied an Istio VirtualService fault injection: 50% of requests
  delayed 4s (exceeding the 3s timeout), 30% aborted with a synthetic 500
- **T+2:00** — SLO dashboard gauge dropped to **67.5%**, crossing the 99% threshold
  and turning red — first real, live SLO breach captured on the dashboard
- **T+2:30** — Reverted the VirtualService to its clean configuration (removed
  fault injection)
- **T+4:00** — SLO gauge recovered to 100%, green

## Impact

Simulated only — no real users affected. Demonstrates what a genuine availability
degradation would look like on the dashboard, and confirms the SLO panel
correctly reflects real traffic conditions rather than always showing a
static "healthy" value.

## Root Cause (of the simulated failure)

Deliberately injected via Istio `VirtualService.spec.http[].fault`:
- `abort`: 30% of requests return HTTP 500 immediately
- `delay`: 50% of requests delayed 4s, exceeding the configured 3s timeout

## Detection

The Grafana SLO dashboard (built in Step 4) correctly surfaced the degradation
within one scrape interval. The panel's threshold coloring (red below 99%)
made the breach immediately visible without needing to read raw numbers.

## A debugging note worth recording

Initially, the SLO panel continued to show 100% even after confirming (via
`istio-proxy` access logs) that faults were actively being injected. Root
cause: Istio emits `istio_requests_total` from **both** the caller's sidecar
(`reporter="source"`) and the destination's sidecar (`reporter="destination"`).
Aborted requests never reach the destination pod, so they only appear in the
`source`-side metrics. The original PromQL query summed both reporters
together, diluting the failure rate. Fix: scope the query to
`reporter="source"` only, since that's the only side with complete visibility
into faulted requests.

## Resolution

- Reverted the VirtualService fault injection
- Confirmed recovery via the SLO dashboard within ~4 minutes

## Follow-ups / Lessons

- Confirmed Kubernetes self-healing (pod replacement) works within ~10 seconds
  with zero manual steps
- Confirmed Istio retries and circuit breaking are active, though under this
  fault pattern (immediate abort at the caller's sidecar) retries could not
  rescue every failed request, since the fault itself intercepts the call
  before it reaches podinfo
- Confirmed the `reporter` label matters when writing PromQL against
  `istio_requests_total` — a good reminder to scope queries by ownership when
  troubleshooting deviating metrics
- Next step: consider a Prometheus alert rule tied to the same SLO query, so a
  future breach pages/notifies rather than requiring someone to be watching
  the dashboard
