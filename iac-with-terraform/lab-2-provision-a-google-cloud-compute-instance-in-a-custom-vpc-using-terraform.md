# Lab 2 - Provision a Google Cloud Compute Instance in a Custom VPC using Terraform

### Overview

In this lab, you will use **Terraform** to provision a **Google Compute Engine VM** inside a **custom Virtual Private Cloud (VPC)**.\
You will also configure **subnets, firewall rules, SSH access**, and deploy a **Linux VM with a public IP**.

This lab mimics a real-world cloud setup where VMs are deployed in **private networks with controlled access**.

### Learning Objectives

By the end of this lab, you will be able to:

1. Write and execute Terraform scripts for **Google Cloud infrastructure**
2. Create and configure a **custom VPC**
3. Create and associate **subnets** with a VPC
4. Define **firewall rules** to control inbound traffic
5. Deploy a **Linux Compute Engine VM** with a public IP
6. Generate and use **SSH keys** via Terraform
7. Apply the complete **Terraform workflow**
8. Destroy cloud resources cleanly using Terraform

### Pre-Requisites

* Google Cloud account with a valid project
* Terraform installed locally (`terraform -version`)
* Google Cloud CLI installed (`gcloud version`)
* VS Code (or any IDE)
* Permissions to create:
  * VPCs
  * Subnets
  * Firewall rules
  * Compute Engine instances

### Authentication (One-Time Setup)

```bash
gcloud init
gcloud auth application-default login
```

Verify project:

```bash
gcloud config get-value project
```

### Deployment Architecture (Logical Flow)

<figure><img src="../.gitbook/assets/unknown (30).png" alt=""><figcaption></figcaption></figure>

```
Custom VPC
 └── Subnet (10.2.0.0/16)
     └── Compute Engine VM
         ├── Public IP
         ├── Firewall (SSH + ICMP)
         └── SSH Key Authentication
```

### Step 1 - Create Project Directory

```bash
mkdir compute_with_networking
cd compute_with_networking
```

Open this folder in VS Code.

### Step 2 - Provider Configuration (`providers.tf`)

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
  project = "terraform-training-422512"
  region  = "us-central1"
}
```

Update:

* `project` → your GCP project ID
* `region` → your preferred region

### Step 3 - Initialize Terraform

```bash
terraform init
```

### Step 4 - Define Variables (`variables.tf`)

```hcl
variable "vpc_network_name" {
  type = string
}

variable "compute_name" {
  type        = string
  description = "Name of the compute instance"
}

variable "machine_type" {
  type        = string
  description = "GCE machine type"
  default     = "n2d-standard-2"
}

variable "zone" {
  type        = string
  description = "GCE zone"
}

variable "image" {
  type = string
}

variable "labels" {
  type = any
}

variable "ssh_user" {
  type = string
}
```

### Step 5 - Create Custom VPC (`vpc.tf`)

```hcl
resource "google_compute_network" "vpc_network" {
  name                    = var.vpc_network_name
  auto_create_subnetworks = false
}
```

Disables default subnets (best practice)

### Step 6 - Create Subnet (`subnet.tf`)

```hcl
resource "google_compute_subnetwork" "subnet" {
  name          = "test-subnetwork"
  ip_cidr_range = "10.2.0.0/16"
  region        = "us-central1"
  network       = google_compute_network.vpc_network.id
}
```

### Step 7 - Define Local Values (`locals.tf`)

```hcl
locals {
  vm_tags = "allow-ssh-icmp"
}
```

Used to attach firewall rules to VMs.

***

### Step 8 - Create Firewall Rules (`firewall.tf`)

```hcl
resource "google_compute_firewall" "default" {
  name    = "test-firewall"
  network = google_compute_network.vpc_network.name

  allow {
    protocol = "icmp"
  }

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_tags = [local.vm_tags]
}
```

Allows:

* ICMP (ping)
* SSH (port 22)

### Step 9 - Generate SSH Keys (`ssh_key.tf`)

```hcl
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

output "tls_private_key" {
  value     = tls_private_key.ssh_key.private_key_pem
  sensitive = true
}
```

### Step 10 - Re-Initialize Terraform

```bash
terraform init
```

(Needed because of the new TLS provider)

### Step 11 - Create Compute Instance (`compute_instance.tf`)

```hcl
resource "google_compute_instance" "compute" {
  name         = var.compute_name
  machine_type = var.machine_type
  zone         = var.zone

  boot_disk {
    initialize_params {
      image  = var.image
      labels = var.labels
    }
  }

  network_interface {
    network    = google_compute_network.vpc_network.id
    subnetwork = google_compute_subnetwork.subnet.id

    access_config {}
  }

  metadata = {
    ssh-keys = "${var.ssh_user}:${tls_private_key.ssh_key.public_key_openssh}"
  }

  tags = [local.vm_tags]
}
```

### Step 12 - Provide Variable Values (`terraform.tfvars`)

```hcl
vpc_network_name = "vpc-saurabh"
compute_name     = "compute-saurabh"
machine_type     = "n2d-standard-2"
zone             = "us-central1-a"
image            = "ubuntu-os-cloud/ubuntu-2204-lts"

labels = {
  environment = "dev"
}

ssh_user = "googleuser"
```

### Step 13 - Terraform Workflow

```bash
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

### Step 14 - Export SSH Key and Connect

```bash
terraform output tls_private_key > key.pem
chmod 400 key.pem
```

Find the VM’s **public IP** from:

* Terraform output
* GCP Console → Compute Engine → VM Instances

SSH into the VM:

```bash
ssh -i key.pem googleuser@<PUBLIC_IP>
```

### Step 15 - Verify in Google Cloud Console

* VPC → Custom VPC exists
* Subnet attached to VPC
* Firewall rule applied
* VM running with external IP

### Cleanup (Destroy Everything)

```bash
terraform destroy --auto-approve
```
