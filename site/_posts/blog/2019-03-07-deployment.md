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
A disaggregated Crail deployment refers to a setup where the Crail datanodes -- or more precisely, the storage resources exposed by the datanodes -- are seperated from the servers running the data processing workloads. At the scale of an entire data center, disaggregation may be implemented by provisioning dedicated compute and storage racks connected via a network. 
 
 
 
 and data is accessed over the network. A disaggregated deployment stands in contrast to a co-located deployment where Crail datanodes and compute servers are deployed on the same set of machines (virtual or physical). 
</p>
<p> 
 
 </p>
</div>
 
<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/deployment/three_options.svg" width="580"></div>
<br> 
<br>
 

