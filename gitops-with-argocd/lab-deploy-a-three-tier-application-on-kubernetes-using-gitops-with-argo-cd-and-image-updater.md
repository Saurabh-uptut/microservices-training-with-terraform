# Lab : Deploy a Three-Tier Application on Kubernetes using GitOps with Argo CD and Image Updater

### Lab Overview

In this lab, you will deploy a complete three-tier application on Kubernetes using **GitOps principles** with **Argo CD** and enable **automatic container image updates** using **Argo CD Image Updater**.

You will not manually apply Kubernetes manifests after setup.\
Git becomes the **single source of truth**.

### Application Architecture

Frontend (NGINX static UI)\
→ Backend (Node.js API)\
→ Database (PostgreSQL)

#### Service Exposure Model

* PostgreSQL: ClusterIP (internal)
* Backend API: LoadBalancer (public)
* Frontend UI: LoadBalancer (public)

### What You Will Build

You will:

* Install Argo CD on Kubernetes
* Prepare a GitOps repository for the three-tier app
* Deploy the application using Argo CD
* Convert backend manifests to Kustomize
* Install Argo CD Image Updater
* Automatically update backend image tags via Git commits
* Observe Git-driven rollouts and self-healing

### Prerequisites

You must have:

* A running Kubernetes cluster (GKE recommended)
* kubectl configured
* Git installed
* Docker installed
* A GitHub repository with push access
* Docker Hub or another supported container registry

## Part 1: Install Argo CD

### Step 1: Create Argo CD namespace

```bash
kubectl create namespace argocd
```

### Step 2: Install Argo CD

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until all pods are running:

```bash
kubectl get pods -n argocd
```

***

### Step 3: Expose Argo CD (lab setup)

```bash
kubectl expose svc argocd-server \
  -n argocd \
  --type=LoadBalancer \
  --name=argocd-server-ext
```

Get the external IP:

```bash
kubectl get svc -n argocd
```

Open in browser:

```
http://<ARGOCD_EXTERNAL_IP>
```

***

### Step 4: Login to Argo CD

Get admin password:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode
```

Login:

* Username: admin
* Password: retrieved value

***

## Part 2: Prepare GitOps Repository

### Step 5: Create GitOps repository structure

Create a GitHub repo, for example:

```
user-management-gitops
```

Directory structure:

```
user-management-gitops/
├── apps/
│   └── user-management/
│       ├── namespace.yml
│       ├── postgres/
│       │   ├── db-secret.yml
│       │   ├── db-pvc.yml
│       │   ├── db-deploy.yml
│       │   └── db-svc.yml
│       ├── backend/
│       │   ├── api-deploy.yml
│       │   ├── api-svc-lb.yml
│       │   └── kustomization.yml
│       └── frontend/
│           ├── ui-configmap.yml
│           ├── ui-deploy.yml
│           └── ui-svc-lb.yml
└── README.md
```

Copy manifests from your earlier lab into the respective folders.

### Step 6: Backend Kustomize setup (required for Image Updater)

Create `apps/user-management/backend/kustomization.yml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - api-deploy.yml
  - api-svc-lb.yml

images:
  - name: YOUR_DOCKERHUB/backend-user-management
    newTag: "1.0.0"
```

Ensure `api-deploy.yml` uses:

```yaml
image: YOUR_DOCKERHUB/backend-user-management:1.0.0
```

Commit and push:

```bash
git add .
git commit -m "Add GitOps manifests with backend Kustomize"
git push origin main
```

***

## Part 3: Create Argo CD Applications

### Step 7: Create backend Argo CD Application (Image Updater scope)

Create `backend-app.yml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: user-management-backend
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: backend=YOUR_DOCKERHUB/backend-user-management
    argocd-image-updater.argoproj.io/backend.update-strategy: semver
    argocd-image-updater.argoproj.io/backend.allow-tags: regexp:^\\d+\\.\\d+\\.\\d+$
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: kustomization
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB_USERNAME/user-management-gitops.git
    targetRevision: main
    path: apps/user-management/backend
  destination:
    server: https://kubernetes.default.svc
    namespace: user-management
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply:

```bash
kubectl apply -f backend-app.yml
```

### Step 8: Create Argo CD Application for remaining resources

Create `user-management-core.yml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: user-management-core
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB_USERNAME/user-management-gitops.git
    targetRevision: main
    path: apps/user-management
  destination:
    server: https://kubernetes.default.svc
    namespace: user-management
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply:

```bash
kubectl apply -f user-management-core.yml
```

## Part 4: Verify GitOps Deployment

```bash
kubectl get all -n user-management
kubectl get pvc -n user-management
kubectl get svc -n user-management
```

Open frontend:

```
http://<FRONTEND_EXTERNAL_IP>
```

## Part 5: Install Argo CD Image Updater

### Step 9: Install Image Updater

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

Verify:

```bash
kubectl get pods -A | grep image-updater
```

***

### Step 10: Configure Git write-back credentials

Create a GitHub Personal Access Token with repo write access.

Create secret:

```bash
kubectl -n argocd-image-updater-system create secret generic git-creds \
  --from-literal=username=YOUR_GITHUB_USERNAME \
  --from-literal=password=YOUR_GITHUB_TOKEN
```

## Part 6: Automatic Image Update Flow

### Step 11: Push a new backend image

```bash
docker build --platform linux/amd64 \
  -t YOUR_DOCKERHUB/backend-user-management:1.0.1 ./api

docker push YOUR_DOCKERHUB/backend-user-management:1.0.1
```

### Step 12: Observe GitOps automation

What happens automatically:

1. Image Updater detects new tag
2. Commits new tag to `kustomization.yml`
3. Pushes commit to Git
4. Argo CD syncs
5. Backend rolls out new pods

Verify:

```bash
git log --oneline
kubectl describe deploy backend-deploy -n user-management | grep Image
```

## Part 7: Drift Detection and Self-Healing

```bash
kubectl scale deploy backend-deploy -n user-management --replicas=1
```

Within seconds, Argo CD restores replicas from Git.

## Cleanup

```bash
kubectl delete application user-management-backend -n argocd
kubectl delete application user-management-core -n argocd
kubectl delete namespace argocd
kubectl delete namespace user-management
```
