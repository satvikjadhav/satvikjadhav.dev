---
title: "Apache Spark In-Memory Computations: A Mini Deep Dive"
date: 2024-08-02T15:44:03+05:30
author: Satvik Jadhav
tags: ["Spark"]
categories: ["Apache Spark"]
ShowToc: true
TocOpen: false
Summary: How Apache Spark utilizes in-memory computations to accelerate data transformations, and the benefits of in-memory processing over disk operations and the impact of hardware choices on performance.
---


In this post, I want to break down some my learnings about how Apache Spark handles in-memory computations, focusing on the interaction between CPU and RAM, and how this affects performance when manipulating large datasets. Let’s start with a practical example and dive into the internals of Spark’s processing.

---

## **What Happens Between the CPU and RAM in Spark?**

To understand Spark’s performance, let’s walk through an example of a Spark transformation on a large dataset and see how the CPU and RAM interact.

### **Practical Example**

Assume we have a large Parquet file containing a table of 100 million rows representing customer transactions. We load this file into a Spark DataFrame and apply a transformation, such as filtering transactions that are over $1000.

Here’s what happens step-by-step during this process:

---

### **Step 1: Reading the Parquet File (Initial Disk Read)**

- Spark doesn’t immediately load the entire Parquet file into memory. Instead, it reads the metadata (column types, partitions, etc.) first.
    - Spark first reads the file's metadata, which is typically small and stored at the end of the file.
    - This metadata provides information about the file's structure, including the number and locations of row groups.
    - Based on the metadata, Spark determines how to split the file into partitions.
    - Each partition typically corresponds to one or more row groups in the Parquet file.
    - The number of partitions is influenced by factors like:
        - The size of the file, Spark configuration (e.g., `spark.sql.files.maxPartitionBytes`), The number of row groups in the Parquet file
    - At any given time, only a fraction of the 512GB will be in memory across the cluster.
- When we execute an action (e.g., `count()` or `collect()`), Spark begins to read the actual data into memory across the cluster’s executors.
    - Spark distributes the reading tasks across the available executors and each executor is responsible for reading one or more partitions.

**Data Flow:**

- **Disk → RAM**: Spark reads chunks (partitions) of the Parquet file from disk into memory (RAM). It doesn't load everything at once, only the partitions required for the transformations.

---

### **Step 2: In-Memory Storage and Decompression**

- If we cache the DataFrame, Spark stores the data in memory (RAM) across executors. Since Parquet files are compressed, Spark may decompress the data before storing it in memory for faster access.
- The partitions are distributed across the available RAM in the executors.

**Data Flow:**

- `Spark` creates an RDD (Resilient Distributed Dataset) or DataFrame to represent the data in memory. Spark stores data in a columnar format in RAM, which allows for efficient access to specific columns without reading unnecessary data.
- **Compressed Parquet Data (Disk) → Uncompressed Rows (RAM)**: Once the data is in memory, it’s typically stored as JVM objects, which are much faster for the CPU to access.

---

### **Step 3: CPU Operations on Data in Memory (Transformation)**

Let’s say we do the following: 
```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

val spark = SparkSession.builder.appName("Example").getOrCreate()

val df = spark.read.parquet("customer_data.parquet")
val result = df.filter(col("age") > 30)
               .groupBy("country")
               .agg(avg("account_balance").alias("avg_balance"))

```

This operation happens directly in memory.

**What happens in the CPU?**
- **RAM Access**: CPU fetches age data from RAM column by column. The CPU reads from RAM into its **L1/L2 cache** (small, high-speed memory inside the CPU) and processes them. 
- **Vectorized Processing**: Modern CPUs can perform operations on multiple data points simultaneously using SIMD (Single Instruction, Multiple Data) instructions, which Spark leverages for performance.
- **Transformation Execution**: The CPU compares each age value to 30 and keeps the rows that match, repeating this for each partition.
- **Cache Optimization**: Spark tries to keep frequently accessed data in CPU cache for faster processing.

**Data Flow:**

- **RAM → CPU Cache → RAM**: The data is processed in the CPU cache, and the filtered result is stored back in RAM.
- CPU compares each `age` value to 30.
- CPU writes a bitmap to RAM indicating which rows meet the condition.

---

### **Step 4: Intermediate Results (Shuffling and Aggregation)**

If you perform operations like `groupBy()` or `join()`, Spark might shuffle data between executors. This involves transferring data between memory (RAM) of different executors.

**Data Flow:**

- **RAM → Network → Other Executor’s RAM**: Data is sent between executors, processed by the CPU on each executor, and stored back in memory.

In the case of the above example: `(groupBy("country"))`
- CPU reads the country column and the bitmap from RAM.
- CPU creates a hash table in RAM, with countries as keys.
- For each qualifying row (based on the bitmap), the CPU adds its index to the appropriate country's list in the hash table.

And for Aggregation: `agg(avg("account_balance"))`
- CPU reads the account_balance column and the grouping hash table from RAM.
- For each country group:
    - CPU sums the account balances and counts the entries.
    - CPU divides the sum by the count to get the average.
- CPU writes the results (country and average balance) to a new area in RAM.

These operations occur in parallel across multiple executor JVMs and CPU cores, and each executor processes a subset of the data in its own RAM.

### **Step 5: Final Result (Action Execution)**

After all transformations are completed, Spark writes the final result to disk (e.g., a new Parquet file) when you issue an action like `df.write.parquet("path")`.

**Data Flow:**

- **RAM → Disk**: The final result is written from memory back to storage.

---

## **Why Is RAM Faster Than Disk?**

Now, why does Spark prefer to store data in RAM rather than using disk storage, even with fast disks like NVMe SSDs?

- **Latency**: RAM operates in nanoseconds, while even the fastest NVMe SSDs operate in microseconds. This difference means RAM is around 1000x faster.
- **Throughput**: RAM can handle 25–35 GB/s in modern systems, whereas NVMe SSDs typically max out at around 3–5 GB/s.
- **Random Access**: RAM is optimized for random access, allowing it to retrieve any byte almost instantaneously. Disk storage, even with SSDs, has higher latency when accessing random data blocks.

---

## **In-Memory Computations in Spark**

Spark’s **in-memory computations** refer to its ability to store data in memory and perform transformations directly in RAM without needing to read from or write to disk for each operation. This can drastically improve performance.

- **RAM stores the data**: Spark stores data partitions in memory (RAM) across executors.
- **CPU reads from RAM**: The CPU fetches data from RAM into its cache and performs the required transformations.
- **CPU processes the data**: The CPU executes operations like filtering, mapping, or aggregating.
- **Results are written back to RAM**: The transformed data is stored back in memory, avoiding costly disk I/O.

---

## **Conclusion**

In-memory processing is what gives Spark its incredible speed for large data sets, but understanding the role of the CPU and RAM is essential to maximizing performance. By caching data in RAM and using CPU power efficiently (via techniques like vectorized processing), Spark minimizes the need for slow disk access, allowing large-scale transformations to happen quickly.

That said, if memory is limited and Spark has to spill to disk, using fast storage like NVMe SSDs can reduce the performance hit, but RAM will always be the fastest option.

---

