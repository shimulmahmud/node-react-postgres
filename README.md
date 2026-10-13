# Node + React + Postgres — Hands-On DevOps Assessment

This repo implements the production-style platform described in the assessment PDF using a local Docker Compose setup, Kubernetes manifests, an API gateway pattern, CI/CD automation, platform services, and observability scaffolding.

## Prerequisites

- Docker Engine with the Compose plugin
- For Kubernetes: `kubectl`, kind, and an ingress-nginx controller
- For autoscaling: Metrics Server installed in the cluster
- For automated Kubernetes deploys: a GitHub Actions self-hosted runner on the machine that can access the local kind cluster

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

- Both Dockerfiles use multi-stage builds to keep final images small.
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
- Frontend deployment + NodePort service
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

- The workflow publishes images to GHCR using the automatically provided `GITHUB_TOKEN`; grant the workflow `packages: write` permission and ensure the GHCR packages are public or pullable by the cluster.
- Build and test jobs use GitHub-hosted runners. Deploy and rollback jobs use a self-hosted GitHub Actions runner on the machine that owns, or can network-reach, the Kubernetes cluster. For this local kind assessment, the runner reads the cluster credentials from its local `~/.kube/config`; protect that host and grant the runner only the Kubernetes permissions it needs. A GitHub-hosted runner cannot reach a kubeconfig endpoint such as `localhost` on the developer's machine.
- Rollback strategy: `kubectl rollout undo deployment/<name> -n <namespace>` or redeploy a previous tagged image.

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

The app intentionally routes writes through `writePool` and reads via `readPool`, which makes the design production-ready for later replica adoption.

---

## 6. Observability, scaling, and rate limiting

The observability folder currently contains example manifests for:

- Prometheus
- Grafana
- Fluent Bit DaemonSet
- Loki
- HPA for the backend

### Recommended alerts

- high error rate
- high CPU usage
- high latency (p95/p99)
- sustained request rate increase without matching scale-out

### Scaling

The backend HPA is configured for 2 to 6 replicas based on CPU and memory utilization. Metrics Server must be installed, and resource requests must be set on the backend containers, for the HPA to calculate utilization. The current backend Deployment does not yet define those requests, so autoscaling is configured but not operationally verified.

### Rate limiting

Rate limiting is best applied at the ingress or gateway layer so the platform can reject excess traffic before it reaches the backend. This repository documents that design choice, but does not yet configure or validate an ingress rate limit.

The Prometheus, Grafana, Fluent Bit, and Loki manifests are assessment scaffolding; dashboards, alert rules, and an end-to-end searchable-log verification still need to be added before claiming Task 6 is fully complete.

## Design decisions and remaining work

- Compose provides a one-command local stack; Kubernetes keeps the database, Redis, and search services internal and exposes the frontend through NodePort/Ingress.
- Read/write routing is in the backend using separate PostgreSQL pools. Reads may lag behind writes when a real replica is introduced, so read-after-write and consistency-sensitive checks must use the writer.
- The local kind cluster cannot be reached directly from a GitHub-hosted runner. CI builds and publishes images on GitHub-hosted runners; deployment runs on the self-hosted runner connected to the cluster.
- With more time: provision isolated preprod/release clusters, add verified dashboards/alerts and searchable logs, enforce ingress rate limits, and capture the required screenshots and passing workflow run under `docs/`.

## Assessment evidence

The assessment requests evidence under `docs/`: `kubectl get all -n dev`, the app at `node-react.local`, Grafana dashboards, Loki log queries, and a passing GitHub Actions run. Add those screenshots/links after capturing them; they are not represented here as completed evidence.

---

## Local verification performed

The app was validated locally by running the Docker Compose stack and confirming the backend and frontend responded successfully:

```bash
curl -fsS http://localhost:4000/api/health
curl -fsS http://localhost:4000/api/health/ready
curl -fsS -X POST http://localhost:4000/api/items -H 'Content-Type: application/json' -d '{"name":"config-check"}'
```

The app successfully returned a ready status and inserted/retrieved the item record.

> Note: this environment had port 8080 already occupied, so the frontend was verified on 8081 during local testing. The project documentation and default Compose setup remain aligned to the assessment standard of 8080 when no conflict exists.
