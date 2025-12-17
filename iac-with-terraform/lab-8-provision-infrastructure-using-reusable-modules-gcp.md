# Lab 8 - Provision Infrastructure using Reusable Modules (GCP)

### Goal architecture

* VPC
* 2 subnets: `public-subnet`, `private-subnet`
* Firewall: allow **tcp 22** + **tcp 80**
* 2 VMs: one per subnet
  * public VM has public IP
  * private VM has no public IP

### 1) Create module folders

```bash
mkdir -p Google/google_compute_instance \
         Google/google_firewall \
         Google/google_subnet \
         Google/google_vpc \
         infra
```

## MODULE 1: VPC (`Google/google_vpc`)

#### `variables.tf`

```hcl
variable "vpc_network_name" {
  type = string
}

variable "auto_create_subnetworks" {
  type = bool
}
```

#### `vpc.tf`

```hcl
resource "google_compute_network" "vpc_network" {
  name                    = var.vpc_network_name
  auto_create_subnetworks = var.auto_create_subnetworks
}
```

#### `outputs.tf`

```hcl
output "id" {
  value = google_compute_network.vpc_network.id
}

output "name" {
  value = google_compute_network.vpc_network.name
}
```

***

## MODULE 2: Subnet (`Google/google_subnet`)

#### `variables.tf`

```hcl
variable "name" { type = string }
variable "ip_cidr_range" { type = string }
variable "region" { type = string }
variable "vpc_network_id" { type = string }
```

#### `subnet.tf`

```hcl
resource "google_compute_subnetwork" "subnet" {
  name          = var.name
  ip_cidr_range = var.ip_cidr_range
  region        = var.region
  network       = var.vpc_network_id
}
```

#### `outputs.tf`

```hcl
output "id" {
  value = google_compute_subnetwork.subnet.id
}

output "name" {
  value = google_compute_subnetwork.subnet.name
}
```

## MODULE 3: Firewall (`Google/google_firewall`) — supports multiple allow blocks

#### `variables.tf`

```hcl
variable "name" { type = string }
variable "network_name" { type = string }
variable "target_tags" { type = list(string) }
variable "source_ranges" { type = list(string) }

variable "ingress" {
  type = list(object({
    protocol = string
    ports    = optional(list(string))
  }))
}
```

#### `firewall.tf`

```hcl
resource "google_compute_firewall" "allow_traffic" {
  name    = var.name
  network = var.network_name

  dynamic "allow" {
    for_each = var.ingress
    content {
      protocol = allow.value.protocol
      ports    = try(allow.value.ports, null)
    }
  }

  source_ranges = var.source_ranges
  target_tags   = var.target_tags
}
```

#### `outputs.tf`

```hcl
output "name" {
  value = google_compute_firewall.allow_traffic.name
}
```

## MODULE 4: Compute Instance (`Google/google_compute_instance`) — supports optional public IP

#### `variables.tf`

```hcl
variable "compute_name" { type = string }
variable "machine_type" { type = string }
variable "zone" { type = string }
variable "image" { type = string }
variable "labels" { type = map(string) }

variable "network_id" { type = string }
variable "subnet_id" { type = string }

variable "ssh_user" { type = string }
variable "public_key_openssh" { type = string }

variable "tags" { type = list(string) }

variable "public_ip" {
  type        = bool
  description = "Attach public IP if true"
}
```

#### `compute_instance.tf`

```hcl
resource "google_compute_instance" "compute" {
  name         = var.compute_name
  machine_type = var.machine_type
  zone         = var.zone
  labels       = var.labels
  tags         = var.tags

  boot_disk {
    initialize_params {
      image = var.image
    }
  }

  network_interface {
    network    = var.network_id
    subnetwork = var.subnet_id

    dynamic "access_config" {
      for_each = var.public_ip ? [1] : []
      content {}
    }
  }

  metadata = {
    ssh-keys = "${var.ssh_user}:${var.public_key_openssh}"
  }
}
```

#### `outputs.tf`

```hcl
output "name" {
  value = google_compute_instance.compute.name
}

output "private_ip" {
  value = google_compute_instance.compute.network_interface[0].network_ip
}

output "public_ip" {
  value = try(google_compute_instance.compute.network_interface[0].access_config[0].nat_ip, null)
}
```

## INFRA PROJECT (`infra/`)

### `providers.tf`

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

  backend "gcs" {
    bucket = "my-tf-state-bkt202527may"
    prefix = "infra-modules-lab"
  }
}

provider "google" {
  project = "terraform-training-422512"
  region  = "us-central1"
}
```

### `ssh_key.tf`

```hcl
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

output "tls_private_key_pem" {
  value     = tls_private_key.ssh_key.private_key_pem
  sensitive = true
}
```

### `locals.tf`

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

### `main.tf`

```hcl
module "vpc_network" {
  source                  = "../Google/google_vpc"
  vpc_network_name         = "vpc-saurabh"
  auto_create_subnetworks  = false
}

module "subnet" {
  for_each = local.subnets
  source   = "../Google/google_subnet"

  name           = each.key
  ip_cidr_range  = each.value.ip_cidr_range
  region         = "us-central1"
  vpc_network_id = module.vpc_network.id
}

module "firewall" {
  source       = "../Google/google_firewall"
  name         = "allow-ssh-http"
  network_name = module.vpc_network.name

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["allow-ssh"]

  ingress = [
    { protocol = "tcp", ports = ["22"] },
    { protocol = "tcp", ports = ["80"] }
  ]
}

module "compute_instance" {
  for_each = local.compute_instances
  source   = "../Google/google_compute_instance"

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

### `outputs.tf`

```hcl
output "public_ips" {
  value = {
    for k, m in module.compute_instance :
    k => m.public_ip
  }
}

output "private_ips" {
  value = {
    for k, m in module.compute_instance :
    k => m.private_ip
  }
}
```

## Run Terraform workflow

From inside `infra/`:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

### What students should observe

* Modules can be reused across projects
* `for_each` at the module level creates multiple subnets/VMs cleanly
* `dynamic` blocks make modules flexible (firewall rules, optional public IP)
* Outputs become much easier to consume when returned as a **map keyed by VM name**

