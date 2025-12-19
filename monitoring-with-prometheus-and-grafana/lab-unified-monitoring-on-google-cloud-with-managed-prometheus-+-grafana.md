# Lab: Unified Monitoring on Google Cloud with Managed Prometheus + Grafana

**Use Cloud Monitoring as the central metrics store, then use Grafana to visualize it.**

Why this is best on GCP:

* **Most Google Cloud services already publish platform metrics into Cloud Monitoring** (Cloud Run, Load Balancing, Cloud SQL, Pub/Sub, etc). So you instantly get broad coverage without running exporters everywhere.
* For **Prometheus-style metrics**, use **Google Cloud Managed Service for Prometheus (GMP)** to ingest Prometheus and OpenTelemetry metrics into **Cloud Monitoring storage**.&#x20;
* In **Grafana**, use the **built-in Google Cloud Monitoring data source** to query and dashboard those metrics.&#x20;

This gives you one consistent dashboarding layer (Grafana) for both:

1. Native GCP service metrics (Cloud Run, Compute, etc)
2. Your custom app metrics (Prometheus and OTEL) stored in Cloud Monitoring via GMP&#x20;

#### Lab outcomes

By the end, students will:

* Visualize **Cloud Run + Compute Engine + GKE** metrics in Grafana using a single data source (Cloud Monitoring).
* Ingest **Prometheus-format metrics from a VM** into Cloud Monitoring using Ops Agent’s Prometheus receiver.&#x20;
* (Optional) Ingest **custom application metrics** (Prometheus or OTEL) and build a dashboard and alert.

#### Architecture you will build

* **Cloud Run**: use built-in Cloud Monitoring metrics (requests, latency, instance count, CPU, memory)
* **Compute Engine VM**: expose Prometheus metrics (node\_exporter), collected by **Ops Agent Prometheus receiver** into Cloud Monitoring&#x20;
* **GKE** (optional but recommended): enable **Managed Service for Prometheus** for cluster and workload metrics&#x20;
* **Grafana**: add **Google Cloud Monitoring data source** and build dashboards&#x20;

## Prerequisites

* A GCP project + billing enabled
* gcloud CLI authenticated
* Permissions (or a prepared student service account) for:
  * Compute Admin
  * Cloud Run Admin
  * Monitoring Admin
  * Service Account Admin
  * (Optional for GKE) Kubernetes Engine Admin

## Task 1: Create a service account for Grafana

1. Create a service account:

```bash
export PROJECT_ID="YOUR_PROJECT"
gcloud config set project $PROJECT_ID

gcloud iam service-accounts create grafana-sa \
  --display-name="Grafana Cloud Monitoring Reader"
```

2. Grant minimal roles (good baseline for labs):

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:grafana-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/monitoring.viewer"
```

3. Create a JSON key (lab convenience):

```bash
gcloud iam service-accounts keys create grafana-sa-key.json \
  --iam-account="grafana-sa@$PROJECT_ID.iam.gserviceaccount.com"
```

***

## Task 2: Deploy Grafana (quick lab option)

Pick one:

#### Option A: Run Grafana on a small Compute Engine VM (simple for students)

* Create VM (e2-medium is fine)
* Install Docker
* Run Grafana:

```bash
docker run -d --name=grafana -p 3000:3000 grafana/grafana:latest
```

Then open `http://VM_EXTERNAL_IP:3000` and login (admin/admin).

#### Option B: Use Managed Grafana (if your org already uses it)

If you want the lab to be more enterprise-like, use a managed Grafana offering, but the rest of the lab stays identical.

***

## Task 3: Add Google Cloud Monitoring as Grafana data source

In Grafana:

1. Connections (or Data sources) -> Add data source
2. Choose **Google Cloud Monitoring**
3. Authentication: upload `grafana-sa-key.json`
4. Save and test

Grafana’s native support for Google Cloud Monitoring is documented here. ([Grafana Labs](https://grafana.com/docs/grafana/latest/datasources/google-cloud-monitoring/?utm_source=chatgpt.com))

***

## Task 4: Cloud Run workload and dashboard panels

#### 4A. Deploy a Cloud Run service

Use any sample container. Example:

```bash
gcloud run deploy hello-unified-monitoring \
  --image=us-docker.pkg.dev/cloudrun/container/hello \
  --region=us-central1 \
  --platform=managed \
  --allow-unauthenticated
```

#### 4B. Generate traffic

```bash
SERVICE_URL=$(gcloud run services describe hello-unified-monitoring \
  --region=us-central1 --format='value(status.url)')

for i in {1..300}; do curl -s $SERVICE_URL >/dev/null; done
```

#### 4C. Build Cloud Run panels in Grafana (Cloud Monitoring datasource)

Create a dashboard “GCP Unified Monitoring” and add panels like:

* Request count (rate)
* Request latency (p95 if available)
* Instance count
* Container CPU / memory utilization

Tip for teaching: show students how to browse metrics in the query builder first, then refine filters by `service_name`, `region`, and `project`.

***

## Task 5: Compute Engine VM Prometheus metrics into Cloud Monitoring

This is the key step that makes “Prometheus + Grafana” work across non-Kubernetes VMs without operating a full Prometheus stack.

#### 5A. Create a VM

```bash
gcloud compute instances create vm-metrics \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud
```

#### 5B. Install node\_exporter (Prometheus endpoint)

SSH into the VM:

```bash
gcloud compute ssh vm-metrics --zone=us-central1-a
```

Install node\_exporter (one straightforward way):

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter || true

curl -LO https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvf node_exporter-1.8.2.linux-amd64.tar.gz
sudo cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/

cat <<'EOF' | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl -s localhost:9100/metrics | head
```

#### 5C. Install and configure Ops Agent Prometheus receiver

On the VM, install Ops Agent and configure it to scrape `node_exporter` and send into Cloud Monitoring.

Google documents collecting Prometheus metrics on Compute Engine using Ops Agent’s Prometheus receiver.&#x20;

High-level steps for the lab:

1. Install Ops Agent
2. Configure a Prometheus scrape job for `localhost:9100`
3. Restart agent
4. Verify metrics appear in Cloud Monitoring metrics explorer, then in Grafana

Verification:

* In Cloud Monitoring, confirm time series appear for node exporter metrics.
* In Grafana (Cloud Monitoring datasource), query a node metric and filter to `vm-metrics`.

Suggested Grafana panels for the VM:

* CPU utilization
* Memory available
* Disk space free
* Network receive and transmit

## Task 6 (Optional but recommended): GKE metrics with Google Managed Service for Prometheus

If you want “kube metrics + app metrics” in a very standard way:

* Create a small GKE cluster
* Enable **Managed Service for Prometheus** collection
* Confirm cluster metrics flow into Cloud Monitoring, then visualize them in Grafana

GMP is Google’s managed Prometheus and OTEL metrics solution and uses Cloud Monitoring storage and APIs. ([Google Cloud Documentation](https://docs.cloud.google.com/stackdriver/docs/managed-prometheus?utm_source=chatgpt.com))

Suggested GKE panels:

* Pod count by namespace
* Container CPU and memory by workload
* Node conditions and allocatable resources

## Task 7: Alerts (Grafana or Cloud Monitoring)

Two good patterns:

* **Grafana alerting** on Cloud Monitoring queries (simple for students)
* **Cloud Monitoring alerting policies** (more “GCP-native”, good for enterprise discussions)

Example alerts:

* Cloud Run 5xx rate above threshold for 5 minutes
* VM memory available below threshold for 10 minutes
* GKE node NotReady count > 0
