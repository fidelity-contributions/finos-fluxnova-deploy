# FluxNova Control Center UI – Helm Charts

This directory contains the Helm chart for deploying the **FluxNova Control Center UI** on Kubernetes.  
The UI is a separate workload from the FluxNova engine (see `helm-charts-engine/`) and runs on a different port.

| Component | Docker image | Port | Path |
|-----------|-------------|------|------|
| Engine (Spring Boot) | `finos/fluxnova-springboot-demo` | 8080 | `/monitoring` |
| UI (Control Center) | `finos/fluxnova-control-center` | 4000 | `/` |

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
  --set env.FXN_REST_API_URL="https://demo.fluxnova.finos.org/monitoring" \
  --set env.FLUXNOVA_UI_HOST="demo.fluxnova.finos.org"
```

### 4. Apply the TLS Ingress and certificate

> Skip if you already applied the engine TLS resources and share the same domain.

```bash
kubectl apply -f certificate.yaml   -n fluxnova-ns
kubectl apply -f ingress-tls.yaml   -n fluxnova-ns
```

### 5. Check deployment and pod status

```bash
kubectl get deployments -n fluxnova-ns
kubectl get pods        -n fluxnova-ns
kubectl get ingress     -n fluxnova-ns
```

---

## Key configuration values (`values.yaml`)

| Key | Default | Description |
|-----|---------|-------------|
| `image.name` | `finos/fluxnova-control-center` | Docker Hub image name |
| `image.tag` | `latest` | Image tag to deploy |
| `service.port` | `4000` | Container port the UI listens on |
| `env.FLUXNOVA_UI_HOST` | `demo.fluxnova.finos.org` | Hostname used in the Ingress rule |
| `env.FXN_REST_API_URL` | `https://demo.fluxnova.finos.org/monitoring` | Engine REST API URL injected into the UI container |
| `replicaCount` | `1` | Number of UI pod replicas |
| `imagePullSecrets` | `[]` | Pull secrets (needed for private Docker Hub images) |

---

## Ingress path routing

Both the engine and the UI share the domain `demo.fluxnova.finos.org`.  
They are separated by Ingress path rules:

```
https://demo.fluxnova.finos.org/             → fluxnova-ui   (port 4000)
https://demo.fluxnova.finos.org/monitoring   → fluxnova       (port 8080)
```

Ensure the engine's `ingress-tls.yaml` (`helm-charts-engine/ingress-tls.yaml`) uses  
`path: /monitoring` instead of `/` so that both Ingresses can coexist without conflict.

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

