# Lab 4: Build Dashboards in Grafana

In this lab, you’ll create Grafana dashboards using metrics collected by Prometheus from Node Exporter (set up in the earlier labs). You’ll turn raw time-series data into readable, actionable visuals.

Prerequisites

* Prometheus and Grafana are installed, configured, and reachable.\
  <br>
* At least two virtual machines are running.\
  <br>
* Node Exporter is installed on each VM, and Prometheus is scraping those targets (they show UP on the Prometheus /targets page).\
  <br>

Note: All of the above was completed in previous labs.

Learning Objectives

* Add/confirm Prometheus as a data source in Grafana.\
  <br>
* Query Node Exporter metrics with PromQL (CPU, memory, disk, network).\
  <br>
* Build panels (Time series, Gauge, Bar gauge) and arrange them into a dashboard.\
  <br>
* Use variables (e.g., instance) to switch between hosts in one dashboard.\
  <br>
* Set refresh intervals, time ranges, and save/share the finished dashboard.

### Steps to follow -

<br>

1. Navigate to the Grafana homepage and login the credentials as configured
2. Create a new Dashboard.

<br>

![](<../.gitbook/assets/unknown (53).png>)

\
<br>

Save the Dashboard with some name say - Virtual Machine Monitoring

<br>

![](<../.gitbook/assets/unknown (55).png>)

<br>

![](<../.gitbook/assets/unknown (57).png>)

<br>

Click on Add Visualization

<br>

![](<../.gitbook/assets/unknown (60).png>)

<br>

Select the Prometheus Source&#x20;

![](<../.gitbook/assets/unknown (62).png>)

<br>

![](<../.gitbook/assets/unknown (63).png>)

<br>

#### 1. Add your first Visualization - Available Memory on each node

Note - Follow the numbers on the screenshot below to execute each step

<br>

![](<../.gitbook/assets/unknown (64).png>)

<br>

Steps -

<br>

1. Select the visualisation type as time-series&#x20;
2. Update the title to - Available Memory on Nodes
3. Update the metric to the following

<br>

| node\_memory\_MemAvailable\_bytes |
| --------------------------------- |

<br>

Query Explanation

This query shows the amount of available (free + reclaimable) memory on each server.

* It includes memory that is currently unused and memory that can be freed up quickly if needed (like cached memory).
* The value is in bytes.

<br>

4. Execute the query.
5. Click Back to the Dashboard

<br>

Great, our first Visualisation is ready.

<br>

![](<../.gitbook/assets/unknown (65).png>)

<br>

Next, Click on Save Dashboard

<br>

Next, add another Visualisation to present “Used CPU Memory”

<br>

#### 2. Used CPU Memory

<br>

Steps -

<br>

1. Add a new Visualization and update the highlighted section below.

<br>

![](<../.gitbook/assets/unknown (66).png>)

<br>

1. Visualisation Type - Time Series
2. Title - Used Node Memory
3. To add the query, Select the  code under the query section, update the query as below.

<br>

| node\_memory\_MemTotal\_bytes - node\_memory\_MemAvailable\_bytes |
| ----------------------------------------------------------------- |

<br>

Query Explanation

This query subtracts the available memory from the total memory to calculate how much memory is currently used on a server.

* node\_memory\_MemTotal\_bytes: Total RAM on the system.
* node\_memory\_MemAvailable\_bytes: RAM that’s still available to use.

4. Execute the query - Run Queries
5. Click Back to Dashboard

\
\
\
\
\
<br>

#### 3. Running processes count

<br>

1. Add a new Visualization and update the highlighted section below.

<br>

![](<../.gitbook/assets/unknown (67).png>)

<br>

1. Update the visualization type - Stat
2. Title - Running processes Count
3. Metrics in the query section as below&#x20;

<br>

| node\_procs\_running |
| -------------------- |

<br>

Query Explanation

This metric shows the number of processes that are currently running on a server.

* A process is any program or task the system is working on.\
  <br>
* This value excludes sleeping or idle processes — only actively running ones are counted.

<br>

4. Run the query&#x20;
5. Back to Dashboard

![](<../.gitbook/assets/unknown (68).png>)

<br>

#### 4. CPU usage % (excluding idle time)

1. Add a new Visualization and update the highlighted section below.

![](<../.gitbook/assets/unknown (69).png>)

1. Update the visualization type - Gauge
2. Title - CPU usage % (excluding idle time)
3. Metrics in the query section as below&#x20;

<br>

| 100 - avg(irate(node\_cpu\_seconds\_total{mode="idle"}\[5m])) \* 100 |
| -------------------------------------------------------------------- |

<br>

**Query Explanation**

This query calculates the overall CPU usage percentage across all CPU cores, based on the last 5 minutes.

Let’s break it down:

* node\_cpu\_seconds\_total{mode="idle"}: Tracks how much time the CPU is doing nothing (idle).
* irate(...\[5m]): Measures how quickly that idle time is increasing — in other words, how idle the CPU is per second.
* avg(...): Averages the idle time across all CPU cores.
* 100 - ... \* 100: Subtracts the idle percentage from 100%, giving you the CPU usage (i.e., time the CPU is doing work).<br>

4. Run the query&#x20;
5. Back to Dashboard

![](<../.gitbook/assets/unknown (70).png>)

Great! We have built a useful dashboard you can practice adding other different type of visualizations

<br>
