# Lab 3 - Add Terraform State File to Remote Storage using Google Cloud Storage

### Overview

In this lab, you will configure Terraform to store its **state file remotely in a Google Cloud Storage bucket**.\
Using a remote backend improves collaboration, enables state locking, and ensures the Terraform state is safely stored outside local machines.

This lab continues from the previous Terraform lab where infrastructure was already provisioned.

***

### Learning Objectives

By the end of this lab, you will be able to:

1. Understand Terraform remote state and why it is required
2. Create a Google Cloud Storage bucket for Terraform state
3. Configure the Terraform backend using Google Cloud Storage
4. Initialize and migrate Terraform state to a remote backend
5. Observe Terraform state locking behavior during plan and apply

***

### Prerequisites

* Google Cloud account with valid credentials
* Terraform installed locally
* Google Cloud CLI installed
* VS Code or any IDE
* Completion of the previous Terraform lab (compute and networking)

***

### Important Notes

* This lab is a continuation of the previous lab
* Reuse the same Terraform project directory
* Do not create new infrastructure for testing
* Ensure you have permission to create Google Cloud Storage buckets

***

### Step 1 - Create a Google Cloud Storage Bucket

Terraform requires a **globally unique bucket name**.

Run the following command:

```bash
gcloud storage buckets create gs://YOUR_BUCKET_NAME \
  --project=YOUR_PROJECT_ID \
  --location=YOUR_REGION
```

Example:

```bash
gcloud storage buckets create gs://my-tf-state-bkt-2025 \
  --project=terraform-training-422512 \
  --location=us-central1
```

Verify the bucket creation:

```bash
gcloud storage buckets list
```

***

### Step 2 - Update Terraform Backend Configuration

Open the existing `providers.tf` file and update it as shown below.

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "6.35.0"
    }
  }

  backend "gcs" {
    bucket  = "my-tf-state-bkt202527may"
    prefix = "tf-state-saurabh"
  }
}

provider "google" {
  project = "terraform-training-422512"
  region  = "us-central1"
}
```

#### What changed

* A `backend "gcs"` block was added
* `bucket` specifies where the Terraform state will be stored
* `prefix` defines a folder-like path inside the bucket to organize state files

***

### Step 3 - Initialize Terraform and Migrate State

Run the following command from the Terraform project directory:

```bash
terraform init
```

Terraform will:

* Detect the new backend configuration
* Ask to migrate the existing local state
* Upload the state file to the Google Cloud Storage bucket

Approve the migration when prompted.

***

### Step 4 - Verify Remote State Storage

1. Open Google Cloud Console
2. Navigate to Cloud Storage
3. Open the bucket you created
4. Navigate to the folder defined by the prefix
5. Confirm the presence of the `terraform.tfstate` file

At this point, Terraform no longer uses a local state file.

***

### Step 5 - Run Terraform Plan or Apply

Execute either of the following:

```bash
terraform plan
```

or

```bash
terraform apply
```

Observe the output carefully.

You will see a line similar to:

```
Acquiring state lock. This may take a few moments...
```

#### What this means

* Terraform locks the remote state before making changes
* This prevents multiple users from modifying the same infrastructure simultaneously
* State locking is a key advantage of using a remote backend

***

### Key Takeaways

* Remote state enables safe collaboration across teams
* Google Cloud Storage provides durable and highly available state storage
* Terraform automatically handles state locking with the GCS backend
* Backend configuration should be done early in a project to avoid state migration later

