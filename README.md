# location-tracking-system-design
An approach to designing a location tracking system at a high level

Task:
>Given a set of 1 million users, <br>
>design a system which will provide location/route<br>
>data for the user hourly/daily/weekly, realtime.<br>

We need more data to decide the configuration, but theoretically with assumptions design may look like below,

 ## System Design Constraints
- Each location event will be : 150 bytes
- Refresh interval on the user device for sending location is : 1s
- Total data per second to stream : 1000000 * 150 = 150 MB/s
 
 ## Replication
 3x Async replication in kafka (replication is default) should suffice this load based on the open source articles 
 [BenchMarkReference](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)
 
 ## Partitions
 - Partitions = Desired Throughput / Partition Speed
 - Considering only this topic,  a single partition for a single Kafka topic runs at 10 MB/s.
 - So to yield 150 MB/s we need 18 partitions.
 
 ## Spark Executors/Tasks
  Again we need to test to decide on the number of cores and executors 
  but assuming no extra config, we can have exact number of cores as kafka partitions
  assuming our memory and system allows it.
  
 with the above config we can assume that, each log record is available for subscription 
 in the same second as they arrive or less than a second.
 
  ## Latency - very optimistic :)
 
Target Allowed Latency for writing data to data store/HDFS  <= 1 min  
Writing Data:
- kafka stream processing <= 10ms
- target latency kafka to spark <= 10 ms
- processing in spark  <= 20 - 30 ms
- writing to cassandra <=10 ms 
- HDFS + Spark (Batch every 1 - 5 min) <= 1m 

Reading Data : 100ms
- Query cassandra for route hourly/range : 20-30ms
- Query for mongo <= 100ms

  
## Through put required
 We may have to try number of configurations before we achieve this throughput
 Through put : 
- 1000000 TPS for streaming/writing
- < =  1000000 TPS for reading (only active users)
               
 ## High Level Design
 
 ![alt text](tracking-design.jpg?raw=true)
 
## Producers
We have to delegate the load to the client applications, so the data must be pushed from the client applications on devices.
Based on the defined interval, assuming user enabled location, client devices will push the location events to kafka streams with kafka producer.
 
 A simple location Object may look like (adding columns to cassandra is easy and no downtime)
 ```
userId uuid,
ts timestamp,
longitude double,
latitude double,
 
 ```
 ## Why We need Kafka Streams for collecting data ?
 
 Kafka processes the records realtime, though spark has the capacity to process the streams it can only process in  micro batches.
 Moreover Data ingestion is way faster with Kafka, and we already know how easily we can scale kafka ,
 and other features like fault tolerance and replication.
 
 if we can afford latency then, Spark streams are also a better option.
 
 But for real time data processing, Kafka is one of the best options.
 
 ## Spark Streaming
 
 In our case it will write the transformed location objects to cassandra and HDFS.
 
 Spark processes records in small batches, in near future it can process real time .
 It gives us the flexibility to transform the events coming from kafka, after transforming , we can write the data to cassandra.
 Spark comes with great analytical tools like MlLib, PySpark. which make dealing analytics a whole lot easier.
 
 ## HDFS
 In our cases we need to feed computed views for hourly/daily/weekly location data to mongo, 
 so we will use HDFS+Spark to batch process and compute these views so the search can be faster.
 
 The Hadoop Distributed File System (HDFS) is the primary data storage system used by Big Data applications. 
 Our project uses HDFS architecture as it provides a reliable way of managing pools of big data set. 
 Most important, with HDFS, we have one centralized location where any Spark worker can access the data.
               
## Cassandra

In our case we store location objects in cassandra, and the computed views/documents for hourly/daily/weekly in mongo.
Its a distributed fault-tolerant database
its a hybrid of Amazon DynamoDB(Key-value store) and Google BigTable(sql), and improves on the partitioning and scalability significantly.

Table in cassandra may look like :

```
CREATE TABLE tracker(
userId uuid,
ts timestamp,
longitude double,
latitude double,
PRIMARY_KEY(userId, ts));
```
>Partition column is -userId<br>
clustering column is - ts

An insert to cassandra may look like 
```
INSERT INTO TRACKER (userId, ts, longitude, latitude ) values (1, 0, 0, 0);
```
Lets assume we made three inserts like above for the same user, then the partition looks like below

 ![alt text](cassandra.jpg?raw=true)
 
 - locating partition is O(1) 
 - search on the basis of clustered column(time stamp) is at worst O(logN), N - number of records.
 
 Locating the partition is super fast because of the consistency hashing, and since the locations are sored on time stamp,
 if we query for the user locations in past one hour, cassandra can respond really fast.<br>
 
 So Yes, we can track the data using cassandra alone, but if we want query on the basis of co-ordinates,
 or the query fits too many objects like querying for a week, then your front end could get slower.<br>
 
 so to avoid that, we have Spark :D , and I think mongo also can help here.,
 But if we want query for a range of timestamps, we will still depend on cassandra<br>
 
 Cassandra is a distributed database, to make the queries faster we can put the cassandra partitions inside spark partitions,
 If Spark executor on data center reads from cassandra partitions it will be local so no latency.
 And spark even allows us to cache the data for fast reads, its for more complicated cases like if you want to know locations a user has been,
 within a radius etc ...,
 
 ![alt text](spark-cassandra.png?raw=true)
 
 ## HDFS + SPARK Batch --> Mongo
 
 In our case it stores 3 documents for each user hourly/daily/weekly.
 structure of the document is discussed in next section.
 
 Spark transform the objects from HDFS and inserts in mongo. mongo stores documents in BSon format or Binary json,
 with the concept of eventual consistency, reads are super fast on mongo, but writes based on the load can have some acceptable latency.
 ## Queries
 
 - Spark SQL is very good to query cassandra/ and there are number of open source libs to extend it, ame it fast with spark.
 - for mongo it can be straight forward mongo query for the users.
 ## Location Documents for User
 We basically need to trace the route with the locations available.
 Spark will take the location data and transforms into a route, with known positions, 
 For hourly it will be small Json , but for daily and weekly it will be a huge json,
 so we need to define the number of positions for those structures.
 
 We can use Google Maps/GeoLocation APIs also for this purpose.
 If our front-end is capable of parsing this json and showing in a map, then the basic design is done.
 
 ```
 {
  "route": [
    {
      "name": "Rixos The Palm Dubai",
      "position": [25.1212, 55.1535],
    },
    {
      "name": "Shangri-La Hotel",
      "location": [25.2084, 55.2719]
    },
    {
      "name": "Grand Hyatt",
      "location": [25.2285, 55.3273]
    }
  ]
}
```

we can go with the GeoJson document as well [RFC 7946](https://tools.ietf.org/html/rfc7946)

Example :
 ```
Coordinates of a MultiLineString are an array of LineString
   coordinate arrays:

     {
         "type": "MultiLineString",
         "coordinates": [
             [
                 [100.0, 0.0],
                 [101.0, 1.0]
             ],
             [
                 [102.0, 2.0],
                 [103.0, 3.0]
             ]
         ]
     }
```

 ## Furthermore.,
 - this design can be extended for functionalities like to know location in a range/ in a radius etc.
 - Open source like [Datastax](https://www.datastax.com/) have contribute libraries to leverage spark/cassandra .
 - open for suggestions :)
