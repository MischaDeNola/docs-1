---
title: Storage Dashboard
toc: false
feedback: false
---

The CockroachDB Admin UI enables you to monitor the storage utilization for your cluster.

<div id="toc"></div>

To view the Storage metrics for your cluster, [access the Admin UI](explore-the-admin-ui.html#access-the-admin-ui), then from the Dashboard drop-down box, select **Storage**. 

#### Cluster and Node View for the Time Series Graphs
By default, the Time Series panel displays the cluster view, which shows the metrics for the entire cluster. 

To access the node view that shows the metrics for an individual node, select the node from the **Graph** drop-down list.

<img src="{{ 'images/admin_ui_select_node.png' | relative_url }}" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:40%" />

The Storage dashboard displays the following time series graphs:

## Capacity
<img src="{{ 'images/admin_ui_capacity.png' | relative_url }}" alt="CockroachDB Admin UI Capacity graph" style="border:1px solid #eee;max-width:100%" />

In the node view, the graph displays the maximum allocated capacity, available storage capacity, and capacity used by CockroachDB for the selected node.

In the cluster view, the graph displays the maximum allocated capacity, available storage capacity, and capacity used by CockroachDB across all nodes in the cluster.

You can monitor this graph to determine when additional storage needs to be provided. 

On hovering over the graph, the values for the following metrics are displayed:

Metric | Description
--------|----
Capacity | The maximum storage capacity allocated to CockroachDB.
Available | The free storage capacity available to CockroachDB.
Used | Disk space used by the data in the CockroachDB store. Note that this value is less than (capacity - available), because Capacity and Available metrics consider the entire disk and all applications on the disk including CockroachDB, whereas Used metric tracks only the store's disk usage.

{{site.data.alerts.callout_info}}If you are running multiple nodes on a single machine (not recommended), then the available capacity displayed in the graph may be incorrect. This is because when multiple nodes are running on a single machine, the machine's hard disk is treated as separate stores for each node, while in reality, only one hard disk is available for all nodes. The total available capacity is then displayed as the hard disk size multiplied by the number of nodes on the machine. {{site.data.alerts.end}}

You can configure the maximum allocated storage capacity for CockroachDB using the --store flag. For more information, see [Start a Node](start-a-node.html#store).

## File Descriptors
<img src="{{ 'images/admin_ui_file_descriptors.png' | relative_url }}" alt="CockroachDB Admin UI File Descriptors" style="border:1px solid #eee;max-width:100%" />

In the node view, the graph displays the number of open file descriptors for that node, compared with the file descriptor limit.

In the cluster view, the graph displays the number of open file descriptors across all nodes, compared with the file descriptor limit.

If Open count is almost equal to the Limit count, increase [File Descriptors](recommended-production-settings.html#file-descriptors-limit).

{{site.data.alerts.callout_info}}If you are running multiple nodes on a single machine (not recommended), the actual number of open file descriptors are considered open on each node. Thus the limit count value displayed on the Admin UI is the actual value of open file descriptors multiplied by the number of nodes, compared with the file descriptor limit. {{site.data.alerts.end}}

For Windows systems, you can ignore the File Descriptors parameter, because the concept of file descriptors is not applicable to Windows. 

{{site.data.alerts.callout_info}}The Storage dashboard displays time-series graphs for other metrics such as Live Bytes, Log Commit Latency, Command Commit Latency, RocksDB Read Amplification, and RocksDB SSTables that are important for CockroachDB developers. For monitoring CockroachDB, it is sufficient to monitor the Capacity and File Descriptors graphs.{{site.data.alerts.end}}