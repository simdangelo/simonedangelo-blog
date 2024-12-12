---
date: 2024-08-12
modified: 2024-12-12T00:10:00+01:00
---
These notes about **Partitioning** will be "notebook-style" and only important and useful operations will be shown in this article (hence, very probably, no such operations will be shown: DataFrame creation, session state creation, etc.). For the complete code, refers to these notebooks I uploaded on a specific repository on [this link](metti!!!).
# 0. Resources
* [*Data with Nikk the Greek* YouTube Channel](https://www.youtube.com/@DataNikktheGreek)
# 1. First sneak peak into Spark Partitions
 General hints:
 * `df.rdd.getNumPartitions()` returns the number of partitions of the current Spark DataFrame;
 * `df.write.format("noop").mode("overwrite")` is used to specify a no-operation (noop) data sink, which effectively performs no action, often for testing or debugging purposes.

Let's understand what's the default parallelism by running:
```python
spark: SparkSession = SparkSession \
    .builder \
    .appName("Partitioning 1") \
    .master("local[4]") \
    .enableHiveSupport() \
    .getOrCreate()

sc = spark.sparkContext
spark.sparkContext.defaultParallelism
```
* The variable `spark.sparkContext.defaultParallelism` returns the level of parallelism and, by default, it's the number of cores available to application. Indeed from the above code we got `4`.
* The number of cores is the maximum number of tasks you can execute in parallel.

In the following three sub-chapters we'll see **two ways to influence the number of partitions during the creation of a DataFrame** (either with `spark.CreateDataFrame()` and `spark.range()` functions):
1. set the number of cores available to application;
2. use the `num_partitions` argument of the `spark.range()` function.
## 1.1. Partition Size based on Cores and Data Amount with `spark.CreateDataFrame()`
Let' use define a function using the `spark.CreateDataFrame()` to create a DataFrame:
```python
def sdf_generator1(num_iter: int = 1) -> DataFrame:
    d = [
        {"a":"a", "b": 1},
        {"a":"b", "b": 2},
        {"a":"c", "b": 3},
        {"a":"d", "b": 4},
        {"a":"e", "b": 5},
        {"a":"e", "b": 6},
        {"a":"f", "b": 7},
        {"a":"g", "b": 8},
        {"a":"h", "b": 9},
        {"a":"i", "b": 10},
    ]

    data = []
    for _ in range(0, num_iter):
        data.extend(d)
    ddl_schema = "a string, b int"
    df = spark.createDataFrame(data, schema=ddl_schema)
    return df
```

If we create a DataFrame by running `sdf_gen1 = sdf_generator1(2)` and we take a look at the number of partitions by runnning `sdf_gen1.rdd.getNumPartitions()`, we'll get `4`:

Also, you can check in the *Executor* tab in the Spark UI:
![](Apache%20Spark/attachments/Pasted%20image%2020240812144551.png)

We can also visualize which partition each row of the DataFrame belongs to:
```python
sdf_part1 = sdf_gen1.withColumn("partition_id", f.spark_partition_id())
sdf_part1.show()
```
Output:
```
+---+---+------------+
|  a|  b|partition_id|
+---+---+------------+
|  a|  1|           0|
|  b|  2|           0|
|  c|  3|           0|
|  d|  4|           0|
|  e|  5|           0|
|  e|  6|           1|
|  f|  7|           1|
|  g|  8|           1|
|  h|  9|           1|
|  i| 10|           1|
|  a|  1|           2|
|  b|  2|           2|
|  c|  3|           2|
|  d|  4|           2|
|  e|  5|           2|
|  e|  6|           3|
|  f|  7|           3|
|  g|  8|           3|
|  h|  9|           3|
|  i| 10|           3|
+---+---+------------+
```

From this output we can see that data is distributed quite uniformly among partitions and we can prove that by running:
```python
row_count = sdf_gen1.count()
sdf_part_count1 = sdf_part1.groupBy("partition_id").count()
sdf_part_count1 = sdf_part_count1.withColumn("count_perc", 100*f.col("count")/row_count)
sdf_part_count1.show()
```
Output:
```
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           0|    5|      25.0|
|           1|    5|      25.0|
|           2|    5|      25.0|
|           3|    5|      25.0|
+------------+-----+----------+
```

This uniformity reflects also on the computation time. Indeed if we run this code
```python
sc.setJobDescription("Gen1_Exp1")
sdf_gen1_1.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```
we can notice that uniformity in this job detail's page in the Spark UI:
![](Apache%20Spark/attachments/Pasted%20image%2020240812150330.png)

Let's repeat this experiment for a bigger dataframe:
```python
sdf_gen1_2 = sdf_generator1(2000)

sdf_part1_2 = sdf_gen1_2.withColumn("partition_id", f.spark_partition_id())
row_count = sdf_gen1_2.count()
sdf_part_count1_2 = sdf_part1_2.groupBy("partition_id").count()
sdf_part_count1_2 = sdf_part_count1_2.withColumn("count_perc", 100*f.col("count")/row_count)
sdf_part_count1_2.show()
```
Output:
```
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           0| 5120|      25.6|
|           1| 5120|      25.6|
|           2| 5120|      25.6|
|           3| 4640|      23.2|
+------------+-----+----------+
```
And let's trigger a job with:
```python
sc.setJobDescription("Gen1_Exp2")
sdf_gen1_2.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

The bejaviour looks very similar to the previous one:
![](Apache%20Spark/attachments/Pasted%20image%2020240812151410.png)

>[!note]
>The size of the data does not have any influence on the number of partitions (if you load a file there will be some influence as we'll see later on).
>
## 1.2. Partition Size based on Cores and Data Amount with `spark.range()`
Now we use another function to create a dataframe:
```python
def sdf_generator2(num_rows: int, num_partitions: int = None) -> DataFrame:
    return (
        spark.range(num_rows, numPartitions=num_partitions)
        .withColumn("date", f.current_date())
        .withColumn("timestamp",f.current_timestamp())
        .withColumn("idstring", f.col("id").cast("string"))
        .withColumn("idfirst", f.col("idstring").substr(0,1))
        .withColumn("idlast", f.col("idstring").substr(-1,1))
        )
        
sdf_gen2 = sdf_generator2(2000000)
sdf_gen2.rdd.getNumPartitions()
```
We got `4` still here.

All the considerations we did in the previous chapter are still valid here and you'll get the same results.
## 1.3. Influence on Spark partitions to the performance
As we mentioned above, the `spark.range()` function allows us to influence the number of partitions. Let's create a bunch of new dataframes:
```python
sdf4 = sdf_generator2(20000000, 4)
print(sdf4.rdd.getNumPartitions())
sc.setJobDescription("Part Exp1")
sdf4.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf8 = sdf_generator2(20000000, 8)
print(sdf8.rdd.getNumPartitions())
sc.setJobDescription("Part Exp2")
sdf8.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf3 = sdf_generator2(20000000, 3)
print(sdf3.rdd.getNumPartitions())
sc.setJobDescription("Part Exp3")
sdf3.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf6 = sdf_generator2(20000000, 6)
print(sdf6.rdd.getNumPartitions())
sc.setJobDescription("Part Exp4")
sdf6.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf200 = sdf_generator2(20000000, 200)
print(sdf200.rdd.getNumPartitions())
sc.setJobDescription("Part Exp5")
sdf200.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf20000 = sdf_generator2(20000000, 20000)
print(sdf20000.rdd.getNumPartitions())
sc.setJobDescription("Part Exp6")
sdf20000.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```
* `sdf4`: the 4 cores work on the 4 partitions in parallel.
* `sdf8`: the 4 cores work on 4 partitions in parallel; once this step is executed the remaining 4 partitions are being executed in parallel again:
	![](Apache%20Spark/attachments/Pasted%20image%2020240812163220.png)
* `sdf3`: only three cores are working and the remaining one is set in IDLE, so we're wasting our resources, hence the performance is not optimized:
	![](Apache%20Spark/attachments/Pasted%20image%2020240812163428.png)
* `sdf6`: the 4 cores work on 4 partitions in parallel; after that only two cores will work on the remaining two partitions while the other two cores are set in IDLE. Again we're wasting our resources, hence the performance is not optimized:
	![](Apache%20Spark/attachments/Pasted%20image%2020240812163715.png)
* `sdf200` and `sdf20000`: the same consideration of `sdf6`.

---
# 2. Coalesce
How **Coalesce** works:
* it's a narrow transformation.
* can only reduce and not increase the number of partitions. it does not return any errors, but it just ignores a value higher than the initial available partitions.
* Coalesce can skew the data within each partition which leads to lower performance and some tasks running way longer.
* Coalesce can help with efficiently **reducing high number of small partitions** and improve performance. Remember a too high number of partitions leads to a lot of scheduling overhead.
* A too small number of partitions (bigger partitions) can result to OOM or other issues. A factor of 2-4 of the number of cores is recommended. But really depends on the memory available. If you can't increase the number of cores the only option of a stable execution not reaching the limits of your memory is increasing the number of partitions. Recommendations in the internet say anything between 100-1000 MB. Spark sets his max partition bytes parameter to 128 MB.

We'll generate a DataFrame with two billion rows distributed across **four partitions**, meaning each partition holds 500 million rows. Then we'll use `.coalesce()` function to **reduce the number of partitions from 4 to 3**:
```python
num_rows = 2000000000
sdf1 = sdf_generator(num_rows, 4)

sdf2 = sdf1.coalesce(3)
sdf2.rdd.getNumPartitions()
```

Let's see the data repartition across these 3 new partitions:
```python
row_count2 = sdf2.count()
sdf_part2 = sdf2.withColumn("partition_id", f.spark_partition_id())
sdf_part_count2 = sdf_part2.groupBy("partition_id").count()
sdf_part_count2 = sdf_part_count2.withColumn("count_perc", 100*f.col("count")/row_count2)
sdf_part_count2.show()
```
Output:
```
+------------+----------+----------+
|partition_id|     count|count_perc|
+------------+----------+----------+
|           0| 500000000|      25.0|
|           1| 500000000|      25.0|
|           2|1000000000|      50.0|
+------------+----------+----------+
```
Basically with coalesce two partitions have been merged into one large partition of 1 million rows.

When using `.coalesce()`, **you can only reduce** the number of partitions in a DataFrame. Spark ignores any request to increase the partition size. If we attempt to **increase** the number of partitions:
```
>>> sdf2.coalesce(2).rdd.getNumPartitions()
2

>>> sdf2.coalesce(1).rdd.getNumPartitions()
1

>>> sdf2.coalesce(5).rdd.getNumPartitions()
4

>>> sdf2.coalesce(10).rdd.getNumPartitions()
4
```
Spark **ignores** the request, keeping the partition size at **4**, the original number defined when the DataFrame was created.
## 2.1. Comparisons between Experiments
Next, we examine how `coalesce` affects data distribution. Since `coalesce` is a **narrow transformation**, it doesn’t involve data shuffling across nodes, making it efficient but potentially leading to uneven data distribution.

We started with **four partitions** of **500 million rows each**, totaling **two billion rows**. After applying `coalesce(3)`, Spark **merged two partitions** into a **single partition** of **one billion rows**, leaving the remaining two partitions unchanged at **500 million rows each**.

This redistribution creates a significant imbalance due to Spark’s merging strategy. It only **unions** partitions, producing one large partition and two smaller ones. Let's look at this by running three job and by inspecting the Spark UI for **three different scenarios**:

```python
sc.setJobDescription("Baseline 4 partitions")
sdf1.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sc.setJobDescription("Coalesce from 4 to 3")
sdf2.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf3 = sdf_generator(num_rows, 3)
sc.setJobDescription("3 Partitions")
sdf3.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

* **Baseline 4 partitions**
	![](Apache%20Spark/attachments/Pasted%20image%2020240812184724.png)
* **Coalesce from 4 to 3**
	![](Apache%20Spark/attachments/Pasted%20image%2020240812184954.png)
* **3 Partitions**
	![](Apache%20Spark/attachments/Pasted%20image%2020241209222127.png)

Recap:
* **Baseline 4 partitions**:
    - Highly uniform execution: each partition had **500M rows**.
    - **Execution time:** **1.6 minutes** (aligned with 4 tasks using 4 cores).
* **Coalesce from 4 to 3**:
    - **Uneven distribution:** Two partitions had **500M rows**, and one had **one billion rows**.
    - **Execution time:** **3.2 minutes** – delayed by the largest partition.
    - **Bottleneck:** The two smaller partitions finished quickly, leaving two cores idle while waiting for the last partition.
* **3 Partitions**:
    - **Balanced distribution:** Three evenly split partitions.
    - **Execution time:** **1.9 minutes**. Using a partition count **not divisible** by the number of cores leads to **resource underutilization** (in this case we are leaving out 1 core, while only three are working).

We explored a more **strategic approach** to using `.coalesce()` by adjusting the initial partition count **before applying `coalesce`**. We generated a DataFrame with **eight partitions** and **two billion rows**, evenly distributing **250 million rows** per partition. Then, we applied **`coalesce(4)`** to reduce the partition count from **8 to 4**:
```python
sdf4 = sdf_generator(num_rows, 8)
print(sdf4.rdd.getNumPartitions())
sc.setJobDescription("8 Partitions")
sdf4.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf5 = sdf4.coalesce(4)
print(sdf5.rdd.getNumPartitions())
sc.setJobDescription("Coalesce from 8 to 4")
sdf5.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

As we did before, let's take a look at the SparkUI and then we'll make a recap:
* **8 Partitions**
	![](Apache%20Spark/attachments/Pasted%20image%2020241209223423.png)
* **Coalesce from 8 to 4**
	![](Apache%20Spark/attachments/Pasted%20image%2020241209223514.png)

Recap:
1. **8 Partitions:**
    - Each partition held **250 million rows**.
    - **Execution time:** **1.5 minutes.**
2. **Coalesce from 8 to 4:**
    - Merged 8 partitions into 4, keeping **500 million rows** in each new partition.
    - **Execution time:** **1.7 minutes.**

Spark UI Insights:
- **Even Data Distribution:** After applying `coalesce(4)`, partitions were **evenly distributed** with **500 million rows each**, utilizing all available cores.
- **Task Overlap:** Since some tasks finished earlier, Spark executed subsequent tasks in **parallel**, reducing overall execution time.
- **Minimal Overhead:** While **`.coalesce()`** introduced a minor delay due to the **narrow transformation**, the overhead was **insignificant**, with a slight increase in runtime from **1.5 to 1.7 minutes.**

We'll explore the impact of **reducing an extremely high number of partitions** on Spark DataFrames. Starting with **200,001 partitions**, we'll apply **`.coalesce(4)`** to examine the potential performance improvement:
```python
sdf6 = sdf_generator(num_rows, 200001)
print(sdf6.rdd.getNumPartitions())
sc.setJobDescription("200001 partitions")
sdf6.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf7 = sdf6.coalesce(4)
print(sdf7.rdd.getNumPartitions())
sc.setJobDescription("Coalesce from 200001 to 4")
sdf7.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

As we did before, let's take a look at the SparkUI and then we'll make a recap:
* **200001 partitions**
	![](Apache%20Spark/attachments/Pasted%20image%2020241209224748.png)
* **Coalesce from 200001 to 4**
	![](Apache%20Spark/attachments/Pasted%20image%2020241209224831.png)

Recap:
- **200,001 Partitions (Unoptimized):**
    - Execution Time: **2.6 minutes**
    - **Highly Scattered Tasks:** Spark scheduled **200,001 small tasks**, causing a **significant scheduling overhead.**
    - **Task Duration:** Tasks ranged from **1 milliseconds to 0.2 seconds**, reflecting **severe load imbalance.**
- **Coalesce from 200001 to 4:**
    - Execution Time: **2.4 minutes** (*Theoretically, it should be very similar to Baseline 4 partitions case, but they aren't. Try to re-run all jobs*).
    - **Improved Data Distribution:**
		- Partitions contained roughly **500 million rows** each.
        - Although the row count varied slightly (~8,000-row difference), this was **negligible** compared to the dataset size.

For our final experiment, we examined how **adjusting partition sizes** could impact Spark's execution performance when dealing with large datasets. The initial configuration ("*Baseline 4 partitions*") has the following characteristics:
- **Total Size:** ~8GB when persisted on disk.
- **Initial Partitioning:** 4 partitions (~2GB per partition).
- **Hardware:** 4 cores, with ~2GB per core when using 4 partitions.

Given the **large partition size**, this configuration is far from optimal. Ideally, each partition should be around **200MB** to **500MB**, ensuring efficient parallel processing and reducing memory spill risks. So, lets' generate a dataset with 40 partitions:
```python
sdf8 = sdf_generator(num_rows, 40)
print(sdf8.rdd.getNumPartitions())
sc.setJobDescription("40 partitions")
sdf8.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

Here's the SparkUI details:
![](Apache%20Spark/attachments/Pasted%20image%2020241210210950.png)

**Optimized Partitioning: 40 Partitions
- **New Partition Size:** ~200MB per partition (50 million rows per partition).
- **Execution Time:** **1.6 minutes** (same as the baseline with 4 partitions).
- **Task Distribution:** Tasks were **well-balanced**, with uniform task execution times.

### Key Insights
1. **Balanced Partition Size Matters:**
    - Using **40 partitions (~200MB each)** prevented **spillovers** and **resource overload**, maintaining **1.6-minute execution time.**
    - This balanced configuration **maximized parallel task execution** while **minimizing scheduling overhead.**
2. **Avoid Too Few or Too Large Partitions:**
    - Having **only 4 large partitions (~2GB each)** risks **worker memory overload** and can cause **disk spillovers** if the dataset size increases further.
# 3. Repartition
Today's topic is a **follow-up** to our previous discussions on **partitioning** and **coalescing.** This time, we’re looking into **another way to repartition data** using a command you probably guessed already: **`.repartition()`**.

We’ll also use this opportunity to **recap why partitioning is important** and **why it really matters** when working with large datasets. After that, we’ll **deep dive** into how **`repartition`** works.

In the **next session**, we’ll explore **how `repartition` and `coalesce`** can be used **together** in **specific scenarios** to **optimize performance**, building on what we covered about **`coalesce`** in the previous episode.

Let's begin our **Spark session** and set up the **data generator**. We'll quickly review the generated data to better understand how it works, especially since we’ll be performing some **group-by operations** later for analysis:
```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as f
from pyspark.sql import DataFrame

spark = SparkSession \
    .builder \
    .appName("Repartition") \
    .master("local[4]") \
    .enableHiveSupport() \
    .getOrCreate()

sc = spark.sparkContext

def sdf_generator(num_rows: int, num_partitions: int = None) -> DataFrame:
    return (
        spark.range(num_rows, numPartitions=num_partitions)
        .withColumn("date", f.current_date())
        .withColumn("timestamp",f.current_timestamp())
        .withColumn("idstring", f.col("id").cast("string"))
        .withColumn("idfirst", f.col("idstring").substr(0,1))
        .withColumn("idlast", f.col("idstring").substr(-1,1))
        )
```

The `id` generation logic is straightforward: Spark generates IDs, and we convert them into strings, taking the **first character** of each id. For example, if we generate **20,000 IDs**, the **first character** will often be "1," meaning **grouping or clustering** based on this first character will create a **data skew**.

We’ll see this in action later during our **partitioning** experiments.

We have **two functions** today that will help us **analyze the data distribution**:
1. `rows_per_partition`: This function **clusters the data** based on its `partition_id` and shows how rows are distributed across partitions:
2. `rows_per_partition_col`: This function **groups the data by both `partition_id` and a specific column**, which in our case will be the `idfirst`.

Let's define and apply the `rows_per_partition` function:
```python
def rows_per_partition(sdf: "DataFrame", num_rows: int) -> None:
    sdf_part = sdf.withColumn("partition_id", f.spark_partition_id())
    sdf_part_count = sdf_part.groupBy("partition_id").count()
    sdf_part_count = sdf_part_count.withColumn("count_perc", 100*f.col("count")/num_rows)
    sdf_part_count.orderBy("partition_id").show()

rows_per_partition(sdf_gen, 20)
```
Output:
```
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           0|    5|      25.0|
|           1|    5|      25.0|
|           2|    5|      25.0|
|           3|    5|      25.0|
+------------+-----+----------+
```

Then `rows_per_partition_col`:
```python
def rows_per_partition_col(sdf: "DataFrame", num_rows: int, col: str) -> None:
    sdf_part = sdf.withColumn("partition_id", f.spark_partition_id())
    sdf_part_count = sdf_part.groupBy("partition_id", col).count()
    sdf_part_count = sdf_part_count.withColumn("count_perc", 100*f.col("count")/num_rows)
    sdf_part_count.orderBy("partition_id", col).show()

rows_per_partition_col(sdf_gen, 20, "idfirst")
```
Output:
```
+------------+-------+-----+----------+
|partition_id|idfirst|count|count_perc|
+------------+-------+-----+----------+
|           0|      0|    1|       5.0|
|           0|      1|    1|       5.0|
|           0|      2|    1|       5.0|
|           0|      3|    1|       5.0|
|           0|      4|    1|       5.0|
|           1|      5|    1|       5.0|
|           1|      6|    1|       5.0|
|           1|      7|    1|       5.0|
|           1|      8|    1|       5.0|
|           1|      9|    1|       5.0|
|           2|      1|    5|      25.0|
|           3|      1|    5|      25.0|
+------------+-------+-----+----------+
```

We see how **rows are distributed** across **partitions**:
- **Partition 0** contains rows with IDs starting with 0, 1, 2, 3, and 4.
- **Partition 1** has rows with IDs starting with 5, 6, 7, 8, and 9.
* **Partitions 2 and 3** have rows with ids starting with `1` since most generated IDs fall into this range (from 10 to 19).
## 3.1. Recap Partitioning
Let's recap why partitioning is essential. I found an [interesting Stack Overflow link](https://stackoverflow.com/questions/64600212/how-to-determine-the-partition-size-in-an-apache-spark-dataframe) related to this, but let's summarize the key points.
### 3.1.1. The most important thing you want a good parallisation
The first thing we already know: **parallelization**. This has **two aspects**:
1. You don’t want **unused cores**, meaning one core does all the work while the others are idle, resulting in wasted resources. We saw earlier that choosing a **partition number** that is **divisible by the number of cores** can help. You can check this using `spark.SparkContext.defaultParallelism`, which returns the number of available cores. There are recommendations to use a **factor of 2 to 4** based on your **default parallelism**, but this depends on your dataset size:
	- For **small datasets**, use the number of cores as the partition count to minimize **scheduling overhead**.
	- For **large datasets**, increase the partition count if adding more cores isn’t possible. This can help avoid running out of memory.
1. You also don’t want **skewed workloads**, where one core has significantly more work than the others. This happens when data isn't evenly distributed. This concept is called **data skew**. If one partition contains **much more data** than others, the job becomes **unbalanced**. Though **small partitions** combined with **one large partition** can sometimes balance out, the safest approach is having **uniform partition sizes** or, in the worst case, a **normal distribution**.

These two principles — **parallelization** and **balanced partitions** — ensure that Spark performs at its best. Since Spark **parallelizes** data processing based on **memory partitions**, which later become **tasks** in each **stage of a job**, proper partitioning is critical.
### 3.1.2. Partition size
When it comes to partition size, you want to avoid making partitions **too small** because this leads to underutilized memory and **overhead** from managing too many small partitions. On the other hand, **very large partitions** can cause memory issues.

From what I found online, there are some general rules: **partitions should not exceed 1 GB**, with recommendations typically falling between **100 MB and 1,000 MB**, depending on available memory.

Even with **high memory capacity**, using partitions that are too large can cause **out-of-memory errors**. This happens because memory availability can fluctuate due to various factors like creating objects, initializing classes, and performing garbage collection.

Staying within a reasonable partition size range **increases stability** and helps **prevent errors**. For example, the default value for `maxPartitionBytes` when loading files from storage in Spark is **128 MB**, which we’ll explore in a future episode.

While larger machines with more memory can support bigger partitions, **sticking to recommended sizes** is generally a safe approach. In the end, it’s something you need to **experiment with** to find what works best for your specific use case.
### 3.1.3. Distribution Overhead Considerations
We ran tests with **2,000 and 20,000 partitions** and noticed some **distribution overhead** causing **time loss**. Having **slightly more partitions** can be beneficial if they aren't too large, but **too many small partitions** negatively impact performance.

If the **execution time** of one of your tasks is **well below 90%** of the total job time—or even **as low as 10%**—while the rest is consumed by scheduling, something is clearly wrong. This **distribution overhead** often results from **too many small partitions**, making scheduling the dominant factor instead of task execution.

An online rule of thumb suggests that if a task runs for **less than 100 milliseconds**, the partition is **too small**, and you should **increase the partition size** to reduce overhead.
## 3.2. How Repartition works
Repartitioning allows adjusting the number of partitions **up or down**, unlike `coalesce`, which only decreases them. This method **requires shuffling**, making it a **wide transformation**, unlike the narrow transformation of `coalesce`. Let’s explore how it works by reviewing the official Spark documentation.

The function `DataFrame.repartition()` takes two main parameters:
1. **Number of Partitions**: Defines the target number of partitions.
2. **Columns**: Specifies one or more columns to partition by.

For example, you can use:
- Just the number of partitions (`df.repartition(4)`).
- A specific column (`df.repartition("column_name")`).
- Both the number of partitions and columns (`df.repartition(4, "column_name")`).

Additionally, **multiple columns** can be passed as input, ensuring flexibility in data distribution.

Let's make a comparison with `coalesce()`:
- **`coalesce()`**: Merges partitions without shuffling, which can result in uneven data distribution due to its union-like behavior.
- **`repartition()`**: Redistributes the data using a **round-robin mechanism**, ensuring **uniform partition sizes**. This makes repartitioning **more balanced** but **more expensive** due to shuffling.

If no specific number of partitions is provided and only a column is used, Spark defaults to its internal **`spark.sql.shuffle.partitions`** setting, which is **200 by default**. This parameter becomes relevant when performing wide transformations like `groupBy` or `join`.

We start by generating a sample dataset of **2,000 rows**, ensuring a uniform distribution during creation. However, inspecting the **ID First Column**, we notice that the data is **not perfectly uniform**, as expected from the earlier discussion.
### 3.2.1. Baseline Job Execution
We run a **baseline job** using **four partitions**. After confirming the initial state, we proceed with repartitioning experiments:
```terminal
>>> sdf1 = sdf_generator(num_rows=20000, num_partitions=4)
>>> sdf1.rdd.getNumPartitions()
4
```

Data distribution after aggregation:
```terminal
>>> rows_per_partition(sdf1, num_rows)
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           0| 5000|      25.0|
|           1| 5000|      25.0|
|           2| 5000|      25.0|
|           3| 5000|      25.0|
+------------+-----+----------+

>>> rows_per_partition_col(sdf1, num_rows, "idfirst")
+------------+-------+-----+----------+
|partition_id|idfirst|count|count_perc|
+------------+-------+-----+----------+
|           0|      0|    1|     0.005|
|           0|      1| 1111|     5.555|
|           0|      2| 1111|     5.555|
|           0|      3| 1111|     5.555|
|           0|      4| 1111|     5.555|
|           0|      5|  111|     0.555|
|           0|      6|  111|     0.555|
|           0|      7|  111|     0.555|
|           0|      8|  111|     0.555|
|           0|      9|  111|     0.555|
|           1|      5| 1000|       5.0|
|           1|      6| 1000|       5.0|
|           1|      7| 1000|       5.0|
|           1|      8| 1000|       5.0|
|           1|      9| 1000|       5.0|
|           2|      1| 5000|      25.0|
|           3|      1| 5000|      25.0|
+------------+-------+-----+----------+
```

### 3.2.2. Repartitioning Down to Three Partitions
We use **`repartition(3)`** on the DataFrame:
```terminal
>>> sdf_3 = sdf1.repartition(3)
>>> sdf_3.rdd.getNumPartitions()
3

>>> rows_per_partition(sdf_3, num_rows)
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           0| 6667|    33.335|
|           1| 6667|    33.335|
|           2| 6666|     33.33|
+------------+-----+----------+
```
- **Number of Partitions**: The DataFrame now has **three partitions**.
- **Data Distribution**: After aggregation, we see that **each partition holds an equal number of rows**, unlike the uneven distribution seen with `coalesce()`.
### 3.2.3. Repartitioning Up to Twelve Partitions
Next, we **increase** the number of partitions to **twelve** using **`repartition(12)`**:
```terminal
>>> sdf_12 = sdf1.repartition(12)
>>> sdf_12.rdd.getNumPartitions()
12

>>> rows_per_partition(sdf_12, num_rows)
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           0| 1667|     8.335|
|           1| 1666|      8.33|
|           2| 1666|      8.33|
|           3| 1666|      8.33|
|           4| 1667|     8.335|
|           5| 1667|     8.335|
|           6| 1667|     8.335|
|           7| 1667|     8.335|
|           8| 1666|      8.33|
|           9| 1667|     8.335|
|          10| 1667|     8.335|
|          11| 1667|     8.335|
+------------+-----+----------+
```
- **Partition Count**: The DataFrame now has **twelve partitions**.
- **Data Distribution**: Each partition holds approximately **8% of the data**, confirming that repartitioning effectively redistributes the rows **evenly** across all partitions.

These tests demonstrate **Spark’s `repartition()` behavior**, ensuring balanced data distribution regardless of whether we **increase** or **decrease** the number of partitions. This contrasts sharply with `coalesce()`, which only **decreases** partitions and may cause skewed distributions.
### 3.2.4. Repartitioning by column
We continue experimenting with **Spark’s `repartition()`** function, focusing on **column-based partitioning** to explore how it manages data distribution.

*Note: For the sake of this example, I have temporarily disabled Spark's Adaptive Query Execution (AQE). If AQE is enabled, Spark may not create 200 partitions(AQE Internally uses Coalesce function to merge the smaller partitions), as this can lead to the generation of many empty partitions, which is not an optimal scenario:*
```python
spark.conf.set("spark.sql.adaptive.enabled", "False")
```

We **repartition** the DataFrame by `idfirst` column using the default **shuffle partitions (200)**:
```terminal
>>> spark.conf.set("spark.sql.shuffle.partitions", 200)
>>> sdf_col_200 = sdf1.repartition("idfirst")
>>> sdf_col_200.rdd.getNumPartitions()
200

>>> rows_per_partition(sdf_col_200, num_rows)
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           3| 1111|     5.555|
|          18| 1111|     5.555|
|          26| 1111|     5.555|
|          35|    1|     0.005|
|          49| 1111|     5.555|
|          75| 1111|     5.555|
|         139| 1111|     5.555|
|         144|11111|    55.555|
|         166| 1111|     5.555|
|         189| 1111|     5.555|
+------------+-----+----------+

>>> rows_per_partition_col(sdf_col_200, num_rows, "idfirst")
+------------+-------+-----+----------+
|partition_id|idfirst|count|count_perc|
+------------+-------+-----+----------+
|           3|      7| 1111|     5.555|
|          18|      3| 1111|     5.555|
|          26|      8| 1111|     5.555|
|          35|      0|    1|     0.005|
|          49|      5| 1111|     5.555|
|          75|      6| 1111|     5.555|
|         139|      9| 1111|     5.555|
|         144|      1|11111|    55.555|
|         166|      4| 1111|     5.555|
|         189|      2| 1111|     5.555|
+------------+-------+-----+----------+
```

Observations:
- The **number of partitions** is confirmed as **200**, but **many partitions are empty**.
- Only **10 partitions** contain data, reflecting the **skewed distribution** of the **ID First** values we noticed earlier.
- This highlights a critical **limitation** of column-based repartitioning: **empty partitions** can occur if the data’s key distribution is uneven.

We **reduce** the shuffle partitions from **200** to **20**:
```terminal
>>> spark.conf.set("spark.sql.shuffle.partitions", 20)
>>> sdf_col_20 = sdf1.repartition("idfirst")
>>> sdf_col_20.rdd.getNumPartitions()
20

>>> rows_per_partition(sdf_col_20, num_rows)
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           3| 1111|     5.555|
|           4|11111|    55.555|
|           6| 2222|     11.11|
|           9| 2222|     11.11|
|          15| 1112|      5.56|
|          18| 1111|     5.555|
|          19| 1111|     5.555|
+------------+-----+----------+

>>> rows_per_partition_col(sdf_col_20, num_rows, "idfirst")
+------------+-------+-----+----------+
|partition_id|idfirst|count|count_perc|
+------------+-------+-----+----------+
|           3|      7| 1111|     5.555|
|           4|      1|11111|    55.555|
|           6|      4| 1111|     5.555|
|           6|      8| 1111|     5.555|
|           9|      2| 1111|     5.555|
|           9|      5| 1111|     5.555|
|          15|      0|    1|     0.005|
|          15|      6| 1111|     5.555|
|          18|      3| 1111|     5.555|
|          19|      9| 1111|     5.555|
+------------+-------+-----+----------+
```

Results:
* The result is similar: the data **only occupies 10 partitions**, though the **specific partition IDs** differ due to **hash partitioning**

We adjust **shuffle partitions** to **10**, expecting **1 `idfirst` value per partition**:
```terminal
>>> sdf_col_10 = sdf1.repartition(10, "idfirst")
>>> sdf_col_10.rdd.getNumPartitions()
10

>>> rows_per_partition(sdf_col_10, num_rows)
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           3| 1111|     5.555|
|           4|11111|    55.555|
|           5| 1112|      5.56|
|           6| 2222|     11.11|
|           8| 1111|     5.555|
|           9| 3333|    16.665|
+------------+-----+----------+


>>> rows_per_partition_col(sdf_col_10, num_rows, "idfirst")
+------------+-------+-----+----------+
|partition_id|idfirst|count|count_perc|
+------------+-------+-----+----------+
|           3|      7| 1111|     5.555|
|           4|      1|11111|    55.555|
|           5|      0|    1|     0.005|
|           5|      6| 1111|     5.555|
|           6|      4| 1111|     5.555|
|           6|      8| 1111|     5.555|
|           8|      3| 1111|     5.555|
|           9|      2| 1111|     5.555|
|           9|      5| 1111|     5.555|
|           9|      9| 1111|     5.555|
+------------+-------+-----+----------+
```

Unexpected Behavior:
- Surprisingly, **only 6 partitions** are assigned data, even though **10 unique ID First values** exist.
- This suggests **hash partitioning** does **not guarantee balanced partitions** if the data distribution is uneven.

The unexpected result of only **6 active partitions** (instead of **10**) occurs because **Spark uses hash partitioning** when a **column** is specified for repartitioning. While we assumed a round-robin mechanism initially, hash partitioning assigns rows based on **hash values** of the partitioning key. This can lead to **empty partitions** if the hash distribution is not uniform. (*This concept needs to be confirmed*)

We set the **number of partitions** even lower than before:
```terminal
>>> sdf_col_5 = sdf1.repartition(5, "idfirst")
>>> sdf_col_5.rdd.getNumPartitions()

>>> rows_per_partition(sdf_col_5, num_rows)
+------------+-----+----------+
|partition_id|count|count_perc|
+------------+-----+----------+
|           0| 1112|      5.56|
|           1| 2222|     11.11|
|           3| 2222|     11.11|
|           4|14444|     72.22|
+------------+-----+----------+


>>> rows_per_partition_col(sdf_col_5, num_rows, "idfirst")
+------------+-------+-----+----------+
|partition_id|idfirst|count|count_perc|
+------------+-------+-----+----------+
|           0|      0|    1|     0.005|
|           0|      6| 1111|     5.555|
|           1|      4| 1111|     5.555|
|           1|      8| 1111|     5.555|
|           3|      3| 1111|     5.555|
|           3|      7| 1111|     5.555|
|           4|      1|11111|    55.555|
|           4|      2| 1111|     5.555|
|           4|      5| 1111|     5.555|
|           4|      9| 1111|     5.555|
+------------+-------+-----+----------+
```

Observation:
- **One partition** still **receives no data**, remaining **empty**.
- **Skewed Data Distribution**: Instead of **assigning data** to the **empty partition**, Spark **fills up** an already heavily loaded partition.
- Specifically, **Partition 4** ends up with **55% of the data**, while others remain underutilized.
## 3.3. SparkUI
Let's run the following three Spark jobs with different repartitioning strategies:
reducing partitions from 4 to 3, increasing partitions from 4 to 12, and repartitioning based on the `idfirst` column from 4 to 5. Here's the code:
```python
sc.setJobDescription("Repartition from 4 to 3")
sdf_3.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sc.setJobDescription("Repartition from 4 to 12")
sdf_12.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sc.setJobDescription("Repartition from 4 to 5 with col")
sdf_col_5.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

Starting with the first job "*Repartition from 4 to 3*" we examined the Spark UI:
![](Apache%20Spark/attachments/Pasted%20image%2020241211212504.png)

Let's see the first stage:
![](Apache%20Spark/attachments/Pasted%20image%2020241211212559.png)

**Stage 1 - Initial Partitions:**
- All **four tasks** executed as expected.
- **Shuffle Write Operation**:
    - Data was **written** by the original tasks.
    - Spark prepared data for redistribution in the next stage using **shuffle**.

Here's the second stage:
![](Apache%20Spark/attachments/Pasted%20image%2020241211212728.png)

**Stage 2 - Repartitioned Partitions:**
- Spark executed **three tasks**, corresponding to the **three new partitions**.
- Tasks **read** and **processed** data after **shuffle read**.
- Each partition received **6,667 rows**, confirming a **balanced partitioning**.

Looking deeper in the SQL/DataFrame tab, we confirmed 20,000 records were shuffled, with Spark using the `RoundRobinPartitioning` method:
![](Apache%20Spark/attachments/Pasted%20image%2020241211213646.png)
![](Apache%20Spark/attachments/Pasted%20image%2020241211213541.png)

Let's move to the second job "*Repartition from 4 to 12*" we examined the Spark UI and we observe similar behavior during the first stage:
![](Apache%20Spark/attachments/Pasted%20image%2020241211214315.png)

In the second stage, Spark targeted 12 partitions. The data distribution across partitions appeared uniform, but the execution times varied significantly. This variation is likely due to scheduling overhead, as Spark needed to coordinate tasks across more partitions:
![](Apache%20Spark/attachments/Pasted%20image%2020241211214555.png)

Additionally, some delay occurred when reloading data from storage. These factors contributed to the uneven execution times despite the uniform data distribution.

Looking at the third job "*Repartition from 4 to 5 with col*" we increased the number of partitions to five while partitioning by the `idfirst` column. In the first stage, we see four tasks performing shuffle writes, as expected. In the second stage, five tasks are loaded, corresponding to the five target partitions:
![](Apache%20Spark/attachments/Pasted%20image%2020241211220332.png)
![](Apache%20Spark/attachments/Pasted%20image%2020241211220347.png)

A notable observation is that in stage 2 one task remains completely empty, as you we can see in *Shuffle Read Size / Records* field, specifically in the *Min* column which shows *0.0 B / 0*. You can see the same thing also in the following image where the *Task ID* 868 is empty in the last column. Additionally, one task receives approximately 75% of all the data, causing a significant processing imbalance:
![](Apache%20Spark/attachments/Pasted%20image%2020241211221341.png)

This skew leads to uneven execution times, with one task dominating the stage's total runtime at 5 milliseconds.

Examining the SQL/DataFrame tab of the Spark UI reveals that partitioning by `idfirst` triggers `hashpartitioning` rather than `RoundRobinPartitioning` distribution:
![](Apache%20Spark/attachments/Pasted%20image%2020241211220710.png)

Hash partitioning uses a hash key to assign records to partitions, causing sorting in the background. This behavior illustrates the critical difference between partitioning by a fixed number of partitions and partitioning by a column. Similar mechanisms will also appear in wide transformations, which we will explore in greater depth later.
## 3.4. When to use repartitioning (and partly coalesce)
To wrap up the partitioning topic, here are key scenarios where using repartitioning makes sense:
- **Adjusting Partitions for Cores:**
    - If the number of partitions isn’t divisible by the number of cores, adjust partitions.
    - Use `coalesce` to decrease partitions.
    - Use `repartition` to increase partitions. Example: From seven to eight partitions, only `repartition` works.
- **Handling Data Skew:**
    - Use `repartitioning` to manage skewed data, as `coalesce` might not be effective.
    - This ensures even data distribution and better parallelism.
- **Optimizing Joins and Shuffle Operations:**
    - Pre-partition tables on a common key before a join to reduce shuffle and sorting.
    - This can also improve operations like `orderBy`, though weigh repartitioning costs against potential gains.
- **Dealing with Empty Partitions:**
    - Filtering can cause many empty partitions.
    - Example: Filtering 10 million rows in 1,000 partitions down to 10 rows leaves 990 empty partitions.
    - Be aware of this for follow-up operations like writing files.
- **File Writing Efficiency:**
    - More partitions result in more files, potentially causing many small files in formats like Parquet.
    - Consider reducing partitions before writing to optimize file sizes.

See also [here](https://medium.com/@zaiderikat/apache-spark-repartitioning-101-f2b37e7d8301]).
## 3.5. References
- [Interesting discussion on Stackoverflow of Coalesce vs Repartition speed](https://stackoverflow.com/questions/31610971/spark-repartition-vs-coalesce)
- [When to use repartition](https://medium.com/@zaiderikat/apache-spark-repartitioning-101-f2b37e7d8301)
- [Factors to consider for no. of partitions](https://stackoverflow.com/questions/64600212/how-to-determine-the-partition-size-in-an-apache-spark-dataframe)
# 4. Spark Partitions in Action
