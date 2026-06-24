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

This file is maintained inside the `terraform-bootstrap/` directory of this repository. Because ArgoCD is explicitly configured to isolate its tracking path to the `apps/` directory, the Terraform infrastructure bootstrapping assets safely coexist within the same repository without interfering with the live GitOps continuous reconciliation loop.

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

Here is the updated Markdown snippet to add directly to the bottom of your `README.md`. It documents the cluster verification steps using both the fast Kubernetes Custom Resource layer and the native Elastic HTTP API.


## 📊 Verification: Checking Cluster & Node Health

Once the cluster has successfully reconciled, you can monitor and verify the health of the stateful Elasticsearch nodes using either the Kubernetes abstraction layer or by hitting the native Elastic cluster APIs directly.

### Method 1: The Fast Kubernetes Check
The ECK Operator continuously aggregates internal cluster health status directly up to the Custom Resource Definition (CRD) status block. View it instantly with:
```bash
kubectl get elasticsearch -n elastic-system

```

* **Expected Output:** The `HEALTH` column should read `green` or `yellow` (yellow is standard for single-node sandboxes because there are no secondary nodes to house index replicas), with `NODES` at `1/1`.

### Method 2: Querying the Native Elasticsearch HTTP API

For comprehensive structural cluster metrics, query the native `/_cluster/health` API endpoint directly from your host terminal.

#### 1. Establish the Network Bridge

Open a port-forwarding channel to expose the cluster's internal secure HTTPS service to your local host (keep this running):

```bash
kubectl port-forward svc/internal-search-es-http -n elastic-system 9200:9200

```

#### 2. Extract and Decode the Admin Credentials

In a separate terminal window, extract the automatically managed `elastic` user bootstrap password from its Kubernetes secret store:

```bash
PASSWORD=$(kubectl get secret internal-search-es-elastic-user -n elastic-system -o jsonpath="{.data.elastic}" | base64 -d)

```

#### 3. Execute the API Query

Run a `curl` request against your local socket. The `-k` flag tells curl to bypass verification of the operator's self-signed TLS certificates:

```bash
curl -k -u elastic:$PASSWORD "https://localhost:9200/_cluster/health?pretty"

```

#### Expected Telemetry Object:

```json
{
  "cluster_name" : "internal-search",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 1,
  "active_shards" : 1,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 1,
  "number_of_pending_tasks" : 0
}

```

*(Note: An internal status of `"yellow"` is the perfectly optimized target state for a single-node local environment. It simply indicates that primary data shards are active and healthy, but index replication loops are safely unassigned due to the lack of secondary hardware instances).*


## 🛑 Teardown: Pausing vs. Full SRE Clean Up

Depending on whether you want to save your progress for later or completely purge the local sandbox environment from your machine, choose one of the following teardown patterns:

### Option A: Pausing the Environment (Preserve State)

If you want to free up your host machine's CPU and RAM right now but want to keep your Elasticsearch data indices, cluster configurations, and ArgoCD states completely intact for next time:

```bash
minikube stop
```

* **To Resume Later:** Simply execute `minikube start`. You **do not** need to re-run your Terraform or bootstrap manifests. The persistent disk states will mount automatically, and the ECK and ArgoCD control loops will self-heal back to a running green status within a couple of minutes.

### Option B: The Full SRE Tear Down (Destructive Purge)

If you are completely finished with this sandbox iteration and want to cleanly wipe out every resource, helm release, data volume, and namespace from your local machine, execute this exact structural teardown sequence:

#### 1. Sever the GitOps Link First

Instruct ArgoCD to stop tracking your tracking repository. This gracefully deletes the root application controller and cascades down to clean up the Elasticsearch workloads and local storage volumes cleanly:

```bash
kubectl delete -f root-application.yaml
```

#### 2. Execute Terraform Infrastructure Destruction

Navigate to your isolated bootstrap directory and allow Terraform to cleanly unprovision the infrastructure orchestrators (ArgoCD and the ECK Operator Helm configurations) and system namespaces:

```bash
cd terraform-bootstrap/
terraform destroy -auto-approve
```

#### 3. Erase the Cluster Node State

Finally, completely destroy the local Minikube virtual machine/container engine profile to wipe out any remaining cache lines or host-path disk allocations:

```bash
minikube delete
```
