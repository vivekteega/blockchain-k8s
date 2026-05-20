# Blockchain Kubernetes Infrastructure

Kubernetes-based blockchain infrastructure for running an Anvil Ethereum development node with Helm and ArgoCD. The repository demonstrates a GitOps deployment model, namespace isolation, externalized runtime configuration, manual secret handling, RBAC, pod health checks, and resource controls.

This project is intentionally scoped as local blockchain infrastructure: it runs a deterministic Anvil node for development and testing while using production-style Kubernetes workflows that can evolve toward real execution and consensus clients.

## Overview

The stack deploys an Anvil node from the Foundry image into a dedicated `blockchain` namespace. Kubernetes manifests are managed through a Helm chart, and ArgoCD continuously reconciles the desired state from Git into the cluster.

Key capabilities:

- Anvil Ethereum development node running on Kubernetes.
- Local cluster workflow using `vind` vCluster in Docker.
- Dedicated `blockchain` namespace for isolation.
- Helm chart with configurable values for chain ID, port, host, block time, replicas, image, service type, and resources.
- ConfigMap-driven node configuration.
- Kubernetes Secret dependency for private key and mnemonic values, created manually and never committed to Git.
- Read-only RBAC role for a `junior-dev` service account.
- CPU and memory requests and limits on the Anvil pod.
- TCP liveness and readiness probes.
- ArgoCD Application for GitOps-based synchronization from GitHub.

## Architecture

```text
                 GitHub Repository
                blockchain-k8s repo
                        |
                        | push / merge
                        v
                  ArgoCD Application
                 argocd namespace
                        |
                        | syncs Helm chart
                        v
        +-----------------------------------+
        | Kubernetes cluster via vind       |
        |                                   |
        |  namespace: blockchain            |
        |                                   |
        |  +-----------------------------+  |
        |  | Deployment: anvil           |  |
        |  | image: ghcr.io/foundry-rs   |  |
        |  | command: anvil              |  |
        |  +-------------+---------------+  |
        |                |                  |
        |                v                  |
        |  +-----------------------------+  |
        |  | Service: anvil              |  |
        |  | port: 8545                  |  |
        |  +-----------------------------+  |
        |                                   |
        |  +-----------------------------+  |
        |  | ConfigMap: anvil-config     |  |
        |  | chain ID, host, port, time  |  |
        |  +-----------------------------+  |
        |                                   |
        |  +-----------------------------+  |
        |  | Secret: anvil-keys          |  |
        |  | private key, mnemonic       |  |
        |  | created manually            |  |
        |  +-----------------------------+  |
        |                                   |
        |  +-----------------------------+  |
        |  | RBAC: junior-dev read-only  |  |
        |  +-----------------------------+  |
        +-----------------------------------+
```

## Repository Structure

```text
.
├── anvil-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── configmap.yaml
│       ├── deployment.yaml
│       ├── namespace.yaml
│       ├── rbac.yaml
│       └── service.yaml
├── anvil-deployment.yaml
├── anvil-service.yaml
├── anvil-configmap.yaml
├── anvil-secret-template.yaml
├── rbac.yaml
├── argocd-app.yaml
└── README.md
```

`anvil-chart/` contains the Helm-managed Kubernetes resources used by ArgoCD.

`anvil-chart/values.yaml` contains environment-specific configuration such as chain ID, replica count, image, service type, namespace, and pod resources.

`anvil-deployment.yaml`, `anvil-service.yaml`, `anvil-configmap.yaml`, and `rbac.yaml` are raw manifest references. The Helm chart is the preferred deployment path.

`anvil-secret-template.yaml` documents the required secret shape without storing real secret values.

`argocd-app.yaml` defines the ArgoCD Application that points to this repository and deploys the Helm chart.

## Prerequisites

- `vind` installed and running a local Kubernetes environment.
- Helm 3.
- `kubectl` configured for the target cluster.
- ArgoCD installed in the `argocd` namespace.
- Access to the GitHub repository configured in `argocd-app.yaml`.
- An `anvil-keys` Kubernetes Secret created manually in the `blockchain` namespace.

## Quick Start

Create or select the local Kubernetes environment:

```bash
vind up
kubectl cluster-info
```

Create the namespace if it does not already exist:

```bash
kubectl create namespace blockchain
```

Create the Anvil wallet secret manually before deploying the workload:

```bash
kubectl create secret generic anvil-keys \
  --namespace blockchain \
  --from-literal=private-key=<YOUR_PRIVATE_KEY> \
  --from-literal=mnemonic="<YOUR_MNEMONIC>"
```

Render the Helm chart locally:

```bash
helm template anvil ./anvil-chart
```

Install or upgrade the chart manually:

```bash
helm upgrade --install anvil ./anvil-chart --namespace blockchain
```

Verify the deployment:

```bash
kubectl get pods -n blockchain
kubectl get svc -n blockchain
kubectl logs -n blockchain deploy/anvil
```

Apply the ArgoCD Application for GitOps-based deployment:

```bash
kubectl apply -f argocd-app.yaml
kubectl get applications -n argocd
```

After ArgoCD takes ownership, changes should be made through Git and reconciled by ArgoCD instead of direct manual edits in the cluster.

## Secret Management

The Anvil pod expects a Kubernetes Secret named `anvil-keys` in the `blockchain` namespace with the following keys:

- `private-key`
- `mnemonic`

Create it manually:

```bash
kubectl create secret generic anvil-keys \
  --namespace blockchain \
  --from-literal=private-key=<YOUR_PRIVATE_KEY> \
  --from-literal=mnemonic="<YOUR_MNEMONIC>"
```

Confirm that the secret exists without printing the secret values:

```bash
kubectl get secret anvil-keys -n blockchain
```

Real private keys and mnemonics must never be committed to this repository. The file `anvil-secret-template.yaml` is a reference only and should not be applied with placeholder values.

## GitOps Workflow

ArgoCD is configured through `argocd-app.yaml` to watch the GitHub repository and deploy the Helm chart from `anvil-chart/`.

The workflow is:

1. Update Helm templates or `anvil-chart/values.yaml`.
2. Commit and push the change to GitHub.
3. ArgoCD detects the new desired state.
4. ArgoCD syncs the chart into the `blockchain` namespace.
5. Kubernetes reconciles the Deployment, Service, ConfigMap, RBAC, and namespace resources.

The Application enables automated sync with pruning and self-healing:

- `prune: true` removes resources that are no longer defined in Git.
- `selfHeal: true` reverts manual drift in the cluster back to the Git-defined state.
- `CreateNamespace=true` allows ArgoCD to create the destination namespace if required.

This keeps Git as the source of truth for infrastructure changes.

## RBAC

The Helm chart includes a read-only access model for junior developers:

- `Role`: `blockchain-viewer`
- `ServiceAccount`: `junior-dev`
- `RoleBinding`: `junior-dev-viewer`

The role grants read-only access to:

- pods
- pod logs
- services
- configmaps

Allowed verbs are:

- `get`
- `list`
- `watch`

The role intentionally does not grant access to Secrets or write operations. This allows developers to inspect workload health and logs without exposing wallet material or allowing changes to cluster state.

## Design Decisions

### TCP probes instead of HTTP probes

Anvil exposes an Ethereum JSON-RPC endpoint. Kubernetes `httpGet` probes are not a good fit because JSON-RPC endpoints expect structured POST requests and may not return probe-compatible responses for plain HTTP GET checks.

The chart uses `tcpSocket` liveness and readiness probes instead. This validates that the Anvil process is listening on the configured RPC port without making assumptions about JSON-RPC HTTP semantics.

### Secrets are not managed by Helm

Wallet private keys and mnemonics are sensitive runtime inputs. They are intentionally excluded from Helm values and Git-tracked manifests.

The chart references the `anvil-keys` Secret, but the secret itself is created manually. This keeps credentials outside the repository and avoids leaking sensitive values through Helm release metadata, Git history, or pull requests.

### Helm for environment configuration

The Helm chart makes the Anvil deployment portable across environments. Values such as namespace, replica count, image tag, port, chain ID, block time, service type, and resource limits can be adjusted without rewriting Kubernetes manifests.

### Namespace isolation

The `blockchain` namespace provides a clear boundary for blockchain infrastructure resources. It makes access control, cleanup, troubleshooting, and future multi-environment deployments simpler.

### Resource requests and limits

The Deployment defines CPU and memory requests and limits so the Anvil pod has predictable scheduling behavior and a bounded resource footprint in shared clusters.

### Raw manifests kept as references

Raw YAML manifests are retained as learning and reference artifacts. The operational deployment path is the Helm chart, especially when used through ArgoCD.

## Configuration

Default values are defined in `anvil-chart/values.yaml`:

```yaml
anvil:
  chainId: "31337"
  port: "8545"
  host: "0.0.0.0"
  blockTime: "12"

replicaCount: 1
```

Resource settings:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

These values can be overridden with a separate values file for different environments:

```bash
helm upgrade --install anvil ./anvil-chart \
  --namespace blockchain \
  --values values-dev.yaml
```

## Operational Checks

Check ArgoCD state:

```bash
kubectl get application anvil -n argocd
```

Check workload status:

```bash
kubectl get deploy,pod,svc,configmap -n blockchain
```

Check Anvil logs:

```bash
kubectl logs -n blockchain deploy/anvil
```

Port-forward the Anvil RPC service locally:

```bash
kubectl port-forward -n blockchain svc/anvil 8545:8545
```

Test the RPC endpoint:

```bash
curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'
```

## Week 2 Plans

The next phase is to move from a local Anvil development node toward production-grade blockchain infrastructure patterns:

- Provision Hetzner infrastructure for a persistent remote Kubernetes environment.
- Deploy a real Ethereum execution client such as Geth.
- Deploy a consensus client such as Lighthouse.
- Add persistent volumes for chain data.
- Introduce node monitoring with Prometheus metrics and Grafana dashboards.
- Add alerting for peer count, sync status, disk usage, restart loops, and RPC availability.
- Separate environment values for local, staging, and remote infrastructure.
- Evaluate external secret management for non-local environments.

## Security Notes

- Do not commit private keys, mnemonics, kubeconfigs, ArgoCD credentials, or generated secret manifests.
- Use short-lived local development keys only.
- Restrict Secret access through RBAC.
- Treat Helm values and ArgoCD Application manifests as public infrastructure configuration.
- Rotate any key that has been exposed in shell history, logs, Git commits, screenshots, or CI output.

## Status

This repository represents Week 1 of a blockchain infrastructure portfolio project: local Kubernetes deployment, Helm packaging, GitOps delivery, secure secret handling, health checks, RBAC, and resource controls for an Anvil-based Ethereum development node.
