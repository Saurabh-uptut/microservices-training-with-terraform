# Lab: Expose the Three-Tier App with a Single Gateway (GKE Gateway Controller)

#### What you will achieve

Route both frontend and backend through a **single public IP** using **Gateway API** on GKE.

* `/` → frontend (ClusterIP)
* `/backend/...` → backend (ClusterIP) with prefix rewrite to `/...`
* Postgres stays internal (ClusterIP)

#### Prerequisites

* GKE cluster and `kubectl` configured
* Namespace `user-management` exists
* App from previous lab running:
  * `postgres-db` (ClusterIP)
  * `backend-deploy`
  * `frontend-deploy`

***

## Step 0: Mandatory platform checks and fixes

### 0.1 Verify Gateway API is available

If this fails:

```bash
kubectl get gatewayclass
```

and you see:\
`error: the server doesn't have a resource type "gatewayclass"`

It means Gateway API is not enabled on the cluster.

#### Enable Gateway API on the cluster

Regional cluster:

```bash
gcloud container clusters update <CLUSTER_NAME> \
  --region <REGION> \
  --gateway-api=standard
```

Zonal cluster:

```bash
gcloud container clusters update <CLUSTER_NAME> \
  --zone <ZONE> \
  --gateway-api=standard
```

Re-check:

```bash
kubectl get gatewayclass
```

You should see `gke-l7-regional-external-managed` (or similar).

***

### 0.2 Fix: “proxy-only subnetwork is required”

If your Gateway events show:\
`An active proxy-only subnetwork is required in the same region and VPC as the forwarding rule`

You must create a **proxy-only subnet** in the same **VPC** and **region**.

#### Find the cluster VPC

```bash
gcloud container clusters describe <CLUSTER_NAME> \
  --region <REGION> \
  --project <PROJECT_ID> \
  --format="value(network)"
```

#### Create the proxy-only subnet (choose a non-overlapping CIDR)

Example:

```bash
gcloud compute networks subnets create proxy-only-<REGION> \
  --project <PROJECT_ID> \
  --region <REGION> \
  --network <VPC_NAME> \
  --range 10.129.0.0/23 \
  --purpose=REGIONAL_MANAGED_PROXY \
  --role=ACTIVE
```

If you are unsure about overlap, list existing subnets first:

```bash
gcloud compute networks subnets list \
  --project <PROJECT_ID> \
  --filter="region:(<REGION>) AND network:(<VPC_NAME>)" \
  --format="table(name,ipCidrRange,purpose,role)"
```

***

## Step 1: Clean up previous per-service LoadBalancers

```bash
kubectl delete svc lb-backend-service -n user-management --ignore-not-found
kubectl delete svc lb-frontend-service -n user-management --ignore-not-found

kubectl get deploy,rs,pods,svc -n user-management -o wide
```

***

## Step 2: Create ClusterIP Services

### 2.1 Backend ClusterIP

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

### 2.2 Frontend ClusterIP

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

Apply:

```bash
kubectl apply -f frontend-clusterip.yml
kubectl get svc -n user-management
```

#### Fix included: Ensure Services have endpoints

If later you see SNEG errors like “no zones found”, validate endpoints immediately:

```bash
kubectl get endpointslice -n user-management -l kubernetes.io/service-name=frontend-clusterip-service
kubectl get endpointslice -n user-management -l kubernetes.io/service-name=backend-clusterip-service
kubectl get pods -n user-management --show-labels
```

If endpoints are empty, your selector labels do not match.

***

## Step 3: Create the Gateway (single public IP)

### 3.1 Create infra namespace

```bash
kubectl create ns gateway-system --dry-run=client -o yaml | kubectl apply -f -
```

### 3.2 Allow routes only from labeled namespaces

```bash
kubectl label ns user-management gateway-access=true --overwrite
```

#### Fix included: If this label is missing, HTTPRoute bind or acceptance can fail.

### 3.3 Create `gateway.yml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: gateway-system
spec:
  gatewayClassName: gke-l7-regional-external-managed
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: "true"
```

Apply:

```bash
kubectl apply -f gateway.yml
kubectl get gateway -n gateway-system
kubectl describe gateway web-gateway -n gateway-system
```

Wait until:

* `PROGRAMMED: True`
* `ADDRESS` is populated

This will be your `<GATEWAY_IP>`.

***

## Step 4: Update frontend to call backend through Gateway

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
      "API_URL": "http://<GATEWAY_IP>/backend/"
    }
```

Apply and restart frontend:

```bash
kubectl apply -f ui-configmap.yml
kubectl rollout restart deploy/frontend-deploy -n user-management
```

***

## Step 5: Create the HTTPRoute (two paths + rewrite)

Create `userapp-route.yml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: userapp-route
  namespace: user-management
spec:
  parentRefs:
    - name: web-gateway
      namespace: gateway-system
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /backend
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: backend-clusterip-service
          port: 80

    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend-clusterip-service
          port: 80
```

Apply and check:

```bash
kubectl apply -f userapp-route.yml
kubectl describe httproute userapp-route -n user-management
```

#### Fix included: “no zones found in ServiceNetworkEndpointGroup”

If you see this temporarily in HTTPRoute events, but your Services have endpoints and NEG status shows zones, it often resolves once NEG and health checks converge. Confirm endpoints and re-check route events.

***

## Step 6: Fix backend 503 by configuring HealthCheckPolicy

#### Symptom we saw

External call returned:

```bash
curl -I http://<GATEWAY_IP>/backend/health
HTTP/1.1 503 Service Unavailable
via: 1.1 google
```

This indicates Google LB has **no healthy backends** (health check mismatch is the typical cause).

### 6.1 Confirm backend works inside the cluster

```bash
kubectl run -n user-management curlpod --image=curlimages/curl:8.5.0 -it --rm -- \
  sh -c 'curl -i http://backend-clusterip-service/health || true; echo; curl -i http://backend-clusterip-service/ || true'
```

### 6.2 Apply HealthCheckPolicy for backend

Because your route strips `/backend`, the backend should receive `/health` when you call `/backend/health`.

Create `backend-hc-policy.yml`:

```yaml
apiVersion: networking.gke.io/v1
kind: HealthCheckPolicy
metadata:
  name: backend-hc
  namespace: user-management
spec:
  targetRef:
    group: ""
    kind: Service
    name: backend-clusterip-service
  default:
    config:
      type: HTTP
      httpHealthCheck:
        requestPath: /health
        port: 3000
```

Apply:

```bash
kubectl apply -f backend-hc-policy.yml
kubectl get healthcheckpolicy -n user-management
```

If your backend health path is not `/health`, change `requestPath` accordingly.

***

## Step 7: End-to-end tests

Replace `<GATEWAY_IP>` with the real Gateway address (in your case it was `35.209.132.154`).

### 7.1 Frontend

```bash
curl -I http://<GATEWAY_IP>/
```

### 7.2 Backend

```bash
curl -I http://<GATEWAY_IP>/backend/health
curl -s http://<GATEWAY_IP>/backend/api/user | head
```

### 7.3 UI

Open:

```
http://<GATEWAY_IP>/
```

***

## Troubleshooting checklist (the exact issues we fixed)

### A) No GatewayClass resource type

* Enable Gateway API on the cluster (`--gateway-api=standard`)

### B) Proxy-only subnet required

* Create a proxy-only subnet with:
  * purpose `REGIONAL_MANAGED_PROXY`
  * role `ACTIVE`
  * same region and VPC as the cluster

### C) “no zones found in ServiceNetworkEndpointGroup”

* Ensure Service has endpoints:
  * EndpointSlice not empty
* Ensure selector matches pod labels
* If endpoints exist, it often resolves after NEG sync

### D) 503 Service Unavailable from `/backend/...`

* Apply HealthCheckPolicy to backend Service
* Ensure health check path and port match the real app behavior

