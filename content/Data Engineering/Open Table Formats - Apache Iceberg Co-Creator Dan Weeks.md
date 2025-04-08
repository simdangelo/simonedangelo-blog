---
date: 2025-03-31
modified: 2025-04-01T22:12:49+02:00
---

*The following one is a transcription/re-elaborated version of a really interesting [speech on YouTube](https://www.youtube.com/watch?v=Hh1MkMBAqqI) about Apache Iceberg taken by Dan Weeks, who is one of the creators of this technology.*

---

We talk about it as a **universal analytic table format**. Iceberg really **upgrades** what a **data lake** is to be **more like a data warehouse**. It provides a lot of the benefits and behaviors of a data warehouse on top of **object storage lake storage**. It also brings a whole new host of capabilities:
- Time travel
- Branching
- Tagging

> [!note] The main idea behind Apache Iceberg
> Iceberg is a **revolutionary way** of storing data and storing table data to provide **SQL behaviors**.

# The Evolution of Data Processing
Excel and primitive ways of thinking about data is where a lot of the data processing industry started, especially in the distributed space. We started off as distributed engineers, not really thinking about data in the context of warehousing but thinking about it as an engineering challenge:
- How do we multi-process data?
- How do we split it up?
- How do we process huge amounts of logs in a reasonable amount of time?

The very earliest iterations were built on top of file systems like **HDFS**. Early papers that came out were the foundation of how we think about data processing today. After that, we got **MapReduce** and concepts for working with that data. However, guarantees and behaviors from the early days of SQL and data warehousing were abandoned to tackle the engineering challenge of distributed processing.

# Netflix's Data Challenges
At Netflix, we started early. Netflix was one of the first **large-scale data warehouses in the cloud**. We were taking technologies that existed in the data center (Hadoop, HDFS) and trying to use those in the cloud, running into many challenges along the way. Some were inherent to those technologies, others were driven by cloud migration.

Three main areas we as a platform team were trying to address (ultimately resulting in **Iceberg** being the solution):
1. **Correctness issues** - Being able to atomically mutate a dataset. Many cases where you would get incremental changes or partial results, or challenges with S3 and eventual consistency that caused problems for accurately interrogating data.
2. **Cost and performance** - Operating at very large scale and processing huge amounts of data was difficult. The cloud exacerbated this problem because operations take longer than in HDFS. A list operation in S3 (which can be inconsistent) can take 100+ milliseconds, and when working with thousands to millions of partitions and directories, that causes real problems.
3. **Data engineering productivity** - All these problems with correctness, scalability, and underlying representation surfaced to data engineers. Every analyst and data engineer had to learn about and deal with these issues as part of their practice, when in reality they just wanted things to behave the way SQL systems behave.

# What is a Table Format?
Iceberg is a **table format**, which is confusing to many people because table formats share conventions with file formats:
- They have schemas
- They have columns
- They have data

**The difference between a Parquet file and an Iceberg table:** A file has some of those conventions, but when you have multiple files sitting next to each other, are they part of the same dataset? If they share the same schema and everything looks the same, you can process them together. However, with many files, if schemas change over time or they have different layouts and locations, what constitutes the dataset?

**What Iceberg does:** Iceberg tracks all this information and provides a way to consistently interrogate your data, know what is part of the dataset and what is not, and then how it changes over time.

# Comparing Iceberg to Legacy Formats
A good way to compare this is against **Legacy formats like Hive tables**. How do those things differ?

**Hive** was really built on top of file systems and directories. They were using lower-level conventions of early distributed processing in order to achieve SQL-like behaviors. Those behaviors came over time, but they still used those fundamental conventions. A good example is: **Hive tracks directories** - whatever files happen to be in those directories are part of your dataset. But changing those things means dropping files into directories, and you can't do that atomically across many different files. You also have to list those directories to figure out what is part of that dataset. There are scale and performance challenges because of these listing operations, and in the cloud, those operations get much slower.

On the other hand, **Iceberg** takes a very different approach:
- Doesn't rely on a file system
- Everything is **immutable**
- Fully **traceable**
- Uses a **tree of metadata** that's tracked
- Changes to the table can be fully ACID-compliant because you're really changing from one tree to another, taking the minimal set of changes needed.
## Real-world Problems with Traditional Approaches
Let's take a couple examples and run through these quickly to see what these problems actually look like, because they manifest as issues that users run into as well.

A good example is listing of directories. If somebody is coming to a dataset that was curated by somebody using Hive, if they were a good data engineer, they laid out their table with a timestamp for when the event actually happened. But they also need to partition the data for better performance, typically with day and hour partitioning:
![](Pasted%20image%2020250401215715.png)

An analyst who's familiar with SQL will sit down and just say, "I'm going to query for the time range that I care about," and maybe include some timezone information. But this is actually quite bad - it's a **full table scan**. This analyst forgot to include the partitioning information along with the actual time they want to use.

The problem is that the time for the actual event and the time for the partitioning can be out of sync. You can run into problems where you have to create a more complicated query to get the range of data you want, sometimes requiring broader ranges especially if those times don't align:
![](Pasted%20image%2020250401215812.png)

There's even a problem when they try to account for this with an OR condition, which causes another problem and results in another full table scan. These are the kind of problems where you're exposing file system behaviors all the way up to the user, and now they have to deal with it as part of their analytics.

You can do this correctly by coming up with complicated statements, and this is about as simple as it can be:
![](Pasted%20image%2020250401215905.png)

If you're working with multiple tables and datasets, you have to construct these kinds of queries to get the accurate information you want.

**Iceberg's solution:** Iceberg takes a very different approach. We don't require that you use partitioning as a separate concept from things like timestamp. You can do identity partitioning, but you can also do transforms. In Iceberg, if you lay out your data correctly and use proper partitioning, you can just do time-based partitioning - something any analyst or SQL person would expect and get the behavior they want.

Another thing we do is optimize the way we lay out data for object stores - what we call the **object store layout**. Hive, having paths that constitute different segments of data, doesn't work well in S3 - you get hot spots, 503 errors, and with small files and other problems, these things compound to get really poor performance.

Iceberg decouples the way you lay out data from the actual logical data itself. This allows us to shard very finely across different partitions, which works very well for S3:
![](Pasted%20image%2020250401220628.png)

If somebody's doing data engineering, all they need to do is partition by timestamp, and Iceberg takes care of things like timezone offsets that the engine can leverage to automatically filter to the correct timestamp range:
![](Pasted%20image%2020250401220701.png)

# Restoring SQL Fundamentals
A lot of what I'm talking about with Iceberg is really just **restoring SQL fundamentals** that have existed since the 1980s when the first SQL standards were released. We're bringing back those conventions now that we've solved distributed processing challenges and making sure people can trust their data.

A good example of what hurts many engineers working in systems like Hive is that many SQL behaviors are not preserved. Is renaming a column in a dataset the same as performing a drop column and add column? No! Data should be preserved! But in many systems that work on files (CSV, Parquet), they have different conventions for operations like this. It can be surprising to anybody using the system because they need to know how the engine behaves and how the file format interacts with the engine. Some project by position, some by name. If you're trying to get multiple engines working together with different conventions, you can never get a system that works across all of them. If somebody makes a change, it may look correct in one engine, but you only find out it's wrong when using a different one - often appearing just as null values.

Another example is the **zombie data problem**. Dropping a column and adding a column with the same name - in a traditional relational database, dropping something should discard those values, even if you add the same named thing back. But in the Hive world, this would often resurrect data because it has the same name in the Parquet file or the same position in a CSV file. You get weird situations where months later, someone makes a change to the dataset and resurrects data that shouldn't be there.

Iceberg addresses these challenges, with the goal of making it a standard to restore SQL behavior.
# Advantages of Using Iceberg
What are the advantages of using Iceberg beyond correcting these problems?
- **Expressive SQL** - Can be used for row-level operations because of the guarantees provided
- Allows engines to build complicated logic on fundamentals and provide guarantees about table state transitions
- **New features** like time travel and rollback - important for data engineers and analysts to compare table states or see incremental changes
- **Better engineering patterns:**
    - Audit data
    - Branching and tagging
    - Experiment without impacting production datasets

**Declarative data engineering:** You tell or describe in the dataset the ideal state, and systems working with it can try to achieve that state. This allows things to come in behind the scenes and make changes to the table as maintenance functions to get closer to the desired state, without requiring everyone interacting with the table to always adhere to those things.

There are many more advantages in terms of performance, flexible strategies for updates, background optimization services - all enabled by the core capabilities of the table format.

# The "Happy Accident"
We call this a **happy accident** because when we started, we didn't really understand what problem we were solving. We were initially just trying to get Spark, Trino, and Flink to all work together on the same dataset.

What that ended up unlocking, which is apparent now, is a **universal analytic storage representation**. Once a few different systems can speak the same language, pretty much everything can. This opens up to any sort of batch processing systems that know how to read and operate with Iceberg.

Now we're seeing a major shift with different vendors supporting it - different technologies from graph databases to streaming systems, all interacting with Iceberg. Once they have a common language, they can all interact with data together.

This solves a much larger problem in the ecosystem: **how to share storage**. Global access to object stores means we now have almost universal storage, but we still need to speak the same language to work with it.

## The Quiet Revolution

We call this a **quiet revolution** because for the longest time, every database really wants to hold your data and do your compute. The joke we have about databases is "nobody wants to give up the data layer." But interoperability is so important - you want to own your data and bring compute to it.

There are still companies that are "drawing 25" and saying "it's best if you store it in our format," but they'll be pushed by the market because everything else is starting to work together at a rapid pace.

**Storage neutrality** is incredibly important. You want to be able to access your data where it lives and not hire data engineers to move data around and restructure it into different systems.

This creates **modularity** where you can build infrastructure using many components that serve your needs. You may have graph use cases, Python use cases for lightweight versions of smaller datasets, but you may also need large ETL tools like Spark and Trino. A growing number of vendors support Iceberg, so you can incorporate specialized capabilities into your ecosystem.

You control your data and build on **open standards** - this is the core philosophy behind Iceberg.

## Predictions for the Future

For a long time, we've talked about separation of compute and storage, but this is really the **unbundling of storage from compute**:

- Early days: Cloud storage + one compute engine
- Now: Cloud storage + any compute

This allows you to construct your platform, bring the tools you need to solve your problems, and unbundle the entire ecosystem. It also lets you focus on specialized use cases while working with centralized data storage.

## Tabular's Role

If you put this picture together, here's where Tabular sits in the ecosystem:

- You have your actual storage and formats
- You need support: maintenance, optimization, security
- Tabular provides these as a SaaS-based product
- You can choose what technologies you want on top