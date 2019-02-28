---
layout: post
title: "Shuffle Disaggregation using RDMA accessible remote DRAM and NVMe Flash"
author: Patrick Stuedi
category: blog
comments: true
---

<div style="text-align: justify"> 
<p>
One of the goals of Crail has always been to enable efficient storage disaggregation for distributed data processing workloads. Separating storage from compute resources in a cluster is known to have several interesting benefits. For instance, it allows storage resources to scale independently from compute resources, or to run storage systems on specialized hardware (e.g., weak CPU but fast networks) for better performance and reduced cost. Storage disaggregation also simplifies system maintenenance as one can uprade compute and storage resources at different cycles. 
</p>
<p>
Today, data processing applications running in the cloud may implicitly use disaggregated storage through cloud storage services like S3. For instance, it is not uncommon for mapreduce workloads in the cloud to use S3 instead of HDFS for storing input and output data. While Crail can offer high-performance disaggregated storage for input/output data as well, in this blog we specifically look at how to use Crail for efficient disaggregation of shuffle data. 
</p>
 
<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/disaggregation/spark_disagg.svg" width="580"></div>
<br> 
<br>
 
<p>
Generally, the arguments for disaggregation hold for any type of data including input, output and shuffle data. One aspect that makes disaggregation of shuffle data particularly interesting is that it gives compute nodes access to ''infinitely'' large pools or storage resources, whereas in traditional ''non-disaggregated'' deployments a compute node is bound by its local resources.
</p>

<p>
Recently, there has been an increased interest in disaggregating shuffle data. For instance, <a href="https://dl.acm.org/citation.cfm?id=3190534">Riffle</a> -- a research effort driven by Facebook -- is a shuffle implementation for Spark that is capable of operating on disaggregated storage resources. Facebook's disaggregated shuffle engine has also been presented at <a href="https://databricks.com/session/sos-optimizing-shuffle-i-o">SparkSummit'18</a>. In this blog, we discuss how Spark shuffle disaggregation is done in Crail using RDMA accessible remote DRAM and NVMe flash. We occasionally also point out certain aspects where our approach differs from Riffle and Facebook's disaggregated shuffle manager. Some of the material shown in this blog has also been presented in previous talks on Crail, namely at <a href="https://databricks.com/session/running-apache-spark-on-a-high-performance-cluster-using-rdma-and-nvme-flash">SparkSummit'17</a> and <a href="https://databricks.com/session/serverless-machine-learning-on-modern-hardware-using-apache-spark">SparkSummit'18</a>. 
</p>
</div>

### Overview

<div style="text-align: justify"> 
<p>
In a traditional shuffle operation, data is exchanged between map and reduce task using direct communication. For instance, in a typical Spark deployment map tasks running on worker machines write data to a series of local files  -- one per task and partition -- and reduce tasks later on connect to all of the worker machines to fetch the data belonging to their associated partition. By contrast, in a disaggregated shuffle operation, map and reduce tasks exchange data with each other via a remote shared storage system. In the case of Crail, shuffle data is organized hierarchically, meaning, each partition is mapped to a separate directory (the directory is actually a ''MultiFile" as we will see later). Map tasks dispatch data tuples to files in different partitions based on a key, and reduce tasks eventually read all the data in a given partition or directory (each reduce task is associated with exactly one partition). 
</p>
</div>

<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/disaggregation/overview.svg" width="350"></div>
<br>

### Challenge: Large Number of Small Files


<div style="text-align: justify"> 
<p>
One of the challenges with shuffle implementations in general is the large number of objects they have to deal with. The number and size of shuffle files depend on the workload, but also on the configuration, in particular on the number of map and reduce tasks in a stage of a job. The number of tasks in a job is often indirectly controlled through the partition size specifying the amount of data each task is operating on. Finding the optimal partition size for a job is difficult and often requires manual tuning. What you want is a partition size small enough to generate enough tasks to exploit all the parallelism available in the cluster. Also, a large number of small task helps to mitigate stragglers, a major issue for distributed data processing frameworks. 
</p>

<p>
Unfortunately, a small partition size often leads to a large number of small shuffle files. As an illustration, we
performed a simple experiment where we measured the size distribution of Spark shuffle (and broadcast) files of individual tasks when executing (a) PageRank on the Twitter graph, and (b) SQL queries on a TPC-DS dataset. As shown in the figure below, the range of shuffle data is large, ranging from a few bytes (for machine learning) to a few GBs (for TPC-DS) per compute task.
</p>
</div>

<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/disaggregation/cdf-plot.svg" width="480"></div>
<br>

<div style="text-align: justify"> 
<p>
From an I/O performance perspective, writing and reading large numbers of small files is much more challenging than, let's say, dealing with a small number of large files. This is true in a 'non-disaggregated' shuffle operation, but even more so in a disaggregated shuffle operation where are I/O requests include both networking and storage.  
</p>
</div> 

### Per-core Aggregation and Parallel Fetching

<div style="text-align: justify"> 
<p>
To mitigate the overheads of writing and reading large numbers of small data sets, the disaggregated Crail shuffler implements two simple optimizations. First, subsequent map tasks running on the same core append shuffle data to a per-core set of files. The number of files in a per-core set correspondings to the number of partitions. Logically, the files in a per core-set are actually part of different directories (one directory per partition as discussed before). For instance, the blue map tasks running on the first core all append their data to the same set of files (marked blue) distributed over the different partitions, same for the light blue and the white tasks. Consequently, the number of files in the system depends on the number of cores and the number of partitions, but not on the number of tasks. 
</p>
</div> 

<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/disaggregation/optimization_tasks.svg" width="380"></div>
<br>

<div style="text-align: justify"> 
<p>
The second optimization we use in our disaggregated Crail shuffler is efficient parallel reading of entire partitions using Crail MultiFiles. One problem with large number of small files is that it makes efficient parallel reading difficult, mainly because the small file size limits the number of in-flight read operations a reducer can issue on a single file. One may argue that we don't necessarily need to parallelize the reading of a single file. As long as we have large numbers of files we can instead read different files in parallel. The reason this is inefficient is because we want the entire partition to be available at the reducer in a virtually contiguous memory area to simplify sorting. If we were to read multiple files concurrently we either have to temporarily store the receiving data of a file and later copy the data to right place within the contiguous memory area, or if we want to avoid copying data we let the different file readers directly receive the data at the correct offset which leads to random writes cache thrashing at the reducer. Both, copying data and cache thrashing are a concern at a network speed of 100 Gb/s or more. 
</p>
<p>
Crail Multifiles offer zero-copy parallel reading of large numbers of files in a sequential manner. From the prespective of a map task MultiFiles are flat directories consisting of files belonging to different per-core sets. From a reducer perspective, a MultFile look like a large file that can be read sequentially using many in-flight operations. For instance, the following shows how a reduce task during a Crail shuffle operation reads a partition from remote storage. 
</p>
</div>  
```
CrailStore fs = CrailStore.newInstance();
CrailMultiFile multiFile = fs.lookup("/shuffle/partition1").get().asMultiFile();
ByteBuffer buffer = Buffer.allocateDirect(multiFile.size());
int batchSize = 16;
CrailBufferedInputStream stream = multiFile.getMultiStream(batchSize);
while (stream.read(buf) > 0);
``` 
<div style="text-align: justify"> 
<p>
Internally, a MultiFile manages multiple streams to different files and maintains a fixed number of active in-flight operations at all times (except in the end when the stream reaches its end). The number of in-flight operations is controlled via the batch size parameter which is set to 16 in the above example. 
</p>
<p>
 <strong>Comparison with Riffle</strong>: In contrast to Riffle, the Crail shuffler does not merge files at the map stage. There are two reasons for this. First, even though Riffle shows how to overlap merge operations with map tasks, there is always a certain number of merge operations at the end of a map task that cannot be hidden effectively, which delays the map phase. Second, in contrast to Riffle which assumes commodity hardware with low network bandwidth and low metadata throughput (file "open" requests per second), the Crail shuffler is built on top of Crail which offers high bandwidth and high metadata throughput implemented over fast network and storage hardware. As shown in previous posts, Crail supports millions of file metadata operations per second and provides a random read bandwidth for small I/O sizes that is close 100Gb/s. At such a storage performance, any optimization involving the CPU is typically just adding overhead. For instance, the Crail shuffler also does not compress shuffle data as compression rates of common compression libraries (LZ4, Snappy, etc.) are slower than the read/write bandwidth of Crail. 
</p>
<p>
The Crail shuffler also differs from Riffle with regard to how file indexes are managed. Riffle, just like the vanilla Spark shuffler, relies on the Spark driver to map partitions to sets of files and requires reduce task to interact with the driver while reading the partitions. In contrast, the Crail shuffler totally eliminates the Spark driver from the loop by encoding the mapping between partitions and files implicitly using the hierarchical namespace in Crail. 
 </p>
</div>  

### Robustness Against Machine Skew

<div style="text-align: justify"> 
<p>
Shuffle operations, being essentially barriers between compute stages, are highly sensitive to task runtime variations. Runtime variations may be caused by skew in the input data which leads to variations in the partitions size, meaning, different reduce tasks get assigned different amounts of data. Dealing with data skew is tricky and typically requires re-paritioning of the data. Another cause of task runtime variation is machine skew. For example, in a heterogeneous cluster some machines are able to process more map tasks than others, thus, generating more data. In traditional non-disaggregated shuffle operations, machines hogging large amounts of shuffle data after the map phase quickly become the bottleneck (red marked links in the figure below) during the all-to-all network transfer phase. 
</p>
</div>
 
<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/disaggregation/machine_skew.svg" width="380"></div>
<br>

<div style="text-align: justify"> 
<p>
One way to deal with this problem is through weighted fair scheduling of network transfers, as shown <a href="https://dl.acm.org/citation.cfm?id=2018448">here</a>. But doing so requires global knowledge of the amounts of data to transfer and also demands fine grained scheduling. In contrast to a traditional shuffle, the Crail disaggregated shuffle naturally is more reobust against machine skew. Even though some machines generate more shuffle data than others during the map phase, the resulting temporary data residing on disaggregated storage is still evenly distributed across storage servers. This is because shuffle data stored on Crail is chopped up in small blocks that get distributed across the servers. Naturally, the different storage servers are serving roughly equal amounts of data at the beginning of the reduce phase. 
 </p>
</div>  


### Hardware Configuration

The specific cluster configuration used for the experiments in this blog:

* Cluster
  * 8 node OpenPower cluster (for Crail)
  * 2 node X86 cluster (for RAMCloud)
* OpenPower Node configuration
  * CPU: 2x OpenPOWER Power8 10-core @2.9Ghz 
  * DRAM: 512GB DDR4
  * Network: 1x100Gbit/s Ethernet Mellanox ConnectX-4 EN (Ethernet/RoCE)
    * RDMA send/recv latency, ib_send_lat (RTT): 3.1us
    * RDMA read latency, ib_read_lat (RTT): 2.3us
* Software
  * RedHat 7.2 with Linux kernel version 4.10.13
  * Crail 1.0, internal version 2842
  * Alluxio 1.4
  * RAMCloud commit f53202398b4720f20b0cdc42732edf48b928b8d7

