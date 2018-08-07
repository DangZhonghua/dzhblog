---
layout: post
title: "ceph monitor study"
date: 2018-08-07 12:55:00
categories: ceph
tags:
---

# ceph monitor study
[Monitor Config Reference](http://docs.ceph.com/docs/master/rados/configuration/mon-config-ref/)

> Ceph Monitors maintain a “master copy” of the cluster map, which means a Ceph Client can determine the location of all Ceph Monitors, Ceph OSD Daemons, and Ceph Metadata Servers just by connecting to one Ceph Monitor and retrieving a current cluster map. Before Ceph Clients can read from or write to Ceph OSD Daemons or Ceph Metadata Servers, they must connect to a Ceph Monitor first. With a current copy of the cluster map and the CRUSH algorithm, a Ceph Client can compute the location for any object. The ability to compute object locations allows a Ceph Client to talk directly to Ceph OSD Daemons, which is a very important aspect of Ceph’s high scalability and performance.
> The primary role of the Ceph Monitor is to maintain a master copy of the cluster map. Ceph Monitors also provide authentication and logging services. Ceph Monitors write all changes in the monitor services to a single Paxos instance, and Paxos writes the changes to a key/value store for strong consistency. Ceph Monitors can query the most recent version of the cluster map during sync operations. Ceph Monitors leverage the key/value store’s snapshots and iterators (using leveldb) to perform store-wide synchronization.

![ceph monitor function](http://docs.ceph.com/docs/master/_images/ditaa-ae8fc6ae5b4014f064a0bed424507a7a247cd113.png)

## Bootstrapping Monitors
A Ceph Monitor requires a few explicit settings:

1. **Filesystem ID**: The fsid is the unique identifier for your object store. Since you can run multiple clusters on the same hardware, you must specify the unique ID of the object store when bootstrapping a monitor. Deployment tools usually do this for you (e.g., ceph-deploy can call a tool like uuidgen), but you may specify the **fsid** manually too.
   
2. **Monitor ID**: A monitor ID is a unique ID assigned to each monitor within the cluster. It is an alphanumeric value, and by convention the identifier usually follows an alphabetical increment (e.g., a, b, etc.). This can be set in a Ceph configuration file (e.g., [mon.a], [mon.b], etc.), by a deployment tool, or using the ceph commandline.
   
3. **Keys**: The monitor must have secret keys. A deployment tool such as ceph-deploy usually does this for you, but you may perform this step manually too. See [Monitor Keyrings](http://docs.ceph.com/docs/master/dev/mon-bootstrap/#secret-keys) for details.

## Configure Monitor
> To apply configuration settings to the entire cluster, enter the configuration settings under [global]. To apply configuration settings to all monitors in your cluster, enter the configuration settings under [mon]. To apply configuration settings to specific monitors, specify the monitor instance (e.g., [mon.a]). By convention, monitor instance names use alpha notation.
> example:

```
[global]

[mon]

[mon.a]

[mon.b]

[mon.c]

```
## minimum cconfiguration
> should configure monitor host name and its ip:port for every monitors in one cluster.

```c++
[mon]
        mon host = hostname1,hostname2,hostname3
        mon addr = 10.0.0.10:6789,10.0.0.11:6789,10.0.0.12:6789

```
or

```c++
[mon.a]
        host = hostname1
        mon addr = 10.0.0.10:6789
```

## Cluster ID
> Each Ceph Storage Cluster has a unique identifier (fsid). If specified, it usually appears under the [global] section of the configuration file. Deployment tools usually generate the fsid and store it in the monitor map, so the value may not appear in a configuration file. The ***fsid makes it possible to run daemons for multiple clusters on the same hardware***.

***Do not set this value if you use a deployment tool that does it for you.***

---

## Initial Members
> We recommend running a production Ceph Storage Cluster with at least three Ceph Monitors to ensure high availability. When you run multiple monitors, you may specify the initial monitors that must be members of the cluster in order to establish a quorum. This may reduce the time it takes for your cluster to come online.

```c++
[mon]
    mon initial members = a,b,c

    Description:The IDs of initial monitors in a cluster during startup. If specified, Ceph requires an odd number of monitors to form an initial quorum (e.g., 3).
    Type:	String
    Default:	None
    A majority of monitors in your cluster must be able to reach each other in order to establish a quorum. You can decrease the initial number of monitors to establish a quorum with this setting.
```


## monitor data
Ceph provides a default path where Ceph Monitors store data. For optimal performance in a production Ceph Storage Cluster, we recommend running Ceph Monitors on separate hosts and drives from Ceph OSD Daemons. As leveldb is using mmap() for writing the data, Ceph Monitors flush their data from memory to disk very often, which can interfere with Ceph OSD Daemon workloads if the data store is co-located with the OSD Daemons.

In Ceph versions 0.58 and earlier, Ceph Monitors store their data in files. This approach allows users to inspect monitor data with common tools like ls and cat. However, it doesn’t provide strong consistency.

In Ceph versions 0.59 and later, ***Ceph Monitors store their data as key/value pairs***(level db). Ceph Monitors require ACID transactions. Using a data store prevents recovering Ceph Monitors from running corrupted versions through Paxos, and it enables multiple modification operations in one single atomic batch, among other advantages.

Generally, we do not recommend changing the default data location. If you modify the default location, we recommend that you make it uniform across Ceph Monitors by setting it in the [mon] section of the configuration file.

## monitor common configuration option list


Option Name | Type | Default | Description
---|:---:|:---:|:---
mon data|string|/var/lib/ceph/mon/```$cluster-$id```|The monitor’s data location
mon data size warn|Integer|`15*1024*0124*1024`|Issue a HEALTH_WARN in cluster log when the monitor’s data store goes over 15GB
mon data avail warn|Integer|30|Issue a HEALTH_WARN in cluster log when the available disk space of monitor’s data store is lower or equal to this percentage.
mon data avail crit|Integer|5|Issue a HEALTH_ERR in cluster log when the available disk space of monitor’s data store is lower or equal to this percentage.
mon warn on cache pools without hit sets|Boolean|true|Issue a HEALTH_WARN in cluster log if a cache pool does not have the hitset type set set
mon warn on crush straw calc version zero|Boolean|Issue| a HEALTH_WARN in cluster log if the CRUSH’s straw_calc_version is zero. See CRUSH map tunables for details.
mon warn on legacy crush tunables|Boolean|true|Issue a HEALTH_WARN in cluster log if CRUSH tunables are too old (older than mon_min_crush_required_version)
mon crush min required version|String|firefly|The minimum tunable profile version required by the cluster. See CRUSH map tunables for details.
mon warn on osd down out interval zero|Boolean|true|Issue a HEALTH_WARN in cluster log if mon osd down out interval is zero. Having this option set to zero on the leader acts much like the noout flag. It’s hard to figure out what’s going wrong with clusters witout the noout flag set but acting like that just the same, so we report a warning in this case.
mon cache target full warn ratio|Float|0.66|Position between pool’s cache_target_full and target_max_object where we start warning
mon health data update interval|Float|60|How often (in seconds) the monitor in quorum shares its health status with its peers. (negative number disables it)
mon health to clog|Boolean|True|Enable sending health summary to cluster log periodically.
mon health to clog tick interval|Integer|3600|How often (in seconds) the monitor send health summary to cluster log (a non-positive number disables it). If current health summary is empty or identical to the last time, monitor will not send it to cluster log.
mon health to clog interval|Integer|60|How often (in seconds) the monitor send health summary to cluster log (a non-positive number disables it). Monitor will always send the summary to cluster log no matter if the summary changes or not.



## Storage Capacity
The following settings only apply on cluster creation and are then stored in the OSDMap.
```c++
[global]

        mon osd full ratio = .80
        mon osd backfillfull ratio = .75
        mon osd nearfull ratio = .70
```
> If some OSDs are nearfull, but others have plenty of capacity, you may have a problem with the CRUSH weight for the nearfull OSDs.
> These settings only apply during cluster creation. Afterwards they need to be changed in the OSDMap using ceph osd set-nearfull-ratio and ceph osd set-full-ratio

Option Name|Type|Default|Description
:---|:---:|:---:|:---
mon osd full ratio|Float|.95|The percentage of disk space used before an OSD is considered full.
mon osd backfillfull ratio|Float|.90|The percentage of disk space used before an OSD is considered too full to backfill.
mon osd nearfull ratio|Float|.85|The percentage of disk space used before an OSD is considered nearfull.

## HeartBeat
Ceph monitors know about the cluster by requiring reports from each OSD, and by receiving reports from OSDs about the status of their neighboring OSDs. Ceph provides reasonable default settings for monitor/OSD interaction; however, you may modify them as needed.

### Monitor Store Synchronization

cluster with multiple monitors (recommended), each monitor checks to see if a neighboring monitor has a more recent version of the cluster map (e.g., a map in a neighboring monitor with one or more epoch numbers higher than the most current epoch in the map of the instant monitor). Periodically, one monitor in the cluster may fall behind the other monitors to the point where it must leave the quorum, synchronize to retrieve the most current information about the cluster, and then rejoin the quorum.
For the purposes of synchronization, monitors may assume one of three roles (this is paxos algorithm role):

1. ***Leader***: The Leader is the first monitor to achieve the most recent Paxos version of the cluster map.
2. ***Provider***: The Provider is a monitor that has the most recent version of the cluster map, but wasn’t the first to achieve the most recent version.
3. ***Requester***: A Requester is a monitor that has fallen behind the leader and must synchronize in order to retrieve the most recent information about the cluster before it can rejoin the quorum.

These roles enable a leader to delegate synchronization duties to a provider, which prevents synchronization requests from overloading the leader–improving performance. In the following diagram, the requester has learned that it has fallen behind the other monitors. The requester asks the leader to synchronize, and the leader tells the requester to synchronize with a provider.
![Synchronization example](http://docs.ceph.com/docs/master/_images/ditaa-215fab4d12b3f0727a4fbc633b58887918820ca9.png)

Synchronization always occurs when a new monitor joins the cluster. During runtime operations, monitors may receive updates to the cluster map at different times. This means the leader and provider roles may migrate from one monitor to another. If this happens while synchronizing (e.g., a provider falls behind the leader), the provider can terminate synchronization with a requester.

Once synchronization is complete, Ceph requires trimming across the cluster. Trimming requires that the placement groups are active + clean.

monitor synchroniztion:

Option Name|Type|Default|Description
:---|:---:|:---:|:---
mon sync trim timeout|Double|30.0|*
mon sync heartbeat timeout|Double|30.0|*
mon sync heartbeat interval|Double|5.0|*
mon sync backoff timeout|Double|30.0|*
mon sync timeout|Double|60.0|Number of seconds the monitor will wait for the next update message from its sync provider before it gives up and bootstrap again.
mon sync max retries|Integer|5|*
mon sync max payload size|32-bit Integer|1045676|The maximum size for a sync payload (in bytes)
paxos max join drift|Integer|10|The maximum Paxos iterations before we must first sync the monitor data stores. When a monitor finds that its peer is too far ahead of it, it will first sync with data stores before moving on.
paxos stash full interval|Integer|25|How often (in commits) to stash a full copy of the PaxosService state. Current this setting only affects mds, mon, auth and mgr PaxosServices.
paxos propose interval|Double|1.0|Gather updates for this time interval before proposing a map update.
paxos min|Integer|500|	The minimum number of paxos states to keep around
paxos min wait|Double|0.05|	The minimum amount of time to gather updates after a period of inactivity.
paxos trim min|Integer|250|Number of extra proposals tolerated before trimming
paxos trim max|Integer|500|The maximum number of extra proposals to trim at a time
paxos service trim min|Integer|250|The minimum amount of versions to trigger a trim (0 disables it)
paxos service trim max|Integer|500|The maximum amount of versions to trim during a single proposal (0 disables it)
mon max pgmap epochs|Integer|500|The maximum amount of pgmap epochs to trim during a single proposal
mon mds force trim to|Integer|0|Force monitor to trim mdsmaps to this point (0 disables it. dangerous, use with care)
mon osd force trim to|Integer|0|	Force monitor to trim osdmaps to this point, even if there is PGs not clean at the specified epoch (0 disables it. dangerous, use with care)
mon osd cache size|Integer|10|The size of osdmaps cache, not to rely on underlying store’s cache
mon election timeout|Float|5|On election proposer, maximum waiting time for all ACKs in seconds.
mon lease|Float|5|The length (in seconds) of the lease on the monitor’s versions.
mon lease renew interval factor|Float|0.6|mon lease * mon lease renew interval factor will be the interval for the Leader to renew the other monitor’s leases. The factor should be less than 1.0.
mon lease ack timeout factor|Float|2.0|The Leader will wait mon lease * mon lease ack timeout factor for the Providers to acknowledge the lease extension.
mon accept timeout factor|Float|2.0|	The Leader will wait mon lease * mon accept timeout factor for the Requester(s) to accept a Paxos update. It is also used during the Paxos recovery phase for similar purposes.
mon min osdmap epochs|32bit int|500|Minimum number of OSD map epochs to keep at all times.
mon max pgmap epochs|32bit int|500|Maximum number of PG map epochs the monitor should keep.
mon max log epochs|32bit int|500|Maximum number of Log epochs the monitor should keep.