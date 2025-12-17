# Lab 7 - Provision a Google Kubernetes Engine Cluster using Terraform

#### Overview

In this lab, you will use Terraform to provision a Google Kubernetes Engine (GKE) cluster and a managed node pool. You will then fetch cluster credentials and verify access using kubectl.

***

### Learning objectives

1. Configure Terraform for Google Cloud
2. Create a GKE cluster using Terraform
3. Create a node pool for worker nodes
4. Fetch kubeconfig and validate cluster access
5. Destroy the infrastructure using Terraform

***

### Prerequisites

* Google Cloud account with a valid project
* Terraform installed locally
* Google Cloud CLI installed locally
* kubectl installed locally
* VS Code or similar IDE
* Permissions to create GKE and networking resources

***

### Step 0: Authenticate to Google Cloud

1. Initialize gcloud (one time if not done already)

```bash
gcloud init
```

2. Authenticate Terraform using Application Default Credentials

```bash
gcloud auth application-default login
```

3. Set your project (recommended)

```bash
gcloud config set project YOUR_PROJECT_ID
```

***

### Step 1: Enable required Google Cloud APIs

Run the following commands:

```bash
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com
```

Optional but commonly needed:

```bash
gcloud services enable iam.googleapis.com
```

***

### Step 2: Create a new Terraform project folder

```bash
mkdir gke_with_terraform
cd gke_with_terraform
```

Create the following files:

* providers.tf
* variables.tf
* network.tf
* gke.tf
* outputs.tf
* terraform.tfvars

***

### Step 3: Configure provider in `providers.tf`

Create `providers.tf`:

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "6.35.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

***

### Step 4: Define variables in `variables.tf`

Create `variables.tf`:

```hcl
variable "project_id" {
  type        = string
  description = "Google Cloud project ID"
}

variable "region" {
  type        = string
  description = "Region for regional resources"
  default     = "us-central1"
}

variable "zone" {
  type        = string
  description = "Zone for zonal resources and node placement"
  default     = "us-central1-a"
}

variable "network_name" {
  type        = string
  description = "VPC network name"
  default     = "gke-vpc"
}

variable "subnet_name" {
  type        = string
  description = "Subnet name"
  default     = "gke-subnet"
}

variable "subnet_cidr" {
  type        = string
  description = "Primary subnet CIDR"
  default     = "10.10.0.0/16"
}

variable "cluster_name" {
  type        = string
  description = "GKE cluster name"
  default     = "gke-cluster-1"
}

variable "node_pool_name" {
  type        = string
  description = "Node pool name"
  default     = "primary-pool"
}

variable "node_machine_type" {
  type        = string
  description = "Machine type for nodes"
  default     = "e2-medium"
}

variable "node_count" {
  type        = number
  description = "Number of nodes in the node pool"
  default     = 2
}
```

***

### Step 5: Create VPC and subnet in `network.tf`

Create `network.tf`:

```hcl
resource "google_compute_network" "vpc" {
  name                    = var.network_name
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "subnet" {
  name          = var.subnet_name
  region        = var.region
  network       = google_compute_network.vpc.id
  ip_cidr_range = var.subnet_cidr

  private_ip_google_access = true
}
```

***

### Step 6: Create the GKE cluster and node pool in `gke.tf`

Create `gke.tf`:

```hcl
resource "google_container_cluster" "gke" {
  name     = var.cluster_name
  location = var.region

  network    = google_compute_network.vpc.id
  subnetwork = google_compute_subnetwork.subnet.id

  remove_default_node_pool = true
  initial_node_count       = 1

  ip_allocation_policy {}

  release_channel {
    channel = "REGULAR"
  }
}

resource "google_container_node_pool" "primary" {
  name       = var.node_pool_name
  location   = var.region
  cluster    = google_container_cluster.gke.name
  node_count = var.node_count

  node_config {
    machine_type = var.node_machine_type

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}
```

Notes:

* This creates a regional GKE cluster.
* The default node pool is removed and replaced with a managed node pool you control.
* `ip_allocation_policy {}` enables VPC-native networking.

***

### Step 7: Add outputs in `outputs.tf`

Create `outputs.tf`:

```hcl
output "cluster_name" {
  value = google_container_cluster.gke.name
}

output "cluster_region" {
  value = var.region
}

output "network_name" {
  value = google_compute_network.vpc.name
}

output "subnet_name" {
  value = google_compute_subnetwork.subnet.name
}
```

***

### Step 8: Provide values in `terraform.tfvars`

Create `terraform.tfvars`:

```hcl
project_id = "YOUR_PROJECT_ID"
region     = "us-central1"
zone       = "us-central1-a"

cluster_name = "gke-saurabh"
node_count   = 2
```

***

### Step 9: Run the Terraform workflow

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

***

### Step 10: Fetch kubeconfig and connect to the cluster

1. Get credentials:

```bash
gcloud container clusters get-credentials gke-saurabh --region us-central1 --project YOUR_PROJECT_ID
```

2. Verify kubectl access:

```bash
kubectl get nodes
kubectl get namespaces
```

If you see nodes in Ready state, the cluster is working.

***

### Step 11: Optional verification by deploying a test app

```bash
kubectl create deployment nginx --image=nginx
kubectl get pods
```

Cleanup the test deployment:

```bash
kubectl delete deployment nginx
```

***

### Step 12: Destroy the infrastructure

```bash
terraform destroy --auto-approve
```

***

### Common troubleshooting

* Quota errors: reduce `node_count` or use a smaller `node_machine_type`
* API not enabled: re-run the `gcloud services enable` commands
* Credentials issue: re-run `gcloud auth application-default login`
* Wrong region in credentials command: use `--region` for regional clusters and `--zone` for zonal clusters

