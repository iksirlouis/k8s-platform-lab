# GitOps Microservices Platform Lab

A local, auto-scaling Kubernetes platform built from scratch in Minikube. This project moves away from manual `kubectl` setups by introducing a complete **GitOps Delivery Pipeline** to handle app deployments, persistent data, and full cluster monitoring.

---

## Architecture Overview

The platform uses a standard GitOps flow where changes pushed to the repository are automatically synced to the local cluster.

```text
       ┌────────────────────────────────────────────────────────┐
       │                   GitHub Repository                    │
       │                    (App Manifests)                     │
       └───────────────────────────┬────────────────────────────┘
                                   │
                                   ▼ (GitOps Sync Loop)
┌──────────────────────────────────────────────────────────────────────────────┐
│ Minikube Local Cluster                                                       │
│                                  ▼                                           │
│                    ┌───────────────────────────┐                             │
│                    │          Argo CD          │                             │
│                    │      (GitOps Engine)      │                             │
│                    └─────────────┬─────────────┘                             │
│                                  │                                           │
│         ┌────────────────────────┼────────────────────────┐                  │
│         ▼                        ▼                        ▼                  │
│  ┌──────────────┐         ┌──────────────┐         ┌──────────────┐          │
│  │  ConfigMap   │         │  HPA Engine  │         │ Storage PVC  │          │
│  │(HTML Inject) │         │ (Auto-Scale) │         │ (1Gi Disk)   │          │
│  └──────┬───────┘         └──────┬───────┘         └──────┬───────┘          │
│         │                        │                        │                  │
│         └───────────────┐        │        ┌───────────────┘                  │
│                         ▼        ▼        ▼                                  │
│                     ┌───────────────────────────┐                            │
│                     │  baseline-web-deployment  │                            │
│                     │     (Nginx Core Pods)     │                            │
│                     └────────────▲──────────────┘                            │
│                                  │                                           │
│                                  │ (Metrics Collection)                      │
│                                  ▼                                           │
│                     ┌───────────────────────────┐                            │
│                     │   Prometheus & Grafana    │                            │
│                     │   (Monitoring Stack)      │                            │
│                     └───────────────────────────┘                            │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Core Features

* **GitOps Pipeline:** Eliminated manual `kubectl apply` commands. Used **Argo CD** to track the `/apps` directory on GitHub so that any git commit automatically triggers a rollout and prunes deleted resources.
* **Self-Healing Cluster:** Enabled `selfHeal: true` inside the root Argo CD controller. If any manual tweaks or configuration drift happen inside the cluster, Argo CD automatically overwrites them to match the Git state.
* **Autoscaling (HPA):** Configured a **Horizontal Pod Autoscaler (HPA)** backed by explicit container resource limits (CPU Request: `50m`, Limit: `100m`). Tested it by running a high-traffic stress loop, successfully forcing the cluster to scale out up to 4 replicas.
* **Argo CD Override Fix:** Resolved the common conflict where Argo CD fights with the live autoscaler by adding an **`ignoreDifferences`** block to the root manifest. This lets the HPA scale up replicas seamlessly without throwing sync errors in Argo.
* **Persistent Storage:** Mounted a **1Gi Persistent Volume Claim (PVC)** using Minikube's local storage provider directly to the web server's logging directory (`/var/log/nginx`) so that access logs survive container restarts.
* **Monitoring & Alerts:** Deployed the **Kube-Prometheus-Stack** via **Helm**. This tracks CPU/Memory quotas, working set sizes, and network throughput, displaying them in real-time on custom **Grafana** dashboards.

---

## Repository Structure

```text
├── apps/
│   ├── 01-test-pod.yaml        # Nginx Web Deployment & resource allocations
│   ├── 02-test-service.yaml    # ClusterIP service for internal routing
│   ├── 03-web-content.yaml     # ConfigMap containing custom HTML content
│   └── 05-web-hpa.yaml         # Autoscaler logic and metrics limits
└── platform/
    ├── argocd-install.yaml     # Local backup of Argo CD manifests
    └── root-application.yaml   # Argo CD App-of-Apps controller
```

---

## Testing & Metrics

### Load Test Results
Idle baseline: The web app runs on a single pod using roughly **9.97 MiB of RAM** and `0%` CPU.

When hitting the service with a heavy load loop:
1. Traffic spikes through the `baseline-web-service`.
2. Average pod CPU usage quickly shoots past the **50% scaling threshold**.
3. Kubernetes instantly intercepts the traffic load, scaling the deployment up to **4 running pods** to balance the incoming requests.
4. After traffic stops, the 5-minute cool-down window triggers and safely spins down the extra pods.

---

## Tech Stack
* **Infrastructure:** Kubernetes, Minikube, WSL2 (Ubuntu)
* **CD & Package Management:** Argo CD, Helm v3
* **Monitoring:** Prometheus Operator, Grafana

---

##Deployment & Teardown

### Quick Start (Rebuild from scratch)
If you want to spin up the entire environment from a blank slate, run these commands in order:

```bash
# 1. Start the local cluster
minikube start --driver=docker

# 2. Deploy Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f platform/argocd-install.yaml

# 3. Apply the root app to bootstrap everything
kubectl apply -f platform/root-application.yaml

# 4. Access the Argo CD UI (in a separate terminal)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Clean Teardown
To completely wipe the cluster and free up your computer's CPU and RAM without losing your local Git code:

```bash
# Delete the local Minikube cluster and all its volumes
minikube delete
```

