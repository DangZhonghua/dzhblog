# Cluster Map
Ceph depends upon Ceph Clients and Ceph OSD Daemons having knowledge of the cluster topology, which is inclusive of 5 maps collectively referred to as the “Cluster Map”:


1. The Monitor Map: Contains the cluster fsid, the position, name address and port of each monitor. It also indicates the current epoch, when the map was created, and the last time it changed. To view a monitor map, execute ceph mon dump.
   
2. The OSD Map: Contains the cluster fsid, when the map was created and last modified, a list of pools, replica sizes, PG numbers, a list of OSDs and their status (e.g., up, in). To view an OSD map, execute ceph osd dump.
   
3. The PG Map: Contains the PG version, its time stamp, the last OSD map epoch, the full ratios, and details on each placement group such as the PG ID, the Up Set, the Acting Set, the state of the PG (e.g., active + clean), and data usage statistics for each pool.
   
4. The CRUSH Map: Contains a list of storage devices, the failure domain hierarchy (e.g., device, host, rack, row, room, etc.), and rules for traversing the hierarchy when storing data. To view a CRUSH map, execute ceph osd getcrushmap -o {filename}; then, decompile it by executing crushtool -d {comp-crushmap-filename} -o {decomp-crushmap-filename}. You can view the decompiled map in a text editor or with cat.

5. The MDS Map: Contains the current MDS map epoch, when the map was created, and the last time it changed. It also contains the pool for storing metadata, a list of metadata servers, and which metadata servers are up and in. To view an MDS map, execute ceph fs dump.


Each map maintains an iterative history of its operating state changes. Ceph Monitors maintain a master copy of the cluster map including the cluster members, state, changes, and the overall health of the Ceph Storage Cluster.

## Where the cluster map store
The anwser is: ceph cluster monitors. The monitors are entries to the cluster. Ceph Monitors maintain a master copy of the cluster map including the cluster members, state, changes, and the overall health of the Ceph Storage Clutser. Each Map maintains an iterative history of its operating state change.
### High Availability Monitors
Since ceph Monitors is the entry point of cluster: before ceph clients can read or write data, they must contract a Ceph 