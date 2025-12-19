# Lab 5 : Monitor Docker Containers with Prometheus, cAdvisor & Grafana

In this lab, you will monitor Docker containers running on a Linux VM. You will install Docker (if required), deploy a multi-tier application in containers, run cAdvisor to expose container-level metrics, configure Prometheus to scrape those metrics, and build Grafana dashboards to visualize container performance.

***

### Learning Objectives

1. Deploy cAdvisor on a VM to expose container metrics
2. Configure Prometheus to scrape container-level metrics
3. Validate metrics using PromQL in Prometheus
4. Build Grafana dashboards for container CPU, memory, network, and storage
5. Gain end-to-end visibility into container workloads

***

### Prerequisites

* Prometheus and Grafana installed and running from earlier labs
* One Linux VM (you may reuse vm1 from previous labs)
* Docker installed on that VM (or install in Step 1)
* Ansible control node configured (optional automation path)
* Completed Lab 2 on VM monitoring with Node Exporter

Keep the earlier repository handy, especially the folder `install_prometheus_agent/`.

***

## High Level Workflow

1. Install Docker on vm1
2. Deploy a three-tier sample application in containers
3. Install cAdvisor on vm1
4. Configure Prometheus to scrape cAdvisor
5. Query container metrics in Prometheus
6. Build a Grafana dashboard for containers

***

## Step 1: Install Docker on vm1

You may install Docker either using Ansible or manually.

***

### Option A: Install Docker using Ansible (recommended)

1. Open the Ansible playbook `install_docker.yml` inside `install_prometheus_agent/`:

```
---
- name: Install Docker on remote servers and configure non-sudo access
  hosts: vm1
  become: true
  tasks:
    # Docker installation tasks
```

2. Run the playbook:

```bash
ansible-playbook install_docker.yml
```

***

### Option B: Install Docker manually

On vm1, run:

```bash
docker version
docker run --rm hello-world
```

If the hello-world container prints its success message, Docker is correctly installed.

***

## Step 2: Deploy the Three-Tier Application in Containers

Clone the sample application repository:

```bash
git clone https://github.com/saurabhd2106/user-management-ansible-solution-ih.git app3tier
cd app3tier
```

***

### Configure the Ansible inventory

Edit `ansible/inventory.ini`:

```
[prometheus_agents]
vm1 ansible_host=<public_ip_address>

[all:vars]
ansible_user=<username>
ansible_ssh_private_key_file=./key.pem
ansible_python_interpreter=/usr/bin/python3.12
```

Copy your SSH key into the `ansible/` folder and ensure permission is correct:

```bash
chmod 400 key.pem
```

***

### Update UI configuration

Edit `ui/config.json`:

```
{
  "API_URL": "http://<backend_ip_address>:3000/"
}
```

***

### Deploy the application using Ansible

From inside the `ansible/` directory:

```bash
ansible-playbook final-app.yml
```

This launches the full three-tier application as Docker containers on vm1.

***

### Validate the application

Open a browser and navigate to:

```
http://<vm1_public_ip>
```

The user-management UI should load successfully.

***

## Step 3: Install cAdvisor on vm1

Install cAdvisor using the Ansible playbook from the earlier repository:

```bash
ansible-playbook install_cadvisor.yml
```

This deploys cAdvisor and exposes container metrics on port 8080.

***

## Step 4: Configure Prometheus to Scrape cAdvisor Metrics

On the VM where Prometheus is running:

1. SSH into the Prometheus VM
2. Navigate to the folder containing `prometheus.yml`
3. Add the following scrape job

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets:
          - '20.16.198.44:9100'
          - '20.16.198.73:9100'

  - job_name: 'cadvisor'
    static_configs:
      - targets:
          - '20.16.198.44:8080'
```

Replace the IP addresses with the public or private IP of vm1 where cAdvisor is installed.

Restart Prometheus:

```bash
docker compose restart prometheus
```

***

### Verify cAdvisor Target

Open Prometheus UI:

```
http://<prometheus_vm_ip>:9090
```

Go to:

Status → Targets

You should see:

* cadvisor target: UP

***

## Step 5: Test PromQL Queries for Container Metrics

In the Prometheus UI, open the Graph tab and test the following queries.

#### CPU usage (per container)

```
rate(container_cpu_usage_seconds_total{image!=""}[1m])
```

#### Memory working set

```
container_memory_working_set_bytes{image!=""}
```

#### Filesystem usage

```
container_fs_usage_bytes{image!=""}
```

#### Network traffic (per second)

```
rate(container_network_receive_bytes_total{image!=""}[1m])
rate(container_network_transmit_bytes_total{image!=""}[1m])
```

If these queries return time series data, cAdvisor and Prometheus are working correctly.

***

## Step 6: Build a Grafana Dashboard for Container Monitoring

1.  Open Grafana:

    ```
    http://<grafana_vm_ip>:3000
    ```
2. Go to:\
   Dashboards → New → Import
3. Choose one of the following options:

#### Option A: Import an existing cAdvisor dashboard

Upload a downloaded JSON file.

#### Option B: Create your own dashboard

Start with these recommended panels.

***

### Suggested Grafana Queries

#### Top 5 containers by CPU usage

```
topk(5, sum by (container_label_com_docker_compose_service) (
  rate(container_cpu_usage_seconds_total{image!=""}[1m])
))
```

#### Top 5 containers by memory usage

```
topk(5, sum by (container_label_com_docker_compose_service) (
  container_memory_working_set_bytes{image!=""}
))
```

#### Container count

```
count(count by (id) (container_last_seen{image!=""}))
```

#### Network receive

```
sum by (container_label_com_docker_compose_service) (
  rate(container_network_receive_bytes_total{image!=""}[1m])
)
```

#### Network transmit

```
sum by (container_label_com_docker_compose_service) (
  rate(container_network_transmit_bytes_total{image!=""}[1m])
)
```

Tip: Label names may vary depending on Docker version. Use Grafana Explore to inspect labels such as:

* container
* name
* id
* container\_label\_com\_docker\_compose\_service

Adjust your groupings accordingly.

***

## Success Criteria

Your implementation is complete when:

1. cAdvisor appears UP in the Prometheus targets list
2. CPU, memory, filesystem, and network PromQL queries return container-level metrics
3. Grafana displays live container dashboards, including Top-N CPU and memory consumers
4. The three-tier application containers are visible in the dashboards

***

## Troubleshooting Guide

#### Prometheus shows cAdvisor target DOWN

* Ensure port 8080 is open on vm1
* Check cAdvisor logs: `docker logs cadvisor`
* Verify the IP in prometheus.yml is correct
* Restart Prometheus

#### Grafana shows no data

* Test PromQL in Prometheus first
* Verify Prometheus datasource URL in Grafana
* Adjust label names based on Explore output

#### cAdvisor shows 0 containers

* Ensure required bind mounts exist for Docker
* If Docker root directory is customized, mount the correct directory

#### Application not reachable

* Check with `docker ps` whether containers are running
* Verify port mappings and firewall settings

***

## Completion

You have successfully monitored Docker containers using:

* cAdvisor for container metrics
* Prometheus for metric scraping
* Grafana for visualization
