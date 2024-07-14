---
tags:
  - APP/CLICKHOUSE
source: https://medium.com/vimeo-engineering-blog/clickhouse-is-in-the-house-413862c8ac28
---




# ClickHouse is in the house



## Insights gained and lessons learned from our long video analytics migration journey.

Video analytics is used by millions of Vimeo users, ranging from individual content creators to large enterprises, facilitating in-depth insights into viewing patterns for hundreds of millions of videos. With real-time data spanning any desired time frame, our analytical platform beautifully handles over a billion viewing log events every day.
In this post, I’ll outline our journey from a traditional architecture anchored in  [Apache Phoenix](https://phoenix.apache.org/)  [HBase](https://hbase.apache.org/)  to our embrace of  [ClickHouse](https://clickhouse.com/docs/en/development/architecture)  just eighteen months ago. Along the way, I’ll pepper in key learnings and tips, making this a read you won’t want to skip.
![](https://miro.medium.com/v2/resize:fit:700/0*RpkkQeCgFAjDSvUj) From Copenhagen Airport in Kastrup, Denmark. Photo by  [Nazrin Babashova](https://unsplash.com/@kurokami04?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)  [Unsplash](https://unsplash.com/photos/silver-and-black-love-free-standing-letters-6jaeAHB4ioU?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) 


# What is video analytics data?

Video analytics are broken down into user sessions. Think of these as individual viewing experiences, characterized by dimensions like device, location, and source to derive viewing metrics such as views, view time, and much more. Keep in mind, a single user can have multiple sessions for a video — they might watch a portion, pause, then resume later or even revisit days later.
On the back end, we capture the micro events that shape each session in real time: plays, loads, regular heartbeats, and user actions. This granular data enables robust video metric construction, enabling aggregation over any time frame and refinement using the myriad of dimensions we collect.


# What problem are we trying to solve

For years, our analytics platform rested on a widespread architectural approach: an ACID key-value storage leveraging HBase, which is a Hadoop JVM-based database, with Apache Phoenix handling the queries. Such setups were the standard in analytical realms, similar to platforms like  [BigQuery](https://cloud.google.com/bigquery)  and sharded  [Postgres](https://www.postgresql.org/)  databases.
Challenges arose as our needs expanded. With increased usage, a growing list of use cases, and more advanced analytical features, we grappled with frequent issues like HBase Garbage Collection hang-ups, region server failures, and regular query timeouts. Our enterprise users, and even some non-enterprise users, faced hindered analytics capabilities, and the system struggled to handle the sheer volume of data ingestion, query demands, aggregations, and feature requests, all without breaking the bank. Balancing performance with cost became a serious challenge, as scaling and support became prohibitively expensive and the ROI increasingly diminished.


# Evaluating alternatives

There’s a saying, “If it ain’t broke, don’t fix it”. For us, it was quite the opposite. Our analytics system was going down the drain, and with our commitment to maintaining our SLA/SLO, modernization wasn’t just a desire — it was an imperative. This catalyzed our pursuit for alternatives.
Our deep dive into research funneled us toward OLAP-based databases, pinpointing key criteria to ensure alignment with both our present and future demands:
-  **Compression** . Can it efficiently handle petabytes of session logs?
-  **Query performance** . Does it cater to current and future query requirements?
-  **Throughput** . Is it capable of managing more than 1 billion session log events daily?
-  **Usability and maintenance** . Does it have a strong open-source community and tool availability?
-  **Infrastructure** . Can it meet our requirements on a host of criteria, everything from CPUs and memory to  [Kubernetes](https://kubernetes.io/)  compatibility and non-JVM preferences, given our previous garbage collection woes.
-  **Cost effectiveness** . While all criteria matter, the balance must lead to sustainable and reasonable costs long-term

We narrowed down our search to four solutions that best matched our use cases with different characterization and contrast to evaluate it from our context above. The contenders were  [Apache Druid](https://druid.apache.org/)  [MemSQL/SingleStoreDB](https://www.singlestore.com/) , enhanced HBase and pre-aggregations, and ClickHouse.
For our proof of concept, we used 25 percent of our entire data. We began with a modest machine count and scaled progressively, monitoring performance. We tested extensive data ingestion, realistic user query loads, unique user cases, and extreme scenarios. We assessed query performance using quantiles to gain a full-spectrum understanding from lightweight to heavy-duty users. Additionally, we evaluated data storage efficiency post-migration and during real-time ingestion. We didn’t stop at performance; resilience was tested through simulated failures, data loss, backup scenarios, and more. Throughout, we iteratively fine-tuned to ensure we were comparing apples to apples.
In the post-evaluation of each database against our criteria (with metrics ranging from query performance to cost), ClickHouse emerged as the unrivaled frontrunner. It excelled across the board, even astonishingly so in certain domains, and proved more cost-efficient. When pitted against Apache Phoenix on HBase, ClickHouse queries were a lightning-fast 10✕ improvement, with storage efficiency magnitudes better at 2–3✕. What used to take many minutes with Phoenix took mere seconds with ClickHouse.


# Setting up ClickHouse Infrastructure on Kubernetes

Our sessions data traces back to 2007, amounting to many petabytes of redacted analytics data, or over 100 billion sessions annually. Figure 1 dives into our setup.
![](https://miro.medium.com/v2/resize:fit:700/0*uzjARFzQUbRljxb5)  **Figure 1.**  Overview of our ClickHouse infrastructure setup, with Apache ZooKeeper, ClickHouse shards and replicas, ClickHouse Kubernetes operation and configs, storage policies, and even Velero for backup of persistent volumes.


## But how do you store it?

In any distributed setup like ClickHouse, fault tolerance is a must-have. How is this accomplished? By having at least one extra replica. For better read and write speeds? That’s where sharding steps in, spreading the work across all ClickHouse instances. So, for our needs, we have a ClickHouse cluster with dozens of shards (not as many as you’d guess!) and two replicas. Add to that a mix of several terabytes of persistent disk, 500 GB solid-state drives, and support from five  [Apache ZooKeeper](https://zookeeper.apache.org/)  nodes, and we’re set for our analytics load.


## Amazing, but how do you deploy it?

We rely on the open-source  [Kubernetes ClickHouse operator](https://github.com/Altinity/clickhouse-operator)  by Altinity. This operator oversees the creation, upgrades, configurations, and management of ClickHouse clusters on Kubernetes. It adeptly manages changes related to pod and service templates, server configurations, user management, and more. We’ve deployed ClickHouse on a self-managed Kubernetes system for  [Google Cloud Platform](https://cloud.google.com/)  with the specified configurations. One essential aspect was ensuring mounted persistent space for transaction logs to guarantee recovery post-restarts. Our strategy also considers shard anti-affinity, ensuring shards and replicas are in separate zones for resilience.


## Can you make it even better?

Absolutely. ClickHouse offers a myriad of configurations, some of which are still experimental. One method to optimize cost and performance is through multi-tier storage policies. Each  ` **MergeTree** `  family table in ClickHouse can be associated with a specific storage policy, effectively directing how and where your data is stored, organizing the policy group disks into one or more volumes of data, and defining write order within a volume and data moves between volumes (see Figure 2).  ` **MergeTree** `  in ClickHouse world is a family of engines for inserting large volumes of table data. To learn more, consult the  [ClickHouse docs](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree) 
![](https://miro.medium.com/v2/resize:fit:600/0*S91TajKW8Rpb8GXb)  **Figure 2.**  A multi-tier storage policy structure indicating how and where data is stored.
Breaking it down: to streamline costs and enhance performance, recent data can be designated to solid-state drives (which are pricier), while older data can be routed to less expensive and slightly less performant disks, based on defined criteria.


# Data ingestion: what and how?

Before I jump into data modeling, let me quickly review the type of data we’re feeding into ClickHouse and how we’re doing it.


## Understanding the data flow

Our system is flooded with session events: loads, heartbeats, plays, finishes, and user interactions. To boost query and storage performance, we send the differences, or  *deltas* , of these sessions as events come in. In short, metrics like total view time are delivered as increments rather than as one lump sum.
Consider a 90-second viewing session: instead of a single 90-second data point, it’s sent in three 30-second bursts, leading to an overall 90 seconds.


## Apache Spark’s role

Given ClickHouse’s mainly append-only behavior and non-ACID nature, we had to rethink our data pipelines. This is where Apache Spark steps in, helping us calculate and send deltas for updates. We’ve set up stateful structured streaming jobs in Apache Spark version 3.3 to store each session’s changes and forward these deltas to ClickHouse. The result? Simplified and faster queries without the need for complex subqueries.


## So how do we insert?

By utilizing ClickHouse’s  [null tables](https://clickhouse.com/docs/en/engines/table-engines/special/null) , that’s how. When writing to a null table, data is ignored, not stored, so we direct our ingestion straight from Spark jobs via JDBC to a load balancer endpoint. This is bolstered by reducing insertion parallelism and increasing batch insertions, optimizing ClickHouse memory allocation and managing data fragmentation.
Here’s an example of a null table:

```
CREATE TABLE vimeo.session_event_ingestion ON CLUSTER analytics
(
    video_owner_id UInt32,
    event_ts UInt64,
    video_id UInt32,
    session_id String,
    country Nullable(FixedString(2)),
    device Nullable(String),
    source_domain Nullable(String),
    ...
    live Nullable(UInt8),
    ...
    )
    ENGINE = Null;
```


Beyond just the mechanics,  [materialized views](https://clickhouse.com/docs/en/guides/developer/cascading-materialized-views)  come into play by inserting into a distributed engine table by sharding keys for adept data distribution within ClickHouse. This setup empowers ClickHouse to distribute data efficiently, freeing our ingestion pipeline from the nuances of distribution. I’ll touch upon materialized views in detail in upcoming sections.


## Is it that easy to ingest?

Ingestion isn’t just about shuttling data. Factors like replication queue, ingestion triggers, and potential duplications due to upstream pipeline hiccups demand vigilance. To combat potential data duplication, we use the  `insert_deduplicate=1`  flag to ensure data uniqueness in our ingestions. For more information on this, see the  [Altinity Knowledge Base](https://kb.altinity.com/altinity-kb-schema-design/insert_deduplication/) 
Every insertion into tables (except null tables) crafts a part. Frequent insertions within a short span can lead to an excessive number of parts. This, in turn, can overload the replication queue, trigger ZooKeeper exceptions, and jeopardize the stability of the entire cluster. Balancing the size of parts, which influences memory and performance, with the quantity of parts is vital, especially as it significantly impacts the replication queue and ZooKeeper operations.


# Data modeling and optimization

This is where the ClickHouse magic happens. Your choices around table engines, configurations, schema, index granularity, codec compressions, TTLs, and key selections are game-changers. Fine-tuning these elements can vastly enhance performance while trimming costs. Figure 3 shows a simplified diagram from a null table that branches out to various user-centric tables.
![](https://miro.medium.com/v2/resize:fit:700/0*P8LaEm0PMukEbPzb)  **Figure 3.**  ClickHouse data modeling overview from Spark ingestion to null tables to other aggregated tables by various materialized views.
Our model includes tables categorized by session, video, and hourly metrics, among other key dimensions. Additionally, we maintain a comprehensive video summary table, aggregating metrics from day one.


## OK, what engine?

From this discussion, it’s evident we lean towards the replicated merge tree engine family. Specifically, we employ the `ReplicatedAggregatingMergeTree`  engine to seamlessly aggregate events into sessions, distributed across shards and replicas by a sharding key that’s encapsulated in the distributed table engine. Here’s a sneak peek:

```
CREATE TABLE vimeo.session_local ON CLUSTER analytics
(
    video_owner_id UInt32 ..,
    video_id UInt32 ..,
    event_ts  DateTime ..,
    view_time SimpleAggregateFunction(sum, Int64)....,
    views SimpleAggregateFunction(sum, Int64) ...,
    ....
    )
    ENGINE = ReplicatedAggregatingMergeTree
    PARTITION BY ..
    PRIMARY KEY ...
    ORDER BY ...
    TTL ts + ... DELETE
SETTINGS index_granularity = ...;
```


These events or ClickHouse parts eventually converge, aggregating based on the functions specified for each column.
The importance of the distribution table cannot be overstated. It’s a delicate dance to distribute data efficiently across machines while minimizing skewness, a challenge often seen with power users. Striking this balance directly influences query and storage performance. Diversifying data too much can strain the replication queue, increase network hops, and hamper query performance. The balance becomes particularly critical when distributing data based on logical keys such that optimal distribution ensures that fewer shards are queried, which streamlines data retrieval. Here’s an example:

```
CREATE TABLE vimeo.session ON CLUSTER analytics AS vimeo.session_local
    ENGINE = Distributed(analytics, vimeo, session_local, cityHash64(video_owner_id,toStartOfHour(event_ts)));
```




## What more can we do?

Our journey with ClickHouse taught us that details matter, and this is particularly true when optimizing data storage and retrieval.
One of our pivotal strategies is  [column compression](https://altinity.com/blog/2019-7-new-encodings-to-improve-clickhouse) . There are more than enough compression algorithms available like DoubleDelta, Delta, and Gorilla; for example,  `CODEC(ZSTD(1))` . Gorilla is excellent for time series, by the way. However, the selection largely depends on the data’s nature — its sparsity, cardinality, and behavior over time.
But if there’s a crown jewel in the realm of query performance, it’s the art of partitioning and key selection. Here’s a concise guide:
-  ` **PARTITION BY** `  ****  This is about data relativity. A single partition should ideally span 1–300 GB of data (or 400 MB to 40 GB for specific aggregations). Aim for single insertions to populate one or just a few partitions and for typical  `SELECT`  queries to touch minimal partitions. Overall, the fewer partitions, the better.
-  ` **ORDER BY** `  ****  ` **PRIMARY KEY** `  ****  Their arrangement is the most important. Opt for one to four columns, organized by cardinality — from the lowest to the highest. The golden rule? The columns most queried and filtered should come first. In ClickHouse, the order directly affects the efficiency of data merges, making this a key aspect to tune in for optimal performance.

An example follows:

```
CREATE TABLE vimeo.session_local ON CLUSTER analytics
(
    video_owner_id UInt32 CODEC(ZSTD(1)),
    video_id UInt32 CODEC(ZSTD(1)),
    event_ts  DateTime CODEC(ZSTD(1)),
    view_time SimpleAggregateFunction(sum, Int64) CODEC(ZSTD(1)) TTL ts + INTERVAL 1 MONTH,
    views SimpleAggregateFunction(sum, Int64) CODEC(ZSTD(1)) TTL ts + INTERVAL 1 MONTH,
    ....
    )
    ENGINE = ReplicatedAggregatingMergeTree
    PARTITION BY toMonth(ts)
    PRIMARY KEY (video_owner_id, toStartOfDay(event_ts), video_id)
    ORDER BY (video_owner_id, toStartOfDay(event_ts), video_id, event_ts)
    TTL ts + toIntervalYear(1) DELETE
SETTINGS index_granularity = 8192
```


Other settings can impact read/write performance, too, such as index granularity.


## Are we missing anything else important?

ClickHouse’s materialized views, which are similar to insertion triggers, are truly revolutionary. Acting in real-time, they aggregate batches of rows from only newly inserted parts and channel them into separate tables. These views shine brightest when tasked with pre-aggregating data, whether that’s time-based like daily aggregates or higher-dimensional aspects like region or device. The payoff? Stellar query performance achieved by interrogating fewer, already aggregated data rows.
To truly harness the potential of materialized views, consider these data optimization techniques:
-  **Data type** . For dimensions boasting less than 10,000 unique values, the  ` **LowCardinality** `  data type can work wonders.
-  **H3 index** . Working with geographic coordinates? Convert latitude and longitude to the concise  [H3 index](https://h3geo.org/docs/highlights/indexing/)  using the  ` **geoToH3** `  function.
-  **UUID optimization** . For UUID strings, the native  ` **** `  type is a no-brainer.
-  **String hashing** . Long strings, especially those serving aggregation or uniqueness purposes, are better transformed into integers using functions like  ` **sipHash64** `  **** 
-  **Hex string compression** . Lengthy hex strings? Halve their size using functions like  ` **unhex** `  **** 

How many of these can you find in the following example?

```
CREATE MATERIALIZED VIEW vimeo.session_mv ON CLUSTER analytics TO vimeo.session
    AS
SELECT
    video_owner_id,
    video_id,
    fromUnixTimestamp(toUInt64(event_ts/1000)) AS event_ts,
    session_id,
    toLowCardinality(coalesce(country, '')) AS country,
    coalesce(geoToH3(longitude, latitude, 15), 0) AS h3Index,
    coalesce(domainWithoutWWW(referer),'') AS source_domain,
    ....
    toUUIDOrZero(lead_id) AS lead_id,
    if(coalesce(viewer_unique_id,'') = '', 0, sipHash64(viewer_unique_id)) AS viewer_unique_id,
   ....
FROM vimeo.session_event_ingestion;
```




## What about uniqueness?

Addressing uniqueness in large-scale database systems is notoriously resource-intensive. Enter ClickHouse with its  [HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog)  utility, a boon for such challenges. Here’s how to effectively harness it:
1.  Instead of using strings, convert the data to unsigned integers using hashing functions like  ` **sipHash64** `  ****  This elevates performance.
2.  Transform your unsigned integers into a HyperLogLog state via  ` **uniqCombinedState(num)** `  ****  The  **  here stands for HyperLogLog precision, which defaults to 17. This translates to an effective 96 KiB of space (given 217 cells at 6 bits each).

For your  ` **ReplicatedAggregatingMergeTree** `  table, define the unique column as  `AggregateFunction(uniqCombined(15),Nullable(UInt64))`  `UInt64` here corresponds to your original column type, for example  `num=15` 
For more information on the  ` **uniqCombined** `  function, see the  [ClickHouse docs](https://clickhouse.com/docs/en/sql-reference/aggregate-functions/reference/uniqcombined) 
What about the materialized view transformation? When mapping to  `state(uniqCombinedState(15))`  in your materialized view, it should look somewhat like this:

```
CREATE MATERIALIZED VIEW vimeo.video_summary_mv ON CLUSTER analytics TO vimeo.video_summary AS
SELECT
    video_owner_id,
    video_id,
    sum(views) as views,
    ..
    uniqCombinedState(15)(sipHash64(viewer_unique_id)) AS uniqe_views
FROM vimeo.session_event_ingestion
GROUP BY
    video_owner_id,
    video_id
```


This process translates your data into a HyperLogLog state. During subsequent operations, the aggregation table combines these states.
What about querying the uniqueness table? To extract data, run the following with the desired precision:

```
select ...,uniqCombinedMerge(15)(uniqe_views) as uniqe_views 
from vimeo.video_summary 
where ....
```




# Data migration and archiving

Migrating data from a fully operational HBase cluster presented a unique set of challenges. Nevertheless, using the tool described in my  [previous blog post](https://medium.com/vimeo-engineering-blog/too-big-to-query-how-to-query-hbase-with-minimal-pain-42221f9a8623) , we extracted data from HBase seamlessly, ensuring uninterrupted performance for our users. Our strategy involved archiving the HBase HFiles into the more manageable Parquet format.
Subsequently, we utilized ClickHouse’s HDFS engine to ingest from these Parquet files where the process involved selecting and inserting to null tables with materialized views to facilitate the necessary transformations, like this:

```
clickhouse-client --receive_timeout=180000 
-q """ insert into vimeo.session_event_ingestion 
       SELECT video_owner_id, ....
       FROM hdfs('hdfs://hostname:8020/tmp/table_dir/*/*/*/*/*','Parquet', 
         video_owner_id UInt32, event_ts UInt64, video_id UInt32, session_id String, ...') """
```


Given the potential for timeouts with extensive Parquet file operations, we divided our data into smaller, more manageable files, up to a few gigabytes each. This segmentation enabled us to ingest data in systematic phases programmatically. In instances where data inconsistencies arose or queries faltered, we implemented corrective measures by either negating and reingesting the data or leveraging ClickHouse dictionaries for updates.


# Optimizing queries

Our primary method for querying ClickHouse is through its JDBC interface. One challenge we faced was dealing with ClickHouse’s eventual merging of parts, especially in aggregated merge tables, which can lead to intermittent issues. However, by adopting alternative query patterns, we learned that these can be effectively navigated.
To enhance our query performance, we’ve integrated parameters like these:
-  ` **os_thread_priority=-10** `  ****  This sets the query thread priority from −20 (the highest) to 19 (the lowest).
-  ` **optimize_skip_unsused_shards=1** `  ****  This skips unused shards for select queries based on the sharding key condition.

These adjustments ensure our ClickHouse queries receive priority over other business-related queries and skip unused shards, thus optimizing overall system efficiency.


# With gratitude

Firstly, kudos for making it to the end of this in-depth exploration. I could go into even more detail about any aspect of what I’ve covered. Ask me a question in the comments and watch what happens.
I also want to thank Vimeans past and present for undertaking this long journey with me: Evan Potter, Rohit Chaudhry, Siddharth Shah, Alexandre Vincent, and Manohar Thimmaraju.


# Join the team

Like our data, our team is growing!  [Check our open positions.](https://vimeo.com/careers) 