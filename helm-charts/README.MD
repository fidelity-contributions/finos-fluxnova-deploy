## Prerequisites

### Install Docker

### Install Minikube

```bash
brew install minikube
minikube start -- vm-driver=virtualbox
eval $(minikube docker-env)
```
The “eval” command above associates your Minikube environment to your current shell

### Setup EKS cluster access

```bash
aws configure

aws eks --region us-east-1 update-kubeconfig --name fluxnova-sandbox-cluster
```
You'll need to input `AWS Access Key` and `AWS Secret Key`, which can be created in [IAM > Users section](https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/users) in the AWS Portal.

To test, run `kubectl get nodes`.

### Install Helm

```bash
brew install helm
```

## Helm Charts

This repository includes Helm charts for deploying the application using Kubernetes. The charts are located in the `fluxnova/` directory and are configured to pull images from GitHub Container Registry.

## Deploying with Helm

1. **Create image pull secret** (if using private images):

   ```bash
   kubectl create secret docker-registry ghcr-secret \
     --docker-server=ghcr.io \
     --docker-username=<your-github-username> \
     --docker-password=<your-personal-access-token> \
     --docker-email=<your-email>
   ```
2. **Clone the repository**:

   ```bash
   git clone https://github.com/finos/fluxnova-deploy
   cd fluxnova-deploy
   ```
   
3. **Install the chart**:

   ```bash
    cd helm-charts
    helm install fluxnova --namespace fluxnova-ns --create-namespace ./fluxnova
   ```

4. **Installing ArgoCD**:
   ```bash
    helm repo add argo "https://argoproj.github.io/argo-helm"
    helm repo update
   
    helm install argocd argo/argo-cd \
    -f argo-cd-values.yaml \
    -n argocd --create-namespace
   ```

5. **Check deployment and pod status**:

   ```bash
   kubectl get deployments -n fluxnova-ns
   kubectl get pods -n fluxnova-ns
   ```