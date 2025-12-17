# Lab 3 - Practice PromQL queries

In this lab, we will practice PromQL queries from the data (metrics) collected from the virtual machines using the Node Exporter.

1. Prometheus and Grafana are installed and configured
2. Multiple virtual machines (at least two) are setup
3. Node Exporter installed on the virtual machines and Prometheus is pulling the metrics from the Node Exporter.

Note - All these steps were completed in the previous lab

#### Learning Objectives -

1. Learning PromQL syntax
2. Writing PromQL query to get meaningful metrics

### Steps to Follow -

Navigate to the Prometheus Server on the browser.

You can write the queries in the query field on the screen.

Let us start with running simple queries

### 1.  Get the Total memory from the Nodes -

```
node_memory_MemTotal_bytes
```

#### Query explanation&#x20;

This query returns the total amount of physical memory (RAM) available on each machine (or node) being monitored.

* The value is in bytes.<br>
* It does not change over time unless the machine's memory is upgraded.

<figure><img src="../.gitbook/assets/unknown (45).png" alt=""><figcaption></figcaption></figure>

This query is a simple, single metric query that returns the memory size of the nodes in Bytes.

### 2. Get the available memory of the nodes&#x20;

```
node_memory_MemAvailable_bytes
```

#### Query Explanation

This query shows the amount of available (free + reclaimable) memory on each server.

* It includes memory that is currently unused and memory that can be freed up quickly if needed (like cached memory).
* The value is in bytes.<br>

<figure><img src="../.gitbook/assets/unknown (46).png" alt=""><figcaption></figcaption></figure>

### 3. Used Memory from each node

Let us now use a mathematical operation to calculate the used memory from the above two queries.

```
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

#### Query Explanation

This query subtracts the available memory from the total memory to calculate how much memory is currently used on a server.

* node\_memory\_MemTotal\_bytes: Total RAM on the system.<br>
* node\_memory\_MemAvailable\_bytes: RAM that’s still available to use.<br>

### 4. CPU usage per second per node

```
node_cpu_seconds_total
```

#### Query Explanation

This metric tells you how much time the CPU has spent doing different types of work, in total, since the system started.

* The time is measured in seconds.
* It includes different modes like:
* user – time spent running applications
* system – time spent on system/kernel tasks
* idle – time the CPU was not doing anything
* iowait, irq, etc.

Each CPU core reports this separately, and the numbers just keep going up (they’re counters).

<figure><img src="../.gitbook/assets/unknown (48).png" alt=""><figcaption></figcaption></figure>

### 5. Idle CPU Memory per second per node

```
node_cpu_seconds_total{mode="idle"}
```

This query shows the total amount of time the CPU has spent in idle mode, for each core, since the system was turned on.

* mode="idle": Filters the data to show only the idle time (when the CPU is doing nothing).
* The numbers keep increasing over time because they are cumulative counters (measured in seconds).

<figure><img src="../.gitbook/assets/unknown (49).png" alt=""><figcaption></figcaption></figure>

Let us say now we want to get the average idle cpu usage per second for each node.

```
avg(node_cpu_seconds_total{mode="idle"}) by (instance)
```

#### Query Explanation -<br>

node\_cpu\_seconds\_total

This is a metric that shows how much time (in seconds) the CPU has spent doing different types of work, like idle, user, or system.<br>

{mode="idle"}<br>

We're filtering the data to only look at the idle time — the time the CPU wasn't doing any work.<br>

avg(...)<br>

Since there are usually multiple CPU cores, this takes the average idle time across all cores.<br>

by (instance)<br>

Groups the results by each machine (or server), where instance usually represents the IP and port of that machine.

<figure><img src="../.gitbook/assets/unknown (50).png" alt=""><figcaption></figcaption></figure>

### 6. Rate at which data is being received over the network (in bytes per second)

```
rate(node_network_receive_bytes_total[5m])
```

#### Query Explanation -

This query shows the rate at which data is being received over the network (in bytes per second) on each network interface of the server.

* node\_network\_receive\_bytes\_total: This is a cumulative counter that tracks the total number of bytes received on each network interface since the system started.
* rate(...\[5m]): Calculates how quickly that number is increasing (i.e., how many bytes are being received per second), averaged over the last 5 minutes.<br>

<figure><img src="../.gitbook/assets/unknown (51).png" alt=""><figcaption></figcaption></figure>

```
rate(node_disk_reads_completed_total[5m])
```

This query tells you how many disk read operations are happening per second, averaged over the last 5 minutes.

* node\_disk\_reads\_completed\_total: A counter that keeps track of the total number of times the system has read data from disk since it started.<br>
* rate(...\[5m]): Calculates how fast that number is growing — basically, the read speed in operations per second over the last 5 minutes.<br>

<figure><img src="../.gitbook/assets/unknown (52).png" alt=""><figcaption></figcaption></figure>

### 7. Percentage of memory currently being used

```
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
```

#### Query Explanation -

This query calculates the percentage of memory currently being used on a server.

* node\_memory\_MemTotal\_bytes: Total memory (RAM) on the system.
* node\_memory\_MemAvailable\_bytes: Memory that's still available for use.
* Subtracting the two gives you the used memory.
* Dividing by the total memory and multiplying by 100 gives you a percentage.

<figure><img src="../.gitbook/assets/unknown (54).png" alt=""><figcaption></figcaption></figure>

### 8. CPU usage % (excluding idle time)

```
100 - avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
```

Query Explanation&#x20;

This query calculates the overall CPU usage percentage across all CPU cores, based on the last 5 minutes.

Let’s break it down:

* node\_cpu\_seconds\_total{mode="idle"}: Tracks how much time the CPU is doing nothing (idle).<br>
* irate(...\[5m]): Measures how quickly that idle time is increasing — in other words, how idle the CPU is per second.<br>
* avg(...): Averages the idle time across all CPU cores.<br>
* 100 - ... \* 100: Subtracts the idle percentage from 100%, giving you the CPU usage (i.e., time the CPU is doing work).

### 9. Filesystem usage % per mount point

```
100 * (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes
```

#### Query Explanation&#x20;

This query calculates the percentage of disk space that is currently used on each filesystem (or mount point).

* node\_filesystem\_size\_bytes: Total size of the disk or partition.
* node\_filesystem\_free\_bytes: Amount of free (unused) space on that disk.
* Subtracting the two gives you the used space.
* Dividing by the total size and multiplying by 100 gives you the used space as a percentage.

<figure><img src="../.gitbook/assets/unknown (56).png" alt=""><figcaption></figcaption></figure>

### 10. Network transmit rate per instance

```
sum(rate(node_network_transmit_bytes_total[5m])) by (instance)
```

#### Query Explanation

This query calculates the total amount of data being sent out (transmitted) per second by each server (instance), averaged over the last 5 minutes.

* node\_network\_transmit\_bytes\_total: Tracks the total bytes sent out over all network interfaces.
* rate(...\[5m]): Measures how fast data is being transmitted per second.
* sum(... by instance): Adds up the transmit rate from all interfaces on the same server.

<figure><img src="../.gitbook/assets/unknown (58).png" alt=""><figcaption></figcaption></figure>

### 11. Top 5 mounts by used disk space

```
topk(5, node_filesystem_size_bytes - node_filesystem_free_bytes)
```

#### Query Explanation

This query finds the top 5 filesystems (disks or partitions) using the most space.

* node\_filesystem\_size\_bytes - node\_filesystem\_free\_bytes: Calculates how much space is used on each filesystem.<br>
* topk(5, ...): Picks the top 5 with the highest used space.

![](<../.gitbook/assets/unknown (59).png>)

### 12. Running processes count

```
node_procs_running
```

#### Query Explanation

This metric shows the number of processes that are currently running on a server.

* A process is any program or task the system is working on.<br>
* This value excludes sleeping or idle processes — only actively running ones are counted.

<figure><img src="../.gitbook/assets/unknown (61).png" alt=""><figcaption></figcaption></figure>

\
<br>
