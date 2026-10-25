# K8s Manifests

This repository defines Kubernetes manifests for BetterHPC services.
Changes are monitored by ArgoCD and automatically deployed/synchronized against the live cluster.

## Prerequisites

The following utilities are required to initialize and maintain deployed resources.

- Helm
- Kubectl
- Kubeseal

## Bootstrapping

### Sealed Secrets: Secret Management

```shell
kubectl kustomize --enable-helm manifests/sealed-secrets/ | kubectl apply --server-side -f -
```

### Traefik: Routing and TLS

Define preliminary secrets:

```shell
kubectl create secret generic porkbun-credentials \
  --namespace kube-system \
  --from-literal=PORKBUN_API_KEY='YOUR_PORKBUN_API_KEY' \
  --from-literal=PORKBUN_SECRET_API_KEY='YOUR_PORKBUN_SECRET_API_KEY' \
  --from-literal=ACME_EMAIL='you@example.com' \
  --dry-run=client -o yaml \
| kubeseal --format yaml > manifests/traefik/porkbun-credentials-sealed.yaml
```

Update the traefik config:

```shell
kubectl apply -f manifests/traefik/
```

usefull commands for traefik:

kubectl -n kube-system rollout restart deployment/traefik

Wait for it to come back up:

```bash
kubectl -n kube-system rollout status deployment/traefik
```

Then watch the logs for certificate issuance:

```bash
kubectl -n kube-system logs deployment/traefik -f
```

### ArgoCD: GitOps

Apply the argocd manifests.

```shell
kubectl create namespace argocd
kubectl kustomize manifests/argocd/ --enable-helm | kubectl apply --server-side -f -
```

Fetch the initial argocd password:

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```