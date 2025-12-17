# Lab 10 - Publish Terraform Modules as Git Submodules for Easy Distribution

> **NOTE**\
> This lab continues from **“Provision Infrastructure using Reusable Modules”**. Complete that lab first.

### Lab Overview

You will:

* Publish Terraform modules to GitHub
* Consume those modules using **Git submodules**
* Use those modules in a Terraform project

### What is a Git Submodule?

A **Git submodule** is a Git repository embedded inside another Git repository. It lets you:

* Reuse shared Terraform modules
* Lock the consumer project to a specific module commit
* Update modules independently and intentionally

***

## TASK 1: Publish Terraform Modules Repository

### Step 1: Create a GitHub repository

Create a repo (public/private), e.g. `terraform-gcp-modules`, and copy the URL.

### Step 2: Push your modules from the `Google/` folder

```bash
cd Google

git config --global user.name "<your_name>"
git config --global user.email "<your_email>"

git init
git add .
git commit -m "Initial GCP Terraform modules"
git branch -M main
git remote add origin <your_repository_url>
git push -u origin main
```

Your modules are now available in GitHub.

***

## TASK 2: Use Modules via Git Submodule in a New Project

### Step 1: Create a consumer project

```bash
mkdir -p module_example/example01
cd module_example/example01
git init
```

### Step 2: Add the modules repo as a submodule

```bash
git submodule add <terraform-modules-repo-url> Google
git submodule status
```

This creates:

* `Google/` directory (submodule code)
* `.gitmodules`

***

### Important: Cloning later (students often miss this)

If someone clones `module_example` later, they must do **either**:

```bash
git clone --recurse-submodules <consumer-repo-url>
```

OR after cloning:

```bash
git submodule update --init --recursive
```

***

## Project Structure

```
module_example/
└── example01/
    ├── Google/      # Git submodule
    ├── main.tf
    ├── locals.tf
    ├── providers.tf
    ├── ssh_key.tf
```

***

## Terraform Configuration

### providers.tf

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "6.35.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "4.0.6"
    }
  }
}

provider "google" {
  project = "terraform-training-422512"
  region  = "us-central1"
}
```

***

### ssh\_key.tf

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

***

### locals.tf

```hcl
locals {
  subnets = {
    public-subnet = { ip_cidr_range = "10.2.0.0/16" }
    private-subnet = { ip_cidr_range = "10.3.0.0/16" }
  }

  compute_instances = {
    public-vm = {
      subnet       = "public-subnet"
      public_ip    = true
      machine_type = "n2d-standard-2"
      zone         = "us-central1-a"
      image        = "ubuntu-os-cloud/ubuntu-2004-lts"
      ssh_user     = "googleuser"
      labels       = { environment = "dev" }
      tags         = ["allow-ssh"]
    }

    private-vm = {
      subnet       = "private-subnet"
      public_ip    = false
      machine_type = "n2d-standard-2"
      zone         = "us-central1-a"
      image        = "ubuntu-os-cloud/ubuntu-2004-lts"
      ssh_user     = "googleuser"
      labels       = { environment = "dev" }
      tags         = ["allow-ssh"]
    }
  }
}
```

***

### main.tf

```hcl
module "vpc_network" {
  source = "./Google/google_vpc"

  vpc_network_name        = "vpc-saurabh"
  auto_create_subnetworks = false
}

module "subnet" {
  for_each = local.subnets
  source   = "./Google/google_subnet"

  name           = each.key
  ip_cidr_range  = each.value.ip_cidr_range
  region         = "us-central1"
  vpc_network_id = module.vpc_network.id
}

module "firewall" {
  source = "./Google/google_firewall"

  name          = "allow-ssh-http"
  network_name  = module.vpc_network.name
  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["allow-ssh"]

  ingress = [
    { protocol = "tcp", ports = ["22"] },
    { protocol = "tcp", ports = ["80"] }
  ]
}

module "compute_instance" {
  for_each = local.compute_instances
  source   = "./Google/google_compute_instance"

  compute_name = each.key
  machine_type = each.value.machine_type
  zone         = each.value.zone
  image        = each.value.image
  labels       = each.value.labels

  network_id = module.vpc_network.id
  subnet_id  = module.subnet[each.value.subnet].id

  ssh_user           = each.value.ssh_user
  public_key_openssh = tls_private_key.ssh_key.public_key_openssh
  tags               = each.value.tags
  public_ip          = each.value.public_ip
}
```

***

## Provision Infrastructure

```bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply --auto-approve
```

***

## Destroy Infrastructure

```bash
terraform destroy --auto-approve
```

***

## Updating the submodule (correct workflow)

#### Option A: Update to latest commit on main

```bash
cd Google
git checkout main
git pull origin main
cd ..

git add Google
git commit -m "Bump terraform modules submodule"
git push
```

#### Option B: Pin to a specific commit/tag (recommended for enterprise)

```bash
cd Google
git fetch --tags
git checkout v1.0.0   # or a specific commit SHA
cd ..

git add Google
git commit -m "Pin modules to v1.0.0"
git push
```

***

### When to Use Git Submodules vs Terraform Registry

* **Git submodules** → internal modules, strict pinning, controlled upgrades
* **Terraform Registry** → easier consumption, nicer UX, semantic versioning

