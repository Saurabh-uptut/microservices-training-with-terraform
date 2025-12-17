# Lab 6 - Troubleshooting Terraform Scripts (Google Cloud)

In this lab you’ll troubleshoot common Terraform errors by **reading error messages**, identifying the root cause, and applying correct fixes.

### Prerequisites

* Google Cloud account with valid credentials
* Terraform installed locally
* `gcloud` CLI installed locally
* VS Code (or similar IDE)

### Learning objectives

* Diagnose Terraform failures using error messages
* Fix common issues: **syntax errors**, **cyclic dependencies**, **invalid expressions**, and **for\_each/output mistakes**

***

### Setup

1. Clone the repository: [**Troubleshooting Scenarios (Google Cloud)**](https://github.com/saurabhd2106/terraform-troubleshoot-google.git)
2.  Switch to the project directory:

    ```bash
    cd <name-of-project>
    ```

***

## Scenario 1: Syntax Errors

### Step 1 — Run formatting

```bash
terraform fmt
```

#### Typical error message

Terraform reports a syntax issue, e.g.:

* file: `compute_instance.tf`
* line: `64`

#### Root cause

Wrong string interpolation syntax (missing quotes).

Correct syntax:

```hcl
name = "${var.vm_name}-learn"
```

Re-run:

```bash
terraform fmt
```

Now initialize the workspace:

```bash
terraform init
```

***

## Scenario 2: Cyclic Dependency Error

### Step 2 - Validate configuration

```bash
terraform validate
```

#### Error meaning

A **cycle** exists: Terraform can’t decide which resource to create first because **Resource A depends on Resource B and vice-versa**.

#### Where it happens

In this case, it’s inside firewall rules where one firewall references another firewall output/input (e.g. `source_ranges` referencing itself or another dependent attribute).

#### Fix

Break the cycle by using a static, valid value for `source_ranges`:

Updated firewall rules:

```hcl
resource "google_compute_firewall" "icmp" {
  name    = "allow-icmp"
  network = google_compute_network.vpc.name

  allow {
    protocol = "icmp"
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = [local.ssh_tag]
}

resource "google_compute_firewall" "tcp_8080" {
  name    = "allow-tcp-8080"
  network = google_compute_network.vpc.name

  allow {
    protocol = "tcp"
    ports    = ["8080"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = [local.http_tag]
}
```

Run:

```bash
terraform validate
```

You should see the cyclic error is gone. New errors may appear—fix them next.

## Scenario 3: Invalid Expression Error (for\_each)

Terraform points to something like:

* `compute_instance.tf` line `62`

#### Problem

The code is trying to use `for_each` with a value that is **not a map/set**.

Best fix: build a map in `locals` for subnet IDs.

### Step 3 — Add locals

```hcl
locals {
  subnets = {
    public  = google_compute_subnetwork.public.id
    private = google_compute_subnetwork.private.id
  }
}
```

### Step 4 - Update compute instances to use for\_each

```hcl
resource "google_compute_instance" "vm" {
  for_each = local.subnets

  name         = "${var.vm_name}-${each.key}"
  machine_type = var.machine_type
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = var.image
    }
  }

  network_interface {
    subnetwork = each.value

    # Give external IP only to "public" VM
    dynamic "access_config" {
      for_each = each.key == "public" ? [1] : []
      content {}
    }
  }

  metadata = {
    ssh-keys = "${var.admin_username}:${tls_private_key.ssh.public_key_openssh}"
  }

  tags = [
    local.ssh_tag,
    local.http_tag,
  ]
}
```

#### Why this is better than the draft

* `each.id` is **invalid** → should be `each.value`
* If you want **only public VM to get public IP**, use a **dynamic access\_config** block

Run:

```bash
terraform validate
```

## Scenario 4: Output errors with for\_each resources

If VMs are created with `for_each`, `google_compute_instance.vm` behaves like a **map**. You must iterate using a **for expression**.

### Step 5 - Fix outputs (output.tf)

Public IPs:

```hcl
output "public_ip_address" {
  value = [
    for instance in values(google_compute_instance.vm) :
    try(instance.network_interface[0].access_config[0].nat_ip, null)
    if try(length(instance.network_interface[0].access_config), 0) > 0
  ]
}
```

Private IPs (fixes the `.0` bug):

```hcl
output "private_ip_address" {
  value = [
    for instance in values(google_compute_instance.vm) :
    instance.network_interface[0].network_ip
  ]
}
```

Run:

```bash
terraform validate
```

At this point, validation should pass.

## Run full Terraform workflow

```bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply --auto-approve
```
