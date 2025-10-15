# Creating the Fluxnova Sandbox Cluster on EKS

1. Make sure that `eksctl` and `aws` cli commands are installed
```
brew install eksctl awscli
```

2. Authenticate using AWS access and secret keys:
```
aws configure
```

3. Create the cluster with `eksctl`
```
eksctl create cluster \
  --name fluxnova-sandbox-cluster \
  --region us-east-1 \
  --version 1.33 \
  --nodegroup-name standard-workers \
  --node-type t3.xlarge \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 1 \
  --node-volume-size 100 \
  --managed \
  --with-oidc \
  --ssh-access \
  --ssh-public-key fluxnova-cluster-nodes
```

4. Install docker-registry secret
```
kubectl create secret docker-registry ghcr-secret \
     --docker-server=ghcr.io \
     --docker-username=<github username> \
     --docker-password=<github pat> \
     --docker-email=<github email address>
```

5. Install Nginx Ingress
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.publishService.enabled=true
```

6. Install cert-manager
```
kubectl apply --validate=false -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

kubectl get pods -n cert-manager
```
Wait for the webhook pod to be Ready (`1/1`).

7. Install fluxnova helm chart
```
cd helm-charts
helm install fluxnova --namespace fluxnova-ns --create-namespace ./fluxnova
cd -
```

8. Configure with cloudflare
```
kubectl create secret generic cloudflare-api-token-secret \
  --from-literal=api-token=<Cloudflare API token> \
  -n cert-manager

kubectl apply -f helm-charts/cluster-issuer.yaml
kubectl apply -f helm-charts/certificate.yaml -n fluxnova-ns
kubectl apply -f helm-charts/ingress-tls.yaml -n fluxnova-ns
```

9. Configure keel (auto-deploy new docker images)

```
helm repo add keel https://charts.keel.sh
helm repo update
helm upgrade --install keel --namespace=kube-system keel/keel
kubectl annotate pod -l app=fluxnova keel.sh/policy=force keel.sh/trigger=poll keel.sh/pollSchedule="@every 10m" -n fluxnova-ns

# Make sure keel is ready
kubectl --namespace=kube-system get pods -l "app=keel"
```

10. Setup ingress basic auth
```
htpasswd -c auth us3r
kubectl create secret generic basic-auth  --from-file=auth
```

You should now be able to access https://demo.fluxnova.finos.org .

## Cert manager debugging

You can then watch the request as it flows through the various stages: certificates -> certificaterequests -> orders -> challenges.

```
kubectl get certificates -A
kubectl get certificaterequests -A
kubectl get orders -A
kubectl get challenges -A
```

To reset cert manager setup:
```
kubectl delete -f helm-charts/cluster-issuer.yaml
kubectl delete -f helm-charts/certificate.yaml
kubectl delete -f helm-charts/ingress-tls.yaml
kubectl delete certificate fluxnova-tls -n fluxnova-ns
kubectl delete certificaterequest fluxnova-tls -n fluxnova-ns
kubectl delete order fluxnova-tls-1-2206662564 -n fluxnova-ns
```