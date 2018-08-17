> 对于某一特定的ceph集群， 与之对应的crush map 是对改集群存储设备的物理位置/部署的描述，并有一套相应的规则(rule)用来计算数据的存储位置。 
> CRUSH 的全称是: Cotrolled Replication Under Scalable Hashing. 利用Hash的随机特性， 并有一套相应的 map来控制hash的随机性，达到 可控的数据分布。
> 在crush map 实际存储数据就只是物理的磁盘(HDD, SSD)。CRUSH map 作为整个集群的物理存储设备的描述：整个的存储设备架构实际上一种层级关系,
> 可以想象是树状的结构，叶子节点是能实际存储数据的的device(HDD, SSD) 或者是OSD. 以下是ceph预先定义的层级类型。
1. osd (or device)
2. host
3. chassis
4. rack
5. row
6. pdu: [Power distribution unit](https://en.wikipedia.org/wiki/Power_distribution_unit)
7. pod: [Point of delivery networking](https://en.wikipedia.org/wiki/Point_of_delivery_(networking))
8. room
10. datacenter
11. region
12. root

> 从这些预先定义的类型可以看出，ceph将数据存储设备划分成了不同的 failure domain. ceph 是数据存储系统， 数据的可靠/安全性是其首要的考量。
> 但是有了这样的划分，实际的数据存储位置还需要规则(rule)来指定。可以想象
> 不同的场景对数据的安全性要求肯定不是统一的。 所以根据物理的分层描述和rule结合可以定制相应的数据安全策略。
> 例如 在同样的物理分布下， 有的规则可以要求数据存储2份， 有的可以要求数据存储3份； 有的要求数据分布要求跨 rack, 有的可以要求跨row.


> Get a simple view the CRUSH hierarchy for your cluster
> 
  ```c++
  ceph osd crush tree
  ```
> Rules for cluster
```c++
ceph osd crush rule ls
```
> view the contents of the rules
```c++
ceph osd crush rule dump
```
> view the contents of crush map
```c++
ceph osd crush dump
```
## Data Placement Overview
Ceph stores, replicates and rebalances data objects across a RADOS cluster dynamically. With many different users storing objects in different pools for different purposes on countless OSDs, Ceph operations require some data placement planning. The main data placement planning concepts in Ceph include:

    Pools: Ceph stores data within pools, which are logical groups for storing objects. Pools manage the number of placement groups, the number of replicas, and the CRUSH rule for the pool. To store data in a pool, you must have an authenticated user with permissions for the pool. Ceph can snapshot pools. See Pools for additional details.
    Placement Groups: Ceph maps objects to placement groups (PGs). Placement groups (PGs) are shards or fragments of a logical object pool that place objects as a group into OSDs. Placement groups reduce the amount of per-object metadata when Ceph stores the data in OSDs. A larger number of placement groups (e.g., 100 per OSD) leads to better balancing. See Placement Groups for additional details.
    CRUSH Maps: CRUSH is a big part of what allows Ceph to scale without performance bottlenecks, without limitations to scalability, and without a single point of failure. CRUSH maps provide the physical topology of the cluster to the CRUSH algorithm to determine where the data for an object and its replicas should be stored, and how to do so across failure domains for added data safety among other things. See CRUSH Maps for additional details.

When you initially set up a test cluster, you can use the default values. Once you begin planning for a large Ceph cluster, refer to pools, placement groups and CRUSH for data placement operations.
![object-pg-pool](http://docs.ceph.com/docs/master/_images/ditaa-1fde157d24b63e3b465d96eb6afea22078c85a90.png)

![object-pg-osd](http://docs.ceph.com/docs/master/_images/ditaa-c7fd5a4042a21364a7bef1c09e6b019deb4e4feb.png)


### [Crush map](docs.ceph.com/docs/master/rados/operations/crush-map/)
The CRUSH algorithm determines how to store and retrieve data by computing data storage locations. CRUSH empowers Ceph clients to communicate with OSDs directly rather than through a centralized server or broker. With an algorithmically determined method of storing and retrieving data, Ceph avoids a single point of failure, a performance bottleneck, and a physical limit to its scalability.

CRUSH requires a map of your cluster, and uses the CRUSH map to pseudo-randomly store and retrieve data in OSDs with a uniform distribution of data across the cluster. For a detailed discussion of CRUSH, see CRUSH - Controlled, Scalable, Decentralized Placement of Replicated Data

CRUSH maps contain a list of OSDs, a list of ‘buckets’ for aggregating the devices into physical locations, and a list of rules that tell CRUSH how it should replicate data in a Ceph cluster’s pools. By reflecting the underlying physical organization of the installation, CRUSH can model—and thereby address—potential sources of correlated device failures. Typical sources include physical proximity, a shared power source, and a shared network. By encoding this information into the cluster map, CRUSH placement policies can separate object replicas across different failure domains while still maintaining the desired distribution. For example, to address the possibility of concurrent failures, it may be desirable to ensure that data replicas are on devices using different shelves, racks, power supplies, controllers, and/or physical locations.

When you deploy OSDs they are automatically placed within the CRUSH map under a host node named with the hostname for the host they are running on. This, combined with the default CRUSH failure domain, ensures that replicas or erasure code shards are separated across hosts and a single host failure will not affect availability. For larger clusters, however, administrators should carefully consider their choice of failure domain. Separating replicas across racks, for example, is common for mid- to large-sized clusters.

### [Placement Group](http://docs.ceph.com/docs/master/rados/operations/placement-groups/)

#### Placement Groups Tradeoffs

Data durability and even distribution among all OSDs call for more placement groups but their number should be reduced to the minimum to save CPU and memory.
Data durability

After an OSD fails, the risk of data loss increases until the data it contained is fully recovered. Let’s imagine a scenario that causes permanent data loss in a single placement group:

    The OSD fails and all copies of the object it contains are lost. For all objects within the placement group the number of replica suddenly drops from three to two.
    Ceph starts recovery for this placement group by choosing a new OSD to re-create the third copy of all objects.
    Another OSD, within the same placement group, fails before the new OSD is fully populated with the third copy. Some objects will then only have one surviving copies.
    Ceph picks yet another OSD and keeps copying objects to restore the desired number of copies.
    A third OSD, within the same placement group, fails before recovery is complete. If this OSD contained the only remaining copy of an object, it is permanently lost.

In a cluster containing 10 OSDs with 512 placement groups in a three replica pool, CRUSH will give each placement groups three OSDs. In the end, each OSDs will end up hosting (512 * 3) / 10 = ~150 Placement Groups. When the first OSD fails, the above scenario will therefore start recovery for all 150 placement groups at the same time.

The 150 placement groups being recovered are likely to be homogeneously spread over the 9 remaining OSDs. Each remaining OSD is therefore likely to send copies of objects to all others and also receive some new objects to be stored because they became part of a new placement group.

The amount of time it takes for this recovery to complete entirely depends on the architecture of the Ceph cluster. Let say each OSD is hosted by a 1TB SSD on a single machine and all of them are connected to a 10Gb/s switch and the recovery for a single OSD completes within M minutes. If there are two OSDs per machine using spinners with no SSD journal and a 1Gb/s switch, it will at least be an order of magnitude slower.

In a cluster of this size, the number of placement groups has almost no influence on data durability. It could be 128 or 8192 and the recovery would not be slower or faster.

However, growing the same Ceph cluster to 20 OSDs instead of 10 OSDs is likely to speed up recovery and therefore improve data durability significantly. Each OSD now participates in only ~75 placement groups instead of ~150 when there were only 10 OSDs and it will still require all 19 remaining OSDs to perform the same amount of object copies in order to recover. But where 10 OSDs had to copy approximately 100GB each, they now have to copy 50GB each instead. If the network was the bottleneck, recovery will happen twice as fast. In other words, recovery goes faster when the number of OSDs increases.

If this cluster grows to 40 OSDs, each of them will only host ~35 placement groups. If an OSD dies, recovery will keep going faster unless it is blocked by another bottleneck. However, if this cluster grows to 200 OSDs, each of them will only host ~7 placement groups. If an OSD dies, recovery will happen between at most of ~21 (7 * 3) OSDs in these placement groups: recovery will take longer than when there were 40 OSDs, meaning the number of placement groups should be increased.

No matter how short the recovery time is, there is a chance for a second OSD to fail while it is in progress. In the 10 OSDs cluster described above, if any of them fail, then ~17 placement groups (i.e. ~150 / 9 placement groups being recovered) will only have one surviving copy. And if any of the 8 remaining OSD fail, the last objects of two placement groups are likely to be lost (i.e. ~17 / 8 placement groups with only one remaining copy being recovered).

When the size of the cluster grows to 20 OSDs, the number of Placement Groups damaged by the loss of three OSDs drops. The second OSD lost will degrade ~4 (i.e. ~75 / 19 placement groups being recovered) instead of ~17 and the third OSD lost will only lose data if it is one of the four OSDs containing the surviving copy. In other words, if the probability of losing one OSD is 0.0001% during the recovery time frame, it goes from 17 * 10 * 0.0001% in the cluster with 10 OSDs to 4 * 20 * 0.0001% in the cluster with 20 OSDs.

In a nutshell, more OSDs mean faster recovery and a lower risk of cascading failures leading to the permanent loss of a Placement Group. Having 512 or 4096 Placement Groups is roughly equivalent in a cluster with less than 50 OSDs as far as data durability is concerned.

Note: It may take a long time for a new OSD added to the cluster to be populated with placement groups that were assigned to it. However there is no degradation of any object and it has no impact on the durability of the data contained in the Cluster.
Object distribution within a pool

Ideally objects are evenly distributed in each placement group. Since CRUSH computes the placement group for each object, but does not actually know how much data is stored in each OSD within this placement group, the ratio between the number of placement groups and the number of OSDs may influence the distribution of the data significantly.

For instance, if there was a single placement group for ten OSDs in a three replica pool, only three OSD would be used because CRUSH would have no other choice. When more placement groups are available, objects are more likely to be evenly spread among them. CRUSH also makes every effort to evenly spread OSDs among all existing Placement Groups.

As long as there are one or two orders of magnitude more Placement Groups than OSDs, the distribution should be even. For instance, 300 placement groups for 3 OSDs, 1000 placement groups for 10 OSDs etc.

Uneven data distribution can be caused by factors other than the ratio between OSDs and placement groups. Since CRUSH does not take into account the size of the objects, a few very large objects may create an imbalance. Let say one million 4K objects totaling 4GB are evenly spread among 1000 placement groups on 10 OSDs. They will use 4GB / 10 = 400MB on each OSD. If one 400MB object is added to the pool, the three OSDs supporting the placement group in which the object has been placed will be filled with 400MB + 400MB = 800MB while the seven others will remain occupied with only 400MB.
Memory, CPU and network usage

For each placement group, OSDs and MONs need memory, network and CPU at all times and even more during recovery. Sharing this overhead by clustering objects within a placement group is one of the main reasons they exist.

Minimizing the number of placement groups saves significant amounts of resources.



### [Pools](http://docs.ceph.com/docs/master/rados/operations/pools/)
When you first deploy a cluster without creating a pool, Ceph uses the default pools for storing data. A pool provides you with:

    Resilience: You can set how many OSD are allowed to fail without losing data. For replicated pools, it is the desired number of copies/replicas of an object. A typical configuration stores an object and one additional copy (i.e., size = 2), but you can determine the number of copies/replicas. For erasure coded pools, it is the number of coding chunks (i.e. m=2 in the erasure code profile)
    Placement Groups: You can set the number of placement groups for the pool. A typical configuration uses approximately 100 placement groups per OSD to provide optimal balancing without using up too many computing resources. When setting up multiple pools, be careful to ensure you set a reasonable number of placement groups for both the pool and the cluster as a whole.
    CRUSH Rules: When you store data in a pool, placement of the object and its replicas (or chunks for erasure coded pools) in your cluster is governed by CRUSH rules. You can create a custom CRUSH rule for your pool if the default rule is not appropriate for your use case.
    Snapshots: When you create snapshots with ceph osd pool mksnap, you effectively take a snapshot of a particular pool.

To organize data into pools, you can list, create, and remove pools. You can also view the utilization statistics for each pool.