
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

Initialize the isolated control plane environment inside the cluster to boot up the delivery loops:
```bash
# Create the bootstrap infrastructure namespace
kubectl create namespace argocd

# Inject the certified stable release definitions directly from the remote cloud
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 3. Verify Control Plane Status
Monitor the startup sequence until all runtime pods show an active, running condition:
```bash
kubectl get pods -n argocd -w
```

## 4. Decode the Admin Authentication Credentials

Extract and unpack the initial administrative access pass string from your cluster's secure memory:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

## 🗺️ Activating Declarative Platform GitOps

Instead of applying resources manually via terminal flags, pass absolute root control to the declarative engine using the App-of-Apps management configuration.

**1. Expose the Graphical Interface Dashboard**

Establish a secure communication pipe to view your platform loops inside your local browser layout:
```bash
kubectl port-forward svc/argocd-server -n argocd 8085:443
```


👉 Open your browser context and go to: https://localhost:8080
• Username: admin
• Password: (The decoded string gathered in the step above)

**2. Inject the Initialization Root Controller**

Inject the master declarative configuration token. This statement forces Argo CD to claim self-ownership over its configurations and automatically scale out your institutional namespaces (hashimoto-core, infra, ops, logs) directly from Git tracking:
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
│       ├── base/                    # Standard workload templates
│       └── overlays/
│           └── local/               # Kustomize tag mutations for GHCR
├── argocd/
│   └── applications/
│       └── hashimoto-local.yaml     # Main App root declarative controller
└── platform/
└── namespaces/
├── kustomization.yaml       # Aggregator configuration
└── namespaces.yaml          # Shared platform division manifests

```

Now that your README.md file is ready to be committed to the root of your manifests repository, let's complete the final step of the pipeline loop!
Once you log into Argo CD on your browser via https://localhost:8080:
• Does the hashimoto-platform-namespaces card appear inside your dashboard?
• Clicking "Sync", does it show a bright green Synced / Healthy status?
Let me know what you see on the screen to verify that your cluster has been fully automated!