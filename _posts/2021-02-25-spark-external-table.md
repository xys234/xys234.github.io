---
layout: post
title: Spark External Table
tags: [spark, data-engineering]
comments: true
---

This post discusses a few lessons I learned while working with external tables in Spark. 

## What is an External Table?
An __external table__ is created when you define a table from files on disk. It is also called __unmanaged table__. Spark only manages the metadata about the external table. On the other hand, if you use `saveAsTable` on a `DataFrame`, you are creating a __managed table__ for which spark will track all relevant information including both data and metadata. The actual data is stored to the location specified by the configuration `spark.sql.warehouse.dir`. The important difference between the two kinds of tables lies with the deletion behavior. When you delete a managed table, the underlying data is also deleted. But for an unmanaged table, only the metadata is deleted while the actual data is intact. 

## When to Use External Table?

Sometimes, there is an external data ingestion process that writes data to a cloud storage (e.g., Amazon s3 or Azure ADLS) according to some predefined folder structure, i,e, partition scheme. It may not be efficient to completely copy the data to a different location by creating a managed table while the underlying data might change. An external table is preferred in this situation and it essentially is a link to the actual data and allows partition pruning in spark SQL at the same time. Therefore, a developer does not need to load the data into a `DataFrame` by using specific paths but uses SQL `where` clause directly on the table. 

## Create an External Table

Suppose we have a small dataset created as follows. 

```python
import pandas as pd

pdf_student = pd.DataFrame({
  'name': ['a', 'b', 'c'],
  'sid': [1,2,3],
  'score': [82, 93, 77]
})

df_student = spark.createDataFrame(pdf_student)

_ = (
  df_student
  .coalesce(1)
  .write
  .format('parquet')
  .partitionBy('sid')
  .mode('overwrite')
  .save('/path/to/data/student')
)

```

We can create an external table pointing to this dataset. 

```python

_ = spark.catalog.createTable(
  'dev.student',
  path='/path/to/data/student', 
  source='parquet',
)

```

If we look at the metadata of table `dev.student`, we can see that spark does partition discovery and specifies `sid` as the partition column as expected. 

```python
spark.sql("DESCRIBE TABLE EXTENDED dev.student;")
```

Even though the partition structure is created after create the external table, the actual partitions are not recovered. We can run the either of the following 2 commands to let spark recursively read the data and finds all the available partitions. 

```python
spark.sql("MSCK REPAIR TABLE dev.student;")      
spark.sql("ALTER TABLE dev.student RECOVER PARTITIONS;")
```

__CAUTION__: Both commands list all the files and subfolders under the root path specified in the `createTable` call. Therefore, it can take some time to run if there are a large number of partitions. In addition, it is important that there is enough driver memory since all the metadata are collected to the driver node. 

## Update an External Table with New Data

This is where things get tricky. 

If we need to write new data to the source location of the table, there are several options. 

First, there is `mode='append'`. This option will __append the data to existing partitions__ and add new partition if necessary.  


```python
pdf_student2 = pd.DataFrame({
  'name': ['b', 'd', 'e'],
  'sid': [2,4,5],
  'score': [63, 72, 100]
})
df_student2 = spark.createDataFrame(pdf_student2)

_ = (
  df_student2
  .coalesce(1)
  .write
  .format('parquet')
  .partitionBy('sid')
  .mode('append')
  .save('path/to/data/student')
)

```

The data in `student2` is appended to the existing data. 

<style scoped>
  .table-result-container {
    max-height: 300px;
    overflow: auto;
  }
  table, th, td {
    border: 1px solid black;
    border-collapse: collapse;
  }
  th, td {
    padding: 5px;
  }
  th {
    text-align: left;
  }
</style><div class='table-result-container'><table class='table-result'><thead style='background-color: grey'><tr><th>name</th><th>score</th><th>sid</th></tr></thead><tbody><tr><td>a</td><td>82</td><td>1</td></tr><tr><td>b</td><td>63</td><td>2</td></tr><tr><td>b</td><td>93</td><td>2</td></tr><tr><td>c</td><td>77</td><td>3</td></tr><tr><td>d</td><td>72</td><td>4</td></tr><tr><td>e</td><td>100</td><td>5</td></tr></tbody></table></div>  

;  

We can see that another student with `sid = 2` is appended.  


Second option is `mode='overwrite'` with the config `spark.conf.get("spark.sql.sources.partitionOverwriteMode") = STATIC`. This config is set by default for a spark session. This would overwrite the entire file.   

```python
_ = (
  df_student2
  .coalesce(1)
  .write
  .format('parquet')
  .partitionBy('sid')
  .mode('overwrite')
  .save('path/to/data/student')

)
```

<style scoped>
  .table-result-container {
    max-height: 300px;
    overflow: auto;
  }
  table, th, td {
    border: 1px solid black;
    border-collapse: collapse;
  }
  th, td {
    padding: 5px;
  }
  th {
    text-align: left;
  }
</style><div class='table-result-container'><table class='table-result'><thead style='background-color: grey'><tr><th>name</th><th>score</th><th>sid</th></tr></thead><tbody><tr><td>b</td><td>63</td><td>2</td></tr><tr><td>d</td><td>72</td><td>4</td></tr><tr><td>e</td><td>100</td><td>5</td></tr></tbody></table></div>
;    

Only data in the new `df_student2` is present. The old data is completely overwritten even for partitions that are not present in `df_student2`.


The third option is `mode='overwrite'` with `spark.conf.get("spark.sql.sources.partitionOverwriteMode") = DYNAMIC`. This would overwrite __existing partition__ and add new partitions.

```python
_ = (
  df_student2
  .coalesce(1)
  .write
  .format('parquet')
  .partitionBy('sid')
  .mode('overwrite')
  .save('path/to/data/student')
)
```

<style scoped>
  .table-result-container {
    max-height: 300px;
    overflow: auto;
  }
  table, th, td {
    border: 1px solid black;
    border-collapse: collapse;
  }
  th, td {
    padding: 5px;
  }
  th {
    text-align: left;
  }
</style><div class='table-result-container'><table class='table-result'><thead style='background-color: grey'><tr><th>name</th><th>score</th><th>sid</th></tr></thead><tbody><tr><td>a</td><td>82</td><td>1</td></tr><tr><td>b</td><td>63</td><td>2</td></tr><tr><td>c</td><td>77</td><td>3</td></tr><tr><td>d</td><td>72</td><td>4</td></tr><tr><td>e</td><td>100</td><td>5</td></tr></tbody></table></div>
;  


We can also add new partitions explicitly. This option is better suited for new data.


```python
pdf_student3 = pd.DataFrame({
  'name': ['x', 'y'],
  'sid': [6,7],
  'score': [88, 99]
})

df_student3 = spark.createDataFrame(pdf_student3)

_ = (
  df_student3
  .coalesce(1)
  .write
  .format('parquet')
  .partitionBy('sid')
  .mode('append')
  .save('path/to/data/student')

)
```

```sql
ALTER TABLE dev.student ADD PARTITION (sid='6') location 'path/to/data/student/sid=6';
ALTER TABLE dev.student ADD PARTITION (sid='7') location 'path/to/data/student/sid=7';
```


### Another syntax with `InsertInto` 

Similar behavior can be realized by using `insertInto` with a spark table with the config change

```

insertInto(tableName, overwrite=None)

Inserts the content of the DataFrame to the specified table.
It requires that the schema of the DataFrame is the same as the schema of the table.
Optionally overwriting any existing data.

```

__CAUTION__: `InsertInto` inserts data by column position. If 2nd column is `score`, it would be treated as `sid` similar to the difference between `union` vs `unionByName`. Therefore, the columnn ordering needs to be carefully maintained.
