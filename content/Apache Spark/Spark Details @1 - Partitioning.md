---
date: 2024-08-12
modified: 2024-12-30T17:22:35+01:00
---
These notes about **Partitioning** will be "notebook-style" and only important and useful operations will be shown in this article (hence, very probably, no such operations will be shown: DataFrame creation, session state creation, etc.). For the complete code, refers to these notebooks I uploaded on a specific repository on [this link](https://github.com/simdangelo/simonedangelo-blog/tree/v4/content/Apache%20Spark/partitioning_notebooks).
# 0. Resources
* [*Data with Nikk the Greek* YouTube Channel](https://www.youtube.com/@DataNikktheGreek)
* [How are the initial number of partitions are determined in PySpark?](https://www.linkedin.com/pulse/how-initial-number-partitions-determined-pyspark-sugumar-srinivasan/) by Sugumar Srinivasan
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
Here's a quick recap about Partitions. Nothing new in this recap, it's just a way to wrap up all the concepts we have to keep in mind:
##  4.1. Recap: Why Partitioning Matters
1. **Good Parallelization:**
    - Ensure all cores in the cluster are utilized.
    - Avoid running on a single core when multiple are available.
2. **Partition Size Considerations:**
    - **Too Small:** Leads to overhead due to excessive scheduling and coordination.
    - **Too Large:** Risks out-of-memory (OOM) errors and garbage collection issues.
    - **Recommended Range:** Between 100 MB to 1 GB per partition, depending on your environment.
3. **Avoid Distribution Overhead:**
    - Having too many small partitions can cause performance loss due to scheduling overhead.
    - A good rule of thumb: Avoid tasks running under 100 milliseconds, as they might indicate an inefficient partitioning strategy.

**Repartitioning vs. Coalescing**:
- **Repartitioning:**
    - A shuffle operation that redistributes data uniformly.
    - Allows **both increasing and decreasing** partitions.
    - Supports partitioning based on specific columns.
    - Suitable for scenarios requiring balanced workloads.
- **Coalescing:**
    - A narrow transformation that combines partitions **without shuffling**.
    - **Only decreases** the number of partitions.
    - Doesn’t rebalance data, so it may cause skewed partitions.

**When to Use Repartitioning and Coalescing**:
1. **Adjusting Partitions for Cores:**
    - If the number of partitions isn’t divisible by available cores, adjust accordingly.
    - Use **Coalesce** to reduce partitions; use **Repartition** to increase them.
2. **Handling Skewed Data:**
    - Skew can cause uneven task execution. Use **Repartitioning** for better distribution.
3. **Join and Shuffle Operations:**
    - Pre-partitioning on a join key can reduce shuffle overhead.
    - Operations like `orderBy` and `groupBy` can benefit from repartitioning.
4. **Filtering Large Datasets:**
    - After filtering, many partitions might be empty. Repartitioning can optimize follow-up tasks.
5. **Dealing with Exploding Data Sizes:**
    - When exploding structured fields or lists, partition sizes can increase unpredictably. Use lower partition sizes to accommodate buffer growth.
6. **Writing Data Efficiently:**
    - Partitioning before writing can reduce file fragmentation.
    - Consider column-based partitioning for faster file access and data querying.

## 4.2. Spark Partitions in actions
We ran several jobs with different partition configurations to explore how **Coalescing** and **Repartitioning** behave when adjusting partitions.
### 4.2.1. Scenario1: Reducing from 30 to 12 Partitions
We tested four approaches:
* **Baseline** with 12 Partitions
* **Baseline** with 13 Partitions
- **Coalesce:** Reduced partitions from 30 to 12
- **Repartition:** Reduced partitions from 30 to 12
```python
sdf = sdf_generator(num_rows, 12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Base line 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 13)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Base line 13")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 13).coalesce(12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Coalesce 13 to 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 13).repartition(12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Repartition 13 to 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

**Baseline Execution (12 and 13 Partitions)**:
- The **12-partition baseline** showed a well-distributed dataset with evenly balanced tasks and execution times.
	![Pasted image 20241213223545](Apache%20Spark/attachments/Pasted%20image%2020241213223545.png)
- The **13-partition baseline**, however, revealed a notable issue: since our Spark cluster has **4 cores**, one partition couldn't be processed in parallel, leaving one core underutilized. This caused a **resource imbalance**, reducing efficiency.
	![Pasted image 20241213224117](Apache%20Spark/attachments/Pasted%20image%2020241213224117.png)

**Coalescing (30 to 12 Partitions)**:
- The process was **slightly slower** than the baseline with 12 partitions, taking about **2 seconds longer**.
- The underlying issue: **data skew**. Spark simply merged smaller partitions without redistributing data, resulting in uneven partition sizes.
- This approach **minimized shuffling**, keeping execution relatively efficient but failing to balance workloads effectively.
	![Pasted image 20241213224332](Apache%20Spark/attachments/Pasted%20image%2020241213224332.png)

**Repartitioning (30 to 12 Partitions)**
- Repartitioning required a **shuffling phase**, increasing the overall **execution time** due to data serialization (in total this job is composed of 2 stages).
- This caused **spill events**, meaning Spark had to temporarily save data to disk, significantly slowing down the process.
- **Final Result:** Despite the initial overhead, the dataset ended up **uniformly partitioned**, closely matching the performance of a pre-distributed dataset.
	![Pasted image 20241213224512](Apache%20Spark/attachments/Pasted%20image%2020241213224512.png)

### 4.2.2. Scenario2: Reducing from 20001 to 12 Partitions
We tested three approaches:
* **Baseline** with 20001 Partitions
- **Coalesce:** Reduced partitions from 20001 to 12
- **Repartition:** Reduced partitions from 20001 to 12
```python
sdf = sdf_generator(num_rows, 20001)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Base line 20001")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 20001).coalesce(12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Coalesce 20001 to 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 20001).repartition(12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Repartition 20001 to 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

**Baseline Execution (20001 Partitions)**:
+ It took 23 seconds and it's characterised by and excessive scheduling overhead caused by too many small partitions.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214130853.png)
**Coalescing (20001 to 12 Partitions)**:
- Surprisingly effective (actually no "surprisingly" at all because with only 12 partitions there no excessive scheduling overhead, so it took only 12 seconds, about 10 seconds less than the baseline).
- Resulted in near-uniform data distribution across partitions because rows per partition varied slightly, which is acceptable (min. 16659167 and max. 16669167).
- Execution time improved significantly compared to the original dataset with 20,001 partitions.
- This demonstrates an ideal scenario where `coalesce` works well, reducing the dataset without shuffling.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214140553.png)
**Repartitioning (20001 to 12 Partitions)**
- Performed worse than expected.
- Shuffling caused additional overhead.
- No memory spill occurred due to the larger partition size, but task execution still took longer (*TODO: i don't understand this point*).
- Despite even data re-distribution after shuffling, the initial small tasks from 20,001 partitions caused extra serialization and processing costs.
- Even if we look only at the second stage, `repartition` works worst than `coalesce` (12s vs 38s).
	![](Apache%20Spark/attachments/Pasted%20image%2020241214141010.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241214141025.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241214141039.png)

### 4.2.3. Scenario3: Reducing from 90 to 40 Partitions
We tested four approaches:
* **Baseline** with 40 Partitions
* **Baseline** with 90 Partitions
- **Coalesce:** Reduced partitions from 90 to 40
- **Repartition:** Reduced partitions from 90 to 40
```python
sdf = sdf_generator(num_rows, 40)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Base line 40")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 90)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Base line 90")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 90).coalesce(40)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Coalesce 90 to 40")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 90).repartition(40)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Repartition 90 to 40")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

**Baseline Execution (40 Partitions)**:
- Data is evenly distributed.
- Slight variations in task durations due to minor scheduling differences.
- Overall execution is efficient and balanced.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214142404.png)

**Baseline Execution (90 Partitions)**:
- Uniform distribution of data.
- Tasks are smaller and shorter, but the total execution time remains close to the 40-partition baseline.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214142443.png)

**Coalescing (90 to 40 Partitions)**:
- Resulted in some skewed partitions:
    - Average partition size: ~4444444 million rows.
    - Largest partition: 6666667 million rows.
- One task took longer (1 second vs 0.8 seconds for the median and 0.7 seconds for the min) and kept one core busy at the end, indicating skew but still acceptable overall.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214142720.png)

**Repartitioning (90 to 40 Partitions)**:
- High overhead due to shuffling during repartitioning.
- After shuffling, execution was fast and evenly distributed.
- The initial shuffling operation was costly, making repartitioning inefficient in this scenario.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214142757.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241214142813.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241214142835.png)

### 4.2.4. Scenario4: Increasing Partitions
We tested five approaches:
* **Baseline** with 1 Partitions
* **Baseline** with 10 Partitions
* **Baseline** with 12 Partitions
- **Repartition:** Reduced partitions from 1 to 12
- **Repartition:** Reduced partitions from 10 to 12
```python
sdf = sdf_generator(num_rows, 1)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Base line 1")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 10)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Base line 10")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Base line 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 1).repartition(12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Repartition 1 to 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 10).repartition(12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Repartition 10 to 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

**Baseline Execution (1 Partition)**:
- Execution runs entirely on **one core**, causing significant delays.
- The process is inefficient due to the lack of parallelism.
	![](Apache%20Spark/attachments/Screenshot%202024-12-14%20at%2023.19.17.png)

**Baseline Execution (10 Partitions)**:
- Two tasks run independently on two cores, leaving the other two cores **idle**.
- Parallelism improves slightly but remains suboptimal.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214231951.png)

**Baseline Execution (12 Partitions)**:
1. The process runs **perfectly parallel**, fully utilizing all cores.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214232015.png)

**Repartitioning (1 to 12 Partitions)**:
- This process involves a **heavy shuffle operation** due to the significant increase in partitions.
- We observe **spills** during the serialization phase, making the shuffling process even more complicated.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214232051.png)
	![](Apache%20Spark/attachments/Screenshot%202024-12-14%20at%2023.42.15.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241214234242.png)

**Repartitioning (10 to 12 Partitions)**:
- Though the initial distribution is better, some **shuffle writes and spills** still occur.
- The resulting execution is efficient after repartitioning, achieving a **well-distributed dataset** with balanced durations.
	![](Apache%20Spark/attachments/Pasted%20image%2020241214234347.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241214234423.png)![](Apache%20Spark/attachments/Pasted%20image%2020241214234436.png)

### 4.2.5. Scenario5: Skewed Dataset
We tested x approaches:
* Baseline of **Skewed Dataset** with 12 partitions
- **Coalesce:** Reduced a **Skewed Dataset** from 12 to 8
- **Repartition:** Repartitioned a **Skewed Dataset** from 12 to 12
- **Repartition:** Repartitioned a **Skewed Dataset** from 12 to 8
```python
sdf = sdf_generator(num_rows, 15)
sdf = sdf.coalesce(12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Base line skewed 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 15)
sdf = sdf.coalesce(12)
sdf = sdf.coalesce(8)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Coalesce for Skew 8")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 15)
sdf = sdf.coalesce(12)
sdf = sdf.repartition(12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Repartition for Skew 12")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 15)
sdf = sdf.coalesce(12)
sdf = sdf.repartition(8)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Repartition for Skew 8")
sdf.write.format("noop").mode("overwrite").save()
sc.setJobDescription("None")
```

**Baseline of Skewed Dataset with 12 partitions**
- Three tasks occupy most of the cores.
- Depending on the specific moment in the timeline, some cores remain **unused**, indicating clear data skew.
- Some partitions are significantly **larger**, causing uneven task durations.
	![](Apache%20Spark/attachments/Pasted%20image%2020241215151108.png)

**Coalesce: Reduced a Skewed Dataset from 12 to 8**
* Slightly improved but still skewed.
- While we use fewer partitions, the tasks remain **unevenly distributed**.
	![](Apache%20Spark/attachments/Pasted%20image%2020241215151144.png)

**Repartition: Repartitioned a Skewed Dataset from 12 to 12**
- Partitions are **evenly distributed** across tasks.
- There is some **spill** due to the shuffle operation, causing minor delays.
	![](Apache%20Spark/attachments/Pasted%20image%2020241215152416.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241215152431.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241215152444.png)

**Repartition: Repartitioned a Skewed Dataset from 12 to 8**
* Comparable to the repartition-to-12 case.
- **Task Distribution:**
	- **Well-balanced** partitions.
	- Uniform execution times across all tasks.
	![](Apache%20Spark/attachments/Pasted%20image%2020241215152539.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241215152551.png)
	![](Apache%20Spark/attachments/Pasted%20image%2020241215152603.png)
#### Query Optimizer Insight:
Upon inspecting Spark’s **physical plan**, we noticed:
- **Coalesce Optimization:**
    - In the "*Coalesce for Skew 8*" job, when coalescing from 15 to 12 and then to 8, Spark **ignored the intermediate step**, applying only **one coalesce** to 8.
    - The second coalesce was **skipped entirely**, as the optimizer combined both operations into one.
	![](Apache%20Spark/attachments/Pasted%20image%2020241215153758.png)
- **Repartition Optimization:**
    - Similarly in the last two jobs, when attempting coalesce followed by a repartition, Spark directly applied **round-robin partitioning**, skipping the coalesce.
	![](Apache%20Spark/attachments/Pasted%20image%2020241215153821.png)

### 4.2.6. Scenario6: Filter operations become more efficient
34.13 to finish!
# 5. Spark Partitions when Saving Files
Let's explore how **data partitioning** affects saving data in Spark using **Parquet** files. We'll examine how the number of partitions influences the number of files created when saving a dataset.
## 5.1. How many files are written based on the number of partitions
We run three different tests with varying partition counts:
```python
base_dir = "base_dir/"

sdf = sdf_generator(num_rows, 1)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Write 1 file")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/test_1_file.parquet")
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 4)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Write 4 file")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/test_4_file.parquet")
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 12)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Write 12 file")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/test_12_file.parquet")
sc.setJobDescription("None")
```

Results:
1. **One Partition:**
    - Save the dataset with **one partition**.
    - Result: **One Parquet file** generated.
    - File Size: ~8MB.
    - Insights: The dataset is saved into a **single Snappy Parquet file**, while additional metadata files may be generated for schema and stats.
    - **Task Count:** 1 task processed. Since only **one task** was assigned, there’s **no parallelism**, resulting in a straightforward, single write operation.
	![](Apache%20Spark/attachments/Pasted%20image%2020241215164542.png)
1. **Four Partitions:**
    - Save the dataset with **four partitions**.
    - Result: **Four Parquet files** created.
    - Output Size per File: ~2MB (split evenly).
    - Insights: Each partition is saved into its **own file**, allowing better parallelism and write efficiency.
    - **Task Count:** 4 tasks processed. Each task handled **one partition**, enabling **parallel writing**.
	![](Apache%20Spark/attachments/Pasted%20image%2020241215164618.png)
1. **Twelve Partitions:**
    - Save the dataset with **twelve partitions**.
    - Result: **Twelve Parquet files** created.
    - Insights: Every partition gets its **own file**, improving **parallel writes**. Each file is processed independently, avoiding write bottlenecks.
    - Output Size per File: ~700KB (split evenly).
    - **Task Count:** 12 tasks processed. Tasks **parallelized efficiently**, though with minimal data, task management overhead could slightly affected performance.
	![](Apache%20Spark/attachments/Pasted%20image%2020241215164826.png)

Here's the visual results:
![](Apache%20Spark/attachments/Pasted%20image%2020241215162905.png)
### Key Takeaways:
- **File Creation Logic:** The number of **Parquet files** created **matches the number of partitions** in Spark.
- **Parallelism:** Spark writes each partition **independently**, ensuring maximum efficiency.
- **Metadata Files:** Additional metadata files might be created, likely containing schema details and file statistics, though their exact content depends on the Parquet specification.
### Key Takeaways from the Spark UI
- **No Shuffling:** Since data saving involves no complex transformations, Spark skips shuffling, resulting in a **single stage job**.
- **Task Distribution:** The number of **tasks equals the number of partitions**, ensuring **maximum parallelism**.
- **Data Size Stats:** The *Spark/DataFrame* tab shows detailed **output file sizes**, **row counts**, and **files created**, confirming that **one file per partition** is created.
## 5.2. Using coalesce and Repartition to save data
Let's dive into how **Coalescing** and **Repartitioning** affect file writes in Spark, particularly when reducing partitions to generate fewer, larger output files.

We ran tests where data was saved with:
- **Coalescing to 4 Partitions**
- **Coalescing to 1 Partition**
- **Repartitioning to 4 Partitions**
- **Repartitioning to 1 Partition**
```python
sdf = sdf_generator(num_rows, 12)
sdf = sdf.coalesce(4)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Write with Coalesce 12 to 4 file")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/test_12_to_4_coalesce_file.parquet")
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 12)
sdf = sdf.coalesce(1)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Write with Coalesce 12 to 1 file")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/test_12_to_1_coalesce_file.parquet")
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 12)
sdf = sdf.repartition(4)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Write with Repartition 12 to 4 file")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/test_12_to_4_repartition_file.parquet")
sc.setJobDescription("None")

sdf = sdf_generator(num_rows, 12)
sdf = sdf.repartition(1)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Write with Repartition 12 to 1 file")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/test_12_to_1_repartition_file.parquet")
sc.setJobDescription("None")
```

**Coalescing to 4 Partitions:**
- **Execution Time:** (Potentially) Faster than coalescing to 1 due to **better core utilization**.
- **Task Utilization:** **Four cores** used evenly, ensuring **balanced processing**.
- **File Output:** **Four files** generated, each about 2MB.
- **Takeaway:** Coalescing is efficient when the target partition count is reasonable and divisible by the core count.

**Coalescing to 1 Partition:**
- **Execution Time:** (Potentially) Slower than coalescing to 4 due to **worse core utilization**.
- **Task Utilization:** Only **one core** is used, leading to slower performance.
- **File Output:** A **single file** of ~8MB generated.
- **Takeaway:** Since **no shuffling** occurs, only one task processes all data, causing slower write performance.

**Repartitioning to 4 Partitions:**
- **Execution Time:** Slightly slower than coalescing due to **shuffle overhead**.
- **Task Utilization:** Fully distributed during the **shuffle stage** and **file writing**.
- **File Output:** **Four files**, each about 2MB.
- **Takeaway:** Repartitioning leverages **parallel processing** but incurs **shuffle costs**, making it more suitable for balancing uneven data.

**Repartitioning to 1 Partition:**
- **Execution Time:** Slightly longer due to **shuffle overhead**.
- **Task Utilization:** Initially uses **all available cores** before **final merging** into one partition.
- **File Output:** A **single file** of ~8MB.
- **Takeaway:** Even though there's **shuffle overhead**, **parallelism** during initial processing improves performance compared to coalescing.
## 5.3. Empty partitions problem when writing
Let's revisit the issue of **empty partitions** when saving data. In a previous session, we discussed how filtering data could result in **empty partitions**, raising concerns about **zero-byte files** being written. This could lead to **unnecessary overhead** in the system, causing file management issues.

To test this, we created a data set with **20 partitions**, containing **1 million rows**. After applying a **filter**, only **200 rows** remained, all located in a **single partition**. The question was: Would Spark still write **19 empty files**?
```python
sdf = sdf_generator(num_rows, 20)
sdf = sdf.filter(f.col("id") < 200)
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("Empty rows")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/emptyRows.parquet")
sc.setJobDescription("None")
```

If we see the storage account, we are pleased to see that Spark only wrote **one file**:
![](Apache%20Spark/attachments/Pasted%20image%2020241215175054.png)

This means that Spark is **smart enough** to avoid writing empty files when saving data. While I’ve occasionally encountered cases where **zero-byte files** were written, this behavior seems **rare** and likely depends on **specific configurations**.

Looking at the **Spark UI**, we noticed that the job ran **significantly faster** compared to the previous save experiments, due to the filtering, given that only **one partition** needed to be saved:
![Screenshot 2024-12-16 at 22.20.07](Apache%20Spark/attachments/Screenshot%202024-12-16%20at%2022.20.07.png)

If we go into the details, we'll see that the partition containing the **200 rows** was the only one processed, and its corresponding task (the first one at the top in the following image) took the most time:
![Pasted image 20241216222327](Apache%20Spark/attachments/Pasted%20image%2020241216222327.png)

In the **SQL query details**, we confirmed:
- **Initial Rows:** 1 million
- **Filtered Rows:** 200
- **File Size:** 3.4KB
- **Number of Files:** 1 file created, as expected.
![Pasted image 20241216222610](Apache%20Spark/attachments/Pasted%20image%2020241216222610.png)

While there was some **deserialization time** observed, likely due to **partition scanning** during the save operation, this was a minor detail compared to the overall performance improvement. This experiment highlights how Spark **efficiently skips empty partitions**, ensuring optimized storage and reducing **system overhead**.
## 5.4. Repartitioning by column idfirst
When **repartitioning by a column**, you might encounter **empty partitions** due to **hash partitioning**. This happens when Spark assigns rows to partitions based on the **hash modulo value** of the partitioning column. If multiple rows resolve to the same hash value, they end up in the **same partition**, potentially leaving others **empty**.

To see this in action, we started with **20 partitions** and **repartitioned down to 10** using the `idfirst` column:
```python
sdf = sdf_generator(num_rows, 20)
sdf = sdf.repartition(10, "idfirst")
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("repartition 10 idfirst")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/repartition_10_idfirst.parquet")
sc.setJobDescription("None")
```

When inspecting the results, only **six partitions** contained data:
```terminal
>>> rows_per_partition(sdf)
+------------+------+----------+
|partition_id| count|count_perc|
+------------+------+----------+
|           3|111111|   11.1111|
|           4|111111|   11.1111|
|           5|111112|   11.1112|
|           6|222222|   22.2222|
|           8|111111|   11.1111|
|           9|333333|   33.3333|
+------------+------+----------+
```

Upon checking the **Parquet files**, we found **six valid files** with data and **one empty file** of **less than 1KB size**:
![Pasted image 20241216223735](Apache%20Spark/attachments/Pasted%20image%2020241216223735.png)

To verify, we loaded the **1KB Parquet file** using Spark's `read.parquet()` method and ran `.show()`.
```terminal
>>> spark.read.parquet(f"{base_dir}/repartition_10_idfirst.parquet/part-00000-c36f2adc-e01d-4f64-84b3-8f06fcfcee4c-c000.snappy.parquet").show()
+---+----+---------+--------+-------+------+
| id|date|timestamp|idstring|idfirst|idlast|
+---+----+---------+--------+-------+------+
+---+----+---------+--------+-------+------+
```

As expected, the file was **empty**, confirming that Spark wrote a file even when no data existed in that partition.

We tried the same experiment, this time repartitioning down to **eight partitions**:
```python
sdf = sdf_generator(num_rows, 20)
sdf = sdf.repartition(8, "idfirst")
print(sdf.rdd.getNumPartitions())
sc.setJobDescription("repartition 8 idfirst")
sdf.write.format("parquet").mode("overwrite").save(f"{base_dir}/repartition_8_idfirst.parquet")
sc.setJobDescription("None")
```

Partitions' details:
```terminal
>>> rows_per_partition(sdf)
+------------+------+----------+
|partition_id| count|count_perc|
+------------+------+----------+
|           0|111111|   11.1111|
|           1|111111|   11.1111|
|           2|222222|   22.2222|
|           3|333334|   33.3334|
|           5|111111|   11.1111|
|           6|111111|   11.1111|
+------------+------+----------+
```

Interestingly, in this run, there were no empty files—Spark only saved files where data existed:
![Pasted image 20241216224541](Apache%20Spark/attachments/Pasted%20image%2020241216224541.png)

This experiment highlighted how **hash partitioning** can cause **imbalances** when repartitioning by a column, potentially leaving some partitions **empty**.
# 6. Spark Partitions when Loading Files
We’ll explore how **data loading with Parquet files in Spark** works, focusing on **partitions** and **data distribution** during loading. After diving deep into this topic recently, I realized it’s **highly complex** with many influencing factors. 

Understanding how **Spark generates partitions** while loading data helps us:
- **Optimize Data Processing:** A good grasp of partitioning can **reduce the need for expensive operations** like **repartitioning** or **coalescing**, which introduce extra overhead.
- **Enhance Performance:** Properly configuring data partitions from the start makes the **entire process more efficient**.
- **Avoid Empty Partitions:** These **waste resources** and **slow down execution**, as we saw in earlier sessions.

To assist with this, I added a useful Spark package from **Biosoft** that helps **analyze data distributions** and the **file system** more easily. This tool prevents us from jumping back and forth into the Spark UI by providing details like **data sizes** and **row counts per partition**.
## 6.1. Setup
Let's list the functions we're going to use for this analysis:
1. **Data Writer Generator:**
    - This function generates uniformly distributed Parquet files based on **row count** and **number of files (partitions)**.
    - Example: If we set **rows = 2000** and **files = 4**, it will create **4 Parquet files** with **500 rows each**.
2. **Configuration Setter:** We predefine Spark configurations with **default values**. We’ll discuss these settings in detail later.
3. **Parquet Analysis Tool:** Using the **Parquet Partitions Function** from the added module, we can:
	- **Calculate partition stats:** Including **bytes per file**, **number of files**, and **rows per partition**.
	- **View Distribution Tables:** Displaying how data is divided across partitions.

Here's the code for this functions:
```python
def sdf_generator(num_rows: int, num_partitions: int = None) -> "DataFrame":
    return (
        spark.range(num_rows, numPartitions=num_partitions)
        .withColumn("date", f.current_date())
        .withColumn("timestamp",f.current_timestamp())
        .withColumn("idstring", f.col("id").cast("string"))
        .withColumn("idfirst", f.col("idstring").substr(0,1))
        .withColumn("idlast", f.col("idstring").substr(-1,1))
        )

BASE_DIR = "base_dir_loading_lesson/"

results_dict = {}
results_list = []
def write_generator(num_rows, num_files):
    sdf = sdf_generator(num_rows, num_files)
    path = f"{BASE_DIR}/{num_files}_files_{num_rows}_rows.parquet"
    sc.setJobDescription(f"Write {num_files} files, {num_rows} rows")
    sdf.write.format("parquet").mode("overwrite").save(path)
    sc.setJobDescription("None")
    print(f"Num partitions written: {sdf.rdd.getNumPartitions()}")
    print(f"Saved Path: {path}")
    return path

def set_configs(maxPartitionsMB = 128, openCostInMB = 4, minPartitions = 4):
    maxPartitionsBytes = math.ceil(maxPartitionsMB*1024*1024)
    openCostInBytes = math.ceil(openCostInMB*1024*1024)
    spark.conf.set("spark.sql.files.maxPartitionBytes", str(maxPartitionsBytes)+"b")
    spark.conf.set("spark.sql.files.openCostInBytes", str(openCostInBytes)+"b")
    spark.conf.set("spark.sql.files.minPartitionNum", str(minPartitions))
    print(" ")
    print("******** SPARK CONFIGURATIONS ********")
    print(f"MaxPartitionSize {maxPartitionsMB} MB or {maxPartitionsBytes} bytes")
    print(f"OpenCostInBytes {openCostInMB} MB or {openCostInBytes} bytes")
    print(f"Min Partitions: {minPartitions}")

    results_dict["maxPartitionsBytes"] = maxPartitionsMB
```
## 6.2. What influences the no. of partitions when loading parquet files
Let's begin by discussing **what influences the partitions** when working with Parquet files in Spark. I've also added some references if you'd like to explore the topic further after this session.
1. **Number of Cores:**
    - The **number of available cores** influences the number of partitions when creating data frames or loading data.
    - This makes sense because **unused cores** would result in less efficient parallelism.
    - Technically, Spark uses the configuration **`spark.sql.files.minPartitionNum`**. If not set, the default value is `spark.sql.leafNodeDefaultParallelism`. This configuration is effective only when using file-based sources such as Parquet, JSON and ORC. ([Note by stackoverflow](https://stackoverflow.com/questions/45704156/what-is-the-difference-between-spark-sql-shuffle-partitions-and-spark-default-pa)  and [by Spark Documentation](https://spark.apache.org/docs/latest/configuration.html#execution-behavior): "`spark.default.parallelism` is the default number of partitions in `RDD`s returned by transformations like `join`, `reduceByKey`, and `parallelize` when not set explicitly by the user. Note that `spark.default.parallelism` seems to only be working for raw `RDD` and is ignored when working with dataframes.". Just to mention a conceptual linked parameter, let's also describe `spark.sql.shuffle.partitions`, which is 200 by default, and represents the number of partitions to use when shuffling data for joins or aggregations.)
2. **File Size or Estimated File Size:**
    - Spark considers the **size of Parquet files** when determining partitioning.
    - We’ll explore this concept more deeply later, but keep in mind that **Parquet files are split into blocks, row groups, and Snappy compressed files**.
    - If files become larger, the splitting process follows its internal logic, which can also affect performance and cause **empty partitions**.
3. **Max Partition Size (`spark.sql.files.maxPartitionBytes`):**
    - This parameter **limits the maximum partition size**.
    - The default value is **128 MB**, meaning that if a file exceeds this size, Spark will split it into multiple partitions accordingly.
4. **Max Cost Per Bytes (`spark.sql.files.openCostInBytes`):**
    - This parameter **represents the cost of creating a new partition**. It defaults to **4 MB**.
    - While **less critical for performance**, its effect is notable when working with **smaller files**.
    - Spark **pads the file size** by adding the value of this parameter, reducing the number of small partitions generated. This results in **fewer, larger partitions**, avoiding idle cores caused by many tiny files.

References:
- [https://stackoverflow.com/questions/70985235/what-is-opencostinbytes](https://stackoverflow.com/questions/70985235/what-is-opencostinbytes)
- [https://stackoverflow.com/questions/69034543/number-of-tasks-while-reading-hdfs-in-spark](https://stackoverflow.com/questions/69034543/number-of-tasks-while-reading-hdfs-in-spark)
- [https://stackoverflow.com/questions/75924368/skewed-partitions-when-setting-spark-sql-files-maxpartitionbytes](https://stackoverflow.com/questions/75924368/skewed-partitions-when-setting-spark-sql-files-maxpartitionbytes)
- [https://spark.apache.org/docs/latest/sql-performance-tuning.html](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
- [https://www.linkedin.com/pulse/how-initial-number-partitions-determined-pyspark-sugumar-srinivasan#:~:text=Ideally%20partitions%20will%20be%20created,resource%20will%20get%20utilised%20properly](https://www.linkedin.com/pulse/how-initial-number-partitions-determined-pyspark-sugumar-srinivasan#:~:text=Ideally%20partitions%20will%20be%20created,resource%20will%20get%20utilised%20properly)
## 6.3. How are the initial number of partitions are determined in PySpark? (by Sugumar Srinivasan on LinkedIn)
*Important note: Before continuing with Jake Callahan's video I want to copy&paste here an interesting [LinkedIn article](https://www.linkedin.com/pulse/how-initial-number-partitions-determined-pyspark-sugumar-srinivasan/) by Sugumar Srinivasan called "How are the initial number of partitions are determined in PySpark?", which provides some practical hints on this topic. I want to discuss first about this article because the next chapters [6.4. Basic Algorithm](#6.4.%20Basic%20Algorithm) and [6.5. Simple Experiments](#6.5.%20Simple%20Experiments) are based on this article.*

It is super interesting topic in Apache Spark, This can be demonstrated using below 3 categories.
- Calculating the partitions with **one single large file** (demonstration in this article).
- Calculating the partitions with **multiple small file**.
- Calculating the partitions with a **file which is non-splittable**.

But I'm going to explain this using the first category and remaining two will try to cover it in my upcoming article (*PS: Unfortunately these upcoming articles have not yet been made*).
### Calculating the partitions with one single large file
Consider we have a single large CSV file in HDFS which is slightly over 1GB (1GB is not that much bigger file that we are going to process in the real world).

Consider we are requesting **2 executors with 2 CPU cores** for processing the data while creating the spark session as mentioned below:
```python {4,5}
spark = SparkSession. \
	builder. \
	config("spark.dynamicAllocation.enabled", False). \
	config("spark.exector.cores", 2). \
	config("spark.executor.instances", 2). \
	config("spark.executor.memory", "2g"). \
	config("spark.sql.warehouse.dir", "/user/hive/warehouse"). \
	enableHiveSupport(). \
	master("yarn"). \
	getOrCreate()
```

Once the spark session gets created, we are good to read the slightly over 1GB CSV file from HDFS to Spark's DataFrame as mentioned below.
```python
sample_data_frame = spark.read \
	.format("csv") \
	.option("header", True) \
	.schema(sample_schema) \
	.load("/user/haoopusr/data/sample_1gb.csv")
```

Once the DataFrame is created, now we can check how many partitions are created in spark's DataFrame through below line.
```terminal
>>> sample_data_frame.rdd.getNumPartitions()
>>> 9
```

From the above result, the CSV file has been split into **9 partitions**. Now you might be thinking what is the logic behind this file splitting into 9 partitions, the answer is simple. The given file will be divided into multiple partitions based the property value `maxPartitionBytes`, by default it is set to **134217728b i.e 128MB**. This property can be tweak if we want.
```terminal
>>> spark.sql.files.maxPartitionBytes
>>> '134217728b'

>>> # Byte to MB Conversion
>>> 134217728/(1024 * 1024)
128.0
```

In this case, since the `maxPartitionBytes` is set to 128MB, the CSV file which is slightly over 1GB has been divided into **9 partitions with 8 partitions size of 128MB** and **9th partition with less in size**.

Now we know the logic of the number of partitions created for the file in question, how can we know **if all 9 partitions will be processed in parallel**? To answer this question, this is totally depending upon how much CPU cores we are allocating for the particular job.

In order to know how many partitions can be processed in parallel based the resource configuration that we set, we can execute the below line either in Spark-Shell or Jupyter Notebook:
```terminal
>>> spark.sparkContext.defaultParallelism
4
```

In this case, **Parallelism** at any given point time can be a **maximum of 4** because you have totally 4 CPU cores.

Consider an another scenario here before moving ahead: let's say we have **slightly more than 1GB CSV file**, but this time we are requesting **6 executors with 2 CPU cores** to process the data:
```terminal
>>> spark.sparkContext.defaultParallelism
12
```
In this case, **Parallelism** at any given point time can be a **maximum of 12** because you have totally 12 CPU cores.

Now you might think that with the default partition size of 128 MB you would have 9 partitions. If this were the case, however, we would only have 9 active cores out of 12 available, which means that the remaining **3 would be inactive**. This would not be an efficient use of resources. However, Spark is intelligent and **divide the file in such a way** that all the resources what we have allocated for the job will get utilised wisely and **none of the resources will be sitting in idle**. In this case Spark will try to create **12 partitions of smaller size** and no longer 9 partitions of 128MB.

Based on that, we can formulate the following rule to compute the **initial number of partitions when loading a single large file**:
> **No of Partitions = min(128MB, (File Size in MB/defaultParallelism))**

**Summary:**
1. If the executor instances and executor cores both are set to 1, then defaultParallelism would be 2 because by default in our cluster defaultParallelism is set to 2.
2. Ideally defaultParallelism can be determined by multiplying number of executor instances and number of cpu cores given per instance.**defaultParallelism = (spark.executor.instances * spark.executor.cores)**
3. To check the default parallelism that we get for the spark job, run the below line in either in spark-shell or jupyter notebook.**spark.sparkContext.defaultParallelism**
4. Ideally partitions will be created based on `maxPartitionBytes` that we set (by default it is '134217728b' 128MB). In some cases, like we have more resources but we have less partitions, in such cases spark decides the partitions size based on the below logic so that resource will get utilised properly: **No of Partitions = min(128MB, (File Size in MB/defaultParallelism))**
5. To check number of partitions get created for any given spark data frame, run the below line either in spark-shell or jupyter notebook upon reading the file from HDFS/Data lake(Amazon s3, Azure Data lake Gen2, etc.)**<your_df_name>.rdd.getNumPartitions()**
## 6.4. Basic Algorithm
Let's break down how Spark calculates the number of partitions when loading data, using this **basic algorithm**:
```python
#Basic algorithm
def basic_algorithm(file_size):
    maxPartitionBytes = int(spark.conf.get("spark.sql.files.maxPartitionBytes")[:-1])    
    minPartitionNum = int(spark.conf.get("spark.sql.files.minPartitionNum"))
    size_per_core = file_size/minPartitionNum
    partition_size = min(maxPartitionBytes, size_per_core)
    no_partitions = file_size/partition_size #round up for no_partitions
    
    print(" ")
    print("******** BASIC ALGORITHM TO ESTIMATE NO PARTITIONS ********")
    print(f"File Size: {round(file_size/1024/1024, 1)} MB or {file_size} bytes")
    print(f"Size Per Core: {round(size_per_core/1024/1024, 1)} MB or {size_per_core} bytes")
    print(f"Partionsize: {round(partition_size/1024/1024, 1)} MB or {partition_size} bytes")
    print(f"EstimatedPartitions: {math.ceil(no_partitions)}, unrounded: {no_partitions}")
```

Let's describe all the variables inside this function:
- `file_size`: The total size of the data file being loaded into Spark.
- `maxPartitionBytes`: Sets the **maximum size per partition**, defaulting to **128 MB**. It is represented by `spark.sql.files.maxPartitionBytes`.
* `minPartitionNum`: Defines the **minimum number of partitions** to be created, defaulting to the number of **available cores**, ensuring proper parallelism. It is represented by `spark.sql.files.minPartitionNum`.
- `partition_size`: Represents the partition size and it's computed choosing from the minimum value from `size_per_core` (= `file_size / minPartitionNum`) and `maxPartitionBytes`, as we saw in the previous chapter [Calculating the partitions with one single large file](#Calculating%20the%20partitions%20with%20one%20single%20large%20file).
* `no_partitions`calculated as `file_size / partition_size` to ensure that the data is divided into an optimal number of partitions.
### 6.4.1. Change `file_size`
Suppose we have:
- A 64 MB file
- A default `maxPartitionBytes` = 128 MB
- A `minPartitionNum` = 4.
```terminal
>>> file_size = 64
>>> set_configs(maxPartitionsMB=128, openCostInMB=4, minPartitions=4)
>>> basic_algorithm(file_size*1024*1024)

******** SPARK CONFIGURATIONS ********
MaxPartitionSize 128 MB or 134217728 bytes
OpenCostInBytes 4 MB or 4194304 bytes
Min Partitions: 4
 
******** BASIC ALGORITHM TO ESTIMATE NO PARTITIONS ********
File Size: 64.0 MB or 67108864 bytes
Size Per Core: 16.0 MB or 16777216.0 bytes
Partionsize: 16.0 MB or 16777216.0 bytes
EstimatedPartitions: 4, unrounded: 4.0
```

To get these results, the algorithm proceeded as follows:
1. **Size Per Core**: `size_per_core = file_size / minPartitionNum` = `64 MB / 4 cores = 16 MB per core`.
2. **Partition Size**: `partition_size = min(maxPartitionBytes, size_per_core)` = `min(128 MB, 16 MB) = 16 MB per partition`.
3. **Number of Partitions**: `no_partitions = file_size / partition_size` = `64 MB / 16 MB = 4 partitions`.

Here, Spark chooses a partition size of **16 MB**, as it ensures parallelism by using all cores.

Let’s consider a **100 MB file**:
```terminal
>>> file_size = 100
>>> set_configs(maxPartitionsMB=128, openCostInMB=4, minPartitions=4)
>>> basic_algorithm(file_size*1024*1024)

******** SPARK CONFIGURATIONS ********
MaxPartitionSize 128 MB or 134217728 bytes
OpenCostInBytes 4 MB or 4194304 bytes
Min Partitions: 4
 
******** BASIC ALGORITHM TO ESTIMATE NO PARTITIONS ********
File Size: 100.0 MB or 104857600 bytes
Size Per Core: 25.0 MB or 26214400.0 bytes
Partion size: 25.0 MB or 26214400.0 bytes
EstimatedPartitions: 4, unrounded: 4.0
```
1. **Size Per Core** `size_per_core = 100 MB / 4 cores = 25 MB per core`.
2. **Partition Size**: `partition_size = min(128 MB, 25 MB) = 25 MB per partition`.
3. **Number of Partitions**: `no_partitions = 100 MB / 25 MB = 4 partitions`.

Here, Spark chooses a partition size of **25 MB**, as it ensures parallelism by using all cores.

**Insights from the Algorithm**:
- Spark optimizes the **partition size** to balance between:
    - **Efficient use of available cores** (avoiding idle cores),
    - **Avoiding oversized partitions** (maintaining the max partition bytes limit).
- By default, the **max partition size (128 MB)** ensures that even large files are divided into smaller, manageable chunks.
- **Minimum partitions** (based on cores) guarantee good parallelism for processing.

Let’s consider a **200 MB file**:
```terminal
>>> file_size = 200
>>> set_configs(maxPartitionsMB=128, openCostInMB=4, minPartitions=4)
>>> basic_algorithm(file_size*1024*1024)

******** SPARK CONFIGURATIONS ********
MaxPartitionSize 128 MB or 134217728 bytes
OpenCostInBytes 4 MB or 4194304 bytes
Min Partitions: 4
 
******** BASIC ALGORITHM TO ESTIMATE NO PARTITIONS ********
File Size: 200.0 MB or 209715200 bytes
Size Per Core: 50.0 MB or 52428800.0 bytes
Partion size: 50.0 MB or 52428800.0 bytes
EstimatedPartitions: 4, unrounded: 4.0
```

Here, Spark chooses a partition size of **50 MB**.
### 6.4.2. Change `maxPartitionBytes`
Suppose we have:
- A 200 MB file
- A default `maxPartitionBytes` = 45 MB
- A `minPartitionNum` = 4.
```
>>> file_size = 200
>>> set_configs(maxPartitionsMB=45, openCostInMB=4, minPartitions=4)
>>> basic_algorithm(file_size*1024*1024)

******** SPARK CONFIGURATIONS ********
MaxPartitionSize 45 MB or 47185920 bytes
OpenCostInBytes 4 MB or 4194304 bytes
Min Partitions: 4
 
******** BASIC ALGORITHM TO ESTIMATE NO PARTITIONS ********
File Size: 200.0 MB or 209715200 bytes
Size Per Core: 50.0 MB or 52428800.0 bytes
Partion size: 45.0 MB or 47185920 bytes
EstimatedPartitions: 5, unrounded: 4.444444444444445
```

**Impact of Non-Splittable Snappy Files** (*TODO: I don't know if it's correct and it's not so clear to me*)
- **Snappy Parquet Compression**:
    - Snappy-compressed Parquet files are **not splittable**.
    - If a Snappy file exceeds the calculated partition size, it **cannot be further split** during partitioning.
- **Effect**:
    - For datasets with only a few files (e.g., 4 files for 200 MB):
        - Partitioning into smaller chunks (e.g., 45 MB) **may not work effectively**.
    - For datasets with many files (e.g., 20 or 30 files):
        - Partitioning into smaller sizes becomes more feasible as each file contributes smaller data chunks.

Now let's increase `maxPartitionBytes` to 200.000 MB:
```terminal
>>> file_size = 200
>>> set_configs(maxPartitionsMB=200000, openCostInMB=4, minPartitions=4)
>>> basic_algorithm(file_size*1024*1024)

******** SPARK CONFIGURATIONS ********
MaxPartitionSize 200000 MB or 209715200000 bytes
OpenCostInBytes 4 MB or 4194304 bytes
Min Partitions: 4
 
******** BASIC ALGORITHM TO ESTIMATE NO PARTITIONS ********
File Size: 200.0 MB or 209715200 bytes
Size Per Core: 50.0 MB or 52428800.0 bytes
Partion size: 50.0 MB or 52428800.0 bytes
EstimatedPartitions: 4, unrounded: 4.0
```

**Observation**: Even though the `maxPartitionBytes` is set higher, the partitioning logic defaults to the size per core to ensure that all available cores are utilized.

**Safety Mechanism for Partitioning** (*TODO: I don't know if it's correct and it's not so clear to me*)
- If the `maxPartitionBytes` value is set too low, Spark ensures that:
    - The **minimum number of partitions** is respected (based on available cores).
    - Partition sizes are adjusted to avoid creating **excessively small partitions** that would lead to overhead or underutilized resources.
- **Example**:
    - For **200 MB file size**, setting `maxPartitionBytes = 4 MB`:
        - Instead of generating **50 partitions** (which could cause inefficiency), Spark would:
            - Default to **4 partitions** (one per core),
            - Assign **50 MB per partition**.
## 6.5. Simple Experiments
*TODO: to finish!*