# Cluster Bootstrapping

This document defines procedures for bootstrapping application manifests for a first-time deployment.

## Prerequisites

The manifests in this repository assume the target cluster is already configured with a Traefik ingress controller.
The following tools must also be installed and configured with access to the desired cluster:

- [Helm](https://helm.sh/) — Kubernetes package management.
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) — Kubernetes cluster administration.
- [kubeseal](https://github.com/bitnami-labs/sealed-secrets) — Sealed Secrets encryption and management.

## Deployment

First time deployments require core cluster services to be provisioned manually.
Once the initial configuration is finished, cluster management is delegated to ArgoCD, which will continuously
monitor the cluster state and automatically reconcile any differences with the state declared in Git.

### 1. Sealed Secrets — Secret Management

The sealed secrets controller enables encrypted secrets to be stored safely in Git and decrypted within the cluster.
Deploy the Sealed Secrets controller and its associated resources:

```shell
kubectl kustomize --enable-helm manifests/sealed-secrets/ \
  | kubectl apply --server-side -f -
```

### 2. Traefik — Routing and TLS

The Traefik manifests configure the cluster ingress controller to request TLS certificates using the ACME protocol.
This requires knowing the authentication keys required by your upstream DNS service.

Generate a Sealed Secret for the DNS provider's ACME protocol.
Replace the placeholder values with the appropriate credentials and contact email.

```shell
kubectl create secret generic porkbun-credentials \
  --namespace kube-system \
  --from-literal=PORKBUN_API_KEY='YOUR_PORKBUN_API_KEY' \
  --from-literal=PORKBUN_SECRET_API_KEY='YOUR_PORKBUN_SECRET_API_KEY' \
  --from-literal=ACME_EMAIL='you@example.com' \
  --dry-run=client -o yaml \
  | kubeseal --format yaml \
  > manifests/traefik/secrets.yaml
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

Keycloak requires two sets of credentials during provisioning: one for the default administrator
account and one for the database connection. The administrator credentials are needed for the
first login and should be recorded for later use. The database credentials do not need to be known by
the operator, and can be set to a random secure value.

```shell
kubectl create secret generic keycloak-credentials \
  --namespace keycloak \
  --from-literal=ADMIN_USERNAME='admin' \
  --from-literal=ADMIN_PASSWORD='admin' \
  --from-literal=DB_NAME='keycloak' \
  --from-literal=DB_USER='keycloak' \
  --from-literal=DB_PASSWORD="$(openssl rand -base64 32 | tr -d '/+=' | cut -c1-32)" \
  --dry-run=client -o yaml \
| kubeseal --format yaml > manifests/keycloak/secrets.yaml
```

Create the target namespace, render the deployment manifests, and apply the changes to the cluster.

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