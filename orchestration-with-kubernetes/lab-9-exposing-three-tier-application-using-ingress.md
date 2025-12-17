# Lab 9 - Exposing three tier application using Ingress

#### What you’ll need (continuation from the previous lab)

* A **GKE cluster** and `kubectl` configured
* Namespace `user-management` already created
* App from the last lab running:
  * `postgres-db` (ClusterIP)
  * `backend-deploy` (was exposed via LoadBalancer in last lab)
  * `frontend-deploy` (was exposed via LoadBalancer in last lab)

***

### Step 0: Connect to your GKE cluster

If you are not already connected:

```bash
gcloud container clusters get-credentials <CLUSTER_NAME> \
  --region <REGION> \
  --project <PROJECT_ID>
```

Confirm:

```bash
kubectl get nodes
kubectl get ns
```

***

### Step 1: Clean up previous LoadBalancers

Replace the two per-service public LBs with a single Ingress public IP.

```bash
kubectl delete svc lb-backend-service -n user-management --ignore-not-found
kubectl delete svc lb-frontend-service -n user-management --ignore-not-found
```

Verify DB + deployments are still healthy:

```bash
kubectl get deploy,rs,pods,svc -n user-management -o wide
```

***

### Step 2: Create ClusterIP Services for backend and frontend

#### 2.1 Backend ClusterIP

Create `backend-clusterip.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-clusterip-service
  namespace: user-management
spec:
  type: ClusterIP
  selector:
    app: backend-app
  ports:
    - port: 80
      targetPort: 3000
```

Apply:

```bash
kubectl apply -f backend-clusterip.yml
```

#### 2.2 Frontend ClusterIP

Create `frontend-clusterip.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-clusterip-service
  namespace: user-management
spec:
  type: ClusterIP
  selector:
    app: frontend-app
  ports:
    - port: 80
      targetPort: 80
```

Apply and confirm:

```bash
kubectl apply -f frontend-clusterip.yml
kubectl get svc -n user-management
```

At this point you should have 3 ClusterIP services:

* `postgres-db`
* `backend-clusterip-service`
* `frontend-clusterip-service`

***

### Step 3: Install NGINX Ingress Controller (once per cluster)

On GKE, this will create **one** `Service type: LoadBalancer`, giving you **one** public IP for all your ingress routes.

#### 3.1 Install Helm (if needed)

Install Helm using the official steps for your OS.

#### 3.2 Add repo and install controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.ingressClassResource.name=nginx \
  --set controller.ingressClassResource.controllerValue=k8s.io/ingress-nginx \
  --set controller.service.type=LoadBalancer
```

Verify:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Get the public IP (this is your `<INGRESS_IP>`):

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

If `EXTERNAL-IP` shows `<pending>`, wait a bit and re-run the command. You can also check events:

```bash
kubectl get events -n ingress-nginx --sort-by=.lastTimestamp | tail -n 30
```

***

### Step 4: Point the frontend to the new backend path

Update the UI config so it calls the backend through the Ingress path `/backend/`.

Create or update `ui-configmap.yml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-configmap
  namespace: user-management
data:
  config.json: |
    {
      "API_URL": "http://<INGRESS_IP>/backend/"
    }
```

Replace `<INGRESS_IP>` with the actual external IP from Step 3.

Apply and restart frontend:

```bash
kubectl apply -f ui-configmap.yml
kubectl rollout restart deploy/frontend-deploy -n user-management
```

***

### Step 5: Create the Ingress routes (single public IP, two paths)

#### 5.1 Frontend Ingress rule

Create `frontend-ingress.yml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  namespace: user-management
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: frontend-clusterip-service
                port:
                  number: 80
```

#### 5.2 Backend Ingress rule

Create `backend-ingress.yml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  namespace: user-management
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /backend(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: backend-clusterip-service
                port:
                  number: 80
```

Apply and check:

```bash
kubectl apply -f frontend-ingress.yml
kubectl apply -f backend-ingress.yml

kubectl get ingress -n user-management
```

***

### Step 6: Test end-to-end

#### 6.1 Frontend via Ingress

```bash
curl -I http://<INGRESS_IP>/
```

Or open:

* `http://<INGRESS_IP>/`

#### 6.2 Backend via Ingress (direct)

```bash
curl -I http://<INGRESS_IP>/backend/health
curl -s http://<INGRESS_IP>/backend/api/user | head
```

#### 6.3 Frontend to Backend (UI wiring)

Open `http://<INGRESS_IP>/` and test CRUD flows.

If API calls fail:

* Confirm `API_URL` is `http://<INGRESS_IP>/backend/` (keep the trailing slash)
* Confirm you restarted the frontend deployment

***

### Troubleshooting (GKE-focused quick fixes)

#### Ingress controller external IP pending

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
kubectl get events -n ingress-nginx --sort-by=.lastTimestamp | tail -n 30
```

#### `/backend/...` returns 404

* Ensure the backend ingress path and annotations match exactly
* Re-apply:

```bash
kubectl apply -f backend-ingress.yml
```

* Check NGINX controller logs:

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=200
```

#### Service has no endpoints

Your service selectors must match pod labels:

```bash
kubectl get pods -n user-management --show-labels
kubectl describe svc backend-clusterip-service -n user-management
kubectl describe svc frontend-clusterip-service -n user-management
```
