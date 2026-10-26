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
These manifests require knowing the authentication keys provided by the upstream DNS service.

Generate a Sealed Secret for the DNS provider's ACME credentials.
The example below uses PorkBun, although any Traefik supported ACME provider will work.

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
account and one for its backend database connection. The administrator credentials are needed for the
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

Configuring ArgoCD to recognize user accounts in Keycloak requires manual changes in the Keycloak UI.
Full instructions are provided in the 
[ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/keycloak/#keycloak-and-argo-cd-with-pkce).

After updating the OIDC configuration in [manifests/argocd/values.yaml](manifests/argocd/values.yaml) to reflect 
values chosen during the Keycloak setup, create the Argo CD namespace and deploy the ArgoCD manifests:

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
