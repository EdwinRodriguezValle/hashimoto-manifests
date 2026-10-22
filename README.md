# Hashimoto Manifests

This repository contains the Argo CD bootstrap configuration and `Application` resources for the Hashimoto platform. Application source code and Kubernetes workload manifests live in [`hashimoto-platform`](https://github.com/EdwinRodriguezValle/hashimoto-platform).

The first deployment target is a K3d cluster on a developer's computer. GitHub Actions builds and publishes images; Argo CD pulls configuration from Git and reconciles the local cluster. No always-on server or inbound connection to the developer's computer is required. The computer and K3d cluster must be running for Argo CD to deploy changes.

## Repository layout

```text
argocd/
├── bootstrap/       # Local Argo CD HTTP and Ingress settings
└── applications/    # Application resources applied to Argo CD
kustomization.yaml   # Kustomize root for this repository
namespaces.yaml      # Platform namespaces
```

The current applications are:

- `hashimoto-platform-namespaces` reads this repository's root Kustomize configuration and creates the platform namespaces.
- `hashimoto-api-gateway` reads `hashimoto-platform`, branch `main`, path `k8s`. That path's root Kustomization deploys the platform resources, despite the Application's current display name.

The namespace Application currently targets `feature/argocd-manifests`. Before using this as the shared team setup, merge the manifests changes to `main` and update its `targetRevision` to `main`, or deliberately select another maintained branch.

## Prerequisites

- Docker, K3d, `kubectl`, Git, and Make installed on the development computer.
- The `hashimoto-platform` and `hashimoto-manifests` repositories cloned locally.
- A K3d cluster with context `k3d-hashimoto-cluster`. The platform repository's [K3d onboarding guide](https://github.com/EdwinRodriguezValle/hashimoto-platform/blob/main/docs/k8s-onboarding.md) creates it.
- GitHub Actions enabled for `hashimoto-platform` and the GHCR packages configured for Kubernetes pulls.

## Local K3d setup

For the full workflow—including branch/PR behavior, GHCR image publication, Argo CD sync, troubleshooting, and local security boundaries—follow [GitOps with Argo CD on local K3d](https://github.com/EdwinRodriguezValle/hashimoto-platform/blob/main/docs/gitops-k3d.md).

From the `hashimoto-platform` repository, create or select the cluster:

```bash
make cluster-create
# If it already exists, use this instead:
# make cluster-use
kubectl config current-context
kubectl get nodes
```

From the root of this repository, install Argo CD and the local ingress settings:

```bash
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f argocd/bootstrap/
kubectl wait --for=condition=Available deployment/argocd-server \
  -n argocd --timeout=300s
```

The install URL follows the current `stable` release. For a repeatable team setup, pin and document a specific Argo CD release. The bootstrap uses local HTTP through Traefik; do not expose it to a public network.

Add `127.0.0.1 argocd.local` to the computer's hosts file, then register both Applications:

```bash
kubectl apply -f argocd/applications/hashimoto-local.yaml
kubectl get application hashimoto-platform-namespaces -n argocd -w
# Wait until it reports Synced and Healthy, then stop watching with Ctrl-C.
kubectl apply -f argocd/applications/api-gateway.yaml
kubectl get applications -n argocd
```

Open [http://argocd.local:8085](http://argocd.local:8085). The initial admin password is in the `argocd-initial-admin-secret` Secret in namespace `argocd`. Change the password after logging in.

To print the initial password in your terminal:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

The two Application files are currently applied manually. The root Application manages namespaces; it does **not** discover other Application files by itself. After editing an Application definition, run `kubectl apply -f` for that file. A future app-of-apps setup can make Application registration fully declarative.

The current `namespaces.yaml` in this repository still declares the `argocd` namespace, and the root Application has pruning enabled. Do not remove that declaration casually: first migrate namespace ownership so an Argo sync cannot prune the namespace Argo CD itself. The application repository's `k8s/namespaces.yaml` intentionally does not declare `argocd`.

## How a deployment reaches K3d

1. Create a `feature/...` branch and open a pull request to `main` in `hashimoto-platform`.
2. Review the backend, frontend, and security checks; merge the pull request.
3. GitHub Actions builds the frontend, API Gateway, and Patient Service images, tags them with the source commit SHA, and publishes them to GHCR.
4. The workflow updates the image tags in `hashimoto-platform/k8s/core/kustomization.yaml` and commits that change to `main`.
5. Argo CD detects the new Git revision and synchronizes the local K3d cluster.

If GHCR packages are public, K3d can pull them without registry credentials. For private packages, configure an image pull Secret in the cluster and reference it from the workload pods. Never store its value in either repository.

To check the deployment:

```bash
kubectl get applications -n argocd
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
```

For rollback, revert the commit that changed the image tag or Kubernetes configuration. Argo CD will reconcile the previous desired state from Git.

## Troubleshooting

- **`cluster ... not found`:** the local cluster destination should be `https://kubernetes.default.svc`. `https://default.svc` is not the in-cluster API address.
- **Application does not update after a push:** check `repoURL`, `targetRevision`, and `path`. Changes to `argocd/applications/*.yaml` also require applying that Application file because there is not yet an app-of-apps parent.
- **Argo cannot render `k8s`:** confirm that `hashimoto-platform/k8s/kustomization.yaml` exists on the selected branch.
- **`ImagePullBackOff`:** check that the image tag exists in GHCR and that the package is public or the namespace has a valid image pull Secret.
- **Argo keeps reverting a manual change:** Git is the desired state. Make the intended edit in Git and let Argo CD sync it.
- **The image-tag workflow cannot push its update:** confirm the workflow's `contents: write` permission is allowed by the repository or organization Actions settings.

## Security notes

The local cluster uses development settings. Do not commit real credentials, access tokens, or personal health data. Keep Argo CD local, rotate its initial admin password, and use a secret-management solution before sharing or promoting the deployment beyond local development.
