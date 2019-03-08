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

### Practical Configurations

<div style="text-align: justify"> 
<p>
A disaggregated Crail deployment refers to a deployment where the Crail datanodes -- or more precisely, the storage resources exposed by the datanodes -- are seperated from the servers running the data processing workloads and data is accessed over the network. 
</p>
</div>
 
<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/deployment/three_options.svg" width="580"></div>
<br> 
<br>
 

