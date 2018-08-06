# ceph file layout when a normal file system file putting into RADOS

> Ceph is object storage system inherently. But how to map your data(like a file located on your local computer) into ceph object if you want
> to store your local file into ceph cluster named RADOS.

## ceph object
> Within ceph RADOS cluster, the object means: its a normal file located on local file system(ext4, xfs) of one of OSDs. But it is NOT equal to user data(like your private file on your local computer). The ceph object size is configuable(e.g., 2MB, 4MB, 2GB, etc.).


## File layout when put data into ceph RADOS cluster

> Ceph distributes (stripes) the data for a given file across a number of underlying objects. Yes, ceph can divide the given file into smaller blocks, then store these blocks into itself objects. This like RAID 0 data strip. which side do this strip function: client. Ceph provides three types of clients: Ceph Block Device, Ceph Filesystem, and Ceph Object Storage. A Ceph Client converts its data from the representation format it provides to its users (a block device image, RESTful objects, CephFS filesystem directories) into objects for storage in the Ceph Storage Cluster.
> The simplest Ceph striping format involves a stripe count of 1 object. Ceph Clients write stripe units to a Ceph Storage Cluster object until the object is at its maximum capacity, and then create another object for additional stripes of data. The simplest form of striping may be sufficient for small block device images, S3 or Swift objects and CephFS files. However, this simple form doesn’t take maximum advantage of Ceph’s ability to distribute data across placement groups, and consequently doesn’t improve performance very much. The following diagram depicts the simplest form of striping:
![simple strip](http://docs.ceph.com/docs/master/_images/ditaa-deb861a26cf89e008006b63d95885b4ed88ba608.png)

> The strip size = strip unit * strip count. The simpest strip IO manner is sequntial way, it can't be parallel to take advantage the rados cluster since the strip only has one strip unit so it can only store one RADOS object. So if the strip is consited of many strip unit(strip count =4, 8, etc.), then there are will be multiple objects( count = strip count) operated(read/write) parallelling. these multi-objects called: **object set**

![fast IO strip](http://docs.ceph.com/docs/master/_images/ditaa-92220e0223f86eb33cfcaed4241c6680226c5ce2.png)

## Strip Terminology

* file
        A collection of contiguous data, named from the perspective of the Ceph client (i.e., a file on a Linux system using Ceph storage). The data for a file is divided into fixed-size “stripe units,” which are stored in ceph “objects.”

* stripe unit
        The size (in bytes) of a block of data used in the RAID 0 distribution of a file. All stripe units for a file have equal size. The last stripe unit is typically incomplete–i.e. it represents the data at the end of the file as well as unused “space” beyond it up to the end of the fixed stripe unit size.

- stripe count
        The number of consecutive stripe units that constitute a RAID 0 “stripe” of file data.

- stripe
        A contiguous range of file data, RAID 0 striped across “stripe count” objects in fixed-size “stripe unit” blocks.
        stripe size = stripe unit * stripe count.

- object
        A collection of data maintained by Ceph storage. Objects are used to hold portions of Ceph client files.

- object set
        A set of objects that together represent a contiguous portion of a file.

> Note that by default, Ceph uses a simple striping strategy in which object_size equals stripe_unit and stripe_count is 1. This simply puts one stripe_unit in each object.


## Data Structure for data striping

```c++
    
    ceph_file_layout - describe data layout for a file/inode
    
    struct ceph_file_layout {
        /* file -> object mapping */
        __le32 fl_stripe_unit;     /* stripe unit, in bytes.  must be multiple
                        of page size. */
        __le32 fl_stripe_count;    /* over this many objects */
        __le32 fl_object_size;     /* until objects are this big, then move to new objects */
        __le32 fl_cas_hash;        /* UNUSED.  0 = none; 1 = sha256 */

        /* pg -> disk layout */
        __le32 fl_object_stripe_unit;  /* UNUSED.  for per-object parity, if any */

        /* object -> pg layout */
        __le32 fl_unused;       /* unused; used to be preferred primary for pg (-1 for none) */
        __le32 fl_pg_pool;      /* namespace, crush ruleset, rep level */
    } __attribute__ ((packed));

```

Here’s a more complex example:

file size = 1 trillion = 1000000000000 bytes

fl_stripe_unit = 64KB = 65536 bytes
fl_stripe_count = 5 stripe units per stripe
fl_object_size = 64GB = 68719476736 bytes

This means:

file stripe size = 64KB * 5 = 320KB = 327680 bytes
each object holds 64GB / 64KB = 1048576 stripe units
file object set size = 64GB * 5 = 320GB = 343597383680 bytes
    (also 1048576 stripe units * 327680 bytes per stripe unit)

So the file’s 1 trillion bytes can be divided into complete object sets, then complete stripes, then complete stripe units, and finally a single incomplete stripe unit:

- 1 trillion bytes / 320GB per object set = 2 complete object sets
    (with 312805232640 bytes remaining)
- 312805232640 bytes / 320KB per stripe = 954605 complete stripes
    (with 266240 bytes remaining)
- 266240 bytes / 64KB per stripe unit = 4 complete stripe units
    (with 4096 bytes remaining)
- and the final incomplete stripe unit holds those 4096 bytes.



