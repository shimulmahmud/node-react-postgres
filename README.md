# Node + React + Postgres — Hands-On DevOps Assessment

This repo implements the production-style platform described in the assessment PDF using a local Docker Compose setup, Kubernetes manifests, an API gateway pattern, CI/CD automation, platform services, and observability scaffolding.

## Prerequisites

- Docker Engine with the Compose plugin
- Node.js 24 LTS for local development
- For Kubernetes: `kubectl`, kind, and an ingress-nginx controller
- For autoscaling: Metrics Server installed in the cluster
- For CI deployments: a persistent Kubernetes cluster with an HTTPS API reachable from GitHub-hosted runners. A local kind API on `localhost` is not reachable from GitHub Actions.

## 1. Local containerized setup

### Start the stack

```bash
docker compose up --build
```

### Access points

- Frontend: http://localhost:8080
- Backend: http://localhost:4000/api/health
- Postgres: localhost:5432

The local stack includes:

- backend API service
- frontend React app behind nginx
- PostgreSQL with a persistent volume
- Redis
- worker service for BullMQ jobs
- OpenSearch for search health checks

### Docker notes

- Both Dockerfiles use multi-stage builds; runtime images use supported Node.js 24 and unprivileged NGINX 1.29.
- The backend runs as the unprivileged `node` user; the frontend uses the unprivileged NGINX image on port 8080.
- Postgres data persists in a Docker volume.
- DB and service credentials are supplied through environment variables instead of hardcoded secrets.
- The frontend app uses `VITE_API_BASE_URL` (default `/api`) so browser requests route through the same origin or through the defined proxy path.

---

## 2. Kubernetes deployment (dev namespace)

Create a local kind cluster and build/load the GHCR-tagged development images before applying the manifests:

```bash
kind create cluster --name kind
docker build -f Dockerfile.backend -t ghcr.io/shimulmahmud/node-react-postgres-platform-backend:dev .
docker build -f Dockerfile.frontend -t ghcr.io/shimulmahmud/node-react-postgres-platform-frontend:dev .
kind load docker-image ghcr.io/shimulmahmud/node-react-postgres-platform-backend:dev --name kind
kind load docker-image ghcr.io/shimulmahmud/node-react-postgres-platform-frontend:dev --name kind
kubectl apply -f k8s/namespace-dev.yaml
kubectl apply -n dev -f k8s/
```

Install ingress-nginx for kind before applying `k8s/ingress-dev.yaml`. If using private GHCR packages instead of locally loaded images, configure an image pull secret in the `dev` namespace and reference it from the pod specs.

### Included Kubernetes resources

- Namespace: `dev`
- Backend deployment + ClusterIP service
- Frontend deployment + ClusterIP service behind Ingress
- PostgreSQL deployment with PVC/PV and ClusterIP service
- Redis deployment + service
- BullMQ worker deployment
- Search deployment + service
- ConfigMap and Secret for app settings
- Ingress for `/` and `/api`

### Health checks

The backend deployment includes:

- readiness probe on `/api/health/ready`
- liveness probe on `/api/health/live`

This ensures traffic is only sent to healthy pods and that DB connectivity issues do not cause pointless restarts.

---

## 3. Ingress / API gateway

Install an ingress controller in the local cluster, then apply the ingress manifest:

```bash
echo "127.0.0.1 node-react.local" | sudo tee -a /etc/hosts
kubectl apply -n dev -f k8s/ingress-dev.yaml
```

Now access:

- http://node-react.local/
- http://node-react.local/api

### Request flow

Browser → Ingress → Service → Pod → PostgreSQL

The ingress routes:

- `/api` and `/api/*` → `backend-service`
- `/` → `frontend-service`

### Kubernetes DNS

Services are resolved through cluster DNS using names like:

```text
backend-service.dev.svc.cluster.local
postgres-service.dev.svc.cluster.local
redis-service.dev.svc.cluster.local
```

This is more stable than using pod IPs because pod IPs change when workloads are rescheduled.

### Debugging notes

1. Ingress is reachable but backend is not
   - Check `kubectl get endpoints backend-service -n dev`
   - Verify the backend pod is ready and the readiness probe passes
   - Review backend pod logs with `kubectl logs -n dev deploy/backend`

2. Frontend cannot talk to backend
   - Confirm the frontend is calling the right API path (`/api`)
   - Ensure the ingress is routing `/api` to the backend service
   - Check ingress controller logs and backend service endpoint status

This design can evolve into a microservice platform by adding more backend services behind the same ingress with additional path-based routing rules.

---

## 4. CI/CD and multi-environment deployment

The repo includes the workflow at [.github/workflows/ci-cd.yml](.github/workflows/ci-cd.yml).

Branch-to-namespace mapping:

- `master` → `dev`
- `pre-release/v*` → `preprod`
- `release/v*` → `release`

### Pipeline flow

1. Install frontend and backend dependencies
2. Validate build/test steps
3. Build Docker images for frontend and backend
4. Tag images with branch name + short SHA
5. Push to GitHub Container Registry (GHCR)
6. Update manifests for the target namespace
7. Apply to Kubernetes

### Secrets and rollback

- Build and publish use GitHub-hosted runners. Deploy targets a persistent cluster using the namespace mapped from the branch; it does not create a temporary cluster.
- Create GitHub Environments named `dev`, `preprod`, and `release`. In each environment, add the `KUBECONFIG_B64` secret and the `APP_URL` variable. The kubeconfig must point to the real cluster's HTTPS API, not `localhost`, and should use a service account limited to that environment's namespace. Require reviewers for `preprod` and `release` in the environment protection settings.
- Store `KUBECONFIG_B64` as the base64-encoded kubeconfig for the environment. For example, generate it locally with `base64 -w0 kubeconfig`; add the output directly to GitHub Environment Secrets and do not commit or paste it into source control.
- The cluster administrator must provision each namespace, its `app-config` ConfigMap and `app-secrets` Secret, the namespace's default ServiceAccount with a durable read-only `ghcr-pull` secret, ingress with TLS, and database/Redis/search dependencies before deploying. Store database URLs/passwords in `app-secrets`, not ConfigMaps. The workflow manages backend/frontend Services and the backend HPA. `APP_URL` must be that environment's HTTPS application URL.
- Scope the deployment service account's namespace Role to get/create/patch Deployments, Services, and HPA; get `app-config`, the `default` ServiceAccount, and only the `app-secrets`/`ghcr-pull` Secrets; and get/list/watch Pods and ReplicaSets for rollout status and rollback. Limit Secret access with `resourceNames`; the workflow never creates or prints secret values or needs cluster-wide permissions.
- The workflow publishes immutable branch-plus-SHA tags to GHCR with the short-lived `GITHUB_TOKEN` and `packages: write`. Cluster image pulls use the separately provisioned read-only GHCR credential, not the short-lived workflow token.
- Run the `CI/CD` workflow manually with a `rollback_namespace` to invoke `kubectl rollout undo` for backend, frontend, and worker. The workflow checks rollout health after rollback.

### Production boundary

The `k8s/` manifests are the assessment's local/dev setup, not a complete production infrastructure definition. In particular, `postgres-pv.yaml` uses `hostPath`, and `app-secrets.yaml` contains development credentials. The production workflow applies only the application Deployments, Services, and backend HPA; it never applies the local database, PV, ConfigMap, or sample Secret manifests. Use managed/stateful platform services, environment-specific secrets, durable storage, TLS, backups, network policies, and an infrastructure-as-code process for real production environments. This demo API has no authentication or authorization and must not be exposed publicly until those controls and rate limiting are implemented.

---

## 5. Platform services: Redis, queue, search, read/write DB

### Redis and queue

- Redis is included for caching, queue backends, and health checks.
- BullMQ is used in the app via `backend/src/queue.js` and `backend/src/worker.js`.
- Every `POST /api/items` enqueues an item-created job, while a separate worker processes it.

### Search

- OpenSearch is included in single-node dev mode.
- The backend checks the search health endpoint at `/api/health/search`.

### Simulated read/write DB split

The app is built with separate write and read connection strings:

- `DB_WRITE_URL` → primary writer
- `DB_READ_URL` → replica read target

This is implemented in `backend/src/db.js` with separate `pg.Pool` instances.

### Why the split matters

Replication lag means a read replica can be behind the primary. Any operation that must reflect its own recent writes should always hit the writer, including:

- validation checks immediately after insert/update
- redirect-after-POST fetches
- uniqueness checks before commit

The app intentionally routes writes through `writePool` and reads via `readPool`, which provides a clear place to add a real read replica later.

---

## 6. Observability, scaling, and rate limiting

The `observability/` manifests deploy Prometheus, kube-state-metrics, Grafana, Fluent Bit, Loki, and the backend HPA into `dev`. Prometheus scrapes ingress-nginx request/latency metrics, kubelet cAdvisor container CPU/memory, and kube-state-metrics replica counts. Fluent Bit tails only `/var/log/containers/*_dev_*.log` and ships them to Loki. Grafana provisions the Prometheus and Loki datasources and the `Backend operations` dashboard, including a backend log panel.

### Install and access

For the existing kind ingress-nginx controller, expose its metrics endpoint internally, install Metrics Server, and create a random local Grafana admin secret:

```bash
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--enable-metrics"},{"op":"add","path":"/spec/template/spec/containers/0/ports/-","value":{"name":"metrics","containerPort":10254,"protocol":"TCP"}}]'
kubectl patch service ingress-nginx-controller -n ingress-nginx --type=json -p='[{"op":"add","path":"/spec/ports/-","value":{"name":"metrics","port":10254,"protocol":"TCP","targetPort":"metrics"}}]'
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl create secret generic grafana-admin -n dev --from-literal="password=$(openssl rand -base64 36)" --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n dev -f observability/
kubectl apply -n dev -f k8s/ingress-dev.yaml
```

The `--kubelet-insecure-tls` option is only for kind's self-signed kubelet certificates. Do not use it in production. The dev Grafana manifest permits anonymous **viewer** access for local evidence; its admin password remains in the `grafana-admin` Secret.

Access Grafana and Prometheus through local tunnels:

```bash
kubectl port-forward -n dev service/grafana 13000:3000
kubectl port-forward -n dev service/prometheus 19090:9090
```

Open `http://localhost:13000/d/backend-operations/backend-operations` for the dashboard. Prometheus targets are available at `http://localhost:19090/targets`.

### Logs and scaling

Loki is searchable in Grafana Explore with:

```logql
{job="fluent-bit", namespace="dev", app="backend"} |= "backend listening"
```

The backend HPA targets 70% CPU and 80% memory, with 2–6 replicas. Backend requests/limits are set in `k8s/backend-deployment.yaml`; Metrics Server must be running for the HPA to calculate utilization. Verify using `kubectl top pods -n dev` and `kubectl get hpa -n dev`.

Ingress rate limiting is configured in `k8s/ingress-dev.yaml` at 10 requests/second per client IP with a burst multiplier of 5. A local 80-request burst returned 53 successful responses and 27 rate-limited `503` responses. Shared NAT addresses therefore share a limit; tune the value for the expected client population.

### Alert rationale

- High 5xx rate: signals failed requests and backend/dependency regressions.
- High CPU or memory: signals resource pressure and saturation risk.
- High p95 latency: catches slow requests hidden by averages.
- Replica count below demand: indicates HPA or cluster capacity is not keeping up.

The dashboard provides the source metrics for these alerts. Alert delivery/notification routing is not configured in this local setup.

## Design decisions and remaining work

- Compose provides a one-command local stack; Kubernetes keeps the database, Redis, and search services internal and exposes the frontend through an Ingress backed by a ClusterIP Service.
- Read/write routing is in the backend using separate PostgreSQL pools. Reads may lag behind writes when a real replica is introduced, so read-after-write and consistency-sensitive checks must use the writer.
- The assessment's demo environment is local-only. The CI deploy target is a persistent remote cluster reachable by GitHub-hosted runners; local kind remains a separate development environment.
- The backend test command now fails on test errors, but the repository currently has no backend test cases; CI relies on syntax and production build validation until tests are added.
- With more time: provision isolated preprod/release clusters, configure alert notification routing, add authentication/authorization before public exposure, and capture a passing remote GitHub Actions deployment after environment secrets are configured.

## Assessment evidence

Evidence captured for this local kind run is in [`docs/evidence/`](docs/evidence/). A passing remote GitHub Actions deployment is still pending the `dev`/`preprod`/`release` environment secrets and externally reachable production cluster described above; it is not claimed as completed evidence.

---

## Local verification performed

The app was validated locally by running the Docker Compose stack and confirming the backend and frontend responded successfully:

```bash
curl -fsS http://localhost:4000/api/health
curl -fsS http://localhost:4000/api/health/ready
curl -fsS -X POST http://localhost:4000/api/items -H 'Content-Type: application/json' -d '{"name":"config-check"}'
```

The app successfully returned a ready status and inserted/retrieved the item record.
