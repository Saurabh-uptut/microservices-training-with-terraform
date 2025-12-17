# Lab 1 - Provision your First Google Cloud Infrastructure using Terraform (Compute VM)

### Goal

Deploy **one Google Compute Engine VM** using Terraform, verify it in Google Cloud Console, and then clean it up.

### Learning objectives

1. Write and run Terraform code using the **Google provider**
2. Authenticate Terraform to Google Cloud (local dev / workstation)
3. Create and configure a **Google Compute Engine instance**
4. Practice the Terraform workflow: `fmt → validate → plan → apply → destroy`

### Prerequisites

* A Google Cloud account with access to a project
* **Terraform** installed (`terraform --version`)
* **Google Cloud CLI** installed (`gcloud version`)
* VS Code (or any IDE)
* Permissions to create Compute Engine resources (e.g., Compute Admin)

### Authentication (Application Default Credentials)

Terraform (via the Google provider) can use **Application Default Credentials (ADC)** from your local machine.

#### Step 1 - Initialize gcloud

```bash
gcloud init
```

#### Step 2 - Create ADC credentials (recommended for labs)

```bash
gcloud auth application-default login
```

#### Step 3 - Confirm which project you’re using

```bash
gcloud config get-value project
```

If it’s not the project you want, set it:

```bash
gcloud config set project <YOUR_PROJECT_ID>
```

### Create the Terraform project structure

#### Step 1 - Create a folder

Create a folder named:

```bash
compute_instance
```

Open it in VS Code.

Recommended files:

* `providers.tf`
* `variables.tf`
* `compute_instance.tf`
* `terraform.tfvars`

### Step 2 - Add `providers.tf`

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

Update these:

* `project` → your GCP Project ID
* `region` → pick a region close to you (example: `asia-southeast1`)

### Step 3 - Initialize Terraform

From inside the folder:

```bash
terraform init
```

This downloads the provider plugin and creates the `.terraform/` directory.

### Step 4 - Add `variables.tf`

Create `variables.tf`:

```hcl
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
  type        = string
  description = "Boot disk image"
}

variable "labels" {
  type        = any
  description = "Labels to apply to the disk"
}

variable "network" {
  type        = string
  description = "VPC network name"
}
```

### Step 5 - Add the VM resource in `compute_instance.tf`

Create `compute_instance.tf`:

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
    network = var.network

    # This gives the VM an external (public) IP
    access_config {}
  }
}
```

### Step 6 - Provide values in `terraform.tfvars`

Create `terraform.tfvars`:

```hcl
compute_name = "compute-saurabh"
machine_type = "n2d-standard-2"
zone         = "us-central1-a"
image        = "ubuntu-os-cloud/ubuntu-2204-lts"

labels = {
  environment = "dev"
}

network = "default"
```

Make sure:

* `zone` belongs to the same “area” as your provider region (example: `us-central1` + `us-central1-a`)
* If you change region to `asia-southeast1`, then zone should be like `asia-southeast1-a`

### Terraform workflow (run in this exact order)

#### 1) Format code

```bash
terraform fmt
```

#### 2) Validate syntax

```bash
terraform validate
```

#### 3) Preview changes

```bash
terraform plan
```

#### 4) Create the VM

```bash
terraform apply --auto-approve
```

### Verify in Google Cloud Console

1. Open **Google Cloud Console**
2. Go to **Compute Engine → VM instances**
3. Confirm you see your VM (example: `compute-saurabh`)
4. Click it and confirm:
   * Zone matches
   * Machine type matches
   * External IP exists (because of `access_config {}`)

### Cleanup (delete everything created by Terraform)

```bash
terraform destroy --auto-approve
```

Verify the VM is gone from **Compute Engine → VM instances**.

### Quick troubleshooting

*   **Auth error / permissions**: re-run

    ```bash
    gcloud auth application-default login
    ```

    and confirm project:

    ```bash
    gcloud config get-value project
    ```
* **Zone/region mismatch**: your `zone` must match the region family (e.g., `us-central1-a` with `us-central1`).
* **Image not found**: try a newer one like:
  * `ubuntu-os-cloud/ubuntu-2204-lts`
