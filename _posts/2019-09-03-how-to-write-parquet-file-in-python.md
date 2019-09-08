---
layout: post
title:  "How to write to a Parquet file in Python"
author: mikulskibartosz
tags: [ 'Parquet', 'Python' ]
description: "Define a schema, write to a file, partition the data"
featured: false
hidden: false
excerpt: "Define a schema, write to a file, partition the data"
---

As you probably know, Parquet is a columnar storage format, so writing such files is differs a little bit from the usual way of writing data to a file.

In this article, I am going to show you how to **define a Parquet schema** in Python, how to manually **prepare a Parquet table and write it to a file**, how to **convert a Pandas data frame into a Parquet table**, and finally how to **partition the data** by the values in columns of the Parquet table.

# Python package

First, we must **install and import the PyArrow** package. If you are using Conda installation looks like this:

```bash
conda install -c conda-forge pyarrow
```

After that, we have to import PyArrow and its Parquet module. Additionally, I import Pandas and the datetime module because I am going to need them in my examples.

```python
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from datetime import datetime
```

# Defining a schema

**Column types can be automatically inferred**, but for the sake of completeness, **I am going to define the schema**.

Imagine that I want to store emails of newsletter subscribers in a Parquet file. I have the timestamp when the person has subscribed to the newsletter, some user id, and the email. The following schema describes a table which contains all of that information.

```python
subscription_schema = pa.schema([
    ('timestamp', pa.timestamp('ms')),
    ('id', pa.int32()),
    ('email', pa.string())
])
```

# Columns and batches

**A batch is a collection of equal-length arrays. Every array contains data of a single column.** Those columns are aggregated into a batch using the schema we have just defined.

In my example, I will store three values in every column. Here are the values. One more time, note that I don't need to specify the type explicitly.

```python
timestamps = pa.array([
    datetime(2019, 9, 3, 9, 0, 0),
    datetime(2019, 9, 3, 10, 0, 0),
    datetime(2019, 9, 3, 11, 0, 0)
], type = pa.timestamp('ms'))

ids = pa.array([1, 2, 3], type = pa.int32())

emails = pa.array(
    ['first@example.com', 'second@example.com', 'third@example.com'],
    type = pa.string()
)

batch = pa.RecordBatch.from_arrays(
    [timestamps, ids, emails],
    names = subscription_schema
)
```

{% include info.html %}

# Tables

We use a Table to define a single logical dataset. It can consist of **multiple batches**. A table is a structure that can be written to a file using the write_table function.

```python
table = pa.Table.from_batches([batch])
pq.write_table(table, 'test/subscriptions.parquet')
```

When I call the **write_table** function, it will write a single parquet file called subscriptions.parquet into the "test" directory in the current working directory.

# Writing Pandas data frames

We can define the same data as a **Pandas data frame**. It may be easier to do it that way because we can generate the data row by row, which is conceptually more natural for most programmers.

```python
dataframe = pd.DataFrame([
    [datetime(2019, 9, 3, 9, 0, 0), 1, 'first@example.com'],
    [datetime(2019, 9, 3, 10, 0, 0), 1, 'second@example.com'],
    [datetime(2019, 9, 3, 11, 0, 0), 1, 'third@example.com'],
], columns = ['timestamp', 'id', 'email'])
```

When the data frame is ready, we can use the **from_pandas** function to convert the data frame into a table. Such a table can be written into a file in exactly the same way as in the previous example.

```python
table_from_pandas = pa.Table.from_pandas(dataframe)
pq.write_table(table_from_pandas, 'test/subscriptions_pandas.parquet')
```

# Data partitioning

To partition the data, we must first decide which column we want to use for partitioning. **It is possible to partition by multiple columns at the same time.**

I am going to partition the subscriptions by the user id, which makes no sense in real life, but that does not matter in an example code ;)

To write partitioned data, we must call the **write_to_dataset** function. It accepts three arguments. The table to be stored, the directory in which it will create the partitioned directory structure, and the columns containing the partitioning keys.

```python
pq.write_to_dataset(
    table,
    root_path='test/subscriptions_partitioned.parquet',
    partition_cols=['id']
)
```

The code above creates the "subscriptions_partitioned.parquet" directory which contains three subdirectories. Every subdirectory contains partitioned parquet files.

```bash
$cd example_partitioned.parquet/
$ls

id=1	id=2	id=3
```