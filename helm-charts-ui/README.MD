# FluxNova Control Center UI – Helm Charts

This directory contains the Helm chart for deploying the **FluxNova Control Center UI** on Kubernetes.  
The UI is a separate workload from the FluxNova engine (see `helm-charts-engine/`) and runs on a different port.

| Component | Docker image | Port | Path |
|-----------|-------------|------|------|
| Engine UI (Spring Boot) | `finos/fluxnova-bpm-platform` | 8080 | `/` |
| Engine REST API | `finos/fluxnova-bpm-platform` | 8080 | `/engine-rest` |
| Control Center UI | `finos/fluxnova-control-center` | 4000 | `/control-center` |

---

## Prerequisites

The following must already be installed / configured:

- `kubectl` configured against your target cluster (`aws eks update-kubeconfig …`)
- `helm` ≥ 3
- cert-manager + `ClusterIssuer` applied (see `../helm-charts-engine/cluster-issuer.yaml`)
- nginx Ingress controller deployed on the cluster

---

## Deploying with Helm

### 1. Clone the repository

```bash
git clone https://github.com/finos/fluxnova-deploy
cd fluxnova-deploy
```

### 2. (Optional) Create an image-pull secret for private Docker Hub images

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-dockerhub-username> \
  --docker-password=<your-dockerhub-token> \
  --docker-email=<your-email> \
  --namespace fluxnova-ns
```

Then set `imagePullSecrets[0].name=dockerhub-secret` in `values.yaml` or pass it via `--set`.

### 3. Install / upgrade the chart

```bash
cd helm-charts-ui

helm upgrade --install fluxnova-ui --namespace fluxnova-ns --create-namespace ./fluxnova-ui \
  --set image.tag=latest \
  --set env.FXN_REST_API_URL="https://demo.fluxnova.finos.org/engine-rest" \
  --set env.FLUXNOVA_UI_HOST="demo.fluxnova.finos.org"
```

### 4. Apply the combined TLS Ingress and certificate

> The combined Ingress (`helm-charts-ui/ingress-tls.yaml`) handles routing for **both**
> the engine and the Control Center UI.  Do **not** apply `helm-charts-engine/ingress-tls.yaml`
> alongside it — that causes nginx to route `/fluxnova-control-center` to the engine instead of the UI.

```bash
kubectl apply -f certificate.yaml          -n fluxnova-ns   # skip if already applied for engine
kubectl apply -f ingress-tls.yaml          -n fluxnova-ns   # combined ingress for engine + UI
```

### 5. Check deployment and pod status

```bash
kubectl get deployments -n fluxnova-ns
kubectl get pods        -n fluxnova-ns
kubectl get ingress     -n fluxnova-ns
```

---

## Troubleshooting 502 Bad Gateway

Run these commands in order to narrow down the root cause:

```bash
# 1. Check the pod is Running and Ready (1/1)
kubectl get pods -n fluxnova-ns -l app=fluxnova-ui

# 2. Check pod logs for startup errors
kubectl logs -n fluxnova-ns -l app=fluxnova-ui --tail=50

# 3. Describe the pod – look at Events for image pull errors or probe failures
kubectl describe pod -n fluxnova-ns -l app=fluxnova-ui

# 4. Verify the Service endpoints (must not be empty)
kubectl get endpoints fluxnova-ui -n fluxnova-ns

# 5. Port-forward directly to the POD (not the service) to bypass service routing
POD=$(kubectl get pod -n fluxnova-ns -l app=fluxnova-ui -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n fluxnova-ns pod/$POD 4000:4000
# Then curl http://localhost:4000/ in another terminal

# 6. If ERR_EMPTY_RESPONSE on port-forward – server is binding to 127.0.0.1 not 0.0.0.0
#    This is already fixed by HOST=0.0.0.0 in values.yaml.  Redeploy and test again.
```

**Common causes:**

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Pod not Running | Image pull failure | Add `imagePullSecrets` or check Docker Hub |
| Pod Running but endpoints empty | Readiness probe failing | Check `kubectl describe pod` Events; verify `service.containerPort` |
| Endpoints present but 502 | Container listens on wrong port | Set `service.containerPort` to the actual EXPOSE port of the image |
| 502 only on first load | App takes >15s to start | Already increased `initialDelaySeconds` to 15/30 |

---

## Key configuration values (`values.yaml`)

| Key | Default | Description |
|-----|---------|-------------|
| `image.name` | `finos/fluxnova-control-center` | Docker Hub image name |
| `image.tag` | `latest` | Image tag to deploy |
| `fullnameOverride` | `fluxnova-ui` | Pins the Service/Deployment name (must match `ingress-tls.yaml`) |
| `service.port` | `4000` | Port exposed by the Kubernetes Service |
| `service.containerPort` | `4000` | Port the container **actually listens on** — override if the Docker image uses a different port |
| `env.FLUXNOVA_UI_HOST` | `demo.fluxnova.finos.org` | Hostname used in the Ingress rule |
| `env.FXN_REST_API_URL` | `https://demo.fluxnova.finos.org/engine-rest` | Engine REST API URL injected into the UI container |
| `ingress.path` | `/control-center(/\|$)(.*)` | Path regex for the nginx Ingress rule |
| `replicaCount` | `1` | Number of UI pod replicas |
| `imagePullSecrets` | `[]` | Pull secrets (needed for private Docker Hub images) |

---

## Ingress path routing

Both the engine and the UI share the domain `demo.fluxnova.finos.org`.
All path rules are kept in **one combined Ingress resource** (`helm-charts-ui/ingress-tls.yaml`)
so that nginx evaluates the more specific `/control-center` path before the engine's catch-all `/`.

```
https://demo.fluxnova.finos.org/                  → engine UI        (port 8080)
https://demo.fluxnova.finos.org/engine-rest        → engine REST API   (port 8080)
https://demo.fluxnova.finos.org/control-center     → control center UI (port 4000, prefix stripped → /)
https://demo.fluxnova.finos.org/control-center/foo → control center UI (port 4000, prefix stripped → /foo)
```

> **Why a single Ingress?**  
> When the engine's `path: /` and the UI's `path: /fluxnova-control-center` live in *separate*
> Ingress objects, nginx may evaluate the engine's catch-all first and route all traffic to the
> engine. A single Ingress ensures correct path ordering.

---

## GitHub Actions – Automated Deployment

The workflow at `.github/workflows/deploy-ui.yml` automatically deploys the UI chart to EKS  
whenever files under `helm-charts-ui/` change on the `main` branch.

### Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `AWS_DEPLOY_ROLE_ARN` | ARN of the IAM role assumed via OIDC for EKS access |
| `FXN_REST_API_URL` | *(optional)* Override for the engine REST URL |
| `FLUXNOVA_UI_HOST` | *(optional)* Override for the UI hostname |

The workflow also supports manual dispatch (`workflow_dispatch`) with a configurable `image_tag` input.

