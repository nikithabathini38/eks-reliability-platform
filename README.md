# eks-reliability-platform
Local Kubernetes reliability &amp; observability platform — ArgoCD, Istio, Prometheus/Grafana, GitHub Actions
## Step 1: Local Cluster (minikube)

Set up a local, 2-node Kubernetes cluster using minikube — free, no cloud cost —
as the foundation for the entire platform:minikube start --nodes2 --cpus4 --memory 6000 --driver=docker 
Two nodes (rather than one) means Kubernetes actually has to schedule pods
across real compute boundaries, matching real multi-node cluster behavior.

## Step 2: GitOps Deployment (ArgoCD + Helm)

Installed ArgoCD to manage cluster state declaratively from Git, rather than
applying changes manually via `kubectl`. Deployed [podinfo](https://github.com/stefanprodan/podinfo)
(a sample microservice) via Helm, with its ArgoCD Application manifest tracked
in [`argocd-apps/podinfo-application.yaml`](argocd-apps/podinfo-application.yaml).

Enabled automated sync with self-healing (`syncPolicy.automated.selfHeal: true`)
so any manual drift from the declared Git state is automatically reverted —
verified this live by manually scaling podinfo to 5 replicas and watching
ArgoCD revert it back to the declared `replicaCount: 2` within seconds.




## Step 3: Istio Service Mesh

Installed Istio (demo profile) on the local minikube cluster and enabled sidecar
injection for the `podinfo` namespace. Configured traffic resilience policies via
a `DestinationRule` and `VirtualService`:

- **Retries**: up to 3 attempts per request, 1s per-try timeout, 3s total timeout,
  retrying on 5xx/connection errors
- **Circuit breaking**: outlier detection ejects a pod after 3 consecutive 5xx
  responses, for a 30s cooldown

### Verification

Simulated failures using podinfo's built-in `/status/500` test endpoint from a
pod inside the mesh, and tailed Envoy's access logs (`istio-proxy` container) to
confirm retry behavior: a burst of 10 client requests generated 21 logged
upstream calls, with matching request IDs appearing up to 3 times each —
confirming Envoy's retry policy is active and working transparently to the caller.

Circuit breaking (outlier detection) was confirmed configured and applied via
`istioctl proxy-config endpoint`. Catching a live ejection snapshot proved
difficult in practice, since Istio load-balances across 2 podinfo replicas —
splitting failed requests between pods makes it harder for either single
endpoint to accumulate 3 *consecutive* failures. This is a good illustration of
how outlier detection interacts with load balancing in a real multi-replica
service.

## Step 4: Observability Stack (Prometheus + Grafana)

Installed the kube-prometheus-stack Helm chart (Prometheus, Grafana, Alertmanager)
into a dedicated `monitoring` namespace. Since the Prometheus Operator only scrapes
targets with a matching ServiceMonitor/PodMonitor, added a custom PodMonitor
(`monitoring-config/istio-podmonitor.yaml`) so Prometheus picks up metrics from
Istio's Envoy sidecars — without it, all dashboards showed "No data" despite
Prometheus and Istio both running correctly.

### SLO Dashboard

Built a Grafana dashboard defining an availability SLO for podinfo:

- **Target**: 99% of requests return a non-5xx response
- **Query**: `sum(rate(istio_requests_total{destination_service_name="podinfo", response_code!~"5.."}[5m])) / sum(rate(istio_requests_total{destination_service_name="podinfo"}[5m])) * 100`
- **Visualization**: Gauge panel with threshold coloring — red below 99%, green at/above 99%, so the panel visually signals SLO breach vs. healthy at a glance

Verified end-to-end by generating live traffic against podinfo from inside the mesh
and confirming the gauge updates in real time, correctly reading ~100% under normal
conditions with no failing requests.

## Step 5: CI/CD Pipeline (GitHub Actions)

Added a GitHub Actions workflow (`.github/workflows/validate-manifests.yaml`) that
runs automatically on every push to `main`:

- **YAML lint** — catches syntax errors in manifests before they reach the cluster
- **Kubeconform validation** — verifies Kubernetes manifests are structurally valid
  against the K8s API schema
- **Trivy security scan** — scans manifests for misconfigurations (missing resource
  limits, running as root, etc.), reporting CRITICAL/HIGH findings

This mirrors how real GitOps teams gate changes before they're synced by ArgoCD —
catching a broken manifest in CI is far cheaper than debugging it after it's live
on the cluster. Verified end-to-end: pipeline runs on push, all steps pass
(lint → validate → scan) in under 30 seconds.

## Step 6: Incident Review

Simulated two failure scenarios against podinfo — a pod deletion and an Istio
fault-injection test (delayed + aborted requests) — to validate the platform's
self-healing and observability end-to-end. Captured a real SLO breach on the
Grafana dashboard (availability dropped to 67.5%, threshold correctly turned
red) and documented the full incident, including a subtle Prometheus
`reporter`-label debugging story.

Full write-up: [incidents/2026-08-13-podinfo-fault-injection.md](incidents/2026-08-13-podinfo-fault-injection.md)
