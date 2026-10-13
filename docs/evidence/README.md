# Assessment Evidence

Evidence captured from the local `kind` cluster on 2026-10-13:

- `app-ingress.png`: React app served through `node-react.local` and ingress-nginx.
- `grafana-dashboard.png`: provisioned Backend operations dashboard with live Prometheus panels and Loki logs.
- `dev-resources.txt`: `kubectl get all -n dev` snapshot.

The local checks verified Prometheus ingress, kubelet/cAdvisor, and kube-state-metrics targets; backend CPU/memory/replicas; request rate, 5xx ratio, and p95 latency; and a backend startup log query in Loki. An 80-request local ingress burst returned 53 HTTP 200 and 27 HTTP 503 responses under the configured rate limit.

A passing GitHub Actions deployment run is not included yet. It requires the remote cluster and the `KUBECONFIG_B64`/`APP_URL` GitHub Environment values described in the root README.
