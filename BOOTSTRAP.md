# Cluster Bootstrapping

This document defines procedures for bootstrapping application manifests for a first-time deployment.

## Prerequisites

The manifests in this repository assume the target cluster is already configured with a Traefik ingress controller.
The following tools must also be installed and configured with access to the desired cluster:

- [Helm](https://helm.sh/) — Kubernetes package management.
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) — Kubernetes cluster administration.
- [kubeseal](https://github.com/bitnami-labs/sealed-secrets) — Sealed Secrets encryption and management.

### Sealed Secrets Key (Redeployments and Migrations Only)

This repository uses SealedSecrets to securely store encrypted values in Git.
Secrets are encrypted using a unique key pair managed by the Sealed Secrets controller.
When migrating to a new cluster, the existing key must be exported from the old cluster before it is decommissioned.
Without the existing key, all secrets will need to be regenerated and replaced.

Use the command below to export the existing secrets key.
The exported file contains secure values in plain text and should be stored in a secure location outside of Git.

```shell
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-key-backup.yaml
chmod 600 sealed-secrets-key-backup.yaml
```

The controller rotates its key periodically, so the backup should be taken or refreshed shortly before migration.
If the key is unavailable, all secrets stored in this repository will need to be regenerated.

## Deployment

First time deployments require core cluster services to be provisioned manually.
Once the initial configuration is finished, cluster management is delegated to ArgoCD, which will continuously
monitor the cluster state and automatically reconcile any differences with the state declared in Git.

### 1. Sealed Secrets — Secret Management

The sealed secrets controller enables encrypted secrets to be stored safely in Git.
Follow the section below that matches how secrets are being handled for the deployment.

#### Reusing Existing Secrets

To redeploy or migrate the SealedSecrets already committed to Git, begin by restoring the original 
encryption key. 

```shell
kubectl apply -f sealed-secrets-key-backup.yaml
```

If the controller was already running before the key was restored, restart it so it loads the key:

```shell
kubectl -n kube-system rollout restart deployment/sealed-secrets-controller
```

Finally, deploy the `kube-system` manifests:

```shell
kubectl kustomize --enable-helm manifests/kube-system/ \
  | kubectl apply --server-side -f -
```

Once the key is restored, the SealedSecrets already committed to Git will decrypt as-is.
The secret generation commands in the following steps can be skipped.

#### Using New Secrets

For first-time deployments, or for deployments where an encryption key backup is unavailable,
the Sealed Secrets Controller is deployed alongside the rest of the Kube-System components.
New encrypted secrets will be issued when deploying each individual service.

To deploy the `kube-system` manifests:

```shell
kubectl kustomize --enable-helm manifests/kube-system/ \
  | kubectl apply --server-side -f -
```

### 2. Traefik — Routing and TLS

The Traefik manifests configure the cluster ingress controller to request TLS certificates using the ACME protocol.
These manifests require knowing the authentication keys provided by the upstream DNS service.

Generate a Sealed Secret for the DNS provider's ACME credentials.
The example below uses PorkBun, although any Traefik supported ACME provider will work.
If the existing secrets key was restored, this command can be skipped.

```shell
kubectl create secret generic porkbun-credentials \
  --namespace kube-system \
  --from-literal=PORKBUN_API_KEY='YOUR_PORKBUN_API_KEY' \
  --from-literal=PORKBUN_SECRET_API_KEY='YOUR_PORKBUN_SECRET_API_KEY' \
  --from-literal=ACME_EMAIL='you@example.com' \
  --dry-run=client -o yaml \
  | kubeseal --format yaml \
  > manifests/kube-system/traefik/secrets.yaml
```

Review the generated manifest, then re-apply the `kube-system` manifests:

```shell
kubectl kustomize --enable-helm manifests/kube-system/ \
  | kubectl apply --server-side -f -
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
If the existing secrets key was restored, this command can be skipped.

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

Render the deployment manifests, which also create the `keycloak` namespace, and apply the changes to the cluster.

```shell
kubectl kustomize --enable-helm manifests/keycloak/ | kubectl apply --server-side -f -
```

### 3. Argo CD — GitOps

Configuring ArgoCD to recognize user accounts in Keycloak requires manual changes in the Keycloak UI.
Full instructions are provided in the
[ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/keycloak/#keycloak-and-argo-cd-with-pkce).

ArgoCD requires credentials to read from the Git repository.
Generate a Sealed Secret for the GitHub App credentials using the command below.
If the existing secrets key was restored, this command can be skipped.

```shell
kubectl create secret generic github-app-credentials \
  --namespace argocd \
  --from-literal=type='git' \
  --from-literal=url='https://github.com/YOUR_ORGANIZATION' \
  --from-literal=githubAppID='YOUR_GITHUB_APP_ID' \
  --from-literal=githubAppInstallationID='YOUR_GITHUB_APP_INSTALLATION_ID' \
  --from-file=githubAppPrivateKey=./YOUR_GITHUB_APP_PRIVATE_KEY.pem \
  --dry-run=client -o yaml \
| kubectl label --local -f - \
  argocd.argoproj.io/secret-type=repo-creds \
  -o yaml \
| kubeseal --format yaml > manifests/argocd/argocd/secrets.yaml
```

After updating the OIDC configuration in [manifests/argocd/argocd/values.yaml](manifests/argocd/argocd/values.yaml) to
reflect
values chosen during the Keycloak setup, deploy the ArgoCD manifests, which also create the `argocd` namespace:

```shell
kubectl kustomize manifests/argocd/ --enable-helm \
  | kubectl apply --server-side -f -
```

Assuming all services have been correctly configured, you should now be able to authenticate with Argo
using credentials configured in Keycloak. However, ArgoCD also provides an initial `admin` user account
with a password that can be retrieved using the following command:

```shell
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

### 4. Synchronization

ArgoCD will automatically deploy any remaining manifests to the cluster. 
Applications that were not deployed manually during bootstrap are installed at this stage. 
No further action is required, although it can take a few minutes for applications to appear and become healthy.

To trigger a synchronization immediately, log in to the ArgoCD UI at https://argo.bhpc.dev, open the application,
and click **Sync**.
