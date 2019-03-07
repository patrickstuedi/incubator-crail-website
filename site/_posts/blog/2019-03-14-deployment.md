---
layout: post
title: "Deployment Modes for Tiered Storage Disaggregation"
author: Patrick Stuedi
category: blog
comments: true
---

<div style="text-align: justify"> 
<p>
One of the goals of Crail has always been to enable efficient storage disaggregation for distributed data processing workloads. Separating storage from compute resources in a cluster is known to have several interesting benefits. For instance, it allows storage resources to scale independently from compute resources, or to run storage systems on specialized hardware (e.g., storage servers with a weak CPU attached to a fast network) for better performance and reduced cost. Storage disaggregation also simplifies system maintenance as one can uprade compute and storage resources at different cycles. 
</p>
<p>
Today, data processing applications running in the cloud may implicitly use disaggregated storage through cloud storage services like S3. For instance, it is not uncommon for map-reduce workloads in the cloud to use S3 instead of HDFS for storing input and output data. While Crail can offer high-performance disaggregated storage for input/output data as well, in this blog we specifically look at how to use Crail for efficient disaggregation of shuffle data. 
</p>
 
<br>
<div style="text-align:center"><img src ="{{ site.base }}/img/blog/disaggregation/spark_disagg.svg" width="580"></div>
<br> 
<br>
 
<p>
Generally, the arguments for disaggregation hold for any type of data including input, output and shuffle data. One aspect that makes disaggregation of shuffle data particularly interesting is that compute nodes generating shuffle data now have access to ''infinitely'' large pools or storage resources, whereas in traditional ''non-disaggregated'' deployments the amount of shuffle data per compute node is bound by local resources.
</p>

<p>
Recently, there has been an increased interest in disaggregating shuffle data. For instance, <a href="https://dl.acm.org/citation.cfm?id=3190534">Riffle</a> -- a research effort driven by Facebook -- is a shuffle implementation for Spark that is capable of operating in a disaggregated environment. Facebook's disaggregated shuffle engine has also been presented at <a href="https://databricks.com/session/sos-optimizing-shuffle-i-o">SparkSummit'18</a>. In this blog, we discuss how Spark shuffle disaggregation is done in Crail using RDMA accessible remote DRAM and NVMe flash. We occasionally also point out certain aspects where our approach differs from Riffle and Facebook's disaggregated shuffle manager. Some of the material shown in this blog has also been presented in previous talks on Crail, namely at <a href="https://databricks.com/session/running-apache-spark-on-a-high-performance-cluster-using-rdma-and-nvme-flash">SparkSummit'17</a> and <a href="https://databricks.com/session/serverless-machine-learning-on-modern-hardware-using-apache-spark">SparkSummit'18</a>. 
</p>
</div>

