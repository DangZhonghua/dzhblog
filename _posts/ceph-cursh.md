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
