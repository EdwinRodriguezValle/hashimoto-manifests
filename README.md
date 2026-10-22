
# Hashimoto Platform - GitOps Manifests 📜

This repository acts as the single source of truth for the infrastructure and deployments of the Hashimoto Platform. It handles all Kubernetes resources, logical naming structures, and sync state controllers via Argo CD utilizing a declarative GitOps strategy.

## 🏗️ Prerequisites

Before replicating the deployment loop, ensure your development machine has the following tools installed:
• Docker / Docker Engine (Running live)
• K3d (Local Kubernetes cluster engine)
• kubectl (Kubernetes CLI manager)

## 🚀 Step-by-Step Cluster Bootstrap

Follow these sequential steps to clean, spin up, and hand over clúster orchestration to GitOps.

## 1. Provision the Local Cluster

Spin up your pre-configured hybrid cluster environment (ensure your terminal execution context matches your orchestration scripts if applicable):

```bash
# Example assuming local cluster provisioning wrappers exist or standard engine startup
k3d cluster create hashimoto-tracker --api-port 6443 -p "8085:80@loadbalancer" --agents 2
```

## 2. Live Install the Argo CD Core Operator

Initialize the control plane environment and apply your local permanent Ingress/HTTP settings to avoid manual port-forwarding:
```bash
# Create the bootstrap infrastructure namespace
kubectl create namespace argocd

# Inject the certified stable release definitions directly from the remote cloud
kubectl apply -n argocd --server-side --force-conflicts -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)

# Apply the local bootstrap configurations (Disables internal TLS & enables Ingress permanent routing)
kubectl apply -f argocd/bootstrap/
```

## 3. Verify Control Plane Status
Monitor the startup sequence until all runtime pods show an active, running condition (1/1 Running):
```bash
kubectl get pods -n argocd -w
kubectl get pods -n argocd
```

## 4. Decode the Admin Authentication Credentials

Extract and unpack the initial administrative access pass string from your cluster's secure memory:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

## 🗺️ Activating Declarative Platform GitOps

Instead of applying resources manually via terminal flags, pass absolute root control to the declarative engine using the App-of-Apps management configuration.

**1. Access the Graphical Interface Dashboard**
Thanks to your permanent Ingress and local host mapping, you no longer need kubectl port-forward.
Make sure you have added 127.0.0.1 argocd.local to your system's hosts file (/etc/hosts or C:\Windows\System32\drivers\etc\hosts).

👉 Open your browser context and go to: http://argocd.local:8085

Username: admin

Password: (The decoded string gathered in step 4 above of run de bellow code)
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

(Note: Accept the browser SSL warning if prompted, though internal HTTP is active via Traefik).

**2. Inject the Initialization Root Controller**

Inject the master declarative configuration token. This statement forces Argo CD to claim self-ownership over its configurations and automatically scale out your institutional namespaces directly from Git tracking:
```bash
kubectl apply -f argocd/applications/hashimoto-local.yaml
````

**📂 Repository File System Layout**

```text
hashimoto-manifests/
├── .github/
│   └── workflows/
│       └── validate-manifests.yaml   # Remote CI schema verification loop
├── apps/
│   └── api-gateway/
│       ├── base/                     # Standard workload templates
│       └── overlays/
│           └── local/                # Kustomize tag mutations for GHCR
├── argocd/
│   ├── bootstrap/                    # Permanent infrastructure configurations (Ingress & Params)
│   │   ├── argocd-params-cm.yaml     # Enables internal HTTP mode (server.insecure) to run in local cluster and avoid TLS certs 
│   │   └── argocd-ingress.yaml       # Permanent Ingress route for Traefik
│   └── applications/                 # Master declarative application controllers
│       └── hashimoto-local.yaml      # Root App-of-Apps declaration
└── platform/
    └── namespaces/
        ├── kustomization.yaml        # Aggregator configuration
        └── namespaces.yaml           # Shared platform division manifests

```