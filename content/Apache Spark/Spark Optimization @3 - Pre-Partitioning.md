---
date: 2024-08-11
modified: 2024-08-12T11:59:39+02:00
---
# 0. Resources
* “Spark Optimization with Scala” course by Daniel Ciocîrlan (link here: [https://rockthejvm.com/p/spark-optimization](https://rockthejvm.com/p/spark-optimization))

---
# 1. Start the Project
If you’re interested in learning how to create a new Spark project in Scala, refer to the [first post on Apache Spark](Apache%20Spark/Spark%20@0%20-%20Run%20Spark%20Applications.md). In this guide, we utilize the same project that was used in previous tutorials and will continue to use in future ones. You can download this project from the following repository: [https://github.com/simdangelo/apache-spark-blog-tutorial](https://github.com/simdangelo/apache-spark-blog-tutorial).

---

# 2. Pre-Partitioning
The main idea behind **Pre-Partitioning** technique is: **partition your data so that Spark doesn’t have to**.
## 2.1. Partition early is a good idea
Let’s start by creating a new Scala file called `Pre-Partitioning` and let’s write the usual configuration:
```scala
val spark = SparkSession.builder
  .appName("Pre-Partitioning")
  .master("local[*]")
  .config("spark.sql.adaptive.enabled", "false")
  .getOrCreate()
  
spark.sparkContext.setLogLevel("WARN")
import spark.implicits._
```

Let’s say by assumption you have 2 Datasets with different number of partitions:
```scala
val initialTable = spark.range(1, 10000000).repartition(10)
val narrowTable = spark.range(1, 5000000).repartition(7)
```
In order to **join** these two tables, Spark will need to **repartition them in the same partitioning schema**.

Let’s define a method that takes an argument of type `T` and a dataset in the form of `Dataset[T]`, which I supposed it has a column called `id` and a number `n` of type `Int`, and this will return a `DataFrame`. The goal of this method is to add `n` new columns to `df`:

```scala
def addColumns[T](df: Dataset[T], n: Int): DataFrame = {
    val newColumns = (1 to n).map(i => s"id * $i as newCol$i")
    df.selectExpr("id" +: newColumns: _*)
  }
```
For instance, if you call`addColumnd(initiaTable, 3)`, you'll have a `DataFrame` with columns `id`, `newCol1`, `newCol2`, `newCol3` which values are, respectively id\*1, id\*2, id\*3.

Let’s present 2 scenarios:
+ **Scenario 1**:
	```scala
	// scenario 1
	val wideTable = addColumns(initialTable, 30)
	val join1 = wideTable.join(narrowTable, "id")
	```
+ **Scenario 2**:
	```scala
	// scenario 2
	val altNarrow = narrowTable.repartition($"id")
	val altInitial = initialTable.repartition($"id")
	val join2 = altInitial.join(altNarrow, "id")
	val result2 = addColumns(join2, 30)
	```
Then let's run an action to trigger the Spark Application by running:
```scala
def main(args: Array[String]): Unit = {  
  join1.show()  
  result2.show()  

  Thread.sleep(1000000000)  
}
```

If we look at the SparkUI, we'll see that *Scenario 1* runs in **16 seconds**, while *Scenario 2* runs in **2 seconds**:
![739489944ed0c22f489210fea10bcc91](Apache%20Spark/attachments/739489944ed0c22f489210fea10bcc91.png)

Let's see the **Query Plan** of *Scenario 1*:
![72db773c5cc91d994c68e31258e6df49](Apache%20Spark/attachments/72db773c5cc91d994c68e31258e6df49.png)

then the **Query Plan** of *Scenario 2:
![6f5b17be7e124e09aa40991ad47229ac](Apache%20Spark/attachments/6f5b17be7e124e09aa40991ad47229ac.png)

* **Scenario 1**. We have a `SortMergeJoin` between the 2 DF, which is a standard join. This `SortMergeJoin` assumes that both branches of this join need to be shuffled and sorted by the column that’s being joined on. So both branches of this query plan end with an `Exchange hashpartitioning` and a `Sort` before the `SortMergeJoin` is being executed. This happens despite the fact that the two DF have been recently repartitioned into 10 and 7 partitions respectively. This happens because the repartition with a number of partitions employs a different kind of partitioner. Indeed we have a `RoundRobinPartitioning` for both DF, which does not guarantee that the rows with identical key are on the same partitions. Summing up: the `RoundRobinPartitioning` is the one we apply manually just to simulate a real case scenario; then the two DF need to be partitioned again so that the rows with the same key will be in the same partitions. In total 4 shuffles: the first two ones done manually and the last two done by Spark to guarantee the join correctness.
- **Scenario2**. This is much simpler. Spark creates a range and it directly performs an `hashpartitioning` because it recognises that we’re simply changing the partitioning scheme for the DF. It automatically changes the partitioned from a `RoundRobinPartitioning` to a repartition by `id`, which is `hashpartitioning` scheme. So, whenever you do a repartition by a column, Spark will use a `hashpartitioning`, that is the rows with the same value for the `id` column will sit on the same partition. This happens for both DF being joined and so we are using the same `hashpartitioning` scheme for both DF. Now, because the two DF have the same partitioner, they’re called co-partitioned:
	![d3bdbbbdfede3292b04dd6fc1a016809](Apache%20Spark/attachments/d3bdbbbdfede3292b04dd6fc1a016809.png)

	So in *Scenario 2* we are doing a **join on co-partitioned DF**, which is much faster. In total we have only 2 shuffles.

The difference between *Scenario 1* and *Scenario 2* is simple: **in *Scenario 2* we partitioned early**.
## 2.2. Another example
It’s always a good idea to **partition early**. To make it clear let’s make a *Scenario 3*, that will be very similar to *Scenario 2* (because we will co-partition the two DF), but for some reasons it will behave like *Scenario 1* in terms of performance.
```scala
// SCENARIO 3
val enhanceColumnsFirst = addColumns(initialTable, 30)
val repartitionedNarrow = narrowTable.repartition($"id")
val repartitionedEnhance = enhanceColumnsFirst.repartition($"id")
val result3 = repartitionedEnhance.join(repartitionedNarrow, "id")
```

Let's look at the Spark UI:
![7a1045aa8bdfd7f3d0e09b7b44be59cf](Apache%20Spark/attachments/7a1045aa8bdfd7f3d0e09b7b44be59cf.png)

and at the physical plan:
![e198990cf0df9f897bc90bf1e6ec7384](Apache%20Spark/attachments/e198990cf0df9f897bc90bf1e6ec7384.png)

At the top we have the `SortMergeJoin` like the one in *Scenario 1* and *Scenario 2*. But before the join, in the first branch we have a `RoundRobinPartitioning` and then a `hashpartitioning` on the biggest DF. This is because we processed the `initialTable` early rather than partitioning it early; and so Spark has no choice but to do the `Project` step after the initial `repartition(10)` (that will add 30 columns), and after that it will do the `hashpartitioning` which is mandatory for the join.

Now, let's create a new DF by joining `repartitionedNarrow` with `enhanceColumnsFirst` and no longer with `repartitionedEnhance`. What we are doing now is not perform the `.repartition($"id")`function:
```scala
// SCENARIO 3  
val enhanceColumnsFirst = addColumns(initialTable, 30)  
val repartitionedNarrow = narrowTable.repartition($"id")  
val repartitionedEnhance = enhanceColumnsFirst.repartition($"id") // <- it's USELESS!!  
val result3 = repartitionedEnhance.join(repartitionedNarrow, "id")  
val result3_bis = enhanceColumnsFirst.join(repartitionedNarrow, "id")
```

Let's take a look at the query plans of both `result3` and `result3_bis`:
![75f82cf25d0d13cb4e4cca2a9a85d9af](Apache%20Spark/attachments/75f82cf25d0d13cb4e4cca2a9a85d9af.png)

The two query plans are exactly the same! This means that the operation `enhanceColumnsFirst.repartition($"id")` is useless! That’s because Spark whenever it doesn't know the partitioning scheme for the DFs that are about to be joined, it forces them into a `hashpartitioning` scheme, which is the shuffle done with the "USELESS" line.

Summing up:
> [!important] Pre-Partitioning
> **Partition early**. Partitioning late is AT BEST what Spark naturally does.  

Let’s highlight another point. The initial goal was to join the `initialTable` with the `narrowTable`. They were both repartitioned respectively with `RoundRobinPartitioning(10)` and `RoundRobinPartitioning(7)`.

Couldn’t we just repartition the narrow table to 10 and just be done with it?

Solution: this idea is TERRIBLE. If we repartition the `narrowTable` into 10 partitions like `initialTable`, neither repartitioning guarantees the fact that all the rows with the same id are on the same partition, which is a mandatory guarantee that must be upheld before the join takes place.

So the query plan produced by `initialTable.join(narrowTable.repartition(10), "id").explain()` would be identical to the one of *Scenario 1*.

---

# 3. To Remember
* Partition your data early so that Spark doesn't have to
	* make the joined DFs share the same partitioner, e.g. partition by the same column
	* decorate the joined DF later (especially if you have lots of transformations)
* Partitioning late is bad
	* at best: same performance as Spark out-of-the-box
	* at worst: worse performance than not partitioning at all