# 🚀 Databricks Performance Optimization

##  What Databricks Optimization Actually Means

Think of it like this:

> **Databricks Optimization = Making your Spark jobs run faster and cheaper.**

In Databricks, you process **huge datasets** — and performance means everything.  
So optimization = reducing:

-  **Time** (speed up queries)
-  **Cost** (use less compute)
-  **Shuffle/IO** (reduce unnecessary data movement)

---

#  💥 CSV and Parquet in PySpark

##  1 What are CSV and Parquet?

| Format | Meaning | Type |
|---------|----------|------|
| **CSV** | Comma-Separated Values | **Text-based** |
| **Parquet** | Columnar storage format by Apache | **Binary (optimized)** |

Both are file formats used to store tabular data —  
but **CSV** is simple and human-readable, while **Parquet** is optimized for **speed and compression**.

---

## 2 Visual Intuition

Imagine your dataset like this:

| Name | Age | City |
|------|-----|------|
| Karan | 25 | Delhi |
| Neha | 30 | Mumbai |
| Ravi | 28 | Pune |

### **CSV stores row by row:**
```
Name,Age,City
Karan,25,Delhi
Neha,30,Mumbai
Ravi,28,Pune
```

If you only need one column (like `Age`), Spark still has to read the entire file.

### **Parquet stores column by column:**
```
Name → [Karan, Neha, Ravi]
Age  → [25, 30, 28]
City → [Delhi, Mumbai, Pune]
```

So if you query only `Age`, Spark reads **just that column** — making it faster.

---

## 3 Real Example in PySpark

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("file_formats").getOrCreate()

data = [("Karan", 25, "IT"), ("Ravi", 30, "HR"), ("Neha", 28, "Finance")]
columns = ["name", "age", "department"]
df = spark.createDataFrame(data, columns)

# Write to CSV
df.write.mode("overwrite").csv("output_csv")

# Write to Parquet
df.write.mode("overwrite").parquet("output_parquet")

# Read back
csv_df = spark.read.csv("output_csv", header=True, inferSchema=True)
parquet_df = spark.read.parquet("output_parquet")

csv_df.show()
parquet_df.show()
```

###  Output Files
- **CSV Folder:** contains plain `.csv` files (open in Excel/Notepad)
- **Parquet Folder:** contains `.parquet` files (binary format — open via Spark or parquet tools)

---

##  4 Performance Comparison

| Operation | **CSV** | **Parquet** |
|------------|----------|-------------|
| Load 1 GB file | ~25–30 sec | ~3–5 sec |
| Disk space used | ~1 GB | ~200 MB (compressed) |
| Schema detection | Must infer | Stored inside file |
| Filtering (few cols) | Reads entire file | Reads only required columns |
| Repeated queries | Slow | Fast (cached metadata) |

---

 **In short:**  
>  **CSV** is simple, easy, but heavy and slow.  
>  **Parquet** is compressed, optimized, and built for big data.

---

# 💥 5 Simple Areas of Optimization (Beginner-Friendly Breakdown)🚀

### 1️ Caching & Persistence (Memory Optimization)

When you use a DataFrame repeatedly, Spark **re-computes** it every time — which is slow!

 **Solution → Cache it in memory**

```python
df.cache()
df.show()
```

Now Spark remembers it in memory — so next time you use it, it’s instant ⚡

 **Analogy:**  
Think of it like storing your school notes in your bag — instead of re-writing them every time you study!

---

### 2️ Partitioning (Data Organization)

Large data is split into **partitions** — smaller chunks of data.  
If not done smartly, Spark has to **scan everything**, even if you only need one part.

 **Solution → Partition by a key column that you often filter on:**

```python
df.write.partitionBy("country").format("delta").save("/tmp/sales")
```

Then, if you query only `country='India'`,  
Spark reads only that partition — not all countries 

💡 **Analogy:**  
Think of it like keeping your clothes in labeled drawers (shirts, jeans, etc).  
You don’t open every drawer to find a shirt!

---

### 3️ File Size & Data Skew (Balanced Workload)

If one partition is huge and others are tiny, Spark executors get uneven work.  
This leads to **data skew** (some executors idle, some overloaded).

 **Solution → Repartition or coalesce** to balance workload:

```python
df = df.repartition(8, "country")  # or df.coalesce(4)
```

💡 **Analogy:**  
Like dividing homework evenly among friends — no one should be overloaded.

---

### 4️ Delta Lake & Z-Ordering (Query Optimization)

**Delta Lake** is Databricks’ special format for reliable, optimized big data storage.  
It supports **ACID**, **time travel**, and **Z-Ordering** (a data-skipping index).

 **Z-Ordering Example:**

```python
OPTIMIZE sales ZORDER BY (customer_id)
```

This helps Spark skip irrelevant data faster — improving read speed 🚀

 **Analogy:**  
Like sorting your contact list alphabetically so you find names faster.

---

### 5️ Join & Shuffle Optimization (Reducing Expensive Moves)

Shuffles happen when Spark has to **rearrange data between executors** — very costly.  
You’ll see this often in **joins** or **groupBy** operations.

 **Solutions →**
- Use **broadcast joins** for small tables:
  ```python
  from pyspark.sql.functions import broadcast
  df_large.join(broadcast(df_small), "id").show()
  ```
- Avoid unnecessary wide transformations.

💡 **Analogy:**  
Instead of asking everyone in the school for one note, just give each group their own copy!

---

# 💥 Databricks Optimization Deep Dive: Z-Ordering & Join Optimization

##  Z-ORDERING — “Find Data Fast”

 **Imagine:**

You have a sales table with millions of records:

| sale_id | customer_id | country | amount |
|----------|--------------|----------|--------|
| 1 | 101 | India | 2000 |
| 2 | 102 | USA | 3000 |
| 3 | 103 | India | 1500 |
| ... | ... | ... | ... |

And most of your queries look like:

```sql
SELECT * FROM sales WHERE customer_id = 101;
```

 **Without Z-ORDER:**  
Data is randomly stored across Parquet files.  
When you query `customer_id = 101`, Spark must scan every file to find matching rows.  
→ Slow, wasteful I/O.

 **With Z-ORDER:**  
You tell Databricks:

```sql
OPTIMIZE delta.`/mnt/sales` ZORDER BY (customer_id);
```

Now, Delta Lake physically reorganizes the data files so that rows with similar `customer_id` values are stored close together.

Next time you query:

```sql
SELECT * FROM sales WHERE customer_id = 101;
```

→ Spark can skip irrelevant data blocks and jump directly to the right files.  
 **Result = way faster filtering.**

**Analogy:**  
Z-Ordering is like sorting your notebook alphabetically —  
so when you look for “Karan,” you flip straight to the K section instead of reading every page.

---

### Mini Example in PySpark

```python
from delta.tables import DeltaTable
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("ZOrderExample").getOrCreate()

# Read a big Delta table
sales_df = spark.read.format("delta").load("/mnt/sales")

# Optimize it by Z-ordering on 'customer_id'
spark.sql("""
OPTIMIZE delta.`/mnt/sales`
ZORDER BY (customer_id)
""")
```

Now your queries with filters on `customer_id` or even `country` will fly ⚡

---

##  JOIN OPTIMIZATION — “Move Less Data”

 **Imagine:**

You have:
- A huge **sales** table (100 million rows)
- A small **customers** table (100 rows)

You want to join them on `customer_id`.

 **Normal Join (bad way):**
```python
sales.join(customers, "customer_id")
```
What happens?  
Spark **shuffles both tables** across worker nodes to match keys.  
Even the small `customers` table moves around.  
Big network cost = slow joins.

---

 **Optimized Join — Broadcast the Small Table**
```python
from pyspark.sql.functions import broadcast

optimized_join = sales.join(broadcast(customers), "customer_id")
```

What happens now?  
Spark sends the small customers table to every worker.  
Only the big sales data stays in place.  
No shuffle needed → super fast 

---

### Mini Example

```python
# Big table
sales = spark.createDataFrame([
    (1, 101, 2000),
    (2, 102, 3000),
    (3, 103, 1500)
], ["sale_id", "customer_id", "amount"])

# Small table
customers = spark.createDataFrame([
    (101, "Karan"),
    (102, "Ravi"),
    (103, "Neha")
], ["customer_id", "name"])

# Optimized join
from pyspark.sql.functions import broadcast
joined = sales.join(broadcast(customers), "customer_id")
joined.show()
```

 **Output:**

```
+------------+-------+------+ 
|customer_id | name  |amount|
+------------+-------+------+
|101         |Karan  |2000  |
|102         |Ravi   |3000  |
|103         |Neha   |1500  |
+------------+-------+------+
```

---

## Recap

| Optimization | What It Does | Example |
|---------------|---------------|----------|
| **Z-Ordering** | Physically reorganizes data so Spark can skip irrelevant files | `OPTIMIZE sales ZORDER BY (customer_id)` |
| **Broadcast Join** | Sends small table to all workers, avoids shuffle | `sales.join(broadcast(customers), "customer_id")` |

---

### ⚡ Combine Them:
If you **Z-ORDER** your Delta tables and use **smart joins**,  
your queries will run **2–10× faster**, often with **half the cost**

---

## ⚡ Final Thought

> Databricks Optimization isn’t about writing new code — it’s about making your existing code **run smarter**.  
> Faster jobs → Lower cost → Happier engineers 

---

# 💥 Liquid Clustering 
### What is Liquid Clustering (in short)?
Liquid Clustering is a new smart way in Databricks (Delta Lake) to organize big data automatically —
so your queries run faster without needing fixed partitions or manual Z-ordering.

### Think of it like this:

* **Old way**: You decide partitions yourself → region=US, region=India, etc.
If data grows or query pattern changes → messy, slow.

* **New way** (Liquid Clustering):
You just say “Hey Databricks, keep my data organized by customer_id”
→ Databricks automatically clusters and maintains that layout.

### Simple Example

```python
CREATE TABLE sales (
  id INT,
  customer_id STRING,
  amount DOUBLE
)
USING DELTA
CLUSTER BY (customer_id);
```

* Now Databricks:
* Stores similar customer_ids together (automatically).
* Keeps the layout optimized as new data comes in.
* Makes filters like WHERE customer_id='C123' run much faster.

### Why It’s Better — Liquid Clustering vs Old Partitioning

| Feature | Old Partitioning | Liquid Clustering |
|----------|------------------|------------------|
| **Partition Handling** | Manual partitions | Automatic, flexible |
| **Change Flexibility** | Hard to change later | Can change clustering keys anytime |
| **Data Distribution** | Data skew problems | Handles evenly |
| **Performance on Updates** | Slow updates | Auto-optimizes |

#### ⚡ In One Line:
```
Liquid Clustering = Automatic, flexible data organization for faster queries — no more manual partitioning or Z-ordering.
```
---

# PySpark Performance Concepts — Skew, Shuffle, Spill, Serialization & UDF(Code Optimization)

## 💥 Data Skew
**Meaning:** When some partitions have significantly more data than others, causing uneven processing load.

```python
# Example: Most records belong to 'IT' department
df.groupBy("Dept").count().show()
```
-> Problem: The 'IT' partition takes much longer, slowing the job.

**Fixes:**
- Use **salting** (add random key prefixes)
- Repartition using better column
- Enable **Adaptive Query Execution (AQE)** — handles skew automatically in Spark 3+

**Tip Line:** 
```
“Data skew means uneven data distribution across partitions, causing slow or stuck tasks.”
```
---

## 💥 Shuffle
**Meaning:** Spark redistributes data between nodes during wide transformations like `groupBy`, `join`, or `distinct`.

```python
df.groupBy("Dept").agg(sum("Salary"))
```

-> This requires all rows for the same Dept to be together → Spark performs a **shuffle**.

**Cost:** Shuffle = Disk I/O + Network I/O → very expensive.

**Optimizations:**
- Minimize `groupBy` and `join` on huge columns.
- Use **broadcast joins** for small tables.
- Enable **AQE** with `spark.sql.adaptive.enabled=true`.

**Tip:**  
```
 “Shuffle means data movement across the cluster, triggered by wide transformations.”
```
---

## 💥 Spill
**Meaning:** When executors run out of memory, intermediate data is spilled to disk.

```python
df.orderBy("Salary").show()
```

-> Large sorts or joins may trigger spills if memory is insufficient.

**Downside:**  
- Disk I/O is slower than RAM → performance drop.

**Optimizations:**
- Increase executor memory.
- Use **cache() / persist()** smartly.
- Avoid wide transformations on massive data.

**Tip:** 
```
 “Spill occurs when Spark writes data to disk due to insufficient memory.”
```

---

## 💥 Serialization
**Meaning:** Converting data objects to byte format for transfer across cluster nodes.

**Why Needed:** Spark is distributed — data must be serialized for network communication.

**Types:**
- `Java Serializer` (default)
- `Kryo Serializer` (faster and compact)

**Enable Kryo:**
```python
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```

**Tip:**  
```
“Serialization improves Spark’s network efficiency; Kryo is faster and uses less space.”
```
---

## 💥 UDF (User Defined Function)
Custom logic not available in built-in Spark functions.

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType

def categorize(salary):
    return "High" if salary > 60000 else "Low"

category_udf = udf(categorize, StringType())

df.withColumn("Category", category_udf("Salary")).show()
```
 **Output:**
```
+------+--------+---------+
| Name | Salary | Category|
+------+--------+---------+
|Karan |50000.57|Low      |
|Ravi  |60000.35|Low      |
|Neha  |70000.79|High     |
|Arjun |45000.11|Low      |
|Meena |80000.96|High     |
+------+--------+---------+
```

 **Note:** UDFs are slower since they bypass Spark’s Catalyst optimizer — prefer **built-in** or **pandas UDFs**.

---

## Summary Table

| Concept | Meaning | Issue | Fix/Optimization |
|----------|----------|-------|------------------|
| **Skew** | Uneven data across partitions | Slow tasks | Salting / AQE |
| **Shuffle** | Data movement between nodes | High I/O cost | Broadcast join / AQE |
| **Spill** | Data written to disk | Slow performance | Tune memory / persist() |
| **Serialization** | Convert data to bytes | Network delay | Use Kryo Serializer |
| **UDF** | Custom logic | Slower execution | Use built-in / pandas UDF |

---

### Quick Recap
```
 “Skew causes imbalance, Shuffle moves data, Spill slows due to disk, Serialization transfers efficiently, and UDF adds flexibility.”
```

