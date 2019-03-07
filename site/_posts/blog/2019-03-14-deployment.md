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
</div> 
 
<br>
<div style="text-align:center"><img src ="{{ site.base }}/img/blog/disaggregation/spark_disagg.svg" width="580"></div>
<br> 
<br>
 

