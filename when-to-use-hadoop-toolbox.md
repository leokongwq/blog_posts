---
layout: post
title: Hadoop-Hbase-Hive-Pig关系
date: 2015-07-12
comments: true
categories:
- bigdata
tags:
- hadoop
- hive
- pig
- sqoop
---

### Hadoop-Hbase-Hive-Pig-Sqoop关系

Eight years ago not even Doug Cutting would have thought that the tool which he's naming after his kid's soft toy would so soon become a rage and change the way people and organizations look at their data. Today Hadoop and Big Data have almost become synonyms to each other. But Hadoop is not just Hadoop now. Over time it has evolved into one big herd of various tools, each meant to serve a different purpose. But glued together they give you a powerpacked combo.

Having said that, one must be careful while choosing these tools for their specific use case as one size doesn't fit all. What is working for someone might not be that productive for you. So, here I will show you which tool should be picked in which scenario. It's not a big comparative study but a short intro to some very useful tools. And, this is based totally on my experience so there is always some scope of suggestions. Please feel free to comment or suggest if you have any. I would love to hear from you. Let's get started :

1- Hadoop : Hadoop is basically 2 things, a distributed file system (HDFS) which constitutes Hadoop's storage layer and a distributed computation framework(MapReduce) which constitutes the processing layer. You should go for Hadoop if your data is very huge and you have offline, batch processing kinda needs. Hadoop is not suitable for real time stuff. You setup a Hadoop cluster on a group of commodity machines connected together over a network(called as a cluster). You then store huge amounts of data into the HDFS and process this data by writing MapReduce programs(or jobs). Being distributed, HDFS is spread across all the machines in a cluster and MapReduce processes this scattered data locally by going to each machine, so that you don't have to relocate this gigantic amount of data.

2- Hbase : Hbase is a distributed, scalable, big data store, modelled after Google's BigTable. It stores data as key/value pairs. It's basically a database, a NoSQL database and like any other database it's biggest advantage is that it provides you random read/write capabilities. As I have mentioned earlier, Hadoop is not very good for your real time needs, so you can use Hbase to serve that purpose. If you have some data which you want to access real time, you could store it in Hbase. Hbase has got it's own set of very good API which could be used to push/pull the data. Not only this, Hbase can be seamlessly integrated with MapReduce so that you can do bulk operation, like indexing, analytics etc etc.

Tip : You could use Hadoop as the repository for your static data and Hbase as the datastore which will hold data that is probably gonna change over time after some processing.

3- Hive : Originally developed by Facebook, Hive is basically adata warehouse. It sits on top of your Hadoop cluster and provides you an SQL like interface to the data stored in your Hadoop cluster. You can then write SQLish queries using Hive's query language, called as HiveQL and perform operations like store, select, join, and much more. It makes processing a lot easier as you don't have to do lengthy, tedious coding. Write simple Hive queries and get the results. Isn't that cool??RDBMS folks will definitely love it. Simply map HDFS files to Hive tables and start querying the data. Not only this, you could map Hbase tables as well, and operate on that data.

Tip : Use Hive when you have warehousing needs and you are good at SQL and don't want to write MapReduce jobs. One important point though, Hive queries get converted into a corresponding MapReduce job under the hood which runs on your cluster and gives you the result. Hive does the trick for you. But each and every problem cannot be solved using HiveQL. Sometimes, if you need really fine grained and complex processing you might have to take MapReduce's shelter.

4- Pig : Pig is a dataflow language that allows you to process enormous amounts of data very easily and quickly by repeatedly transforming it in steps. It basically has 2 parts, the PigInterpreter and the language, PigLatin. Pig was originally developed at Yahoo and they use it extensively. Like Hive, PigLatin queries also get converted into a MapReduce job and give you the result. You can use Pig for data stored both in HDFS and Hbase very conveniently. Just like Hive, Pig is also really efficient at what it is meant to do. It saves a lot of your effort and time by allowing you to not write MapReduce programs and do the operation through straightforward Pig queries.

Tip : Use Pig when you want to do a lot of transformations on your data and don't want to take the pain of writing MapReduce jobs.

5- Sqoop : Sqoop is a tool that allows you to transfer data between relational databases and Hadoop. It supports incremental loads of a single table or a free form SQL query as well as saved jobs which can be run multiple times to import updates made to a database since the last import. Not only this, imports can also be used to populate tables in Hive or HBase. Along with this Sqoop also allows you to export the data back into the relational database from the cluster.

Tip : Use Sqoop when you have lots of legacy data and you want it to be stored and processed over your Hadoop cluster or when you want to incrementally add the data to your existing storage.

6- Oozie : Now you have everything in place and want to do the processing but find it crazy to start the jobs and manage the workflow manually all the time. Specially in the cases when it is required to chain multiple MapReduce jobs together to achieve a goal. You would like to have some way to automate all this. No worries, Oozie comes to the rescue. It is a scalable, reliable and extensible workflow scheduler system. You just define your workflows(which are Directed Acyclical Graphs) once and rest is taken care by Oozie. You can schedule MapReduce jobs, Pig jobs, Hive jobs, Sqoop imports and even your Java programs using Oozie.

Tip : Use Oozie when you have a lot of jobs to run and want some efficient way to automate everything based on some time (frequency) and data availabilty.

7- Flume/Chukwa : Both Flume and Chukwa are data aggregation tools and allow you to aggregate data in an efficient, reliable and distributed manner. You can pick data from some place and dump it into your cluster. Since you are handling BigData, it makes more sense to do it in a distributed and parallel fashion which both these tools are very good at. You just have to define your flows and feed them to these tools and rest of things will be done automatically by them.

Tip : Go for Flume/Chukwa when you have to aggregate huge amounts of data into your Hadoop environment in a distributed and parallel manner.

8- Avro : Avro is a data serialization system. It provides functionalities similar to systems like Protocol Buffers, Thrift etc. In addition to that it provides some other significant features like rich data structures, a compact, fast, binary data format, a container file to store persistent data, RPC mechanism and pretty simple dynamic languages integration. And the best part is that Avro can easily be used with MapReduce, Hive and Pig. Avro uses JSON for defining data types.

Tip : Use Avro when you want to serialize your BigData with good flexibility.

The list is actually pretty big, but I have covered only the most significa