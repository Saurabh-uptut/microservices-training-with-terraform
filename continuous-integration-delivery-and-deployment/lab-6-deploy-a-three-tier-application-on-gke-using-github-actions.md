# Lab 6: Deploy a Three-Tier Application on GKE using GitHub Actions

### Lab Overview

In this lab, you will deploy a complete three-tier application on **Google Kubernetes Engine (GKE)** using **GitHub Actions for CI/CD**.

The pipeline will:

* Build frontend and backend Docker images
* Push images to Docker Hub
* Authenticate to Google Cloud
* Deploy the application to GKE using Kubernetes manifests

### Application Architecture

Frontend (NGINX static UI)\
→ Backend (Node.js API)\
→ Database (PostgreSQL)

#### Service exposure

* PostgreSQL: ClusterIP
* Backend API: LoadBalancer
* Frontend UI: LoadBalancer

### What You Will Build

You will:

* Configure GitHub Secrets for GCP and Docker Hub
* Create a GitHub Actions workflow
* Build and push container images
* Deploy Kubernetes manifests to GKE
* Validate the application end to end

### Prerequisites

You must have:

* A GKE cluster already running
* kubectl configured locally (for verification)
* Docker Hub account
* GitHub repository with the application code
* gcloud CLI installed locally (to create credentials)

## Part 1: Prepare GCP Service Account for GitHub Actions

### Step 1: Create a GCP service account

```bash
gcloud iam service-accounts create github-actions-gke \
  --display-name "GitHub Actions GKE Deployer"
```

### Step 2: Grant required IAM roles

```bash
PROJECT_ID=<your-project-id>

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-actions-gke@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/container.developer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-actions-gke@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

### Step 3: Create service account key

```bash
gcloud iam service-accounts keys create key.json \
  --iam-account github-actions-gke@$PROJECT_ID.iam.gserviceaccount.com
```

You will upload this JSON to GitHub Secrets.

## Part 2: Configure GitHub Secrets

In your GitHub repository, add the following secrets:

| Secret Name          | Value                   |
| -------------------- | ----------------------- |
| GCP\_PROJECT\_ID     | your project id         |
| GCP\_CLUSTER\_NAME   | gke cluster name        |
| GCP\_CLUSTER\_REGION | cluster region          |
| GCP\_SA\_KEY         | contents of key.json    |
| DOCKERHUB\_USERNAME  | docker hub username     |
| DOCKERHUB\_TOKEN     | docker hub access token |

## Part 3: Prepare Kubernetes Manifests

Your repository structure should look like:

```
.
├── api/
├── ui/
├── k8s_solution/
│   ├── namespace.yml
│   ├── db-secret.yml
│   ├── db-pvc.yml
│   ├── db-deploy.yml
│   ├── db-svc.yml
│   ├── api-deploy.yml
│   ├── api-svc-lb.yml
│   ├── ui-configmap.yml
│   ├── ui-deploy.yml
│   └── ui-svc-lb.yml
└── .github/
    └── workflows/
```

Ensure image references in manifests use placeholders like:

```yaml
image: YOUR_DOCKERHUB/backend-user-management:latest
image: YOUR_DOCKERHUB/frontend-user-management:latest
```

## Part 4: Create GitHub Actions Workflow

Create `.github/workflows/deploy-gke.yml`:

```yaml
name: Build and Deploy to GKE

on:
  push:
    branches:
      - main

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  CLUSTER_NAME: ${{ secrets.GCP_CLUSTER_NAME }}
  CLUSTER_REGION: ${{ secrets.GCP_CLUSTER_REGION }}
  DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push backend image
        run: |
          docker build -t $DOCKERHUB_USERNAME/backend-user-management:latest ./api
          docker push $DOCKERHUB_USERNAME/backend-user-management:latest

      - name: Build and push frontend image
        run: |
          docker build -t $DOCKERHUB_USERNAME/frontend-user-management:latest ./ui
          docker push $DOCKERHUB_USERNAME/frontend-user-management:latest

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up gcloud
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure kubectl
        run: |
          gcloud container clusters get-credentials $CLUSTER_NAME \
            --region $CLUSTER_REGION \
            --project $PROJECT_ID

      - name: Deploy to GKE
        run: |
          kubectl apply -f k8s_solution/namespace.yml
          kubectl apply -f k8s_solution/db-secret.yml
          kubectl apply -f k8s_solution/db-pvc.yml
          kubectl apply -f k8s_solution/db-deploy.yml
          kubectl apply -f k8s_solution/db-svc.yml
          kubectl apply -f k8s_solution/api-deploy.yml
          kubectl apply -f k8s_solution/api-svc-lb.yml
          kubectl apply -f k8s_solution/ui-configmap.yml
          kubectl apply -f k8s_solution/ui-deploy.yml
          kubectl apply -f k8s_solution/ui-svc-lb.yml
```

## Part 5: Trigger the Pipeline

Commit and push:

```bash
git add .
git commit -m "Add GitHub Actions CI/CD for GKE"
git push origin main
```

Navigate to:

```
GitHub → Actions → Build and Deploy to GKE
```

Watch:

* Image build
* Image push
* Deployment to GKE

## Part 6: Verify Deployment

```bash
kubectl get pods -n user-management
kubectl get svc -n user-management
```

Open frontend:

```
http://<FRONTEND_LB_IP>
```

Test API:

```bash
curl http://<BACKEND_LB_IP>/health
```

## Cleanup

```bash
kubectl delete namespace user-management
```

(Optional)

```bash
gcloud iam service-accounts delete github-actions-gke@$PROJECT_ID.iam.gserviceaccount.com
```
