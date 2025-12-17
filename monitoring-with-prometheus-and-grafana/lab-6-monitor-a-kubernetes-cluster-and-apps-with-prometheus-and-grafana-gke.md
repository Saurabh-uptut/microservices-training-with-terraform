# Lab 6: Monitor a Kubernetes Cluster and Apps with Prometheus and Grafana (GKE)

### Lab Overview

In this lab, you will set up **full-stack monitoring on Google Kubernetes Engine (GKE)**. You will deploy Prometheus and Grafana inside the cluster, collect metrics from Kubernetes control plane and nodes, and visualize both **cluster health** and **application performance**.

### Application Architecture

#### Stack overview

UI (NGINX static site)\
→ API (Node.js backend)\
→ Database (PostgreSQL)

#### Service exposure model

* PostgreSQL: ClusterIP (internal only)
* API: LoadBalancer (public)
* UI: LoadBalancer (public)

### What You Will Build

You will:

* Provision a **GKE cluster using Terraform**
* Install **Prometheus and Grafana using Helm**
* Deploy a **three-tier application**
* Build dashboards to monitor:
  * Kubernetes nodes, pods, CPU, memory
  * Application performance and pod behavior

### Prerequisites

You must have:

* Google Cloud account with billing enabled
* Terraform installed
* Helm installed
* kubectl installed
* gcloud CLI installed and authenticated
* Access to a running GKE cluster (created in this lab or existing)

## Part 1: Provision a GKE Cluster using Terraform

### Step 1: Clone the GKE Terraform repository

Clone or prepare your Terraform scripts for GKE.

Example:

```bash
git clone <GKE_TERRAFORM_REPO>
cd gke-terraform
```

Make sure the Terraform code provisions:

* GKE Standard cluster
* Node pool with at least 2 nodes
* Region or zone (example: us-central1)

### Step 2: Update terraform.tfvars

Example values:

```hcl
project_id = "your-gcp-project-id"
region     = "us-central1"
cluster_name = "devops-gke"
node_count = 2
```

### Step 3: Execute Terraform

```bash
terraform init
terraform plan
terraform apply --auto-approve
```

### Step 4: Configure kubectl for GKE

Terraform usually outputs cluster details, but you can always connect manually:

```bash
gcloud container clusters get-credentials devops-gke \
  --region us-central1 \
  --project your-gcp-project-id
```

Verify:

```bash
kubectl get nodes
```

***

## Part 2: Install Prometheus and Grafana using Helm

### Step 5: Create monitoring namespace

```bash
kubectl create namespace monitoring
```

***

### Step 6: Install Prometheus

#### Add Helm repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

#### Install Prometheus

```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring
```

#### Expose Prometheus (for lab visibility)

```bash
kubectl expose service prometheus-server \
  --namespace monitoring \
  --type=LoadBalancer \
  --target-port=9090 \
  --name=prometheus-server-ext
```

Get the external IP:

```bash
kubectl get svc -n monitoring
```

Open in browser:

```
http://<PROMETHEUS_EXTERNAL_IP>:9090
```

***

### Step 7: Install Grafana

#### Add Grafana Helm repo

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

#### Install Grafana

```bash
helm install grafana grafana/grafana \
  --namespace monitoring
```

#### Get Grafana admin password

```bash
kubectl get secret grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

#### Expose Grafana

```bash
kubectl expose service grafana \
  --namespace monitoring \
  --type=LoadBalancer \
  --target-port=3000 \
  --name=grafana-ext
```

Get the IP:

```bash
kubectl get svc -n monitoring
```

Open in browser:

```
http://<GRAFANA_EXTERNAL_IP>:3000
```

Login:

* Username: admin
* Password: retrieved above

***

## Part 3: Connect Prometheus to Grafana

### Step 8: Add Prometheus as Data Source

In Grafana UI:

1. Go to Connections → Data Sources
2. Add new data source
3. Select Prometheus
4. Set URL:

```
http://prometheus-server.monitoring.svc.cluster.local
```

5. Click Save and Test

***

## Part 4: Import Kubernetes Cluster Dashboard

### Step 9: Import Dashboard

Use your existing JSON dashboard:

* `k8s_cluster_monitoring.json`

In Grafana:

1. Go to Dashboards → Import
2. Upload JSON
3. Select Prometheus as data source
4. Click Import

You should now see:

* Node CPU and memory
* Pod status
* Namespace usage
* Workload health

***

## Part 5: Deploy Three-Tier Application on GKE

### Step 10: Deploy application resources

Clone your application repo:

```bash
git clone https://github.com/saurabhd2106/usermanagement-lab-ih.git
cd usermanagement-lab-ih/k8s_solution
```

Apply manifests in order:

```bash
kubectl apply -f namespace.yml
kubectl apply -f db-secret.yml
kubectl apply -f db-deploy.yml
kubectl apply -f db-svc.yml
kubectl apply -f api-deploy.yml
kubectl apply -f api-svc-lb.yml
```

Get backend LB IP:

```bash
kubectl get svc -n user-management
```

***

### Step 11: Update UI ConfigMap

Update `ui-configmap.yml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-configmap
  namespace: user-management
data:
  config.json: |
    {
      "API_URL": "http://<BACKEND_LB_IP>/"
    }
```

Apply UI resources:

```bash
kubectl apply -f ui-configmap.yml
kubectl apply -f ui-deploy.yml
kubectl apply -f ui-svc-lb.yml
```

Get frontend IP:

```bash
kubectl get svc -n user-management
```

Open in browser:

```
http://<FRONTEND_LB_IP>
```

***

## Part 6: Observe Application Metrics in Grafana

### Step 12: Verify monitoring

In Grafana:

* Open Kubernetes Cluster dashboard
* Navigate to Pods Monitoring section
* Confirm:
  * backend-deploy pods visible
  * frontend pods visible
  * resource usage updating

You now have:

* Cluster-level monitoring
* Workload-level visibility
* Application traffic reflected in dashboards

## Cleanup

To remove everything:

```bash
kubectl delete namespace user-management
kubectl delete namespace monitoring
```

(Optional)

```bash
terraform destroy --auto-approve
```
