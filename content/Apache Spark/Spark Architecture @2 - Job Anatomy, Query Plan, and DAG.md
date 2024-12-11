---
date: 2024-08-11
modified: 2024-08-11T20:45:56+02:00
---
Although some of the following concepts have already been outlined in another post (link here), let us recap some of them.

# 0. Resources
 * “Spark Optimization with Scala” course by Daniel Ciocîrlan (link here: [https://rockthejvm.com/p/spark-optimization](https://rockthejvm.com/p/spark-optimization))

---

# 1. Core
## 1.1. Spark Architecture
**Spark Architecture** consists of a **Cluster of Worker nodes** that performs the data processing. Main concepts here:
* **Executor**: worker logical node (JVM)
	* performs work for a single application
	* usually more than one per application
	* launched in JVM containers wit their own memory/CPU resources
	* can be 0 or more deployed on the same physical machine
* **Driver**: Spark Application main JVM
	* one per application
	* starts the application and sends work to the executors
* Cluster managed the deployment of the executors
	* driver requests executors & resources from the cluster manager
* For **performance**, a bunch of recommendations:
	- driver should be close to the worker nodes in the sense that they will share the same physical rack or the same physical location so that the data transfer doesn’t take a lot of time.
	- the worker nodes need to be close to each other because otherwise the shuffle operations will be even more expensive that they need to be.
## 1.2. APIs
**RDD API**:
* Distributed typed collections of JVM objects
* The "first citizens" of Spark: all higher-level APIs reduce to RDDs
* Pros: can be highly customized
	* distribution can be controlled
	* order of elements can be controlled
	* arbitrary computation hard/impossible to express with SQL
* Cons: hard to work with
	* for complex operations, need to know the internals of Spark
	* poor APIs for quick data processing

**DataFrame API**:
* high-level distributed data structures
	* contain rows
	* have additional API for querying
	* support SQL directly on top of them
	* generate RDDs after Spark SQL planning & optimizing
* Pros:
	* easy to work with, support SQL
	* already heavily oprimized by Spark
* Cons:
	* type-unsafe
	* unable to compute everything
	* hard to optimize further

**Dataset\[T]**:
* Distributed typed collections of JVM objects
	* support SQL functions of DataFrames
	* support functional operators like RDDs
* DataFrame = Dataset\[Row]
* Pros:
	* easy to work with, support for both SQL and functional programming
	* some Spark optimizations out of the box
	* type-safe
* Cons:
* memory and CPU expensive to create JVM objects
* unable to optimize lambdas

**Performance tips**:
* Use DataFrames most of the time
	* express almost anything with SQL
	* Spark already optimizes most SQL functions
* Use RDDs only in custom processing
* Do not switch types
	* Switching between DFs and RDD\[YourType] (or Dataset\[YourType]) is expensive
	* in Python switching types is disastrous
## 1.3. Lazy evaluation and Planning
Spark has a **Lazy Evaluation** computing model in the sense that Spark will wait until the last moment to execute **Transformations**. We defined several transformations and Spark will decide the dependencies between the data structures, but they will not be actually evaluated until you call an **Action**.

Before you call an action, Spark will do what is called **Planning**, that it Spark will compile DF or SQL transformations to RDD transformations if necessary. So if you’re using the DataFrame API, that will be transformed to RDD dependencies. Spark will compile the RDD transformations into a graph (**DAG**).

Spark will also do some **Logical Planning** in that you will see the RDD dependencies graph and narrow and wide transformations (we’ll do later on). The logical plan will then be transformed into a **Physical Plan**, which is an optimised sequence of steps, dedicated for nodes in the actual cluster. So Spark will know in advance what each of the nodes will do throughout the computation.

Summing up:
- Lazy evaluation
	- Spark waits until the last moment to execute the DF/RDD transformations
- Planning
	- Spark compiles DF/SQL transformations to RDD transformations (if necessary)
	- Spark compiles RDD transformations into a graph before running any code
	- logical plan = RDD dependency graph + narrow/wide transformations sequence
	- physical plan = optimized sequence of steps for noeds in the cluster
- Transformations vs Actions
	- transformation describe gow new DFs are obtained
	- actions start executing Spark code
	- transformations return RDDs/DFs
	- actions return something else e.g. Unit, a number etc.
## 1.4. Spark App Execution and Job Decomposition
* An **Action** triggers a **Job**
* A job is split into **Stages**
	* each stage is dependent on the stage before it
	* a stage must fully complete before the next stage can start
	* for performance: (usually) minimize the number of stages
* A stage has **Tasks**:
	* task = smallest unit of work
	* tasks are run by executors
* An RDD/DataFrame/Dataset has partitions

Let's see how **these concepts are related** to each other:
* App decomposition
	* 1 job = 1 or more stages
	* 1 stage = 1 or more tasks
* Tasks & Executors
	* 1 task is run by executor
	* each executor can run 0 or more tasks
* Partitions & Tasks
	* processing one partition = one task
* Partition & Executors
	* 1 partition stays on 1 executor
	* each executor can lead 0 or more partitions in memory or on disk
* Executors & Nodes
	* 1 executor = 1 JVM on 1 physical node
	* each physical node can have 0 or more executors

---
# 2. Spark Job Anatomy
The objective here is to:
* understand how Spark splits jobs into computational chunks
* make the distinction between narrow and wide transformations
* understand shuffles
## 2.1. How Spark splits jobs into computational chunks
Let’s define a small RDD:
```scala
>>> val rdd1 = sc.parallelize(1 to 1000000)
```
Let’s kick off a Spark Job by running an action:
```scala
>>> rdd1.count()
```

Let’s take a look at the detail of this new job in the SparkUI:
![bb54449a581a384a41beac522b61d811](Apache%20Spark/attachments/bb54449a581a384a41beac522b61d811.png)

This Job has **1 Stage** and that stage has **8 Tasks**. A task is the fundamental unit of computation and a task is being performed on a **Partition** of the RDD in question. So we have an RDD with a million elements and apparently this RDD has already been split into 8 partition and 1 of these partitions corresponds to 1 task.

Let’ run another job:
```scala
>>> rdd1.map(_ * 2).count()
```
![e790134d4b60b46cf4f4c51bd256eb51](Apache%20Spark/attachments/e790134d4b60b46cf4f4c51bd256eb51.png)
This job has a single stage because a map operation does not need repartitioning of the RDD. It can just multiply each element by 2 in the same structure that this RDD already conforms to.

Let’s do a repartition:
```scala
>>> rdd1.repartition(23).count()
```
This action takes a little bit of time because repartitioning means that the RDD is now split differently, not into 8 partitions, but into 23 partitions. Let’s look at the detail of this new job in the SparkUI:
![362d61ddc9611366f2e59dda0dbc946c](Apache%20Spark/attachments/362d61ddc9611366f2e59dda0dbc946c.png)

It took more than the previous jobs and it is much larger than the previous ones. Indeed we have **2 stages** and **31 tasks** in total. Why 31?

In *StageId 2*, Spark computes 8 tasks because when you start off an RDD, that’s usually splits into 8 partitions because this is the default number of partitions that Spark will pick. You can visualize this default number by:
```scala
>>> sc.defaultParallelism
```
that in our case returns 8. (*I think that will number is set by default as the number of cores of your laptop. It seems confirmed by many sources, like these ones [1](https://www.1week4.com/it/big-data/apache-spark-partizioni/), [2](https://www.linkedin.com/pulse/how-initial-number-partitions-determined-pyspark-sugumar-srinivasan/), [3](https://stackoverflow.com/questions/44222307/spark-rdd-default-number-of-partitions))

In *StageId 3*, Spark computes 23 tasks because we kicked off a repartition and Spark will need to change the way that these RDD is being structured and we chose that this RDD is being repartitioned into 23 partitions.

Because we kicked off a repartition, Spark will need to rearrange this data and that will trigger another stage. So whenever an RDD or a DF needs to be repartitioned, Spark will start a **Shuffle**, that is a **data exchange in between the executors in this cluster**. A shuffle means the limit between two stages, so whenever Spark need to do a shuffle, that is the limit where a stage ends and the next stage starts. Many operations trigger a shuffle, not only the repartition shuffle.

> [!tip] Shuffle are expensive
> **Shuffles** are **expensive** because the data transfer usually takes a lot of time.

Let's run a more realistic computation on a CSV file:me more realistic computation:
```scala {6}
>>> val employees = sc.textFile("/tmp/data/employees/employees.csv") -> this is an RDD of Strings
>>> val empTokens = employees.map(line=>line.split(",")) -> this is an RDD of Array of Strings
>>> val empDetails = empTokens.map(tokens => (tokens(4), tokens(7))) -> this is an RDD of tuple of Strings
>>> val empGroups = empDetails.groupByKey(2) // 2 is the number of partitions
>>> val avgSalaries = empGroups.mapValues(salaries => salaries.map(_.toInt).sum /salaries.size)
>>> avgSalaries.collect().foreach(println)
```
The result:
![03935911c1dfb7d5382c4fe47c148bf7](Apache%20Spark/attachments/03935911c1dfb7d5382c4fe47c148bf7.png)

The details of this job:
![bd41839c774bbb1c80c8520a9aa6af05](Apache%20Spark/attachments/bd41839c774bbb1c80c8520a9aa6af05.png)

The **2 map operations** in the DAG correspond to the 2 map operation that we have performed in the code and they are executed into a single stage. So whenever you have an operation that does not need to change the structure of an RDD, then you have this operation in the same stage, and this is why you have the 2 map operations in the same stage.

Then you have the `collect()` function that contains the **`groupByKey()` function and the `map()` function in another stage**. This is because when you need to do a group, which is a `groupByKey()` in our case, then you need to change the structure of the RDD. Indeed in the `groupByKey` case, you need to make sure that all the rows with the same key are on the same partitions. So groups will incur a **Shuffle** and, as we mentioned earlier, shuffle is the limit between two stages. The reason why we have 2 tasks is because we used the number `2` in the `groupByKey()` function, so we forced that RDD to have 2 partitions and because we have  partitions we have 2 tasks because, as we said, a task corresponds to processing one partition in the cluster.

Let’s split the code we wrote in the shell into a diagram:
![59b6984f9d27c9a6d5707ae3256f4456](Apache%20Spark/attachments/59b6984f9d27c9a6d5707ae3256f4456.png)
![051eb5e8ec36b2cdd84079117bd3d51a](Apache%20Spark/attachments/051eb5e8ec36b2cdd84079117bd3d51a.png)
![85b077cf23dd2523423b4125a1415397](Apache%20Spark/attachments/85b077cf23dd2523423b4125a1415397.png)
![4319367bd80cb1a4eb1e0fcdc9f2eb01](Apache%20Spark/attachments/4319367bd80cb1a4eb1e0fcdc9f2eb01.png)
![049044f6755a80fa5463b7064cb4178a](Apache%20Spark/attachments/049044f6755a80fa5463b7064cb4178a.png)
![fd5a2389f727421ccdf875dbc32ad6ce](Apache%20Spark/attachments/fd5a2389f727421ccdf875dbc32ad6ce.png)

Let's summarize some concepts:
* **Task**
	* the smallest unit of computation
	* executed once, for one partition, by one executor
* **Stage**
	* contains tasks
	* enforces no exchange of data = no partitions need data from other partitions
	* depends on the previous stage = previous stage must complete before this one starts
* **Shuffle**
	* exchange of data between executors
	* happens in between stages
	* must complete before next stage starts
* An **Application** contains **Jobs**
* a job contains **Stages**
* s stage contains **Tasks**
## 2.2. Narrow and Wide Transformations
* **Narrow Dependencies**:
	* one input (parent) partition influences **a single** output (child) partition
	* fast to compute
	* examples: map, flatMap, filter, projections
+ **Wide Dependencies**:
	* one input partition influences **more than one** output partitions
	* involve a shuffle = data transfer between Spark executors
	* are costly to compute
	* examples: grouping, joining, sorting

Graphically:
![](Apache%20Spark/attachments/Spark%20Diagrams-Page-12.drawio%20(2).svg)
## 2.3. Shuffles
* Data exchange between executors in the cluster
* Expensive because of:
	* transferring data
	* serialization/deserialization
	* loading new data from shuffle files
* Shuffles are performance bottlenecks because:
	* exchanging data takes time
	* they need to be fully completed before next computations starts
* Shuffle limit parallelization

---
# 3. Reading Query Plans
Objectives:
* understand how Spark "compiles" a SQL/DataFrame job
* read query plans
* predict job performance based on query plans

How a SQL **job** is being so-called **compiled** before it’s actually run on the cluster? This sequence diagram shows all the steps that Spark will perform before a SQL job will actually run and the middle and largest part of this diagram is called **Catalyst Query Optimizer**:
![](Apache%20Spark/attachments/Spark%20Diagrams-Page-12.drawio%20(3).svg)

When you run a SQL job:
* Spark knows the DF dependencies in advance - **unresolved logical transformation plan**
* Catalyst resolves references and expression types - **resolved logical plan**
* Catalyst compresses and pattern matches on the plan tree - **optimized logical plan**
* Catalyst generates **physical execution plans**

The **Selected Physical Plan** is the one that we can see by running `.explain()` function in the console. After the selected physical plan is produced, Spark will proceed to generate some Java and JVM byte-code so that the actual RDDs, that will back up those DF, will we produced and executed throughout the cluster.

Let’s define a small DF:
```scala
>>> val simpleNumbers = spark.range(1, 1000000)
```
This returns a Dataset:
![2afa750077fa1ca0fbd95587564e69c1](Apache%20Spark/attachments/2afa750077fa1ca0fbd95587564e69c1.png)

and, as we know, Datasets support the SQL language on Spark, so:
```scala
>>> val times5 = simpleNumbers.selectExpr("id * 5 as id")
```
Before triggering a job, let’s call the `.explain()` method that will show the query plan that the operation `id * 5` will end up executing if I run a job:
```scala
>>> times5.explain()
```
![591694fb4762de02963681db111daa8a](Apache%20Spark/attachments/591694fb4762de02963681db111daa8a.png)

The bottom step is construct a Range from 1 to 1000000 with a step 1, and `splits=8` means 8 partitions in this DF.  The next step is a Projection, where “project” means “select”. `0L` is  the internal Spark identifier of column `id` and `2L` is the identifier of the new column.

Let’s construct a more complicated DF:
```scala
>>> val moreNumbers = spark.range(1, 1000000, 2)
>>> val split7 = moreNumbers.repartition(7)
>>> split7.explain()
```
![38bb2dc7de9710cfe8a6079c00f593fc](Apache%20Spark/attachments/38bb2dc7de9710cfe8a6079c00f593fc.png)

The first step is the same as above. The `Exchange` keyword means a shuffle, and after that, we will have access to implementation of that exchange which is a `RoundRobinPartitioning`, which means a partition scheme in which every element is being put into a partition in turn (so first number on the first partition, second number on the second partition, and so on). Now let's execute:
```scala
>>> split7.selectExpr("id * 5 as id").explain()
```

Another physical plan:
![fbaf73e64fb2a3a03c041c5f3a06e39f](Apache%20Spark/attachments/fbaf73e64fb2a3a03c041c5f3a06e39f.png)

A more complicated arrangement:
```scala
>>> val ds1 = spark.range(1, 10000000)
>>> val ds2 = spark.range(1, 20000000, 2)

>>> val ds3 = ds1.repartition(7)
>>> val ds4 = ds2.repartition(9)

>>> val ds5 = ds3.selectExpr("id * 3 as id")

>>> val joined = ds5.join(ds4, "id")

>>> val sum = joined.selectExpr("sum(id)")
```
then:
```scala
>>> sum.explain()
```
Output:
![03578f3c08577b56cec2ae8b039665bb](Apache%20Spark/attachments/03578f3c08577b56cec2ae8b039665bb.png)

Let’s read it from the bottom to the top. The `SortMergeJoin` has 2 branches. Let’s start from the bottom one:
- range between 1 to 20000000 with a split of 2 → `ds2`
- `Exchange` with `RoundRobinPartitioning` with 9 → `ds4`
- another `Exchange`, that is another Shuffle with another partitioning scheme called `hashparitioning` over the column `id` with identifier `2L` into 200 partitions.
- after 2 sequential partitioning there is a `Sort` by the column `id` with identifier `2L` in ascending order putting the NULLS first. This is the preparation for the join that we are doing later on. So `Exchange hashparitioning` + `Sort` are preliminary operations to join, which is something that happens in the other DF as well.

Let’s move to the second branch:
- range between 1 to 10000000 with a split of 1 → `ds1`
- `Exchange` with `RoundRobinPartitioning` with 7 → `ds3`
- multiply each element by 3 and the new `id` column has `8L` as identifier → `ds5`
- another `Exchange` with `hashpartitioning` scheme over the column `id` with identifier `8L` into 200 partitions
- `Sort` by the column `id` with identifier `8L` in ascending order putting the NULLS first.

Note: the reason why Spark exchanges data with `Exchange hashpartitioning` before a join is that, in order to do a join in a distributed fashion, the rows with the same keys need to be on the same partition. The goal is to minimize the data movement during the join operation. And this is why both DF need to be partitioned with the exact same partitioning scheme by the column with which we are doing the join. Then we are sorting them with `Sort` so that the join is being done in a sorted fashion. Comparing items in a sorted fashion is much faster than comparing items in a non-sorted fashion.

Coming back to the physical plan, now there is the join:
- the implementation of the join is called `SortMergeJoin` and there is the list of the column by which we’re doing the join (id with identifier `8L` and id with identifier `2L`). The join is an `inner` join.
- then there is a `Project`, meaning that Spark is selecting one of those columns because  an inner join will simply produce duplicates data into these two columns, so Spark will simply select one of them, in our case Spark select the leftmost column `id` with identifier `8L`.

After the join, Spark need to create the DF `sum`:
- there is `HashAggregate` with a `partial_sum` function: Spark operates a partial sum function on all the values  on each partition. And then with the obtained RDD, then it will `Exchange` with a single partition so that it will bring all the values onto a single partition. And then it will run the same `HashAggregate` again with a final `sum` function with all the intermediate results to obtain a single value.

Let’s show some additional information that we can get out of this query plan, in particular **stage and task decomposition**:
* **Task**: remember that is the fundamental unit of computation that is being performed on a single partition. Because you have partitioning information in the query plans (for example we have seen the number of splits, or the number of partitions after a repartition function, or the `hashpartitioning` with the number of partitions, and so on), you can predict the number of tasks that this job will take. So you can count the number of tasks by summing up all those numbers specified in the query plan corresponding to partitions.
+ **Stage**. The numbers inside the parentheses in front of almost every lines indicated the Stage Identifier. So in Stage1 we will construct a range; in Stage2 we will do a Project and in between stages we have a shuffle (that is why exchanges do not have a stage because shuffles happens in between stages). So from query plans you can probably tell how many Stages and Tasks we have in this job if we were to trigger it. In particular, looking at the number inside the parentheses, we should have 7 Stages.

Note. Even though it is recommended to read a query plan from bottom to top, you can also do the opposite by reading the operations at the top in terms of “depending on” the operations at the bottom.

**Summing up:**
* a **Query Plan**:
	* describes all the operations Spark will execute when the action is triggered
	* has information about partitioning scheme
	* has information about the number of partitions in advance
* Explain (true) will give:
	* the parsed logical plan
	* the analyzed logical plan
	* the optimized logical plan (via Catalyst)
	* the physical execution plan (generated by Catalyst)
* Spark works like a compiler

**To Remember:**
* Query plans = layout of Spark computations (before they run)
* whenever you see "exchange", that's a shuffle
* number of shuffles = number of stages
* number of tasks = number of partitions of each intermediate DF
* sometimes Spark already optimized some plans!

---
# 4. Reading DAGs
Objective:
+ correctly interpret the information in the Spark UI
+ know where to go for information
+ read and understand DAGs

As we already know, every `SparkSession` that you start will open a port, which will open a web server (**SparkUI**) where you can inspect the information related to that Spark Session. Let's start by running:
```scala
>>> val rdd1 = sc.parallelize(1 to 1000000)
>>> rdd1.map(_*2).count()
```
In the Spark UI:
![181cc948ad9a539dee6de5c638fc07d1](Apache%20Spark/attachments/181cc948ad9a539dee6de5c638fc07d1.png)

This job is executed in one single stage, because the `map` operation can execute in parallel, so it doesn’t need to change the structure of the RDD and in the DAG we notice two blue boxes corresponding to `sc.parallelize()` and `.map()`. We have 8 tasks because the RDD is split in 8 partitions.

Then:
```scala
>>> rdd1.repartition(23).count()
```
`.repartition()` is a shuffle and it changes the structure of the RDD so I will expect to see a different stage:
![b26ef6fa99838c958a9fae4109e9d4b0](Apache%20Spark/attachments/b26ef6fa99838c958a9fae4109e9d4b0.png)

Notice the Shuffle Read and Shuffle Write specs, that is because Spark changed the structure of the RDD and that is why a stage needs to read the data that the previous stage has written.

Then:
```scala
>>> val ds1 = spark.range(1, 10000000)
>>> val ds2 = spark.range(1, 20000000, 2)
>>> val ds3 = ds1.repartition(7)
>>> val ds4 = ds2.repartition(9)
>>> val ds5 = ds3.selectExpr("id * 3 as id")
>>> val joined = ds5.join(ds4, "id")
>>> val sum = joined.selectExpr("sum(id)")

>>> sum.show()
```

In the Spark UI:
![74d6e45485866affb4eb2417a1ae54eb](Apache%20Spark/attachments/74d6e45485866affb4eb2417a1ae54eb.png)

Note that numbers inside the blue boxes correspond to the numbers inside parentheses in the query plan. If you follow both, it’s easier to understand:
![a91f7c824bfeee20e5a6ba8ded2fe05d](Apache%20Spark/attachments/a91f7c824bfeee20e5a6ba8ded2fe05d.png)

Let’s read the DAG step by step:
- *Stage 0* and *Stage 1*: 8 tasks each because they correspond to the creation of the first two DFs (`ds1` and `ds2`) and they are created independently from each other.
- *Stage 2* and *Stage 3*: after the creation of `ds1` and `ds2`, I’m forcing their repartition into `ds3` and `ds4` and this is a shuffle. A shuffle automatically create another stage that are *Stage 2* and *Stage 3*. So, *Stage 0* and *Stage 1* are both forcing an `Exchange` (notice the `Exchange` blue boxes in the *Stage 1* and *Stage 2*) and this `Exchange` is the preparation phase for whatever come next. *Stage 2* and *Stage 3* have, respectively, 7 and 9 tasks, that are the partitions I wanted. Note that *Stage 2* and *Stage 3* are different because *Stage 2* has an intermediate blue box, that corresponds to the `.selectExpr()`.
- At the end of *Stage 2* and *Stage 3* we have a shuffle that is what happens prior to the join. These 2 exchanges before the join has the purpose of bringing all the rows with the same key on the same partitions, that is on the same executors, so that the join can take place (*I’m not sure about this part. I think that the executor is only 1, that is my local machine. Maybe Daniel would say “same cores” and not “same executors”*). What we have just said correspond to the beginning of *Stage 4*. The we have the `Sort` operation that we have already seen. Then we have the join. And at the very end of *Stage 4* we have an exchange because there is an aggregation with `partial_sum`.
- In *Stage 5* all the intermediate resulting from the `partial_sum` in the aggregation function are put together. This is why we have another `Exchange` at the very first blue box of *Stage 5*:
    ![2e3e0e2e37c046f28bdd8561efce885e](Apache%20Spark/attachments/2e3e0e2e37c046f28bdd8561efce885e.png)
       
    `MapPartitionRDD` in the middle simply compresses an entire partition of an RDD into a single value and then the `MapPartitionRDD` at the end will combine the intermediate values into one.

---
# 5. The Different Spark APIs
Objective:
* how to make RDDs, DataFrames and Datasets interoperate
* understand performance implications for using each

Let's start:
```scala
>>> val rdd = sc.parallelize(1 to 2000000000)
>>> rdd.count()
```

Then:
```scala
>>> import spark.implicits._
>>> val df = rdd.toDF("id")

>>> df.count()
```

Then:
```scala
>>> df.selectExpr("count(*)").show()
```

The SparkUI:
![88ac77365ee5a74232e18b7f59188eac](Apache%20Spark/attachments/88ac77365ee5a74232e18b7f59188eac.png)

Let’s see *JobId 1* and *JobId 2*, corresponding to `rdd.count()` and `df.count()` and let’s see why the second one took twice as long as the first one (2 seconds vs 1 second):
![16595be093ce0d20e72f8c3380c57d88](Apache%20Spark/attachments/16595be093ce0d20e72f8c3380c57d88.png)

![5eb28ecdb9b6fd3159ebb85dc43c5dd9](Apache%20Spark/attachments/5eb28ecdb9b6fd3159ebb85dc43c5dd9.png)

Notice that *StageId 2* took only 45 ms, that is irrelevant compared to *StageId 1*; most of the computation  happened in *StageId 1*. Let’s see the details:
![ca39e9bc39bc75fb1bf7de41c4260d64](Apache%20Spark/attachments/ca39e9bc39bc75fb1bf7de41c4260d64.png)

The 8 tasks are executed in parallel and take roughly between 3 and 4 seconds.

Spark took  a long time to run this stage because whenever you need to convert between an RDD to a DF (in our case `.toDF()` function), that is a **performance hit**.
```scala
>>> val ds = spark.range(1, 2000000000)

>>> ds.count()
>>> ds.selectExpr("count(*)").show()
```

In the Spark UI:
![c0fdb86dcbe29f85ddf662821af1341d](Apache%20Spark/attachments/c0fdb86dcbe29f85ddf662821af1341d.png)

Notice that these 2 actions run jobs that are basically instant compared to the previous ones. That is  because the amount of data that’s actually being shuffled is on the order of bytes which is pretty much instant (*I don’t understand that. The value of bytes is the same for all the jobs, so this shouldn’t be the reason for being so instant).*

Let’s explore the *SQL/DataFame* tab that you have access when you kick off jobs with DataFrame or Dataset APIs where you will notice the jobs that have SQL query plans:

![2db4a83665b71c43f323ab23634f27bf](Apache%20Spark/attachments/2db4a83665b71c43f323ab23634f27bf.png)

Let’s explore the latest job corresponding to `ds.count()` (but it’s the same for `ds.show()`), where we can see all the operations in their implementation and all the collections in their implementation as they’re being used in this computation:
![c456480284abf951c7e5015cfb0a792f](Apache%20Spark/attachments/c456480284abf951c7e5015cfb0a792f.png)

This particular computation starts with a **Range** Collections (in the first block) and Range Collections doesn’t actually evaluate all the elements in the range and so a Range is a very simple data structure when not evaluate element by element, and this is why this particular job takes so little time.

However, if you go back in the SQL/DataFrame tab and click on one of the jobs that took more time (`df.count()` or `df.show()`), we’ll see something different:
![9b5268cb7c301acfa19c66e136055d1d](Apache%20Spark/attachments/9b5268cb7c301acfa19c66e136055d1d.png)

The first operation is a **Scan** and whenever you do a Scan, you will need to consider every element in turn. And whenever you take a look at all these boxes (the ones in the picture, but also the ones below not captured because it would be an huge picture), you will also see the time it takes for every single box to be computed and the box called “WholeStageCodegen” takes the most amount of time (*non mi torna, dice 17.9 secondi ma nella foto sopra \[StageId 1] dice Duration 2 secondi*). It takes 17.9 seconds out of the 3 seconds of the job (*non ha senso 17.9 secondi su 3 totali*). That is because every single element needs to be evaluated individually in the **SerializeFromObject** operation. So, whenever you see **SerializeFromObject** that means an already made collection will be transformed to the DF by considering every single element in turn and this takes a lot of time.

Summing up, the serialisation and the conversion between an RDD and a DF actually takes the most time.

Now:
```scala
>>> val rddTimes5 = rdd.map(_ * 5)

>>> rddTimes5.count()
```

It took 5 seconds:
![1dc9036d7653ea0b89cbecd25b4ebe40](Apache%20Spark/attachments/1dc9036d7653ea0b89cbecd25b4ebe40.png)

and now let’s do the same for `df`:
```scala
>>> val dfTimes5 = df.selectExpr("id * 5 as id")
>>> dfTimes5.count()
```

![a78a490397804a1ec56ec8df4db5a368](Apache%20Spark/attachments/a78a490397804a1ec56ec8df4db5a368.png)

It took 4 seconds that is almost as much time as the original `df.count()`. So why `df.count()` takes 4 seconds and `df.selectExpr("id * 5 as id")` takes almost the same time, when we have just demonstrated that multiplying a billion elements by 5 takes 5 seconds? Let’s look at both the query plans:
![85ad62470adcb822044e0ec18f471c8c](Apache%20Spark/attachments/85ad62470adcb822044e0ec18f471c8c.png)

They’re obviously pretty similar.

In order to run an actual `.count()` computation, a DF will actually perform a count aggregation before returning the value contained in that DF. So this query plan is not really fair. What’s more fair is:
```scala
>>> val dfTimes5Count = dfTimes5.selectExpr("count(*)")
>>> dfTimes5Count.count()
```

Then let’s save the previous result:
```scala
>>> val dfCount = df.selectExpr("count(*)")
```

Let's see the Physical plans:
![95dd9bca41e8056041605d203785faf5](Apache%20Spark/attachments/95dd9bca41e8056041605d203785faf5.png)

They take exactly the same amount of time because the two query plans are 100% identical. So in `dfTimes5.selectExpr("count(*)")` there is no Project with `values*5` in this query plan. Spark eliminated that altogether.

---
# 6. Deploy Modes
Spark has what is called **Execution Modes** or **Deploy Modes** and we specify those when we launch a Spark Application because Spark has to know this in advance.

There are three possibile options and they have performance implications:
- **Cluster Mode**
- **Client Mode**
- **Local Mode**
## 6.1. Cluster Mode
Characteristics:
- the Spark Driver is launched on a worker node
- the Cluster Manager is responsible for Spark processes

If the blue box is the Cluster Manager and the green boxes are the physical machines executing the work, then the driver (the red bullet) will be deployed on one of those machines alongside some of the executors on the physical machines. So the physical machines will also have the Driver and the executors inside the Spark Cluster:
![a1ef8514241509605220d92e8072b34f](Apache%20Spark/attachments/a1ef8514241509605220d92e8072b34f.png)
## 6.2. Client Mode
Characteristics:
- the Spark Driver is on the client machine. So the client machine will talk to the Cluster Manager and it will allocate the driver on its machine
- the Client is responsible for the Spark processes and state management

So if you run a `spark-submit` on Client Mode, then the Driver is on your local laptop and the client is responsible for the spark processes and state management whereas the Cluster Manager is responsible deploying the executors. In this case, the Driver that sits on your local computer will communicate directly with the executors in the physical cluster:
![3ca13c68b3539351f4b47ced4cca60b6](Apache%20Spark/attachments/3ca13c68b3539351f4b47ced4cca60b6.png)
## 6.3. Local Mode
The entire application and the entire Spark “Cluster” runs on the same physical machine that is you laptop.
## 6.4. Comparisons
Let’s compare all these Deployment Modes.
### 6.4.1. Cluster Mode
The Driver is a dedicated JVM container on the cluster, so it shares a physical machine with potentially one or more executors.

**Pros**:
- the Driver is a dedicated JVM Container on the cluster, so it shares a physical machines with potentially one or more executors. The pros of that is that you usually have more memory for the driver because the cluster machines are usually pretty beefy machines
- faster communication between driver and executors because they share the same physical location
- because of those pros, it usually leads to **faster performance**.

**Cons**:
- the failure of the node with the driver means that the entire application will fail
- shipping the driver on the same physical machines with the executors means that you have fewer resources that are allocated to the executors; but it’s not a big downside because the driver usually don’t have that many resources

Summing up:
![ac7a6685512e4d824c5997dc2c43d9c3](Apache%20Spark/attachments/ac7a6685512e4d824c5997dc2c43d9c3.png)

### 6.4.2. Client Mode
The Driver is created on the machine which submits the job.

**Pros**:
- more resources to executors, although, as we mentioned before, it’s not a huge upside
- the node root failure doesn’t crash the entire application, so if one of the green boxes fails, the cluster manager will simply allocate more resources on the other green boxes that have available; so the application is still alive.
- results are immediately available on the machine that is on your client machine. So whenever you return some results to the driver you have immediate access to them rather than shipping them from the cluster to your local machine once the application is done.

**Cons**:
- you have usually fewer resources available to the driver because it’s spun up on your local machine which is probably a less beefy machine that the machine on the cluster
- because the client machine and the worker machines are not in the same physical location that means the communication between the driver and the executors is much much slower
- because of those cons, it usually leads to slower performance.

Summing up:
![42c39f38cff799cfe3d2e8f536cd9bea](Apache%20Spark/attachments/42c39f38cff799cfe3d2e8f536cd9bea.png)

---
# 7. Three ways of configuring Spark Applications
Now let’s see how to configure a spark application in 3 different ways.

1. How to configure a spark application through code, before the Spark Application actually starts running. When you construct a Spark Session method:
	```scala
	val spark = SparkSession.builder()
		.appName("Test Deploy App")
		// method 1
		.config("spark.executor.memory", "1g") // 1 gigabyte of memory
		.getOrCreate()
	```
2. Again through code as method 1, but it will allow you to configure a Spark Application as it’s currently running:
	```scala
	spark.conf.set("spark.executor.memory", "1g") // warning - not all configurations available
	```
    Warning: some configurations are not available to be set while the Spark Application is actually running. So, this particular configuration `"spark.executor.memory"` is illegal!
3. Pass the configuration directly into `spark-submit` tool. So, when you open the command line in the Docker Container, you can add come optional arguments:
    ```scala
    >>> /spark/bin/spark-submit \
    --class part2foundations.TestDeployApp \
    --master spark://b5cc3756826e:7077 \
    -- conf spark.executor.memory 1g \ (or --executor-memory 1g)
    --deploy-mode client \
    --verbose \
    --supervise \
    /opt/spark-apps/spark-optimization.jar /opt/spark-data/movies.json /opt/spark-data/goodComedies.json \
    ```