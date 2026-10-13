# Node + React + Postgres — Hands-On DevOps Assessment

This repo implements the production-style platform described in the assessment PDF using a local Docker Compose setup, Kubernetes manifests, an API gateway pattern, CI/CD automation, platform services, and observability scaffolding.

## 1. Local containerized setup

### Start the stack

```bash
cd /home/sharetrip/Downloads/node-react-postgres-platform
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

Use a local cluster such as kind or minikube and load the images locally or push to a registry the cluster can resolve.

```bash
kubectl apply -f k8s/namespace-dev.yaml
kubectl apply -n dev -f k8s/
```

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

- The workflow publishes images to GHCR using the automatically provided `GITHUB_TOKEN`; grant the workflow `packages: write` permission.
- Cluster access uses a base64-encoded kubeconfig secret such as `KUBECONFIG_B64`.
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

The observability folder includes example manifests for:

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

The backend HPA scales from 2 to 6 replicas based on CPU and memory utilization.

### Rate limiting

Rate limiting is best applied at the ingress or gateway layer so the platform can reject excess traffic before it reaches the backend. This protects upstream services and keeps the worker and database from being overwhelmed under burst traffic.

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
