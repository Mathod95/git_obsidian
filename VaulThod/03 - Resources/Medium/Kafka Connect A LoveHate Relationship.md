---
tags:
  - APP/KAFKA
source: https://medium.com/@leal901/kafka-connect-a-love-hate-relationship-3fa69361f6e6
---
# Kafka Connect: A Love/Hate Relationship

![](https://miro.medium.com/v2/resize:fit:700/0*tXfeQpF8qszRlOEU) 
Apache Kafka is the de facto streaming platform for businesses today, and this popularity has elevated its associated sub-project — Kafka Connect. Kafka Connect enables you to use premade Connectors to send and receive data to and from Kafka. Connectors require no code but rather are configuration driven. This provides a lot of advantages but also a lot of headaches. In this article, we’ll talk about both, and we’ll specifically discuss how to get around the headaches. By the end, hopefully, you will have enough information to make good decisions about Kafka Connect’s usage.
We will reference the community’s  [Debezium Postgres Connector](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)  to talk about sourcing data from Postgres to Kafka, and Confluent’s  [JDBC Sink Connector](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc)  to talk about sending data from Kafka to another database.


# A primer on how Kafka Connect works

The advent of cloud has abstracted away lots of details regarding how things run under the hood, and  [Confluent Cloud](https://www.confluent.io/confluent-cloud/)  does a good job of hiding its complexity. However, I believe that understanding how something actually works allows you to make the best decisions about whether it is a good fit for you. So here is a quick primer:
1.  Kafka Connect is a distributed system, separate from Kafka, but powered by it
2.   *Kafka Connect*  is the platform, and  *Connectors*  run on it
3.  Kafka Connect has  *source connectors*  that bring data from other systems into Kafka, and  *sink connectors*  that send data from Kafka into end systems
4.  Kafka Connect uses an internal topic in Kafka to coordinate source connectors, and the  *consumer group protocol*  to coordinate sink connectors
5.  Connectors aren’t made by one organization but rather are made by the community, and the community is strong: It provides hundreds of connectors for organizations to use. However, this also means that quality and feature sets vary greatly by connector

If you want to learn more depth,  [Confluent](https://developer.confluent.io/courses/kafka-connect/intro/)  has a pretty good course on it. That should give us a good enough base to start talking about how things work. Let’s get started.
![](https://miro.medium.com/v2/resize:fit:700/0*T_IP_X8y08yvMKSe) 


# Source Connectors



# On sourcing from databases

Postgres is a very popular database, which makes the Postgres connector quite popular as well. Thankfully, the community has chosen  [Debezium](https://debezium.io/)  as the project to rally around for very important connectors, and Postgres is one of them.
Sourcing from databases is hard, so if you take some time to look into how  [Debezium has standardized](https://github.com/debezium/debezium)  reading from Write Ahead Logs (WALs), you’ll realize it’s no joke to ensure zero data loss, no data duplication, as well as high uptime. Once you’ve made this realization you are set: You should probably use the community’s standard connector to send data from Postgres to Kafka instead of doing your own thing, which would most likely not do as good of a job (I’ve seen plenty of companies that do this, for some reason). Now that you’ve made the commitment to use the connector, you have to understand it. How does it all work?
To read data from a database, it would be extremely inefficient and wasteful to issue query reads and then send that data back to Kafka (although there is a  [connector](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc)  that does just that, if you so wish). Instead, what most connectors do is listen to the database’s Write Ahead Log. Some DBs provide direct and easy access to their WAL, which is nice! In fact, it would be so  [jerkish not to!](https://docs.oracle.com/en/database/oracle/oracle-database/18/upgrd/deprecated-features-oracle-database-12c-r2.html#GUID-87A754A3-AC6B-4F84-8C36-AB90AC5032D4)  Anyways, Postgres does, and it does so through “logical decoding” (their fancy name for it). Herein lies our first blessing/problem.


## Reading from WALs

There are three common problems that plague users of Kafka Connect whenever they use a Change Data Capture (CDC) connector such as the Debezium Postgres connector:
-  **An old version of Postgres** : Postgres has been around for a while, but logical decoding hasn’t, and formal first-class support is even more recent. Users of old versions of Postgres will have to utilize a community decoder plugin, and these have been found to have  [many issues](https://issues.redhat.com/browse/DBZ-2175?jql=project+%3D+DBZ+AND+resolution+%3D+Unresolved+AND+text+%7E+%22decoderbuf+OR+wal2json%22+ORDER+BY+priority+DESC%2C+updated+DESC) . Although some of them have workarounds (too many for us to get into in this post), their usage will surely slow down deployment speed and resilience. It is worth noting that newer versions of Postgres have a standard decoder called pgoutput that performs very well and is backed by the Postgres project as a whole.
-  **Size of the WAL** : Obviously databases have to be space conscious, so they don’t keep WALs forever; However, using a streaming protocol means we’re plugged directly into the WAL, and one day the connector might fail but the database will keep chugging along. This means that when the connector comes back online, it has to catch up. But what if the data that it needs to catch up to has already been deleted by Postgres? Thus using the Postgres connector also means negotiating with your DB Admin how long the WAL can be kept, and this becomes the maximum amount of time you have to resolve an issue with the connector — before you need to start recovery actions.
-  **Most source connectors don’t offer  *exactly once* **  [yet](https://issues.redhat.com/browse/DBZ-5467) ): This is true for  [Debezium connectors](https://issues.redhat.com/browse/DBZ-5467) , which means that when all goes wrong, you will have duplicates (but no data loss, as they do offer  *at least once* ). For MOST applications, this is sufficient, but a lot of people go into this connector expecting  *exactly-once*  deliveries.

The above issues make the primordial task of the connector, getting data out of the system, quite the math equation that needs to be solved. What is important is that it CAN be solved, but it does require careful tuning and testing. The pain is exacerbated by the very common situation that the people developing data streaming flows aren’t typically DB Admins, so they would prefer not to have to worry about these issues.
Anyways, now that you’ve tamed the sourcing of your data, you need to ensure it is actually sent in a way that is useful, which is where pain point number two pops up.


## Delivery and interpretation of database records in Kafka

Data Types: They make your life as a developer harder and easier at the same time. You can’t live without them.
In order to make data sourced from databases usable somewhere else, maintaining data types or at least converting them properly is paramount. Keeping such requirements in mind, the Kafka community has been evolving ideas on the best way to handle types, but there are some challenges that come with the territory:
- Data interpretation and evolution is hard: Everything that goes into Kafka is serialized as a byte array, which means that when it is read out, it requires a way to be deserialized. By default, Kafka provides a JSON serializer, but as you probably know, sending large amounts of data with naive JSON serialization is actually really inefficient. Confluent has had the  [Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)  for a while to address this issue along with Data Governance and Data Quality problems. However, there are also various solutions in the community that aim to help solve these problems. (I go quite into depth on this issue in a  [podcast](https://www.youtube.com/watch?v=ZIJB8cs-cJU)  with the great  [Kris Jenkins](https://www.youtube.com/@DeveloperVoices) ). This is actually a multi-tiered issue:

1.  When consumed from a database, data comes with the DB’s data type
2.  When converted into a Kafka Connect record, it gets cast into a set of types that are predefined (and available in the  [documentation](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-data-types) 
3.  When sent into Kafka, the data is typeless
4.  When consumed, you have to cast again, either to Connect’s internal data types, which shouldn’t be a problem since it was sourced with Connect, or to a data type that is useful for the application. (ksqlDB famously does not have good compatibility with all of the possible data types a DB may have.)

Long story short, be careful regarding what your data gets converted to, and ensure you are comfortable with the casting. There are various connector configurations to change the behaviors here, especially those having to do with time (always the easiest problem to solve in production systems :) /s). Also, I highly recommend utilizing Schema Registry here.
- In-flight transforms: One of the more useful features of Kafka Connect is its ability to modify every record that passes through it  *slightly*  — in order to fit your needs. These transformations are pluggable and the community has a sprawl of them. For example,  [Debezium has some that are very helpful](https://debezium.io/documentation/reference/stable/transformations/index.html) . However, one of the problems that arises with developer teams is that because of how useful it would be to just modify data on the fly before even storing it, they try to fit all of their data transformations into these — getting pretty creative by correctly chaining transforms and predicates. However, the reality is that these transforms are meant to be lightweight because they can severely hamper the performance of the connector. If you really need more involved transformations, I would try Kafka Streams, Apache Flink, Apache Beam, or whatever flavor of stream processing you like.
- Scalability: More often than not, the connector will be faster than whatever system you are sourcing from, which is mostly great! However, something to note is that most source connectors will be limited to one task ( *tasks*  are the unit of parallelism in Kafka Connect), because they need to ensure a consistent overall order of updates from the database. This may hamper you, particularly if you have expensive transforms. A good rule of thumb is to separate table updates into various connectors in order to ensure being able to keep up with high-traffic databases.



## Maintenance

Long-running source connectors can run into issues as your organization evolves. These are largely rare, as long as there isn’t anything funny going on at the source datastore (like constant failover tests or whatnot). Commonly, the source connector maintenance story primarily consists of:
- Rehydrating every now and then
- Rerunning the connector when it fails due to network or connectivity issues
- Scaling the connector due to more data being pumped through it

 **Rehydrating** 
This has become such a common occurrence that most Debezium connectors now implement snapshots while running as a common operation that can be triggered by command messages to the connector. Rehydration is necessary in cases where the target data store loses the data, and you must reload all of the data from the source DB to bring the complete state to the target (this only happens if you don’t leverage  *compacted topics* , but that’s another story).
If you are using the aforementioned Debezium connectors, you are likely in a good spot to do this easily. Other connectors don’t allow this very easily unless you stand up a completely new connector, due to how hard it is to modify source-state progress in Kafka for source connectors.
 **Rerunning the connector** 
This is the most commonly scripted maintenance operation out there. Connectors fail all of the time due to network issues. The reality is that the unreliable network is the only reliable truth in the networking realm. Projects out there like Confluent for Kubernetes and Strimzi provide auto-restart capabilities in their K8s operators. Do yourself a favor and implement these from day one.
 **Scaling the connector** 
Scaling the connector in tasks up and down is likely the maintenance you’ll face the most. Most of the community would like connectors to have policy-based scaling, similar to what  [KEDA](https://keda.sh/)  did for Kafka Clients. The project doesn’t address this, and neither do commercial offerings out there. I’d recommend making the application team own scaling actions of the connector or implementing your own scaler that listens for connector metrics and scales up/down according to them.


# Last note on source connectors

I know it seems like I am only talking doom and gloom, but in reality, connectors are one of the best things to happen to real-time data processing. I’m trying to highlight that they are unfortunately not that easy to productionalize, but once you have them down, they work really, really well.
To summarize, connectors involve a lot of upfront work and a medium level of maintenance (relatively), but just remember that doing it yourself would be worse.


# Sink Connectors



# On Sending Data to End Systems

Only way you can make the data in Kafka useful is if you eventually send it out to another system for interpretation / further processing. Given that, databases are the common target of this data. However, databases haven’t historically been developed to be performant in receiving large amounts of streaming data. (though some cool entrants in this space like  [Druid](https://druid.apache.org/)  and  [Pinot](https://pinot.apache.org/)  are setting the stage for a bright future)
There is a popular protocol to send data onto databases in a standard way:  [J/ODBC](https://en.wikipedia.org/wiki/Java_Database_Connectivity) . Therefore, we’ll be focusing on discussing the  [JDBC Sink Connector](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc)  in our discussion of Sink Connectors. However, just like source connectors optimized sourcing through WALs, some database projects have optimized writing through their own delivery system, and there are project-specific connectors that make use of these optimizations. Good examples of these are  [MongoDBs Sink](https://www.confluent.io/hub/mongodb/kafka-connect-mongodb)  (leveraging the mongo client),  [Snowflake Sink](https://www.confluent.io/hub/snowflakeinc/snowflake-kafka-connector)  (leveraging snowpipes),  [BigQuery Sink ](https://www.confluent.io/hub/wepay/kafka-connect-bigquery) (leveraging BigQuery’s bulk api), etc.
No matter the implementation, some challenges remain in order to run a stable and production-safe sink connector.


# Message Interpretation

The number one problem that surfaces with sink connectors is data interpretation. It is common for developers to try to deserialize a message in a format it didn’t come in, or to try and sink a message with a schema that isn’t supported by the target system. For both these issues, I highly suggest integrating with Confluent’s Schema Registry from the get-go in order to avoid most of the problems.
Data SerDes (Serialization / Deserialization) is always a big topic when I speak with a customer starting their data streaming journey, and my recommendation is always (in this order):
1.  If your company already has a preference for Avro/Protobuf/JSON, stick with that
2.  If there is no preference, use Protobuf
3.  If you can’t use Protobuf, use Avro
4.  If you can’t use Protobuf or Avro, you might as well give up, because JSON Schema is trash

Now that you have a well-defined schema, I’d look into what the connector expects the schema of the message to be. This is basically different for every single sink connector, so research it well, and if you need to modify data, leverage SMTs ( *single-message transforms* ) or a stream processor like Kafka Streams, Beam, Flink, etc.


# Target system success/failure tracking

A lot of organizations would like to keep track of whether a record that was consumed by a connector was successfully delivered to its target system, and if it wasn’t, why wasn’t it? The Kafka project sought to address this with  [KIP-298](https://cwiki.apache.org/confluence/display/KAFKA/KIP-298%3A+Error+Handling+in+Connect) , which was then expanded in  [KIP-610](https://cwiki.apache.org/confluence/display/KAFKA/KIP-610%3A+Error+Reporting+in+Sink+Connectors) . However, you should keep in mind that while KIP-298 work is available in all connectors because it concerns error reporting by the framework, KIP-610 is an optional implementation for the connector, and not all connectors offer it.
A big builder of connectors for the community is Confluent, and they sought to address this issue faster with their Connect Error Reporter, which was implemented before KIP-610 ever made it to the project. This reporter is largely better than the framework implementation, but it is not available in all connectors, and certainly not in all Confluent connectors. YMMV, but keeping these options in mind when using a sink connector is important. To summarize:
- ALL connectors have KIP-298
- SOME connectors have KIP-610
- SOME Confluent-built connectors have the Confluent Error Reporter



# Optimization for large datasets

A very common scenario is that an organization’s connector will consume data from Kafka very fast, but the insert/upsert to its target will be extremely slow. This causes a slow pipeline, or even worse, a timeout of message delivery that ends up with the connector being considered dead by poll loop constraints.
There are a few ways to mitigate these issues at a high level:
- In database target systems, ensure you implement indices to make upserts faster
- For HTTP endpoints, ensure the target system can access parallel requests and that you are batching these in an efficient manner (hello Salesforce Sinks)
- For vendor end systems that don’t behave like datastores, it ultimately ends up being a game of slowing down the connector to match what the end system can tolerate, and to batch as efficiently as possible to ensure high throughput of data transfers. You can slow down the connector by tuning the fetch settings of the underlying consumers of the connector, and you can batch better by tuning the underlying delivery system the connector uses — if the connector makes it available.



# Last note on sink connectors

Sink connectors are extremely useful, and to be honest, often more resilient in terms of maintenance than source connectors, which is quite nice. Their state tracking is mostly through the Kafka consumer group offset management system, which makes it a breeze to modify their starting points.
A common request I see is how to do data completeness reconciliation between the data extracted at the source and the data delivered at the target. This is a big topic in distributed systems with many hard answers, but commonly a flavor of  [ *distributed tracing*  ](https://www.datadoghq.com/knowledge-center/distributed-tracing/) is used to ensure that you have a reliable reconciliation story.


# Summary

We have discussed the Kafka Connect framework, source and sink connectors. Overall, Connectors are extremely useful, but they require you to consider how they are affecting your source and target systems, and they are limited in the ways in which they expect data to be handled. However, they do enforce best practices with respect to integrating systems from and to Kafka, so they are an excellent tool for organizations that use Apache Kafka in their backends.