# Lab 1: Setup Prometheus and Grafana Using Docker Compose

In this hands-on lab, you will deploy Prometheus and Grafana using Docker Compose, configure Prometheus to scrape metrics, and connect Grafana to visualize those metrics.

Prometheus will collect time-series metrics and Grafana will display them in dashboards.

***

### Learning Objectives

1. Deploy Prometheus and Grafana using Docker Compose
2. Configure Prometheus to scrape metrics from a server or application
3. Connect Grafana to Prometheus as a data source
4. Create and run basic metric visualizations in Grafana

***

### Pre-Requisites

* A Linux virtual machine
* Docker and Docker Compose installed
* Ports 3000 and 9090 open on the VM (accessible via browser)

If Docker requires sudo, enable rootless usage:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

***

## Step-by-Step Guide

***

### Step 1: Setup Prometheus and Grafana Using Docker Compose

1. Connect to the virtual machine using SSH.
2. Clone the repository provided for this lab:

```bash
git clone <repository-url>
cd <repository-folder>
```

3. Inside the project directory, create two folders:

```bash
mkdir prometheus_data grafana_data
```

4. Set correct folder permissions so Prometheus and Grafana can write data.

Prometheus runs as UID 65534:

```bash
sudo chown -R 65534:65534 prometheus_data
sudo chmod -R 775 prometheus_data
```

Grafana runs as UID 472:

```bash
sudo chown -R 472:472 grafana_data
sudo chmod -R 775 grafana_data
```

5. Deploy both services:

```bash
docker compose up -d
```

This starts Prometheus and Grafana containers in the background.

***

### Step 2: Access the Applications

#### Grafana

Open your browser and go to:

```
http://<public-ip>:3000
```

#### Prometheus

Open your browser and go to:

```
http://<public-ip>:9090
```

***

## Step 3: Understanding the Docker Compose Setup

The repository contains two important files:

1. docker-compose.yml
2. prometheus.yml

Below is the explanation.

***

### docker-compose.yml

#### Prometheus Service

```
prometheus:
  image: prom/prometheus:latest
  container_name: prometheus
  restart: unless-stopped
  ports:
    - "9090:9090"
  volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml
    - ./prometheus_data:/prometheus
  networks:
    - monitoring
```

Explanation:

* image: Downloads the Prometheus image
* container\_name: Names the container
* restart: Ensures it restarts on VM reboot
* ports: Exposes Prometheus on port 9090
* volumes:
  * First volume mounts the Prometheus config file
  * Second volume stores Prometheus metrics on the VM
* networks: Joins the monitoring network

#### Grafana Service

```
grafana:
  image: grafana/grafana:latest
  container_name: grafana
  restart: unless-stopped
  ports:
    - "3000:3000"
  volumes:
    - ./grafana_data:/var/lib/grafana
  environment:
    - GF_SECURITY_ADMIN_USER=admin
    - GF_SECURITY_ADMIN_PASSWORD=admin123
  networks:
    - monitoring
```

Explanation:

* image: Downloads Grafana
* ports: Allows access at port 3000
* volumes: Stores dashboards and settings
* environment: Default admin login credentials
* networks: Same network as Prometheus

#### Docker Network

```
networks:
  monitoring:
    driver: bridge
```

Both services communicate on the monitoring bridge network.

***

## Step 4: Understanding the prometheus.yml Configuration

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

Explanation:

* global.scrape\_interval: Prometheus scrapes every 15 seconds
* scrape\_configs:
  * job\_name: Name for this scrape configuration
  * targets: Servers Prometheus will scrape
  * Here it scrapes itself on localhost:9090

***

## Step 5: Add Prometheus as a Data Source in Grafana

1. Login to Grafana using admin / admin123
2. Navigate to:\
   Connections > Add new connection
3. Search for: Prometheus
4. Click Add new data source
5. Enter URL:

```
http://prometheus:9090
```

Why service name?\
Because both containers share the same Docker network and can resolve each other by container name.

6. Scroll down and click Save and Test\
   You should see a success message.

***

## Step 6: Verify Data Collection

#### On Prometheus

1. Navigate to:

```
http://<public-ip>:9090
```

2. In the query box, enter:

```
process_cpu_seconds_total
```

3. Click Execute\
   Metrics should appear in the graph below.

#### On Grafana

1. Login into Grafana
2. Navigate to: Explore
3. Choose Prometheus as data source
4. Enter the metric:

```
process_cpu_seconds_total
```

5. Add label filter:

```
instance = localhost:9090
```

6. Run the query\
   A graph should appear showing live CPU metrics from Prometheus.

***

## Completion

You have successfully:

1. Installed Prometheus and Grafana using Docker Compose
2. Configured Prometheus to scrape metrics
3. Added Prometheus as a data source in Grafana
4. Queried metrics in both Prometheus and Grafana

This completes the first monitoring lab setup.
