# Lab 4 - Practice count and for\_each Meta-Arguments in Terraform

### Overview

In this lab, you will learn how to create **multiple Google Compute Engine instances** using a **single Terraform resource block**.\
Terraform provides two meta-arguments for this purpose: `count` and `for_each`.

You will implement three scenarios:

1. Create multiple resources using a fixed `count`
2. Create multiple resources using `count` with a list
3. Create multiple resources using `for_each`

### Learning Objectives

By the end of this lab, you will be able to:

1. Create multiple resources using Terraform meta-arguments
2. Understand how the `count` meta-argument works and when to use it
3. Understand how the `for_each` meta-argument works and when to use it
4. Compare `count` vs `for_each` in real-world scenarios

***

### Pre-Requisites

* Google Cloud account with valid credentials
* Terraform installed locally
* Google Cloud CLI installed locally
* VS Code or any similar IDE
* Permissions to create Compute Engine instances

***

## PART 1: Using `count` with a Fixed Number

### Task 1: Provision Multiple Compute Instances Using `count`

#### Step 1: Create Project Directory

```bash
mkdir count_example
cd count_example
```

***

#### Step 2: Provider Configuration (`providers.tf`)

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
  project = "terraform-training-422512"
  region  = "us-central1"
}
```

***

#### Step 3: Define Variables (`variables.tf`)

Create `variables.tf`:

```hcl
variable "compute_name" {
  type        = string
  description = "Base name of the compute instance"
}

variable "machine_type" {
  type        = string
  description = "Machine type for the VM"
  default     = "n2d-standard-2"
}

variable "zone" {
  type        = string
  description = "Zone where the VM will be created"
}

variable "image" {
  type = string
}

variable "labels" {
  type = any
}

variable "network" {
  type = string
}
```

***

#### Step 4: Create Compute Instances (`compute_instance.tf`)

Create `compute_instance.tf`:

```hcl
resource "google_compute_instance" "compute" {
  count = 3

  name         = "${var.compute_name}-${count.index}"
  machine_type = var.machine_type
  zone         = var.zone

  boot_disk {
    initialize_params {
      image  = var.image
      labels = var.labels
    }
  }

  network_interface {
    network = var.network
    access_config {}
  }
}
```

Explanation:

* `count = 3` creates three VM instances
* `count.index` generates unique names such as compute-saurabh-0, compute-saurabh-1, compute-saurabh-2

***

#### Step 5: Provide Values (`terraform.tfvars`)

Create `terraform.tfvars`:

```hcl
compute_name = "compute-saurabh"
machine_type = "n2d-standard-2"
zone         = "us-central1-a"
image        = "ubuntu-os-cloud/ubuntu-2004-lts"

labels = {
  environment = "dev"
}

network = "default"
```

***

#### Step 6: Terraform Workflow

Run the following commands in order:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

Verify the VMs in Google Cloud Console under Compute Engine.

***

#### Step 7: Destroy Resources

```bash
terraform destroy --auto-approve
```

***

## PART 2: Using `count` with a List

### Task 2: Provision Multiple Compute Instances by Iterating Over a List

#### Step 1: Create Project Directory

```bash
mkdir count_example_2
cd count_example_2
```

***

#### Step 2: Provider Configuration (`providers.tf`)

Same as Part 1.

***

#### Step 3: Define Variables (`variables.tf`)

```hcl
variable "compute_name" {
  type        = list(string)
  description = "List of compute instance names"
}

variable "machine_type" {
  type        = string
  default     = "n2d-standard-2"
}

variable "zone" {
  type = string
}

variable "image" {
  type = string
}

variable "labels" {
  type = any
}

variable "network" {
  type = string
}
```

***

#### Step 4: Compute Instance Definition (`compute_instance.tf`)

```hcl
resource "google_compute_instance" "compute" {
  count = length(var.compute_name)

  name         = var.compute_name[count.index]
  machine_type = var.machine_type
  zone         = var.zone

  boot_disk {
    initialize_params {
      image  = var.image
      labels = var.labels
    }
  }

  network_interface {
    network = var.network
    access_config {}
  }
}
```

***

#### Step 5: Variable Values (`terraform.tfvars`)

```hcl
compute_name = ["compute-saurabh-1", "compute-saurabh-2"]
machine_type = "n2d-standard-2"
zone         = "us-central1-a"
image        = "ubuntu-os-cloud/ubuntu-2004-lts"

labels = {
  environment = "dev"
}

network = "default"
```

***

#### Step 6: Terraform Workflow

```bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply --auto-approve
```

***

#### Step 7: Destroy Resources

```bash
terraform destroy --auto-approve
```

***

## PART 3: Using `for_each`

### Task 3: Provision Multiple Compute Instances Using `for_each`

#### Step 1: Create Project Directory

```bash
mkdir for_each_example
cd for_each_example
```

***

#### Step 2: Provider Configuration (`providers.tf`)

Same as earlier examples.

***

#### Step 3: Define Variables (`variables.tf`)

```hcl
variable "compute_name" {
  type        = set(string)
  description = "Set of compute instance names"
}

variable "machine_type" {
  type        = string
  default     = "n2d-standard-2"
}

variable "zone" {
  type = string
}

variable "image" {
  type = string
}

variable "labels" {
  type = any
}

variable "network" {
  type = string
}
```

***

#### Step 4: Compute Instance Definition (`compute_instance.tf`)

```hcl
resource "google_compute_instance" "compute" {
  for_each = var.compute_name

  name         = each.value
  machine_type = var.machine_type
  zone         = var.zone

  boot_disk {
    initialize_params {
      image  = var.image
      labels = var.labels
    }
  }

  network_interface {
    network = var.network
    access_config {}
  }
}
```

Explanation:

* `for_each` creates one resource per value in the set
* Each VM is tracked individually by name

***

#### Step 5: Variable Values (`terraform.tfvars`)

```hcl
compute_name = ["compute-saurabh-1", "compute-saurabh-2"]
machine_type = "n2d-standard-2"
zone         = "us-central1-a"
image        = "ubuntu-os-cloud/ubuntu-2004-lts"

labels = {
  environment = "dev"
}

network = "default"
```

***

#### Step 6: Terraform Workflow

```bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply --auto-approve
```

***

#### Step 7: Destroy Resources

```bash
terraform destroy --auto-approve
```

***

### Key Takeaways

* Use `count` when resources are identical and order matters
* Use `count` with lists carefully, as index changes can cause recreation
* Use `for_each` when resources need stable, unique identities
* `for_each` is preferred for production-grade infrastructure

