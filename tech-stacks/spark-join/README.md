# Spark Joins
Apache Spark is an open-source distributed computing system designed for processing and analyzing large-scale data sets. It provides an interface for programming clusters with implicit data parallelism and fault tolerance.

When talking about processing and analyzing data sets, **JOIN** is an evitable term since the introduction of relationship data modeling.

We will not be talking about the types of joins here, like left/right/inner/outer. You should have grasped these concepts and there are plenty of studying materials online. Instead, we are talking about join strategies - which are the underlying operations closely related to the actual performance. 

Understanding this is not a must-have if you just want to write workable Spark codes, but it is essential if you want to go to the next level.

## Broadcast Join
Spark program runs on a multi-node distributed cluster. Let's say a worker node wants to process some data, we need to *send* that data to the worker node (this process is called **shuffling** and it's an intensive network I/O operation). Now, imagine you have a small metadata table that is used repeatedly to join with other tables. We are wasting a lot of network I/O resources if we send that data every time the node needs it. The solution is **Broadcast Join**.

Spark Broadcast Join is a join strategy used in Apache Spark to optimize joining datasets where one dataset is small enough to fit in memory across all worker nodes. This strategy leverages the ability to broadcast the smaller dataset to all worker nodes, eliminating the need for data shuffling.

Here's how the Spark Broadcast Join works:

- Identify the smaller dataset: The first step is to identify which dataset is smaller and can be broadcasted. Typically, the smaller dataset is determined based on its size in memory compared to the available memory on the worker nodes.

- Broadcast the smaller dataset: The smaller dataset is broadcasted to all worker nodes. This means that a copy of the entire dataset is sent to each worker node. Spark optimizes the broadcast by compressing the data and using efficient data transfer mechanisms.

- Join locally: Once the smaller dataset is broadcasted, each worker node can perform the join operation locally. The larger dataset remains partitioned across the worker nodes, and the join is performed by matching the records with the broadcasted dataset.

The benefits of using Spark Broadcast Join are:

- Reduced data shuffling: By broadcasting the smaller dataset, there is no need to shuffle the data across the network, which significantly reduces the network traffic and improves the join performance.

- Memory efficiency: Since the smaller dataset is replicated across all worker nodes, it can be accessed directly from memory without any additional disk I/O, resulting in faster access times.

- Improved performance for small datasets: Broadcast Join is especially effective when the smaller dataset can fit comfortably in memory across all worker nodes. It avoids the overhead of partitioning and shuffling the data, resulting in faster join operations.

However, it's important to note that Spark Broadcast Join is suitable only for small datasets. If the smaller dataset is too large to fit in memory on the worker nodes, using this strategy can lead to out-of-memory errors and degraded performance. In such cases, other join strategies like Shuffle Hash Join or Sort Merge Join should be considered.

In Spark, you can specify the use of Broadcast Join explicitly by calling the `broadcast()` method on the smaller dataset before performing the join operation. Spark's query optimizer will then determine whether to use the Broadcast Join strategy based on the dataset sizes and available resources.

The property `spark.sql.autoBroadcastJoinThreshold` can be configured to set the Maximum size in bytes for a dataframe to be broadcasted.
Here, `spark.sql.autoBroadcastJoinThreshold=-1` will disable the broadcast Join whereas default `spark.sql.autoBroadcastJoinThreshold=10485760`, i.e `10MB`.

## Shuffle Hash Join

When the table is relatively large, the use of broadcast may cause driver- and executor-side memory issues. In this case, the Shuffle Hash Join will be used. It is an expensive join as it involves both shuffling and hashing. Also, it requires memory and computation for maintaining a hash table.

This is the default join strategy in Spark. It involves shuffling the data across the network to ensure that the data with the same join key is co-located on the same worker node. It requires a full shuffle of the data, which can be expensive, but it performs well when the data is evenly distributed and fits in memory.

Here's how Shuffle Hash Join works:

- Partitioning the datasets: In Shuffle Hash Join, both datasets involved in the join operation are partitioned based on the join keys. Each partition consists of records with the same join key.

- Shuffling the data: After partitioning, Spark performs a shuffle operation, which involves redistributing the data across the worker nodes based on the join keys. This ensures that records with the same join key are collocated on the same worker node.

- Joining the data: Once the data is shuffled and co-located on the worker nodes, Spark performs the join operation by matching the records with the same join key. This can be done efficiently as the required data is present on the same worker node, reducing the data movement across the network.

The advantages of Shuffle Hash Join include:

- Scalability: Shuffle Hash Join can handle large datasets as it partitions the data across multiple worker nodes, allowing parallel processing.

- Flexibility: Shuffle Hash Join works well with various data distributions, including skewed data, where some join keys have a significantly higher number of records than others.

- Fault tolerance: Spark's RDDs (Resilient Distributed Datasets) provide fault tolerance by keeping track of the lineage information. In case of failures, the lost or damaged partitions can be recomputed based on the lineage information.

However, Shuffle Hash Join also has some considerations:

- Data shuffling: The process of shuffling the data can be costly in terms of network overhead and disk I/O, especially for large datasets. It requires moving the data across the network, which can impact the performance.

- Memory utilization: Shuffle Hash Join requires sufficient memory to store the shuffled partitions on each worker node. If the data does not fit in memory, it may result in disk spills and degrade the performance.

## Sort-Merge Join
This strategy is used when both datasets are large and cannot fit in memory. It involves sorting both datasets based on the join keys and then merging them based on the sorted order. It requires reading and writing the data multiple times, which can be expensive, but it performs well when the data is skewed or the join keys have a high cardinality.

Here's how Sort-Merge Join works:

- Partitioning and sorting: Both datasets involved in the join operation are partitioned and sorted based on the join keys. Spark ensures that the data with the same join key is co-located within the same partition.

- Merging the sorted data: After the datasets are sorted, Spark performs a merge operation, which combines the sorted partitions of both datasets based on the join keys. This step is similar to merging two sorted lists, where the join keys are used to determine the order of merging.

- Joining the data: Once the sorted data is merged, Spark performs the join operation by matching the records with the same join key. As the data is sorted, the join can be done efficiently by comparing the join keys in a sequential manner.

The advantages of Sort-Merge Join include:

- Efficient for large datasets: Sort-Merge Join works well with large datasets that do not fit in memory. It avoids excessive data shuffling and can handle datasets that exceed the available memory by utilizing disk-based sorting and merging techniques.

- Suitable for skewed data: Sort-Merge Join performs well even when the data has skewed distributions or when there is a high cardinality of join keys. The sorted order allows efficient matching of records with the same join key.

However, Sort-Merge Join also has some considerations:

- Disk I/O: Sort-Merge Join involves reading and writing the data multiple times, which can result in increased disk I/O operations. This can be a potential bottleneck when dealing with large datasets, and it's important to ensure sufficient disk throughput.

- Extra sorting step: Sort-Merge Join requires sorting the datasets before merging them. Sorting can be computationally expensive, especially for large datasets, and it adds additional overhead to the join operation.

## Cartesian Join 
This strategy performs a cross-join between two datasets, creating a Cartesian product of their records. It can be very expensive in terms of computational and storage resources, and it should be used with caution, especially with large datasets, as it can result in a significant increase in the number of records.

Here's how Cartesian Join works:

- Pairing the datasets: Cartesian Join does not require any specific conditions or join keys. It simply pairs each row of the first dataset with every row of the second dataset, resulting in a combination of all records.

- Generating the Cartesian product: Once the datasets are paired, Spark generates a new dataset with all possible combinations of records from both datasets. For example, if the first dataset has m rows and the second dataset has n rows, the resulting Cartesian join will have m x n rows.

The key characteristics of Cartesian Join are:

- Computational and storage overhead: Cartesian Join can result in a significant increase in the number of records, especially if the datasets being joined are large. The resulting dataset can be much larger than the original datasets, leading to increased computational and storage requirements.

- Limited use cases: Cartesian Join is generally not recommended for large datasets due to its computational cost and the potential for generating a vast number of records. It is more suitable for specific use cases where all combinations of records are required, such as generating test data or performing certain types of analysis.

It's important to note that Cartesian Join should be used with caution, especially with large datasets, as it can have a substantial impact on performance and resource utilization. In most cases, other join strategies such as Shuffle Hash Join, Broadcast Join, or Sort-Merge Join are preferred for efficient and targeted data combining.


## Conclusion
In addition to these strategies, Spark also provides optimizations like bucketing and sorting to improve the performance of joins. Bucketing involves partitioning the data based on the join keys, while sorting ensures that the data within each partition is ordered. These techniques can significantly reduce the amount of data shuffling and improve join performance.

When writing Spark code, you can explicitly specify the join strategy using the join method with the appropriate join type and join hint. However, in most cases, Spark's query optimizer will automatically choose the optimal join strategy based on the dataset characteristics and available resources.