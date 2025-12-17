# Lab 5 - Building Reusable Components in Terraform (GCP)

### Learning objectives

* Build reusable resources using **`for_each`**
* Build reusable parts of resources using **`dynamic` blocks**

### Prerequisite

Complete: **Lab - Provision a Google Compute Instance using Terraform** (VPC + one subnet + VM + firewall)

### Target architecture (updated)

* 1 VPC
* **Multiple subnets** in the same VPC
* **1 VM per subnet**
* 1 firewall rule with **multiple allow blocks** (dynamic)

## Folder structure (suggested)

```
.
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── network.tf
├── subnet.tf
├── firewall.tf
├── compute_instance.tf
├── outputs.tf
└── locals.tf
```

***

## Task 1 — Create multiple subnets (for\_each)

### 1) variables.tf

```hcl
variable "subnets" {
  type = map(object({
    ip_cidr_range = string
  }))
  description = "Subnets to create in the VPC"
}
```

### 2) terraform.tfvars

```hcl
subnets = {
  public-subnet = {
    ip_cidr_range = "10.2.0.0/16"
  }
  private-subnet = {
    ip_cidr_range = "10.3.0.0/16"
  }
}
```

### 3) subnet.tf

```hcl
resource "google_compute_subnetwork" "subnet" {
  for_each = var.subnets

  name          = each.key
  ip_cidr_range = each.value.ip_cidr_range
  region        = var.region
  network       = google_compute_network.vpc_network.id
}
```

#### Common validation error you’ll hit (and why)

If your old VM code still referenced:

```hcl
subnetwork = google_compute_subnetwork.subnet.id
```

Terraform will fail because `subnet` is now a **map of resources**, not a single resource.

#### Temporary fix&#x20;

Pick one key explicitly:

```hcl
subnetwork = google_compute_subnetwork.subnet["public-subnet"].id
```

Later (Task 3) we’ll do the **correct fix**: choose subnet per VM dynamically.

## Task 2 — Multiple firewall allow rules (dynamic block)

### 1) variables.tf

```hcl
variable "ingress" {
  type = list(object({
    protocol = string
    ports    = optional(list(string))
  }))
}
```

### 2) terraform.tfvars

```hcl
ingress = [
  {
    protocol = "tcp"
    ports    = ["22", "8080"]
  },
  {
    protocol = "icmp"
  }
]
```

### 3) firewall.tf

```hcl
resource "google_compute_firewall" "default" {
  name    = "test-firewall"
  network = google_compute_network.vpc_network.name

  dynamic "allow" {
    for_each = var.ingress
    content {
      protocol = allow.value.protocol
      ports    = try(allow.value.ports, null)
    }
  }

  target_tags = [local.vm_tags]
}
```

***

## Task 3 — Create multiple Compute Instances (for\_each + dynamic access\_config)

### Fix typo first

Your variable name is written as `compute_insance` (typo).\
You _can_ keep it, but it’s better to correct it now to avoid confusion.

#### variables.tf

```hcl
variable "compute_instances" {
  type = map(object({
    subnet       = string
    public_ip    = bool
    machine_type = string
    zone         = string
    image        = string
    labels       = map(string)
  }))
}
```

#### terraform.tfvars

```hcl
compute_instances = {
  public-vm-saurabh = {
    subnet       = "public-subnet"
    public_ip    = true
    machine_type = "n2d-standard-2"
    zone         = "us-central1-a"
    image        = "ubuntu-os-cloud/ubuntu-2004-lts"
    labels = {
      environment = "dev"
    }
  }

  private-vm-saurabh = {
    subnet       = "private-subnet"
    public_ip    = false
    machine_type = "n2d-standard-2"
    zone         = "us-central1-a"
    image        = "ubuntu-os-cloud/ubuntu-2004-lts"
    labels = {
      environment = "dev"
    }
  }
}
```

#### compute\_instance.tf

```hcl
resource "google_compute_instance" "compute" {
  for_each     = var.compute_instances
  name         = each.key
  machine_type = each.value.machine_type
  zone         = each.value.zone

  boot_disk {
    initialize_params {
      image = each.value.image
      labels = each.value.labels
    }
  }

  network_interface {
    network    = google_compute_network.vpc_network.id
    subnetwork = google_compute_subnetwork.subnet[each.value.subnet].id

    # Create public IP only when public_ip=true
    dynamic "access_config" {
      for_each = each.value.public_ip ? [1] : []
      content {}
    }
  }

  metadata = {
    ssh-keys = "${var.ssh_user}:${tls_private_key.ssh_key.public_key_openssh}"
  }

  tags = [local.vm_tags]
}
```

This is the **correct fix** for Task 1’s earlier “temporary fix”.

***

## Outputs — Multiple public & private IPs

Your `private ip` output had a small syntax issue (`instance.network_interface.0.network_ip`).

Use this:

### outputs.tf

```hcl
output "public_ip_addresses" {
  value = [
    for vm in values(google_compute_instance.compute) :
    try(vm.network_interface[0].access_config[0].nat_ip, null)
    if try(length(vm.network_interface[0].access_config), 0) > 0
  ]
}

output "private_ip_addresses" {
  value = [
    for vm in values(google_compute_instance.compute) :
    vm.network_interface[0].network_ip
  ]
}
```

***

## Required supporting vars/locals (if not already present)

### variables.tf (add if missing)

```hcl
variable "region" {
  type    = string
  default = "us-central1"
}

variable "ssh_user" {
  type    = string
  default = "ubuntu"
}
```

### locals.tf (add if missing)

```hcl
locals {
  vm_tags = "allow-ssh-http"
}
```

***

## Terraform workflow

Run these in order:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

***

### Expected outcome

* Two subnets created: `public-subnet`, `private-subnet`
* Two VMs created: one per subnet
* Firewall has **two allow rules** (tcp 22/8080 + icmp)
* Outputs show:
  * list of public IPs (only for public VM)
  * list of private IPs (for both)

