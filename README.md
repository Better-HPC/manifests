# K8s Manifests

This repository contains the Kubernetes manifests used to deploy and manage BetterHPC services.
Deployments are managed through Argo CD, which monitors the repository and automatically synchronizes changes with the
live cluster.

## Prerequisites

Install the following tools before bootstrapping or maintaining the cluster:

- [Helm](https://helm.sh/) — Kubernetes package management.
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) — Kubernetes cluster administration.
- [kubeseal](https://github.com/bitnami-labs/sealed-secrets) — Sealed Secrets encryption and management.

## Bootstrapping

Complete the following steps in order when initializing services for the first time.

### 1. Sealed Secrets — Secret Management

The sealed secrets controller enables encrypted secrets to be stored safely in Git and decrypted within the cluster.
Deploy the Sealed Secrets controller and its associated resources:

```shell
kubectl kustomize --enable-helm manifests/sealed-secrets/ \
  | kubectl apply --server-side -f -
```

### 2. Traefik — Routing and TLS

Generate a Sealed Secret for the DNS provider and ACME certificate registration.
Replace the placeholder values with the appropriate credentials and contact email.

```shell
kubectl create secret generic porkbun-credentials \
  --namespace kube-system \
  --from-literal=PORKBUN_API_KEY='YOUR_PORKBUN_API_KEY' \
  --from-literal=PORKBUN_SECRET_API_KEY='YOUR_PORKBUN_SECRET_API_KEY' \
  --from-literal=ACME_EMAIL='you@example.com' \
  --dry-run=client -o yaml \
  | kubeseal --format yaml \
  > manifests/traefik/porkbun-credentials-sealed.yaml
```

Review the generated manifest, then apply the Traefik configuration:

```shell
kubectl apply -f manifests/traefik/
```

Restart Traefik to pick up configuration changes and wait for the deployment to become ready:

```shell
kubectl -n kube-system rollout restart deployment/traefik
kubectl -n kube-system rollout status deployment/traefik

```

### Keycloak - Authentication

```shell
kubectl create secret generic keycloak-credentials \
  --namespace keycloak \
  --from-literal=ADMIN_USERNAME='admin' \
  --from-literal=ADMIN_PASSWORD='K6mR7tjo7!UPgA' \
  --from-literal=DB_PASSWORD='Ml#yti@3X5JUSY' \
  --dry-run=client -o yaml \
| kubeseal --format yaml > manifests/keycloak/secrets.yaml
```

```shell
kubectl create namespace keycloak
kubectl kustomize --enable-helm manifests/keycloak/ | kubectl apply --server-side -f -
```

### 3. Argo CD — GitOps

Create the Argo CD namespace and deploy its manifests:

```shell
kubectl create namespace argocd
kubectl kustomize manifests/argocd/ --enable-helm \
  | kubectl apply --server-side -f -
```

Retrieve the initial administrator password:

```shell
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```
