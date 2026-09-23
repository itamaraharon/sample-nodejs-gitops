
## Helm Chart

Built with `helm create` and customized to satisfy the assignment's requirements:

- **Deployment** with 2 replicas (see Design Decisions in the application repo's README for the Deployment-vs-StatefulSet reasoning)
- **Liveness & readiness probes** against `/live` and `/ready`
- **Service** (ClusterIP, port 80 → target port 8080)
- **Ingress** (nginx class, host `sample-nodejs.local`)
- **Resource requests/limits** (`100m`/`128Mi` requests, `250m`/`256Mi` limits)
- **ConfigMap** for non-sensitive config (`LOG_LEVEL`, `APP_ENV`)
- **Secret** for sensitive config (`API_KEY`, base64-encoded via Helm's `b64enc`)
- **Hardened pod/container securityContext**: `runAsNonRoot`, non-root UID, all Linux capabilities dropped
- **imagePullSecrets** (`dockerhub-creds`) — required because the image repository is private

### Validate locally

```bash
cd sample-nodejs
helm lint .
helm template .
```

### Install manually (without ArgoCD)

```bash
helm install sample-nodejs-test . -n sample-nodejs --create-namespace
```

## GitOps / ArgoCD

**Design decision: separate GitOps repo, not ArgoCD pointed at the application repo directly.**

This repo exists specifically so that:
- Application source code and deployed desired-state are cleanly separated — day-to-day app commits (refactors, docs, tests) never touch what's actually running in the cluster.
- ArgoCD only needs read access to one narrow, purpose-built repo, rather than the whole application repo.
- This repo's commit history is a clean, complete audit log of exactly what was deployed and when — every commit here is a real deployment event, not incidental noise.
- The application repo's CI never needs cluster credentials; it only needs write access to this repo (via a scoped PAT), which then hands off to ArgoCD.

### ArgoCD Application configuration

| Field | Value |
|---|---|
| Project | `default` |
| Repo URL | `https://github.com/itamaraharon/sample-nodejs-gitops` |
| Path | `sample-nodejs` |
| Cluster | `https://kubernetes.default.svc` |
| Namespace | `sample-nodejs` |
| Sync Policy | Automatic, Self Heal, Prune |

### Flow

1. A commit lands on `main` in the application repo.
2. CI runs SAST (SonarCloud), bumps the version, builds the Docker image, scans it with Trivy, and pushes it to Docker Hub — all with hard-fail gates.
3. CI updates `image.tag` in this repo's `values.yaml` and pushes.
4. ArgoCD detects the change and auto-syncs the cluster to match.
5. New pods roll out; Self Heal reverts any manual drift from this desired state; Prune removes resources no longer declared here.

## Prerequisites for the private image

Because the Docker Hub repository (`itamara/sample-nodejs`) is private, the target namespace needs a pull secret before ArgoCD can successfully roll out pods:

```bash
kubectl create secret docker-registry dockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<dockerhub-username> \
  --docker-password=<dockerhub-access-token> \
  -n sample-nodejs
```

This secret is referenced by the chart via `imagePullSecrets` in `values.yaml`, and is **not** committed to this repo (it's created manually in the cluster, out of band, as a security best practice).
