# eks-reliability-platform
Local Kubernetes reliability &amp; observability platform — ArgoCD, Istio, Prometheus/Grafana, GitHub Actions
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
