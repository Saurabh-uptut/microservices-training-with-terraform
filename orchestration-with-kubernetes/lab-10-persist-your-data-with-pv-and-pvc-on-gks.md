# Lab 10 - Persist Your Data with PV & PVC on GKS

In this lab, we will move the Postgres storage from “node local” behavior to **Google Cloud Filestore using dynamic provisioning** on GKE. No manually created PV, only a PVC that triggers Filestore provisioning through the Filestore CSI driver.

This uses the GKE-provided Filestore StorageClass **`standard-rwx`** (Basic HDD) and **ReadWriteMany**. GKE creates the Filestore instance and PV automatically.&#x20;

### Changes to the lab

#### What stays the same?

* Backend and frontend manifests stay unchanged
* Postgres Deployment stays mostly the same (still mounts a PVC at `/var/lib/postgresql/data`)

#### What changes

1. Enable Filestore CSI driver (Standard cluster)
2. Replace your current `db-pvc.yml` with a Filestore dynamic PVC:
   * `storageClassName: standard-rwx`
   * `accessModes: ReadWriteMany`
   * increase storage request to meet Filestore minimums (Basic HDD supports 100 GiB+).&#x20;

***

### Step 0: Enable Filestore CSI (one-time per cluster)

#### 0.1 Enable Filestore API

```bash
gcloud services enable file.googleapis.com --project <PROJECT_ID>
```

#### 0.2 Enable Filestore CSI driver add-on (GKE Standard)

```bash
gcloud container clusters update <CLUSTER_NAME> \
  --region <REGION> \
  --update-addons=GcpFilestoreCsiDriver=ENABLED
```

GKE then installs Filestore provisioning StorageClasses (including `standard-rwx`).&#x20;

Verify:

```bash
kubectl get storageclass
```

You should see `standard-rwx` (Basic HDD) and possibly `premium-rwx` (Basic SSD).&#x20;

***

### Step 5.1: Replace the DB PVC with Filestore dynamic PVC

Replace your existing `db-pvc.yml` with this:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: user-management
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: standard-rwx
  resources:
    requests:
      storage: 100Gi
```

Why 100Gi: `standard-rwx` (Basic HDD) supports PVC sizes from 100 GiB to 64 TiB.&#x20;

Apply:

```bash
kubectl apply -f db-pvc.yml
kubectl get pvc -n user-management
```

Important: Filestore dynamic provisioning often uses “wait for first consumer”, so the PVC may stay Pending until the Postgres pod is created and scheduled.&#x20;

***

### Step 5.3: PostgreSQL Deployment (only small note)

Your existing `db-deploy.yml` is fine. Keep this part as-is:

```yaml
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-pvc
```

Apply the DB resources (same as your lab flow):

```bash
kubectl apply -f db-secret.yml
kubectl apply -f db-deploy.yml
kubectl apply -f db-svc.yml
```

***

### Verification for Filestore dynamic PV

#### 1) Check PVC and PV

```bash
kubectl get pvc -n user-management
kubectl get pv | grep postgres-pvc -n || true
```

#### 2) Confirm the Filestore-backed volume is mounted

```bash
kubectl describe pod -n user-management -l app=db | sed -n '/Volumes:/,/Events:/p'
```

#### 3) Validate persistence

```bash
kubectl exec -n user-management deploy/postgres-db -- sh -c 'echo hello-$(date) >> /var/lib/postgresql/data/persist.txt && tail -n 3 /var/lib/postgresql/data/persist.txt'
kubectl delete pod -n user-management -l app=db
kubectl get pods -n user-management -w
kubectl exec -n user-management deploy/postgres-db -- sh -c 'tail -n 3 /var/lib/postgresql/data/persist.txt'
```
