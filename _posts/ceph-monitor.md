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

## monitor configuration option list

```c++
***mon data***
    Description:	The monitor’s data location.
    Type:	String
    Default:	/var/lib/ceph/mon/$cluster-$id
```

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