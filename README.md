# pyspark-kmetamodes

## Ensemble-based incremental distributed k-modes/k-metamodes clustering for PySpark

Ensemble-based incremental distributed k-modes clustering for PySpark (Python 3), similar to the algorithm proposed by Visalakshi and Arunprabha in "Ensemble based Distributed K-Modes Clustering" (IJERD, March 2015) to perform K-modes clustering in an ensemble-based way.

In short, k-modes will be performed for each partition in order to identify a set of *modes* (of clusters) for each partition. Next, k-modes will be repeated to identify modes of a set of all modes from all partitions. These modes of modes are called *metamodes* here.

This module uses several different distance functions for k-modes:

1) Hamming distance.
2) Frequency-based dissimilarity proposed by He Z., Deng S., Xu X. in Improving K-Modes Algorithm Considering Frequencies of Attribute Values in Mode.
3) My own (Andrey Sapegin) dissimilarity function, which is used for calculation of metamodes only. This distance function keeps track of and takes into account all frequencies of all unique values of all attributes in the cluster, and NOT only most frequent values that became the attributes of the mode/metamode. This work is planned to be published in the future. Preprint is available at https://arxiv.org/abs/1909.13721.

This package was originally based on the work of `Marissa Saunders <marissa.saunders@thinkbiganalytics.com>` (https://github.com/ThinkBigAnalytics/pyspark-distributed-kmodes). However, due to the fact that the original package contained several major issues leading to incorrect incremental k-modes implementation and seems to be not maintained for several years, it was decided to perform a major refactoring fixing these issues, adding new distance functions (besides the existing hamming distance), etc.

The refactoring work was performed at Hasso Plattner Institute (www.hpi.de)

## Usage

This module has been developed and tested on Spark 2.3 and Python 3.

### Example on how to run k-metamodes clustering on data:

```python
n_modes=36
partitions=10
max_iter=10
psize = 50000
# optionally select a subset of data:
if data.count() > partitions * psize:
	fraction = 50000 * partitions / (data.count() * 1.0)
	data = data.rdd.sample(False,fraction).toDF()

method=IncrementalPartitionedKMetaModes(n_partitions = partitions, partition_size = 50000, n_clusters = n_modes,max_dist_iter = max_iter,local_kmodes_iter = max_iter, similarity = "frequency", metamodessimilarity = "hamming")
    	
cluster_metamodes = method.calculate_metamodes(data)
modes = method.get_modes()
mode_ids = method.get_mode_indexes()
```

* Possible values for similarity are: "hamming" and "frequency"

* Possible values for metamodessimilarity are: "hamming", "frequency" and "meta" ("meta" can only be used if similarity = "frequency")

Now the metamodes can be used, for example, to find the distance from each original data record to all metamodes using one of the existing distance functions:

```python
def distance_to_all(record):
	sum_distance = 0
	for diss in frequency_based_dissim(record, cluster_metamodes):
		sum_distance += diss
	drow = record.asDict()
	drow["distance"] = sum_distance
	return Row(**drow)
data_with_distances = data.repartition(partitions).rdd.map(lambda record: distance_to_all(record))
```

`method.get_modes()` will return you the list of modes (not metamodes!), i.e. all modes of all clusters in all data subsets (that were clustered individually).

`method.get_mode_indexes()` will return you a list with corresponding mode ID (which is globally unique) for each original record.

Thus, `data.withColumn("modeID", method.get_mode_indexes())` should add a column with mode ID to your original data.
