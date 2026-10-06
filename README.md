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

### How to Run the Infrastructure Stress Test

To validate the elasticity of the cloud platform and witness the HPA automatically multiply your pods in real time, run the following validation loop:

1. **Open a terminal window** to stream live cluster resource telemetry:
   ```bash
   kubectl get hpa -w
   ```

2. **Open a second terminal window** and launch the high-speed traffic generator pod:
   ```bash
   kubectl run cloud-traffic-stressor --image=alpine --restart=Never -- \
     sh -c "while true; do wget -q -O- http://baseline-web-service; done"
   ```

3. **Observe the Metrics & Observability Wave:**
   * Watch your monitoring terminal or your **Grafana Dashboard**. Within 45 seconds, the CPU utilization metric will spike past your designated `50%` threshold.
   * Watch **Argo CD** or your terminal display your single deployment spin up **4 active pod replicas** to safely load-balance the influx.

4. **Cool Down and Scale Back Down:**
   * Kill the traffic loop to simulate the traffic surge ending:
     ```bash
     kubectl delete pod cloud-traffic-stressor
     ```
   * The platform will safely process its 5-minute cool-down stabilization countdown before cleanly terminating the extra containers and returning to its single-pod baseline.


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

## Deployment & Teardown

## Prerequisites, Deployment & Telemetry Access

### Required Local Tools
Before starting, ensure you have the following installed on your local machine:
* **Minikube** & **Docker** (configured as the Minikube driver)
* **kubectl** (Kubernetes CLI)
* **Helm v3** (Kubernetes package manager)

---

### Quick Start (Full Rebuild from Scratch)
Because `minikube delete` wipes everything, follow these exact steps to rebuild the cluster, re-initialize your monitoring stack, and spin up your apps via GitOps:

```bash
# 1. Start a fresh local cluster with optimized resource limits and enable the metrics server
minikube start --cpus 2 --memory 4096 --driver docker
minikube addons enable metrics-server

# 2. Install Helm and deploy the Prometheus & Grafana Monitoring Stack
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install my-monitor prometheus-community/kube-prometheus-stack --namespace monitoring

# 3. Deploy the GitOps Engine (Argo CD)
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 4. Bootstrap your apps using the Root Application Controller
# Once applied, Argo CD pulls and syncs everything else from Git automatically
kubectl apply -f platform/root-application.yaml
```

---

### Accessing the Dashboards (Port-Forwarding & Login)

Kubernetes isolates your cluster services by default. To view your platforms in your web browser, open separate terminal windows and run these port-forward tunnels:

#### 1. Argo CD Dashboard
* **Port-Forward Command:**
  ```bash
  kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```
* **URL:** Go to `https://localhost:8080` (bypass the browser SSL warning).
* **Username:** `admin`
* **Password:** Retrieve your unique local password by running:
  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  ```

#### 2. Grafana Dashboards
* **Port-Forward Command:**
  ```bash
  kubectl port-forward svc/my-monitor-grafana 3000:80 --namespace monitoring
  ```
* **URL:** Go to `http://localhost:3000`
* **Username:** `admin`
* **Password:** Retrieve your unique local password by running:
  ```bash
  kubectl --namespace monitoring get secrets my-monitor-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
  ```

#### 3. Web Content Port-Forward
* **Port-Forward Command:**
  ```bash
  kubectl port-forward svc/baseline-web-service 8085:80
  ```
* **URL:** Go to `http://localhost:8085`

---

### Clean Teardown
To completely wipe the cluster and free up your computer's CPU and RAM:

```bash
# Deletes the entire cluster, metrics engine, and all Helm installations
minikube delete
```

