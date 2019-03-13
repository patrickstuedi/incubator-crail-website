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
In a traditional "non-disaggregated" Crail deployment, the Crail storage servers are deployed co-located with the compute nodes running the data processing workloads. By contrast, a disaggregated Crail deployment refers to a setup where the Crail storage servers -- or more precisely, the storage resources exposed by the storage servers -- are seperated (via a network) from the compute servers running the data processing workloads. Storage disaggregation may be implemented at the level of an entire data center (by provisioning dedicated compute and storage racks), or at the level of individual racks (by dedicating some nodes in a rack exlusively for storage). 
</p>
</div>

<div style="text-align: justify"> 
<p>
Remember that Crail is a tiered storage system where each storage tier consists of a subset of storage servers (and their storage resources respectively). Crail permits each storage tier (e.g., RDMA/DRAM, NVMf/Flash) to be deployed and configured independently. This means we can decide to disaggregate one storage tier but not another. For instance, it is more natural to disaggregate the flash storage tier than to disaggregate the memory tier. High-density all-flash storage enclosures are commonly available and often provide NVMe over Fabrics (NVMf) connectivity, thus, exposing them as disaggregated NVMe Flash in Crail is straightforward. High-density memory servers on the other hand (e.g., AWS x1e.32xlarge), would be wasted if we were not using the CPU to run memory intensive computations. Exporting the memory of compute servers into Crail, however, may still make sense as it allows any server to operate on remote memory as soon as it runs out of local memory. The figure below illustrates three possible configurations of Crail in a single rack: 
 </p>
</div>

<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/deployment/three_options.svg" width="580"></div>
<br> 
<br>

* Non-disaggrageted (left): each each compute server exports some of its local DRAM and Flash into Crail by running one Crail storage server instance for each storage type.
* Complete disaggregation (middle): the compute servers do not participate in Crail storage. Instead, dedicated storage servers for DRAM and Flash are deployed. The storage servers export their storage resources into Crail by running corresponding Crail storage servers.
* Mixed disaggregation (right): each compute server exports some of its local DRAM into Crail. The Crail storage space is then augmented by disaggregated Flash. 

<div style="text-align: justify"> 
<p>
Remember that a Crail storage server is entirely a control path entity, responsible only for registering storage resources (and corresponding access endpoints) with Crail metadata servers and for monitoring the health of the storage resources. Therefore, a storage server does not necessarily need to run co-located with the storage resource it exports. For instance, one may export an all-flash storage enclosure in Crail by deploying a Crail storage server on one of the compute nodes. 
 </p>
 </div>
 
### Fine-grained Tiering using Storage and Location Classes 

<div style="text-align: justify"> 
<p>
In all of the previously discussed configurations there is a one-to-one mapping between storage media type and storage tier. There are situations, however, where it can be useful to configure multiple storage tiers of a particular media type. For instance, consider a setup where the compute nodes have access to disaggregated flash but are also attached to some amount of local flash. In this case, you may want to priotize the use of local flash over flash in the same rack (local flash of a different compute node) over disaggregated flash in a different rack. And of course you want to also priortize local DRAM over DRAM in the same rack over any flash if DRAM is available. The way this is done in Crail is through storage and location classes. In this case, a reasonable configuration would be to create three storage classes. The first storage class contains of the local DRAM, the second storage class constains all of the local flash, and the third storage class represents disaggregated flash. The figure below illustrates a configuration with three storage classes in a simplified single-rack deployment.
</p> 
</div>  

<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/deployment/storage_class.svg" width="400"></div>
<br> 
<br>

<div style="text-align: justify"> 
<p>
Storage classes can easily be defined in the slaves file as follows (see the <a href="https://incubator-crail.readthedocs.io/en/latest/config.html#storage-tiers">Crail documentation</a> for details):
</p> 
</div>   

```
crail@clustermaster:~$ cat $CRAIL_HOME/conf/slaves
clusternode1 -t org.apache.crail.storage.rdma.RdmaStorageTier -c 0
clusternode2 -t org.apache.crail.storage.rdma.RdmaStorageTier -c 0
clusternode1 -t org.apache.crail.storage.nvmf.NvmfStorageTier -c 1
clusternode2 -t org.apache.crail.storage.nvmf.NvmfStorageTier -c 1
disaggnode -t org.apache.crail.storage.nvmf.NvmfStorageTier -c 2
```   

<div style="text-align: justify"> 
<p>
One can also manually attach a storage server to a particular storage class:
 </p>
 </div>
 
```
crail@clusternode2:~$ $CRAIL_HOME/bin/crail datanode -t org.apache.crail.storage.nvmf.NvmfStorageTier -c 2
```    

<div style="text-align: justify"> 
<p>
Remember that the storage class ID is implicitly ordering the storage tiers. During writes, Crail either allocates blocks from the highest priority tier that has free space, or from a particular tier if explicitly requested. The following timeline shows a set of Crail operations and a possible resource allocation in a 3-tier Crail deployment (<font face="Courier" color="blue">W-A</font> refers to the create and write operation of a file <font face="Courier" color="blue">A</font>, <font face="Courier" color="blue">D-A</font> refers to the deletion of file <font face="Courier" color="blue">A</font>). Note that at time <font face="Courier" color="blue">t10</font> the system runs out of DRAM space across the entire rack, forcing file <font face="Courier" color="blue">C</font> to be partially allocated in local flash. At time <font face="Courier" color="blue">t11</font> the system runs out of tier 1 storage, forcing file <font face="Courier" color="blue">D</font> to be partially allocated in disaggregated flash. Subsequently, a set of delete operations (time <font face="Courier" color="blue">t13</font> and <font face="Courier" color="blue">t14</font>) free up space in tier 0, allowing file <font face="Courier" color="blue">F</font> to be allocated in DRAM. 
</p>
</div>

<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/deployment/timeline.svg" width="300"></div>
<br> 
<br>

<div style="text-align: justify"> 
<p>
If applications want to further prioritize the specific local resource of a machine over any other resource in the same storage class they can do so via the location class parameter when creating an object in Crail. 
</p>
</div>

```
CrailLocationClass local = fs.getLocationClass();
CrailFile file = fs.create("/tmp.dat", CrailNodeType.DATAFILE, CrailStorageClass.DEFAULT, local).get().asFile();
``` 
<div style="text-align: justify"> 
<p>
In this case, Crail would first try to allocate storage blocks local the client machine from the highest priority storage tier, once the local resources run out it would allocate remote resource following the priority list of the storage tiers. 
 </p>
</div> 

### Resource Provisioning

<div style="text-align: justify"> 
<p>
During the deployment of Crail, one has to decide on the storage capacity of each individual storage tier or storage class, which is a non-trivial task. One approach is to provision sufficient capacity in the highest performing storage class to satisfy the demands during normal operation, and use the local and the disaggregated flash tiers to absorb the high demands during peak times. 
</p>
</div> 

<br>
<div style="text-align:center"><img src ="http://127.0.0.1:4000/img/blog/deployment/resource_provisioning.svg" width="400"></div>
<br> 
<br>

<div style="text-align: justify"> 
<p>
Ideally, we would want individual storage tiers to be elastic in a way that storage capacities can be adjutsed dynamically (and automatically) based on the load. Currently, Crail does not provide elastic storage tiers. In a <a href="https://www.usenix.org/system/files/osdi18-klimovic.pdf">recent research</a> work, however, we have been exploring how to build elastic storage in the context of serverless computing and in the future we might integrate some of these ideas into Crail as well. 
</p>
</div>  
 
