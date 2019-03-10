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
Crail permits each storage tier (e.g., RDMA/DRAM, NVMf/Flash) to be deployed and configured independently. This means we can decide to disaggregate one storage tier but not another. For instance, it is more natural to disaggregate the flash storage tier than to disaggregate the memory tier. High-density all-flash storage enclosures are commonly available and often provide NVMe over Fabrics connectivity, thus, exposing them as disaggregated NVMe Flash in Crail is straightforward. 
 
 
 This is because DRAM is still associated with a CPU -- unless you are running custom hardware such as custom <a href="https://web.eecs.umich.edu/~twenisch/papers/isca09-disaggregate.pdf">memory blades</a> -- and not making use the CPU and its local memory for computation would be waste. 
</p>
</div>
 
<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/deployment/three_options.svg" width="580"></div>
<br> 
<br>
 

