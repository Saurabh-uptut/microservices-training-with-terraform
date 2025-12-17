# Lab 2: Monitor Multiple Virtual Machines with Prometheus, Node Exporter and Grafana

In this lab, you will provision virtual machines, deploy Node Exporter on them, configure Prometheus to scrape metrics from each VM, and build a Grafana dashboard to visualize CPU, memory, disk, and network usage across your infrastructure.

***

### Learning Objectives

1. Provision multiple VMs using Terraform
2. Install Node Exporter on all VMs using Ansible
3. Configure Prometheus to collect metrics from each VM
4. Visualize VM metrics in Grafana using a prebuilt dashboard

By the end of the lab, you will have a complete end-to-end monitoring solution for multiple virtual machines.

***

### Prerequisites

* Prometheus and Grafana installed on a separate monitoring VM
* Terraform installed and configured
* Ansible installed and configured
* Network connectivity between Prometheus and each VM on port 9100
* SSH connectivity configured to allow Ansible access

***

## Step-by-Step Workflow

You will complete the lab in three major stages:

1. Provision two virtual machines using Terraform
2. Install Node Exporter on both machines using Ansible
3. Update Prometheus configuration and build Grafana dashboards

All tasks are performed from a VM that already has Terraform and Ansible installed.

***

## Step 1: Clone the Repository and Prepare Files

Clone the lab repository containing Terraform and Ansible automation.

```bash
git clone <repository-url>
cd <repository-folder>
```

This repository contains the automation for future labs as well.

***

## Step 2: Provision Virtual Machines Using Terraform

1. Navigate to the Terraform solution folder:

```bash
cd provision_vm_terraform/local_solution
```

2. Open the file `local.tf` and update the prefix variable. Example:

```hcl
prefix = "devops2-week8-prometheus-lab"
```

This prefix is used to name resource groups, network interfaces, and other components.

3. Initialize Terraform:

```bash
terraform init
```

4. Preview the infrastructure:

```bash
terraform plan
```

5. Create the VMs:

```bash
terraform apply --auto-approve
```

6. Retrieve the public IP addresses of the VMs:

```bash
terraform output public_ip
```

Two IPs will be displayed for VM1 and VM2.

7. Retrieve the SSH private key and save it:

```bash
terraform output tls_private_key >> key.pem
```

Move the key into the Ansible folder:

```bash
mv key.pem ../../install_prometheus_agent
```

Set correct permissions:

```bash
cd ../../install_prometheus_agent
chmod 400 key.pem
```

***

## Step 3: Configure the Ansible Inventory

Edit the `inventory.ini` file and update it with the public IP addresses of the newly provisioned VMs.

```
[prometheus_agents]
vm1 ansible_host=20.16.198.44
vm2 ansible_host=20.16.198.73

[all:vars]
ansible_user=azureuser
ansible_ssh_private_key_file=./key.pem
ansible_python_interpreter=/usr/bin/python3.12
```

Test SSH connectivity:

```bash
ssh -i ./key.pem azureuser@<public-ip>
```

***

## Step 4: Verify Ansible Connectivity

Check connectivity to both hosts:

```bash
ansible all -m ping
```

If successful, proceed to install Node Exporter.

***

## Step 5: Install Node Exporter on All VMs

Run the playbook provided in the repository:

```bash
ansible-playbook install_agent.yml
```

This installs and configures Node Exporter on port 9100 for both VMs.

***

## Step 6: Update Prometheus Configuration

Now connect to the VM where Prometheus and Grafana are installed.

```bash
ssh <prometheus-vm-user>@<prometheus-vm-ip>
```

Navigate to the folder where the monitoring configuration files are stored.

Update the `prometheus.yml` file to add the new scrape targets:

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['20.16.198.44:9100', '20.16.198.73:9100']
```

Remember to update the IPs with the ones provisioned using Terraform.

Make sure that port 9100 is open. The Terraform module automatically configures this.

***

## Step 7: Restart Prometheus

Restart only the Prometheus service:

```bash
docker compose stop prometheus
docker compose start prometheus
```

***

## Step 8: Verify Metrics Collection

#### In Prometheus

Open the Prometheus UI:

```
http://<prometheus-ip-address>:9090
```

Run the following query:

```
node_cpu_seconds_total
```

You should see time series from both VM1 and VM2, confirming metrics ingestion.

***

## Step 9: Build a Multi-VM Dashboard in Grafana

1. Download the official Node Exporter Full dashboard template.\
   The commonly used template is available online.\
   The current version reference in this lab is **version 40**.
2. Save the JSON file to your local system.
3. Access the Grafana UI:

```
http://<prometheus-ip-address>:3000
```

Login with your credentials.

4. Navigate to:

Dashboard > New > Import

5. Upload the downloaded JSON file.
6. Choose **Prometheus** as the data source.
7. Click **Import**.

Grafana will import a complete multi-VM monitoring dashboard.

This dashboard contains:

* CPU load
* RAM usage
* Disk utilization
* Network throughput
* System uptime
* Node Exporter metrics grouped per instance

You can filter by host using the instance dropdown at the top of the dashboard.

***

## Completion

You have successfully completed the full lab by:

1. Creating multiple VMs with Terraform
2. Installing and configuring Node Exporter using Ansible
3. Updating Prometheus to scrape metrics from all VMs
4. Verifying metrics collection via Prometheus
5. Importing and visualizing metrics using Grafana dashboards

You now have a multi-server observability setup suitable for production-like monitoring environments.
