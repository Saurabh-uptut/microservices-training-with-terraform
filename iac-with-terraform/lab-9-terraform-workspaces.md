# Lab 9 - Terraform Workspaces

In this lab, you’ll practice Terraform **workspaces** to keep **separate state files** (and therefore separate environments) inside the **same codebase**.

### Learning Objectives

* Create, list, select, and delete Terraform workspaces
* Manage multiple **non-overlapping** environments from the same Terraform configuration
* Maintain multiple **terraform.tfstate** files using workspaces

### Prerequisites

* Terraform installed
* `gcloud` CLI installed and authenticated
* Any working GCP Terraform codebase (e.g., a `google_compute_instance` project)

## Step 0: Prepare the project

Format and initialize the working directory:

```bash
terraform fmt
terraform init
```

***

## Part 1: Default Workspace

### 1) List workspaces

```bash
terraform workspace list
```

You’ll see one workspace:

* `default` (current)

### 2) Plan and apply in default

```bash
terraform plan
terraform apply --auto-approve
```

Result:

* Resources are created in GCP
* A local state file is created: `terraform.tfstate`
* This is the state for the **default** workspace

***

## Part 2: Create and Use “development” Workspace

### 3) Create a new workspace

```bash
terraform workspace new development
```

### 4) Confirm current workspace

```bash
terraform workspace list
```

You’ll notice:

* `development` is selected
* A folder appears: `terraform.tfstate.d/development/`
* This is where the `development` state file will live

***

## Part 3: Use a separate tfvars file per workspace

### 5) Create `dev.terraform.tfvars`

Create a new file in the same directory:

```hcl
compute_name = "compute-saurabh-dev"
machine_type = "n1-standard-2"
zone         = "us-central1-a"
image        = "ubuntu-os-cloud/ubuntu-2004-lts"

labels = {
  environment = "dev"
}

network = "default"
```

> Update values based on your setup (especially `network` if you don’t use default VPC).

### 6) Plan/apply in development workspace using var-file

```bash
terraform plan -var-file=dev.terraform.tfvars
terraform apply -var-file=dev.terraform.tfvars --auto-approve
```

Result:

* A separate state file is created under:
  * `terraform.tfstate.d/development/terraform.tfstate`
* Now you have **two environments** managed from the same code:
  * `default` workspace
  * `development` workspace

***

## Part 4: Destroy Resources Cleanly (Workspace by Workspace)

### 7) Destroy development resources

Make sure you’re still in `development`:

```bash
terraform workspace list
terraform destroy -var-file=dev.terraform.tfvars --auto-approve
```

> Important: if the workspace used a var-file to create resources, use the same var-file to destroy them.

### 8) Switch to default and destroy default resources

```bash
terraform workspace select default
terraform destroy --auto-approve
```

***

## Part 5: Delete the workspace

### 9) Delete development workspace

```bash
terraform workspace delete development
```
