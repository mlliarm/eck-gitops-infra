# eck-gitops-infra

## Local Kubernetes GitOps Sandbox (ECK + ArgoCD + Terraform)

A declarative, production-patterned infrastructure platform blueprint that provisions a localized Kubernetes GitOps playground using **Terraform**, **ArgoCD**, and **Elastic Cloud on Kubernetes (ECK)** on a single-node **Minikube** cluster.

This repository demonstrates the platform engineering pattern of separating initial infrastructure bootstrapping (Push-based IaC via Terraform) from continuous application lifecycle management (Pull-based GitOps via ArgoCD).

---

## 🏗️ Architecture Design


```

```
                 [ Push-Based Infrastructure Setup ]
                                  │
                     (1) Run: terraform apply
                                  ▼
           ┌──────────────────────────────────────────────┐
           │               MINIKUBE CLUSTER               │
           │                                              │
           │  ┌──────────────────┐    ┌────────────────┐  │
           │  │ ArgoCD Namespace │    │ ECK Operator   │  │
           │  └────────┬─────────┘    └────────┬───────┘  │
           └───────────┼───────────────────────┼──────────┘
                       │                       │
                       │ (2) Continuously      │ (3) Manages
                       │     Polls & Syncs     │     Lifecycle
                       ▼                       ▼
                 ┌───────────┐           ┌───────────┐
                 │ Public    │           │ State     │
                 │ Git Repo  │           │ Resources │
                 └───────────┘           └───────────┘
                 [ Pull-Based Continuous Reconciliation ]

```

```

### 1. The Infrastructure Foundation (Terraform)
Local machine configurations target a single-node Minikube instance using the native `hashicorp/kubernetes` and `hashicorp/helm` providers. Terraform initializes the core platform requirements, provisions the system namespaces, and injects the baseline orchestrators (`argo-cd` and `eck-operator` via official Helm charts) into the cluster, then hands off control entirely.

### 2. The Application Lifecycle Loop (ArgoCD & ECK)
An ArgoCD root Application controller registers and watches the target repository paths over the public internet. When state modifications are pushed to the tracking branches, ArgoCD triggers an automated reconciliation cycle. It applies the declarations directly to the API server, instructing the ECK Operator to coordinate stateful workload rollouts (such as our resource-managed single-node Elasticsearch cluster).

---

## 📁 Repository Structure

```text
.
├── README.md                     # Comprehensive platform documentation
├── root-application.yaml         # ArgoCD App-of-Apps parent declaration
├── terraform-bootstrap/
│   └── main.tf                   # Terraform infrastructure blueprint
└── apps/
    └── elasticsearch/
        └── cluster.yaml          # GitOps target Elasticsearch cluster manifest

```

---

## 💻 Configuration Blueprints

### 1. Terraform Infrastructure Provisioning (`terraform-bootstrap/main.tf`)

This file must be kept in a local folder **outside** of the tracking paths used by ArgoCD to keep your infra bootstrapping distinct from your GitOps continuous reconciliation loop.

```hcl
terraform {
  required_version = ">= 1.0.0"
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.24"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "minikube"
}

provider "helm" {
  kubernetes {
    config_path    = "~/.kube/config"
    config_context = "minikube"
  }
}

# 1. Create dedicated namespace for Elastic workloads
resource "kubernetes_namespace" "elastic_system" {
  metadata {
    name = "elastic-system"
  }
}

# 2. Deploy the Elastic Cloud on Kubernetes (ECK) Operator
resource "helm_release" "eck_operator" {
  name       = "eck-operator"
  repository = "[https://helm.elastic.co](https://helm.elastic.co)"
  chart      = "eck-operator"
  namespace  = kubernetes_namespace.elastic_system.metadata[0].name
}

# 3. Deploy ArgoCD via official Helm Chart (Standard Profile)
resource "helm_release" "argocd" {
  name             = "argocd"
  repository       = "[https://argoproj.github.io/argo-helm](https://argoproj.github.io/argo-helm)"
  chart            = "argo-cd"
  namespace        = "argocd"
  create_namespace = true
}

```

### 2. ArgoCD Root Application Link (`root-application.yaml`)

Apply this file locally via `kubectl` to instruct ArgoCD to lock onto your repository layout.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: elasticstack-platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: '[https://github.com/YOUR_USERNAME/eck-gitops-infra.git](https://github.com/YOUR_USERNAME/eck-gitops-infra.git)' # Replace with your public GitHub URL
    targetRevision: HEAD
    path: 'apps/elasticsearch'
  destination:
    server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)'
    namespace: elastic-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```

### 3. GitOps Target Cluster Manifest (`apps/elasticsearch/cluster.yaml`)

Place this configuration file inside your tracking repository. The formatting separates JVM heap configurations from structural container limit allocations to bypass runtime bootstrap errors.

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: internal-search
  namespace: elastic-system
spec:
  version: 9.0.0
  nodeSets:
  - name: master-data
    count: 1
    config:
      node.roles: ["master", "data", "ingest"]
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          env:
          - name: ES_JAVA_OPTS
            value: "-Xms1536m -Xmx1536m"
          resources:
            requests:
              memory: 2Gi
              cpu: "1"
            limits:
              memory: 2.5Gi
              cpu: "1.5"

```

---

## 🚀 Execution & Operational Steps

### Step 1: Tune Host Virtual Memory Kernels

Elasticsearch utilizes Lucene memory mappings extensively. Set the system limits on your host operating system before spinning up Minikube:

```bash
sudo sysctl -w vm.max_map_count=262144

```

### Step 2: Provision Minikube Resource Pools

Launch the local cluster with an optimized resource configuration profile tailored to comfortably run the control loop and the data nodes:

```bash
minikube start --cpus=3 --memory=5632 --driver=docker

```

### Step 3: Run the Infrastructure Bootstrap

Navigate into your local bootstrap directory, initialize the providers, and execute the execution block:

```bash
cd terraform-bootstrap/
terraform init
terraform apply -auto-approve

```

### Step 4: Hook Up the GitOps Automation Loop

With the operators active, execute the root bridge configuration to map the platform back to your code definitions:

```bash
cd ..
kubectl apply -f root-application.yaml

```

### Step 5: Extract Access Secrets & Open Dashboards

Fetch the automatically generated cluster administrator authentication details from your cluster state:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

```

Forward the UI service traffic lines directly to your host machine:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443

```

Open `https://localhost:8080` in your browser, connect using user `admin`, and watch your architecture render live!

---

## 🛡️ Day-2 SRE Drill: Drift Detection & Automated Healing

To verify that the continuous delivery engine is actively tracking and protecting your desired state configuration, execute an intentional runtime state modification by manually deleting a structural cluster resource via the command line:

```bash
kubectl delete service internal-search-es-http -n elastic-system

```

### Reaching Reconciled State:

1. ArgoCD's background application loops flag a drift divergence instantly against the tracking commit.
2. The controller blocks out the out-of-band mutation and forcefully regenerates the deleted configuration objects.
3. The cluster automatically self-heals back to the immutable configuration state defined within your Git repository.
