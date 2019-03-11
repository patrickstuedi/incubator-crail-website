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
In a traditional "non-disaggregated" Crail deployment, the Crail datanodes are deployed co-located with the compute nodes running the data processing workloads. By contrast, a disaggregated Crail deployment refers to a setup where the Crail storage servers -- or more precisely, the storage resources exposed by the storage servers -- are seperated (via a network) from the compute servers running the data processing workloads. Storage disaggregation may be implemented at the level of an entire data center (by provisioning dedicated compute and storage racks), or at the level of individual racks (by dedicating some nodes in a rack exlusively for storage). 
</p>
<p>
Crail permits each storage tier (e.g., RDMA/DRAM, NVMf/Flash) to be deployed and configured independently. This means we can decide to disaggregate one storage tier but not another. For instance, it is more natural to disaggregate the flash storage tier than to disaggregate the memory tier. High-density all-flash storage enclosures are commonly available and often provide NVMe over Fabrics connectivity, thus, exposing them as disaggregated NVMe Flash in Crail is straightforward. High-density memory servers on the other hand (e.g., AWS x1e.32xlarge), would be wasted if we were not using the CPU to run memory intensive computations. Exporting the memory of compute servers into Crail, however, may still make sense as it allows any server to operate on remote memory as soon as it runs out of local memory. 
 </p>
<p>
The figure below illustrates three possible configurations of Crail in a single rack: 
 </p>
</div>
* Non-disaggrageted (left): each each compute server exports some of its local DRAM and Flash into Crail by running one Crail storage server instance of both types.
* Complete disaggregation: where the compute servers do not participate in Crail storage but all 
* Mixed disaggregation: 
 
<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/deployment/three_options.svg" width="580"></div>
<br> 
<br>
 

