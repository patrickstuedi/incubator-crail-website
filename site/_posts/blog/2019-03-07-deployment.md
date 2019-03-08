---
layout: post
title: "Deployment Options for Tiered Storage Disaggregation"
author: Patrick Stuedi
category: blog
comments: true
---

<div style="text-align: justify"> 
<p>
In the last <a href="http://crail.incubator.apache.org/blog/2019/03/disaggregation.html">blog post</a> we discussed the basic design of the Crail disaggregated shuffler as well as its performance under different configurations for two workloads. In this short follow-up blog we briefly describe the various options in Crail for deploying disaggregated storage. 
</p>
</div>

### Mixing Disaggregated and Co-located Configurations

<div style="text-align: justify"> 
<p>
In a traditional "non-disaggregated" Crail deployment, the Crail datanodes are deployed co-located with the compute nodes running the data processing workloads. By contrast, a disaggregated Crail deployment refers to a setup where the Crail datanodes -- or more precisely, the storage resources exposed by the datanodes -- are seperated (via a network) from the servers running the data processing workloads. Storage disaggregation may be implemented at the level of an entire data center (by provisioning dedicated compute and storage racks), or at the level of individual racks (by dedicating some nodes in a rack exlusively for storage). 
</p>
<p>
Crail permits each storage tier (e.g., RDMA/DRAM, NVMf/Flash) to be deployed and configured indepenently.  
 </p>
<p>
In a traditional "non-disaggregated" Crail deployment, the Crail datanodes are deployed co-located with the compute nodes running the data processing workloads. By contrast, a disaggregated Crail deployment refers to a setup where the Crail datanodes -- or more precisely, the storage resources exposed by the datanodes -- are seperated (via a network) from the servers running the data processing workloads. Storage disaggregation may be implemented at the level of an entire data center (by provisioning dedicated compute and storage racks), or at the level of individual racks (by dedicating some nodes in a rack exlusively for storage). 
 
 
 At the scale of an entire data center, storage disaggregation is typically implemented by provisioning dedicated compute and storage racks. Even though Crail is designed to scale horizontally and can be deployed at the level of an entire data center, its architecture favors deployments with fewer fault domains (e.g., a rack or two). Storage disaggregation may also be an attractive deployment model at the level of a single rack. Rather than attaching local storage resources to each compute node (e.g., SATA attached SSDs), a few nodes in a rack may be dedicated as storage nodes serving the remaining (storage-free) compute nodes. A disaggregated Crail deployment stands in contrast to a co-located deployment where Crail datanodes and compute servers are deployed on the same set of machines (virtual or physical). 
 </p><p> 
 
 </p>
</div>
 
<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/deployment/three_options.svg" width="580"></div>
<br> 
<br>
 

