# Delta Lake Merge Optimizations on Structured Streaming

**Author**: Himanshu Amodwala (hiamodwala@microsoft.com)

[[_TOC_]]

## Problem Context

Customer (Cx) approached us to build a near real-time data pipeline that is able to capture the change data from their on-premises SAP ERP system and push it on to an ADLS Bronze layer with a defined (and rigid) business SLA of 30 seconds because that's their current on-premises SLA and they achieve it by using SAP HANA's in-memory capability. Moreover, to define the magnitude of the problem statement, the source table in question has approximately 7 billion records and is increasing at a very rapid rate (It grew by another billion records by the time we completed this experiment in a couple of weeks).

## High Level Architecture Diagram

![image.png](/.attachments/image-cf09579a-f957-4530-9409-ef330fc920fb.png)

**Commentary:**

_On-Premises:_

Change Data from SAP ERP systems are captured by Oracle Golden Gate and pushed to On-Premises Kafka Topic in real time. Each Kafka Topic is configured with 5 Partitions and has a retention period of 3 days.

_Connectivity:_ 

Cx has a dedicated Express Route Connectivity between their On-Premises Data Center and Azure Region that terminates on their Virtual WAN on the Hub Subscription (Following Enterprise Scale Landing Zone Architecture), there are peering's established between the Hub and Spoke Subscriptions Vnet. 

_Data Platform:_

Synapse Workspace is hosted in a Managed VNet and has Data Exfiltration Protection turned on ergo we have setup a Forwarding VNet to connect Synapse Workspace to their On-Premises Data Sources (Please follow this [article](https://techcommunity.microsoft.com/t5/azure-synapse-analytics-blog/connect-dep-enabled-synapse-spark-to-on-prem-apache-kafka/ba-p/3379023) from my colleague explaining the steps).

_Consumption Layer:_

Trino is hosted on AKS ([article]()) to provide the data analysts SQL Interface on top of Delta Lake and Consumption via PowerBI Desktop / Server Located On-Premises for wider distribution.

## Cx's Initial Approach

Historical Data Load: Cx Loaded all the Historical Data (approx. 7 Bn records) in Delta Format on ADLS Gen 2 using Parquet dumps extracted from their on-premises systems and Synapse Spark Notebooks.

![image.png](/.attachments/image-9b75b3d5-c0ed-4e8c-8bf3-21d08927bf68.png)

<p align=center> Note: Code Snippets are for illustration purposes

Data Processing: Data from On-Premises Kafka Topic is read by Synapse Spark Pools using Spark Structured Streaming and merged with Historical Delta Table sitting on ADLS Bronze layer during every Micro-batch.

![image.png](/.attachments/image-ed656234-2441-46d0-972c-2b3153e3c883.png)

![image.png](/.attachments/image-43d6282b-6d30-492e-91b3-55a559dc7d5e.png)

![image.png](/.attachments/image-b00a3959-8252-401c-b85e-ee1f3396fe69.png)

<p align=center> Note: Code Snippets are for illustration purposes

Spark Environment:

- Synapse Spark 3.1 Runtime
- Scala 2.12
- Driver Size: 8 Cores & 64 GB RAM (Medium Pool)
- Executor Size: 8 Cores & 64 GB RAM (Medium Pool)
- Executor Count: 5
- Intelligent Cache Turned On & at 50% capacity of the cluster

### Outcomes of Initial Approach

- The Processing Time for each MicroBatch was in minutes (approximately 21 minutes) which violated their 30 seconds business sla.
- Increased Lag on Kafka Topic in order of several hundred thousand to millions of offsets because of late processing times.
- Jobs would often crash due to OOM errors.

## Optimizations Performed

Post understanding the Cx Problem Statement, Business Rationale / Motivation for using Spark & Delta as a potential replacement for SAP HANA, Microsoft Synapse Performance Optimization documentation ([link](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/apache-spark-performance)) and going through the codebase through a fine-tooth comb, we categorized the optimizations to be made under following buckets (not specifically following this particular order, most of the times 2 or more of the optimizations would overlap)

- Delta Lake Optimizations
- Scala Code Optimizations 
- Spark Cluster Optimizations
- Kafka Optimizations

### Delta Lake Optimizations

_Main Delta Table Optimizations:_ The Historical Data Delta Table that was created lacked best practices ([documentation](https://docs.delta.io/latest/best-practices.html#:~:text=Best%20practices%201%20Choose%20the%20right%20partition%20column,a%20Delta%20table.%20...%204%20Spark%20caching%20)) around Partitioning, Optimize & Z-Order. So we recreated the delta table by identifying the partition, optimize / z-order columns. The characteristics of a good partition column are

- If the cardinality of a column will be very high, do not use that column for partitioning. For example, if you partition by a column userId and if there can be 1M distinct user IDs, then that is a bad partitioning strategy.
- Amount of data in each partition: You can partition by a column if you expect data in that partition to be at least 1 GB and most importantly
- Query Predicate: Try to partition your data based on heavily used columns in your query where clauses to achieve data skipping.

_Data Partitioning_: Based on the above selection criteria and extensive discussion with the Business Teams, we picked 2 columns for partitioning the data, ColumnA and ColumnB (names redacted to protect identity) which are frequently used as query predicates and has a cardinality of about 40,000 and 10,000 respectively (Given the volume of Data i.e., approx. 7 Bn records it justifies the cardinality numbers). 

![image.png](/.attachments/image-ef47e63a-d318-48e9-8dbc-8702646108c2.png)
<p align=center> Note: Code Snippets are for illustration purposes

_Delta Optimize_: We also performed an Optimize on the mainDeltaTable and selected another frequently queried column i.e., ColumnC (name redacted to protect identity) with a cardinality of 42 million for ZOrdering to colocate the data. We had to do this on Synapse Spark 3.2 Runtime .

![image.png](/.attachments/image-8951831f-180d-41b9-b017-167461a0efe9.png)
<p align=center> Note: Code Snippets are for illustration purposes

_Compaction & Vacuum_: Given the volume of the data and the mechanism of capturing incremental data aka. streaming data coming from micro batches, we also setup a separate job running every 12 hours that compacts the smaller parquet files into manageable chunks for faster data reads and vacuums the delta table to keep last 7 days history for time travel.

![image.png](/.attachments/image-dd8c5c34-e588-4af9-885d-f873a08c1464.png)

![image.png](/.attachments/image-93682895-44dd-4411-a5f8-bd270e840838.png)

<p align=center> Note: Code Snippets are for illustration purposes

### Scala Code Optimizations

Merge Optimizations:

_Partition Pruning_: Based on our discussions with the developers and stakeholders and as a recommended best practice we decided to put partition pruning to use. We are using the same columns as query predicates for merge as that we used for partitioning the main delta lake. Moreover, we are also filtering out the distinct values of the partitioning columns and passing along into the merge code for data skipping effectively making merge go faster. 

_Broadcasting_: In order to reduce the data shuffles between the executors we broadcasted the micro batch dataframe.

![image.png](/.attachments/image-43d6282b-6d30-492e-91b3-55a559dc7d5e.png)
<p align=center> Pre-Optimization

![image.png](/.attachments/image-31307e34-8206-44b3-a692-6e5b52bf6093.png)
<p align=center> Post-Optimization
<p align=center> Note: Code Snippets are for illustration purposes

ReadStream Optimizations: 

We also observed that the data coming into the Kafka topic had a variable throughput (spiking significantly during peak business hours). In order to have a better control over the sudden spike in data loads we also added properties of maxOffsetPerTrigger, minOffsetPerTrigger and maxTriggerDelay (documentation [link](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html))

![image.png](/.attachments/image-29628352-7b80-4b3c-af5a-fe7340e0a8e7.png)

Spark Configuration Optimizations:

Since we were broadcasting the micro batch dataframe and as a best practice, it made sense to increase the default value of autoBroadcastJoinThreshold from default 10 MB to 1 GB.

![image.png](/.attachments/image-912032f5-08bf-4d36-a0a2-c36459494692.png)

### Kafka Optimizations

Cx's Kafka admin team initially created the Kafka topic with few partitions that caused an under-utilization of spark executors during read stages and limited the parallelization capabilities. To increase the read performance and effectively combating the lag on the topic, it is always a good practice to have a decent number of partitions that reciprocates the data throughput received by the topic and also matches the number of executors. 

In our example, given the very high input throughput of the data, business sla requirements of 30 seconds and 5 topic partitions, it was advised to increase the number of partitions to 15 to better balance the incoming data across each partition.

### Spark Cluster Optimizations

We originally planned for 15 medium executors (8 cores per executor, 1 executor per topic partition) which increased the read parallelization and provide the necessary compute for the merge operation during every micro batch. However, looking at the Spark UI, there were too many tasks generated during the merge operation and so to effectively control the processing time for the tasks, we gradually increased the executors to 26 which provided a sweet spot around processing the tasks within time and cost requirements. We also increased the Intelligent cache to 50% to better help synapse spark runtime manage the delta table reads from ADLS ([link](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/apache-spark-intelligent-cache-concept#when-to-use-the-intelligent-cache-and-when-not-to))

```
Effective Spark Cluster Configuration:

Driver: 1 count, Medium Node, 8 cores, 64GB RAM. Total 8 Cores
Executors: 26 count, Medium Node, 8 cores 64GB RAM. Total 208 Cores
```
Note: For the above process, we also tried using heavier nodes (Large 16 cores, 128GB RAM & XLarge Nodes 32 cores, 256GB RAM) keeping the overall cores constant across the experiment's i.e., 208 cores and it was observed that their utilization spiked momentarily during the merge operation but for rest of the spark job they were heavily underutilized and lacked read parallelization due to lower executor counts. This observation syncs up with Synapse Spark Performance Optimizations documentation ([link](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/apache-spark-performance#select-the-correct-executor-size)) 

## Outcomes of Optimizations

_ToDo_:

We met the Business SLA for 30 seconds (exactly 30 seconds)

![image.png](/.attachments/image-4fa2ec84-e185-4403-ae14-30ffff275aaa.png)
