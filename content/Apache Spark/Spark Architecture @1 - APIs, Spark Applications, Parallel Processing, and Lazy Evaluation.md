---
date: 2024-05-09
modified: 2024-08-11T19:33:08+02:00
---
# 0. Resources
Before I begin, I want to list all the resources I used to write these notes. Please note that I may have forgotten some of them because I wrote these notes over the past months, and I didn't cite all the sources I used. Really sorry about that. The main ones are:

- LearningJournal Youtube Channel: [https://www.youtube.com/watch?v=fyTiJLKEzME&ab_channel=LearningJournal](https://www.youtube.com/watch?v=_piYXmAXHW8&ab_channel=LearningJournal) (3 main videos, Apache Spark series)
- Learning Spark: Lightning-Fast Data Analytics - Jules S. Damji, Brooke Wenig, Tathagata Das, and Denny Lee (Second Edition 2020, O'Reilly Media)
- Spark: The Definitive Guide - Bill (B.) Chambers, Matei (M.) Zaharia (2018, O'Reilly Media

---

# 1. High-Level Spark Architecture
There are some tasks that our personal computers are not powerful enough to perform. One particular area is **data processing**, where huge amounts of information are involved.
> [!important] Cluster
> A **cluster of computers** pools the resources of many machines together, giving us the ability to use all the cumulative resources as if they were a single computer.  

A cluster is not enough because you need a framework to coordinate work across them: **Spark** does just that, managing and coordinating the execution of tasks on data across cluster of computers. The cluster of machines that Spark will use to execute tasks is managed by a **Cluster Manager,** but we’ll see it later on.

So, let’s review some Spark definition coming from several sources:
> [!important] Apache Spark Definitions
> **spark.apache.org** definition: Apache Spark is a fast and general engine for large-scale data processing.
> 
> **Databricks** definition: Apache Spark is a powerful open source processing engine built around speed, ease of use, and sophisticated analytics.
> 
> **Another definition**: Apache Spark is a fast and general purpose engine for large-scale data processing. Under the hood, it works on a cluster of computers. 

We can define Apache Spark in another way as well. Apache spark is the **combination between 2 elements**:

- a **cluster computing engine**
- a **set of libraries, APIs, and DSLs**.

Look at this diagram, that is **Apache Spark ecosystem**:

![[Apache Spark/attachments/Untitled 28.png|Untitled 28.png]]

**Apache Spark Core** itself has 2 parts:
- a **Computing Engine**
- a set of **Core APIs**

![[Apache Spark/attachments/Untitled 7 5.png|Untitled 7 5.png]]

Apache Spark is a distributed processing engine, but it **doesn’t come** with a **built-in cluster resource manager** and a **distributed storage system**; so, you have to plugin:
- a **cluster manager** (i.e. Apache YARN, Mesos, and Kubernetes)
- a **storage system** (i.e. HDFS, Amazon S3, GCS, CFS [Cassandra file system])

![[Apache Spark/attachments/Untitled 8 4.png|Untitled 8 4.png]]

So the Engine heavily depends upon a third-party **cluster resource manager** and the **storage system** and both of them are of your choice.

> [!summary] Quick Recap about Apache Spark Ecosystem:
> Remember what we have said so far, that is **Apache Spark doesn’t offer**:
> - **cluster management service**
> - **storage management service**
> 
> However, it has a **Compute Engine** as part of Spark Core: it provides some basic functionalities, such as **memory management**, **task scheduling**, **fault recovery** and most importantly **interacting with the cluster manager and storage system**.

It’s the **Spark Core**, or we can say, the **Spark Compute Engine** that executes and manages our Spark jobs and provides a seamless experience to the end user: you just submit job to Spark, and the Spark Core takes care of everything else.

Although you can use Spark with a variety of programming languages (as we'll see in the [2. Spark’s Language APIs](#2.%20Spark’s%20Language%20APIs)) chapter), what it makes available in those languages is worth mentioning. Spark has two fundamental sets of APIs:
- **Structured API:** it consists of **DataFrames** and **Datasets**. They are designed and optimised to work with structured data.
- **Unstructured APIs**: they are the lower level APIs, including **RDDs**, Accumulators and Broadcast variables.

Outside of the Spark Core, we have 4 different set of libraries and packages:
- **Spark SQL** - it allows to use SQL queries for structured data processing
- **Spark Streaming** - it allows to consume and process continuous data streams
- **MLlib** - it is a machine learning library that delivers high-quality algorithms
- **GraphX**

They offer APIs, DSLs, and algorithms in multiple languages.

---

# 2. Spark’s Language APIs
Spark’s **language APIs** make it possible to run Spark code using various programming languages. For the most part, Spark presents some core “concepts” in every language; these concepts are then translated into Spark code that runs on the cluster of machines. If you use just the Structured APIs, you can expect all languages to have similar performance characteristics. Here’s a brief rundown:

- **Scala**. Spark is primarily written in Scala, making it the Spark’s default language.
- **Java**.
- **Python**. It supports nearly all constructs that Scala supports.
- **SQL**. Spark supports a subset of the ANSI SQL 2003 standard.
- **R**. Spark has two commonly used R libraries: one as a part of Spark core (SparkR) and another as an R community-driven package (sparklyr).

Each language API maintains the same core concepts that we described earlier. There is a **SparkSession** object available to the user, which is the entrance point to running Spark code. When using Spark from Python or R, you don’t write explicit JVM instructions; instead, you write Python and R code that Spark translates into code that it then can run on the executor JVMs.

![[Apache Spark/attachments/Untitled 9 4.png|Untitled 9 4.png]]

---

# 3. Spark Applications
**Spark Applications** consists of:
- a **driver process,** responsible for orchestrating **parallel operations** on the Spark Cluster. It accesses the distributed components in the cluster (the Spark Executors and Cluster Manager) through a **Spark Session**. Main functions:
    - it communicates with the cluster manager;
    - it requests resources (CPU, memory, etc.) from the cluster manager for Spark’s executors (JVMs);
    - it transforms all the Spark operations into **DAG** computations, schedules them, and distributes their execution as tasks across the Spark executors.
- a set of **executor,** responsible for carrying out the work that the driver assigns them. In most deployments modes, only a single executor runs per node. They have 2 tasks:
    - executing the code assigned to it by the driver
    - reporting the state of the computation on that executor back to the driver node.

![[Apache Spark/attachments/Untitled 10 3.png|Untitled 10 3.png]]
> [!summary]
> So far we learned how to answer to the following 2 questions:
> 
> - _What is Apache Spark?_ A distributed computing platform.
> - _What do we do with Apache Spark?_ We create program and execute them on a Spark Cluster.

Now we’ll answer to the question:
## 3.1. How do we execute programs on a Spark Cluster?
There are 2 methods:
1. **Interactive clients**
2. **Submit a job**

After Spark installation on your local machine, you can access Spark using Scala Shell, PySpark Shell, and Jupyter notebooks. All of these are **interactive clients**. Most of the people use interactive clients during the learning or development processes. Interactive clients are best suitable for exploration purpose.

But ultimately all explorations will end up into a Spark applications like streaming applications or batch job. In both cases, you must package your **application and submit it to Spark Cluster for execution**. This second method is suitable for production cases. Apache Spark comes with a `spark-submit` script in Spark’s `bin` directory**.**

So coming back to the question: how does Spark execute our programs on a cluster?

Spark is a distributed processing engine, and it follows the **master-slave architecture**. So for every Spark Application, it will create:
- **one master process**
- **multiple slave processes**

In the Spark terminology the master is the **Driver**, and the slaves are the **Executors**.

Let’s make an example: suppose you are using the `spark-submit` script.
- you submit an **Application A1** using `spark-submit`;
- Spark will create one **driver** process and some **executors** for A1.

![[Apache Spark/attachments/Untitled 12 2.png|Untitled 12 2.png]]

This entire set is exclusive for the application A1.

Say now that:
- you submit another **Application A2**;
- Spark will create one more **driver** process and some **executors**

![[Apache Spark/attachments/Untitled 13 2.png|Untitled 13 2.png]]

So, for every Applications, Spark creates:
- one **driver;**
- a **bunch of executors.**

Since the **Driver** is the Master, it is responsible for:
- analysing work across the executors
- distributing work across the executors
- scheduling work across the executors
- monitoring work across the executors,

. It is also responsible for:
- maintaining all the necessary information during the lifetime of the application.

On the other hand, the Executors are only responsible for:
- executing the code assigned to them by the driver and reporting the status back to the driver.

Next question:
## 3.2. Who executes where?
We have:
- a cluster
- a local client machine (our personal computer)
- we start the Spark Application from our client machine.

What goes where?

The executors are always going to run on the cluster machines. There is no exception to this. But we have the flexibility to start the driver on your local machine or on the cluster itself:

![[Apache Spark/attachments/Untitled 14 1.png|Untitled 14 1.png]]

![[Apache Spark/attachments/Untitled 15 1.png|Untitled 15 1.png]]

When you start an application, you have a choice to specify the execution mode between 2 options:
- **Client Mode**: it will start the driver on you local machine
- **Cluster Mode**: it will start the driver on the cluster

As we know, the driver is responsible for the whole application and if anything goes wrong with the driver, the application state is gone.

So, if you start the driver on the local machine, the application is directly dependent on your local computer. This is the **Client Mode**. This is useful when you are exploring things or debugging an application. You can easily debug it, or at least, it can throw back the output on your terminal. But we don’t want this dependency in a production application.

After all, you have a dedicated cluster to run the job. So the **Cluster Mode** makes perfect sense for production deployment. This is because, after `spark-submit`, you can switch off you local computer and the application executes independently within the cluster.

When you start a Spark Shell, or any other interactive client, you would be using Client Mode.

So, if you are running a Spark Shell, your driver is running locally within the same JVM process. The driver is embedded within the shell.

Let’s answer the very first question, putting together all we did.

> [!important]  
> How does the Spark execute our programs on cluster?  

- Spark creates one Driver and a bunch of Executors for each application.
- Spark offer 2 deployment modes for an application:
    - Client Mode - driver on client machine and Executors on the cluster
    - Cluster Mode - Driver and Executors on the cluster.

Next question:
## 3.3. Who controls the cluster? How Spark gets the resources for the driver and the executors?

We need a **Cluster Manager**. It is responsible for managing and allocating resources for the cluster of nodes on which your Spark application runs. Apache Spark supports 4 different cluster managers:
- Apache YARN (cluster manager for Hadoop. It is the most widely used cluster manager for Apache Spark)
- Apache Mesos
- Kubernetes
- Standalone

All of them deliver the same purpose. Let’s take the **YARN** as an example to understand the resources allocation process.

A Spark Application begins by creating a **Spark Session**.

If you are using a Spark Client tool, like Spark Shell or PySpark, they automatically create a Spark Session for you. You can think of Spark Session as a data structure where the driver maintains all the information including the executors location and their status.

> [!info]- SparkSession details
> You can control your Spark application through a driver process called the SparkSession. The SparkSession instance is the way Spark executes user-defined manipulations across the cluster. It provides a single unified entry point to all of Spark’s functionality.
> 
> *(Find the difference between Sparkcontext vs Sparksession so that you can understand the word “unified” in the previous definition.)*
> 
> In Scala and Python, the variabile is available as `spark` when you start the console because the SparkSession is created for you. If you type `spark` in the PySpark console, you get:
 >   ```PowerShell
 >   <pyspark.sql.session.SparkSession at 0x7efda4c1ccd0>
 >   ```

## 3.4. Client Mode case
Now, assume you are starting an application in Client Mode, or you are starting a Spark Shell: in this case your driver starts on the local machine and then, as soon as the driver create a Spark Session, a request goes to YARN resource manager to create a YARN application. The YARN resource manager starts an Application Master. For the client mode the Application Master acts as an Executor Launcher. So, the Application Master will reach out to YARN resource manager and request for further container .

![[Apache Spark/attachments/Untitled 16 1.png|Untitled 16 1.png]]

The resource manager will allocate new containers:

![[Apache Spark/attachments/Untitled 17 1.png|Untitled 17 1.png]]

and the Application Master starts an executor in each Container:

![[Apache Spark/attachments/Untitled 18 1.png|Untitled 18 1.png]]

After the initial setup, these executors directly communicate with the driver:

![[Apache Spark/attachments/Untitled 19 1.png|Untitled 19 1.png]]

This is how Apache Spark creates a driver and the executors for Client Mode Application.
## 3.5. Cluster Mode case
The process for Cluster Mode application is slightly different.
- You submit the packaged application using the `spark-submit` tool;
- the `spark-submit` tool will send a YARN application request to the YARN resource manager;
- the resource manager start an Application Master;
- the driver start in the Application Master container;

![[Apache Spark/attachments/Untitled 20 1.png|Untitled 20 1.png]]

> [!important] **Client Mode** and the **Cluster Mode**
> That’s where the **Client Mode** and the **Cluster Mode** differs:
> - in the Client Mode, the YARN Application Master acts as an executor launcher and the driver resides on your local machine
> - in the Cluster Mode, the YARN Application Master start the driver, and you don’t have any dependency on your local computer
> - once started, the driver will reach out to Resource Manager with a request for more Containers.
>
> The rest of the process is the same as the Client Mode:
> - the Resource Manager will allocate new Containers;
> - the driver starts an executor in each Container.

Actually there is a third option as well, the **Local Mode**.
## 3.6. Local Mode
When you don’t have enough infrastructure to create a multi-node cluster, and you still want to setup Apache Spark, maybe just for learning purpose, you can use **Local Mode**.

It’s the most comfortable method to start a Spark Application. In this mode you don’t need any cluster, neither YARN or Mesos, nothing. You just need a local machine and the Spark binaries.

![[Apache Spark/attachments/Untitled 21 1.png|Untitled 21 1.png]]

When you start an Application, it begins in a JVM and everything else, including driver and executors, runs in the same JVM. The Local Mode is the most convenient method for setting up a quick learning environment.

> [!summary] Recap
> Recap. We talked about 5 topics:
> 1. Driver
> 2. Executors
> 3. Client Mode
> 4. Cluster Mode
> 5. Local Mode

Let’s see them visually.
## 3.7. Local Mode Example
This is a machine where Spark is installed. No cluster manager like YARN or Mesos or Standalone is configured. We have no cluster as well.

All we can do is to start a **Spark Application** in **Local Mode**. We’ll see:
1. an interactive method using Spark Shell;
2. fire an application using `spark-submit` utility.

Remember that both of these are going to start an Application in Local Mode because Client and Cluster mode is only applicable on a Cluster.
### Interactive method using Spark Shell
Let’s start the **Interactive Method** with starting the spark-shell by the terminal:  
```PowerShell
spark-shell
```

Here’s the result:

![[Apache Spark/attachments/Untitled 22 1.png|Untitled 22 1.png]]

Let’s look at some of the messages appeared in the shell:
- every Spark application offers a **Web UI** for monitoring and tracking (available at a certain address, look at the picture above).
- the Spark Shell has already created and started a **Spark Session**.

Let’s go in the Spark UI:

![[Apache Spark/attachments/Untitled 23 1.png|Untitled 23 1.png]]

It says: “Executor driver added”. This means that there is only one process and it is acting as an **executor and driver both**. That’s what we expected from a Local Mode application where **everything runs inside a single JVM**.

If you exit the application and refresh the Spark UI, you’re not able anymore to see that UI because the UI exists only for the life of the Spark Application.
### Using `spark-submit`utility
Let’s go with the `spark-submit` utility:
```Shell
spark-submit --class org.apache.spark.examples-SparkPi spark/spark-2.2.0-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.0.0.jar 1000
```

This example comes along with Spark and it approximates the value of pi.
- The first parameter `--class` is the name of the main class;
- the second argument is the example’s jar file;
- `1000` is just to ensure that the application runs for a while.

Start the Application and look at the Spark UI:

![[Apache Spark/attachments/Untitled 24 1.png|Untitled 24 1.png]]

The processing of these many slices is complete, and we see only 1 executor driver.
## 3.8. Client & Cluster Mode Example
We need a cluster environment and you can setup a free Hadoop Cluster in Google Cloud:

![[Apache Spark/attachments/Untitled 25 1.png|Untitled 25 1.png]]

This cluster is composed of:
- 1 master node
- 3 worker nodes. Each worker machine has got 2 CPU cores, and hence we have 6 YARN cores available at our disposal.

Here’s the **YARN resource manager UI**:

![[Apache Spark/attachments/Untitled 26 1.png|Untitled 26 1.png]]

You can see that there is no application running on this cluster. We’re going to repeat the same exercise as we did for Local Mode, that is:
1. Start an interactive method using the spark-shell (**Client Mode**).
2. Fire an application using `spark-submit` utility (**Cluster Mode**).
### Client Mode
Click on the SSH button of the image above to start a console. The **spark-shell** will start a **Spark Application** in **Client Mode**.

After typing `spark-shell` in the console you can see how the driver and the executors are getting started on this cluster:

![[Apache Spark/attachments/Untitled 27 1.png|Untitled 27 1.png]]

Good, it started!

Now, let’s reach out the Spark UI using YARN Resource Manager again and you will see an application link:

![[Apache Spark/attachments/Untitled 28 1.png|Untitled 28 1.png]]

Drag the horizontal slide at the most right side and click on `ApplicationMaster` under the Tracking UI column:

![[Apache Spark/attachments/Untitled 29 1.png|Untitled 29 1.png]]

You can see that the YARN started those 5 executors in the yellow box, but soon it removed all of them except one (in red). That’s because of the dynamic allocation feature of YARN: since we’re not doing anything and the executors are IDLE, YARN has released all of them except one.

Now let’s close the spark-shell and start another one where we’ll specify the number of executors explicitly with:
```Shell
spark-shell --num-executors 3
```

Now, I expect that YARN should begin with 3 executors and to prove that, go again in the Spark UI and click on `ApplicationMaster` of the new application:

![[Apache Spark/attachments/Untitled 30.png|Untitled 30.png]]

3 executors are started and 2 are released soon.

Where is the **Driver**? Click on the Executors tab in the navbar of the Spark UI and you’ll see that the Driver is running on the Master Node:

![[Apache Spark/attachments/Untitled 31.png|Untitled 31.png]]

This is where we did the SSH (_what does that precisely mean?_). So the Driver started on my local computer. There were 3 executors: one on the node-0, one on the node-1, one on the node-2. Executors on nodes 0 and 2 died, while the one on node 1 is still active.

### Cluster Mode
The command is the same as we used earlier, but this time we’re explicitly specifying the `--deploy-mode` parameter:
```Shell
spark-submit --class org.apache.spark.examples.SparkPi --deploy-mode cluster file://usr/lib/spark/examples/jars/spark-examples.jar 1000
```

Now let’s check the application on the YARN UI:

![[Apache Spark/attachments/Untitled 32.png|Untitled 32.png]]

Let’s check the driver in the Executors tab:

![[Apache Spark/attachments/Untitled 33.png|Untitled 33.png]]

We used spark-submit from the master node, and the Driver is running on one of the worker nods, meaning that the driver is also running in the cluster and it has no dependency on my local computer.

---

# 4. Internal mechanism of parallel processing
We learned about Application Driver and Executors. We know that Apache Spark breaks our application into many smaller tasks and assign them to executors. So **Spark executes the application in parallel**. Let’s dive into the internal mechanism to understand how Spark breaks out code into a set of tasks and run it in parallel.

All that you are going to do in Apache Spark is to read some data from a source and load it into Spark, process the data and hold the intermediate results. Finally, write the results back to a destination. But in this process, you need a data structure to hold the data in Spark. We have 3 alternatives:
- **DataFrame**
- **DataSet**
- **RDDs**

Spark 2.x recommends to use the first 2 and avoid using RRDs. But Both DataFrames and DataSets are ultimately compiled down to an RDD. So, under the hood, everything in Spark is an RDD. For this reason, let’s start with RDDs and try to explain the mechanism of parallel processing.
## 4.1. RDD
The name **RRD** stands for **Resilient Distributed Dataset**.
> [!important]  Spark RDD
> We can descrive Spark RDD as a resilient, partitioned, distributed, and immutable collection of data:
> - **Collection** of data: it holds data and appears to be a Scala Collection.
> - **Resilient**: it is fault-tolerant.
> - **Partitioned**: Spark breaks the RDD into smaller chunks of data. These pieces are called partitions.
> - **Distributed**: instead of keeping these Partitions on a single machine, Spark spreads them across the cluster.
> - **Immutable**: once defined, you can’t change it. Spark RDD is a read-only data structure.

You can create an RDD using 2 methods:
1. Load some data from a source;
2. Create an RDD by transforming another RDD.

## 4.2. Example
Let’s load some data from a file to create an RDD, and then we’ll show the partitions.

After opening the Spark Shell:
```Scala
val flistRDD = sc.textFile("flist.txt")
```
Let's run:
```Scala
flistRDD.getNumPartitions
```
Output:
![[Apache Spark/attachments/Untitled 34.png|Untitled 34.png]]

If you want, you can override the defaults and create as many partitions as you want. Then:
```Scala
val flistRDD = sc.textFile("flist.txt", 5)
```
Output:
![[Apache Spark/attachments/Untitled 35.png|Untitled 35.png]]

You can interate to all partitions and count the number of elements for each partition.

```Scala
flistRDD.foreachPartition(p => println("No of items in parition-" + "p.count(y=>true)))
```
Output:
![[Apache Spark/attachments/Untitled 36.png|Untitled 36.png]]

Let' go back to the main question of this paragraph.
## 4.3. How does the Spark breaks the code into a set of task and run it in parallel?
Firstly, let’s answer to this question: given the following RDD, if I want to count the number of lines in this RDD, can we do it in parallel?

![[Apache Spark/attachments/Untitled 37.png|Untitled 37.png]]

We have 5 partitions. We’ll give one partition to each executor and ask them to count the lines in the given partition.

![[Apache Spark/attachments/Untitled 38.png|Untitled 38.png]]

Then, I’ll take the counts back from these executors and sum it.

Let’s take again the previous example:
![[Apache Spark/attachments/Untitled 39.png|Untitled 39.png]]

Now, let’s check the SparkUI to see what happens under the hood:
![[Apache Spark/attachments/Untitled 40.png|Untitled 40.png]]

We see:
- 1 job
- 1 stage
- 5 tasks

Here's the **explanation**. We asked for the count, and hence Spark started one job. Spark breaks this job into 5 tasks because we had 5 partitions: one counting task per partition. A task is the smallest unit of work, and it is performed by an executor. We are running in local mode, hence we have a single executor. Hence all of these tasks are executed by the same executor.

Before going ahead, let’s dive into this terminology more in-depth:
- **Job**. A parallel computation consisting of multiple tasks that gets spawned in response to a spark action (i.e. `save()`, `collect()`).
    
    During interactive sessions with Spark Shells, the driver converts your Spark applications into one or more Spark Jobs:
    
    ![[Apache Spark/attachments/Untitled 41.png|Untitled 41.png]]
    
    It then transforms each job into a DAG.
    
- **Stage**. Each job gets divided into smaller sets of tasks called stages that depend on each other.
    
    As part of the DAG noes, stages are created based on what operations can be performed serially or in parallel. Not all Spark operations can happen in a single stage, so they may be divided into multiple stages.
    
    ![[Apache Spark/attachments/Untitled 42.png|Untitled 42.png]]
    
- **Task**. A single unit of work or execution that will be sent to a Spark executors.
    
    Each stage is comprised of Spark tasks (a unit of execution), which are then federated across each Spark executor; each task maps to a single core and works on a single partition of data. As such, an executor with 16 cores can have 16 or more tasks working on 16 or more partitions in parallel, making the execution of Spark’s tasks exceedingly parallel.
    
    ![[Apache Spark/attachments/Untitled 43.png|Untitled 43.png]]
    

Let’s check the details:
![[Apache Spark/attachments/Untitled 44.png|Untitled 44.png]]

![[Apache Spark/attachments/Untitled 45.png|Untitled 45.png]]

We can see the 5 tasks and the driver itself executed all of them.
## 4.4. How to control Parallelism
There are 2 main variables to control the degree of parallelism:
1. the number of partitions
2. the number of executors

If you have 10 partitions, you can achieve 10 parallel processing at most. However, if you have just 2 executors, all those 10 partitions will be queued to those 2 executors.

Look at this code:
```Scala
val flistRDD = sc.textFile("flist.txt", 5)
flistRDD.count()
```

Look at the second line: the data is already partitioned and distributed over the executors. Now, all that Apace Spark has to do is to send this `count()` function over to those executors. They run the function on the partition that they have and return the result.

Let’s make something more complicated. The following is the same code written in Scala and in Python:

![[Apache Spark/attachments/Untitled 46.png|Untitled 46.png]]

and the following is a sample from the input data file:

![[Apache Spark/attachments/Untitled 47.png|Untitled 47.png]]

Code explaination:
- Line 1: loads a text file into an RDD. We want to make 5 partitions of this file.
- Line 2: executes a map method on the first RDD and returns a new RDD. We already know that RDD are immutable, so we can’t modify the first RDD. This map operation splits each line into an array of words. Hence, the new RDD is a collection of arrays. This RDD should look like this:
	``` terminal
	[”etc”, “abrt”]
	["etc", "abrt", "plugins"]
	["etc", "abrt", "plugins", "CCpp.conf"]
	...
	```
- Line 3: it executes another map method over the arrayRDD. This time a tuple is generated, a key-value pair. The first element of the array is the key, while the value is hardcoded to 1. What we are trying to do is to count the number of files in each different directory (for example we have 8 files in the directory “etc”). In this case, I’ll obtain:	
	```
    (etc, 1)
    (etc, 1)
    (etc, 1)
    ...
    ```
- Line 4: now, all we have to do is to group all the values by the key and sum up these 1s.
- Line 5: finally, I collect all the data back from the executors to the driver.

(*Find information about MAP & REDUCE method in Learning Journal in Scala tutorial channel.*)

Let’s execute this code (except the `collect()`function) in a multi-node cluster (6-node cluster) and we’ll see what happens under the hood. Check the SparkUI:
![[Apache Spark/attachments/Untitled 48.png|Untitled 48.png]]

What is strange is that there are no jobs here, only executors. We created 4 RDD and in fact, there are 4 different RDD. The reason is that:

> [!important]  
> All these functions that we executed on various RRDs are lazy. They don’t perform anything until you run a function that is not lazy.  

We call:
- lazy functions as **Transformations**
- non-lazy functions as **Actions**

Let’s formalize these two terms.

RDDs offer 2 types of operations: **Transformations** and **Actions**.
- The **Transformation** operations create a **new distributed dataset** from an existing distributed dataset; so they create an RDD from an existing RDD.
- The **Actions** are mainly performed to send results back to the driver, and hence they produce a **non-distributed dataset**.

The `map` and `reduceByKey` are Transformations, whereas `collect`is an Action.

All transformation in Spark are lazy, meaning that they don’t compute results until an action requires them to provide results.

That’s the reason why we don’t see any jobs in SparkUI. Indeed, if we perform the `collect()`function, here’s what we see in the SparkUI:

![[Apache Spark/attachments/Untitled 49.png|Untitled 49.png]]

We can see 1 Job, 2 Stages and 10 Tasks.

Apache Spark completed this job in 2 stages (it couldn’t do it in a single Stage, and we’ll look at the reason later on). We had 5 partitions, so we expected 5 tasks. Since the job went into 2 Stages, we have 10 Tasks (5 Tasks for each Stage).

Let’s see why we had 5 Tasks per Stage, by looking the **DAG**, that show the whole process in a nice visualization:

![[Apache Spark/attachments/Untitled 50.png|Untitled 50.png]]

In Stage 0:
- file load,
- first map operation
- second map operation

Spark was able to complete all of these 3 activities in a single Stage.

In Stage 1:
- reduceByKey function

Spark took a new Stage for `reduceByKey` operation. Why? Let’s make a logical diagram to understand it.

Here’s the initial RDD and the 5 partitions we created:

![[Apache Spark/attachments/Untitled 51.png|Untitled 51.png]]

and Spark assigned these partitions to 5 executors.

![[Apache Spark/attachments/Untitled 52.png|Untitled 52.png]]

Now we perform our first map operation: split each line and split it into individual words. Each executor performs this map function on the partition that they hold:

![[Apache Spark/attachments/Untitled 53.png|Untitled 53.png]]

After completing the first transformation, they keep the transformed data with themselves. They don’t send to me because I haven’t asked for it yet.

Now we perform the second map operation, where a key-value pair for each row is created:

![[Apache Spark/attachments/Untitled 54.png|Untitled 54.png]]

Again, they keep the new transformed data with them. There is no need for any data movement yet. Hence, Spark was able to perform all these operations in a **single stage**.

But now, we want to group the whole dataset by key and then count the number of ones for each key:

![[Apache Spark/attachments/Untitled 55.png|Untitled 55.png]]

But, the keys are spread across the partitions! So, how is Spark going to do that?

To accomplish this task, we need to repartition the data in a way that we get all the records for a single key in one partition. This new partitioning is based on the key. So, essentially, these old partitions will send the data to a new partitions.

![[Apache Spark/attachments/Untitled 56.png|Untitled 56.png]]

These exchange of data happens in a way that it collects all the data for the same key into a single partition. We don’t have to write code for all this exchange because Spark performs this action behind the scene.

We call it a **shuffle and sort activity**.

During this exercise, Spark moves the data and creates new partitions. That’s why it needs a **second Stage**.

> [!important]  
> So, whenever there is a need to move data across the executors, we need a new stage.  

Spark engine will identify such needs and break the job into 2 stages.

Whenever you do something that needs moving data (i.e. a groupBy operation or a join operation) you will notice a new Stage.

Once we have these key-based partitions, it is simple for each executor to calculate the count for the key that they own.

Finally, they send these results to the driver because we executed the `collect()` action:

![[Apache Spark/attachments/Untitled 57.png|Untitled 57.png]]

---

# 5. Transformations, Actions, and Lazy evaluation
Let’s go back to the concepts of DataFrames and Partitions.
## 5.1. DataFrames
As we explain before, Spark has two fundamental sets of APIs:
- the **low-level “unstructured” APIs.** It consists of **DataFrames** and **Datasets**. They are designed and optimised to work with structured data.
- the **higher-level structured APIs**. They are the lower level APIs, including **RDDs**, Accumulators and Broadcast variables.

A ==**DataFrame**== is the most common Structured API and simply represents a table with row and columns. The list that defines the columns and the types within those columns is called the **Schema**. You can think of a DataFrame as a spreadsheet with named columns.

The fundamental difference between a spreadsheet and a Spark DataFrame is that this last one can span thousands of computers, whereas a spreadsheet sits on one computer in one specific location.

The DataFrame concept is not unique to Spark. **R** and **Python** both have similar concepts. However, Python/R DataFrames (with some exception) exist on one machine rather than multiple machine. However, because Spark has languages interfaces foro both Python and R, it’s quite easy to convert Pandas (Python) DataFrames, and R DataFrames to Spark DataFrames.
## 5.2. Partitions 
To allow every executors to perform work in parallel, Spark breaks up the data into chunks called ==**partitions**==. A partition is a collection of rows that sit on one physical machine in your cluster:
- If you have 1 partition, Spark will have a parallelism of 1, even if you have thousands of executors.
- If you have many partition, but only 1 executor, Spark have a parallelism of 1.

PS. You do not manipulate partitions manually or individually. You simply specify the high-level transformations of data, and Spark determines how this work will actually execute on the cluster.
## 5.2. Transformations
In Spark, the core data structures are _immutable_, meaning they cannot be changed after they are created. To “change” a DataFrame, you need to instruct Spark how you would like to modify it. to do what you want. These instructions are called ==**transformations**==.

Here’s a simple transformation to find all even numbers:
```Python
divisBy2 = myRange.where('number % 2 = 0')
```
This code returns no output! Spark will not act transformations until we call an **action**.

There are 2 types of transformations:
1. **narrow transformations**: are those for which each input partition will contribute to only one output partition. The `where()` statement specifies a narrow dependency, where only one partition contributes to at most one output partition:
    
    ![[Apache Spark/attachments/Untitled 58.png|Untitled 58.png]]
    
2. **wide transformations**: are those for which input partitions contributing to many output partitions. It is also knows as **shuffle** whereby Spark will exchange partitions across the cluster:
    
    ![[Apache Spark/attachments/Untitled 59.png|Untitled 59.png]]
    

With narrow transformations, Spark will automatically perform an operation called pipelining, meaning that if we specify multiple filters on DataFrame, they’ll be performed in-memory.

On the other hand, when we perform a shuffle, Spark writes the results to disk. Discussions about shuffle optimisation across the web leads to a topic called **lazy evaluation**.
## 5.3. Lazy evaluation
> [!important]  
> Lazy evaluation means that Spark will wait until the very last moment to execute the graph of computation instructions.  

Instead of modifying the data immediately when you express some operation, you build up a plan of transformations that you would like to apply to your data. By waiting until the last minute to execute the code, Spark compiles this plan from your DataFrame transformations to a streamlined physical plan that will **run as efficiently as possibile across the cluster**. This provides immense benefits because Spark can optimise the entire data flow from end to end. An example of this is called **predicate pushdown** on DataFrames. If we build a large Spark job but specify a filter at the end that only requires to fetch one row from our source data, the most efficient way to execute this is to access the single record that we need and only after apply the transformation.
## 5.4. Actions
Transformations allow us to build up our logical transformation plan. To trigger the computation, we run an **action**. An action instructs Spark to compute a result from a series of transformations. The simplest action is `count()`, which gives us the total number of records in the DataFrame.