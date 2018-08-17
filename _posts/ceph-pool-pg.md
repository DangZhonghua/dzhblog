
#ceph pool and PG
[ceph pool](http://docs.ceph.com/docs/master/rados/operations/pools/#fast-read)
Compared with local file system, the pool can be considered as volume, the PG can be considered as folder.
Ceph client retrive a Cluster Map from a ceph Monitor, the Cluster Map contain detail information about the pools of the cluster, such as the poolid(pool id is used to form the pg name), the pool size(number of replica), the CRUSH rule and the number of placement groups determine how ceph will place the data.
![overall ceph client IO](http://docs.ceph.com/docs/master/_images/ditaa-778c68aa73d182d88236b73e1a4a40bd78fb32d6.png)


## ceph pool
Features of pool:
1. Resilience: You can set how many OSD are allowed to fail without losing data. For replicated pools, it is the desired number of copies/replicas of an object. A typical configuration stores an object and one additional copy (i.e., size = 2), but you can determine the number of copies/replicas. For erasure coded pools, it is the number of coding chunks (i.e. m=2 in the erasure code profile)
   
2. Placement Groups: You can set the number of placement groups for the pool. A typical configuration uses approximately 100 placement groups per OSD to provide optimal balancing without using up too many computing resources. When setting up multiple pools, be careful to ensure you set a reasonable number of placement groups for both the pool and the cluster as a whole. PG number can only be enlarged, can't make it smaller!!!

3. CRUSH Rules: When you store data in a pool, placement of the object and its replicas (or chunks for erasure coded pools) in your cluster is governed by CRUSH rules. You can create a custom CRUSH rule for your pool if the default rule is not appropriate for your use case.
4. Snapshots: When you create snapshots with ceph osd pool mksnap, you effectively take a snapshot of a particular pool.

All these features are refected on the definition of pool(source code)

```c++
struct pg_pool_t {
  static const char *APPLICATION_NAME_CEPHFS;
  static const char *APPLICATION_NAME_RBD;
  static const char *APPLICATION_NAME_RGW;

  enum {
    TYPE_REPLICATED = 1,     // replication
    //TYPE_RAID4 = 2,   // raid4 (never implemented)
    TYPE_ERASURE = 3,      // erasure-coded
  };
  static const char *get_type_name(int t) {
    switch (t) {
    case TYPE_REPLICATED: return "replicated";
      //case TYPE_RAID4: return "raid4";
    case TYPE_ERASURE: return "erasure";
    default: return "???";
    }
  }
  const char *get_type_name() const {
    return get_type_name(type);
  }

  enum {
    FLAG_HASHPSPOOL = 1<<0, // hash pg seed and pool together (instead of adding)
    FLAG_FULL       = 1<<1, // pool is full
    FLAG_EC_OVERWRITES = 1<<2, // enables overwrites, once enabled, cannot be disabled
    FLAG_INCOMPLETE_CLONES = 1<<3, // may have incomplete clones (bc we are/were an overlay)
    FLAG_NODELETE = 1<<4, // pool can't be deleted
    FLAG_NOPGCHANGE = 1<<5, // pool's pg and pgp num can't be changed
    FLAG_NOSIZECHANGE = 1<<6, // pool's size and min size can't be changed
    FLAG_WRITE_FADVISE_DONTNEED = 1<<7, // write mode with LIBRADOS_OP_FLAG_FADVISE_DONTNEED
    FLAG_NOSCRUB = 1<<8, // block periodic scrub
    FLAG_NODEEP_SCRUB = 1<<9, // block periodic deep-scrub
    FLAG_FULL_NO_QUOTA = 1<<10, // pool is currently running out of quota, will set FLAG_FULL too
    FLAG_NEARFULL = 1<<11, // pool is nearfull
    FLAG_BACKFILLFULL = 1<<12, // pool is backfillfull
  };

  static const char *get_flag_name(int f) {
    switch (f) {
    case FLAG_HASHPSPOOL: return "hashpspool";
    case FLAG_FULL: return "full";
    case FLAG_EC_OVERWRITES: return "ec_overwrites";
    case FLAG_INCOMPLETE_CLONES: return "incomplete_clones";
    case FLAG_NODELETE: return "nodelete";
    case FLAG_NOPGCHANGE: return "nopgchange";
    case FLAG_NOSIZECHANGE: return "nosizechange";
    case FLAG_WRITE_FADVISE_DONTNEED: return "write_fadvise_dontneed";
    case FLAG_NOSCRUB: return "noscrub";
    case FLAG_NODEEP_SCRUB: return "nodeep-scrub";
    case FLAG_FULL_NO_QUOTA: return "full_no_quota";
    case FLAG_NEARFULL: return "nearfull";
    case FLAG_BACKFILLFULL: return "backfillfull";
    default: return "???";
    }
  }
  static string get_flags_string(uint64_t f) {
    string s;
    for (unsigned n=0; f && n<64; ++n) {
      if (f & (1ull << n)) {
	if (s.length())
	  s += ",";
	s += get_flag_name(1ull << n);
      }
    }
    return s;
  }
  string get_flags_string() const {
    return get_flags_string(flags);
  }
  static uint64_t get_flag_by_name(const string& name) {
    if (name == "hashpspool")
      return FLAG_HASHPSPOOL;
    if (name == "full")
      return FLAG_FULL;
    if (name == "ec_overwrites")
      return FLAG_EC_OVERWRITES;
    if (name == "incomplete_clones")
      return FLAG_INCOMPLETE_CLONES;
    if (name == "nodelete")
      return FLAG_NODELETE;
    if (name == "nopgchange")
      return FLAG_NOPGCHANGE;
    if (name == "nosizechange")
      return FLAG_NOSIZECHANGE;
    if (name == "write_fadvise_dontneed")
      return FLAG_WRITE_FADVISE_DONTNEED;
    if (name == "noscrub")
      return FLAG_NOSCRUB;
    if (name == "nodeep-scrub")
      return FLAG_NODEEP_SCRUB;
    if (name == "full_no_quota")
      return FLAG_FULL_NO_QUOTA;
    if (name == "nearfull")
      return FLAG_NEARFULL;
    if (name == "backfillfull")
      return FLAG_BACKFILLFULL;
    return 0;
  }

  /// converts the acting/up vector to a set of pg shards
  void convert_to_pg_shards(const vector<int> &from, set<pg_shard_t>* to) const;

  typedef enum {
    CACHEMODE_NONE = 0,                  ///< no caching
    CACHEMODE_WRITEBACK = 1,             ///< write to cache, flush later
    CACHEMODE_FORWARD = 2,               ///< forward if not in cache
    CACHEMODE_READONLY = 3,              ///< handle reads, forward writes [not strongly consistent]
    CACHEMODE_READFORWARD = 4,           ///< forward reads, write to cache flush later
    CACHEMODE_READPROXY = 5,             ///< proxy reads, write to cache flush later
    CACHEMODE_PROXY = 6,                 ///< proxy if not in cache
  } cache_mode_t;
  static const char *get_cache_mode_name(cache_mode_t m) {
    switch (m) {
    case CACHEMODE_NONE: return "none";
    case CACHEMODE_WRITEBACK: return "writeback";
    case CACHEMODE_FORWARD: return "forward";
    case CACHEMODE_READONLY: return "readonly";
    case CACHEMODE_READFORWARD: return "readforward";
    case CACHEMODE_READPROXY: return "readproxy";
    case CACHEMODE_PROXY: return "proxy";
    default: return "unknown";
    }
  }
  static cache_mode_t get_cache_mode_from_str(const string& s) {
    if (s == "none")
      return CACHEMODE_NONE;
    if (s == "writeback")
      return CACHEMODE_WRITEBACK;
    if (s == "forward")
      return CACHEMODE_FORWARD;
    if (s == "readonly")
      return CACHEMODE_READONLY;
    if (s == "readforward")
      return CACHEMODE_READFORWARD;
    if (s == "readproxy")
      return CACHEMODE_READPROXY;
    if (s == "proxy")
      return CACHEMODE_PROXY;
    return (cache_mode_t)-1;
  }
  const char *get_cache_mode_name() const {
    return get_cache_mode_name(cache_mode);
  }
  bool cache_mode_requires_hit_set() const {
    switch (cache_mode) {
    case CACHEMODE_NONE:
    case CACHEMODE_FORWARD:
    case CACHEMODE_READONLY:
    case CACHEMODE_PROXY:
      return false;
    case CACHEMODE_WRITEBACK:
    case CACHEMODE_READFORWARD:
    case CACHEMODE_READPROXY:
      return true;
    default:
      assert(0 == "implement me");
    }
  }

  uint64_t flags;           ///< FLAG_*
  __u8 type;                ///< TYPE_*
  __u8 size, min_size;      ///< number of osds in each pg
  __u8 crush_rule;          ///< crush placement rule
  __u8 object_hash;         ///< hash mapping object name to ps
private:
  __u32 pg_num, pgp_num;    ///< number of pgs


public:
  map<string,string> properties;  ///< OBSOLETE
  string erasure_code_profile; ///< name of the erasure code profile in OSDMap
  epoch_t last_change;      ///< most recent epoch changed, exclusing snapshot changes
  epoch_t last_force_op_resend; ///< last epoch that forced clients to resend
  /// last epoch that forced clients to resend (pre-luminous clients only)
  epoch_t last_force_op_resend_preluminous;
  snapid_t snap_seq;        ///< seq for per-pool snapshot
  epoch_t snap_epoch;       ///< osdmap epoch of last snap
  uint64_t auid;            ///< who owns the pg
  __u32 crash_replay_interval; ///< seconds to allow clients to replay ACKed but unCOMMITted requests

  uint64_t quota_max_bytes; ///< maximum number of bytes for this pool
  uint64_t quota_max_objects; ///< maximum number of objects for this pool

  /*
   * Pool snaps (global to this pool).  These define a SnapContext for
   * the pool, unless the client manually specifies an alternate
   * context.
   */
  map<snapid_t, pool_snap_info_t> snaps;
  /*
   * Alternatively, if we are defining non-pool snaps (e.g. via the
   * Ceph MDS), we must track @removed_snaps (since @snaps is not
   * used).  Snaps and removed_snaps are to be used exclusive of each
   * other!
   */
  interval_set<snapid_t> removed_snaps;

  unsigned pg_num_mask, pgp_num_mask;

  set<uint64_t> tiers;      ///< pools that are tiers of us
  int64_t tier_of;         ///< pool for which we are a tier
  // Note that write wins for read+write ops
  int64_t read_tier;       ///< pool/tier for objecter to direct reads to
  int64_t write_tier;      ///< pool/tier for objecter to direct writes to
  cache_mode_t cache_mode;  ///< cache pool mode

  bool is_tier() const { return tier_of >= 0; }
  bool has_tiers() const { return !tiers.empty(); }
  void clear_tier() {
    tier_of = -1;
    clear_read_tier();
    clear_write_tier();
    clear_tier_tunables();
  }
  bool has_read_tier() const { return read_tier >= 0; }
  void clear_read_tier() { read_tier = -1; }
  bool has_write_tier() const { return write_tier >= 0; }
  void clear_write_tier() { write_tier = -1; }
  void clear_tier_tunables() {
    if (cache_mode != CACHEMODE_NONE)
      flags |= FLAG_INCOMPLETE_CLONES;
    cache_mode = CACHEMODE_NONE;

    target_max_bytes = 0;
    target_max_objects = 0;
    cache_target_dirty_ratio_micro = 0;
    cache_target_dirty_high_ratio_micro = 0;
    cache_target_full_ratio_micro = 0;
    hit_set_params = HitSet::Params();
    hit_set_period = 0;
    hit_set_count = 0;
    hit_set_grade_decay_rate = 0;
    hit_set_search_last_n = 0;
    grade_table.resize(0);
  }

  uint64_t target_max_bytes;   ///< tiering: target max pool size
  uint64_t target_max_objects; ///< tiering: target max pool size

  uint32_t cache_target_dirty_ratio_micro; ///< cache: fraction of target to leave dirty
  uint32_t cache_target_dirty_high_ratio_micro; ///<cache: fraction of  target to flush with high speed
  uint32_t cache_target_full_ratio_micro;  ///< cache: fraction of target to fill before we evict in earnest

  uint32_t cache_min_flush_age;  ///< minimum age (seconds) before we can flush
  uint32_t cache_min_evict_age;  ///< minimum age (seconds) before we can evict

  HitSet::Params hit_set_params; ///< The HitSet params to use on this pool
  uint32_t hit_set_period;      ///< periodicity of HitSet segments (seconds)
  uint32_t hit_set_count;       ///< number of periods to retain
  bool use_gmt_hitset;	        ///< use gmt to name the hitset archive object
  uint32_t min_read_recency_for_promote;   ///< minimum number of HitSet to check before promote on read
  uint32_t min_write_recency_for_promote;  ///< minimum number of HitSet to check before promote on write
  uint32_t hit_set_grade_decay_rate;   ///< current hit_set has highest priority on objects
                                       ///temperature count,the follow hit_set's priority decay 
                                       ///by this params than pre hit_set
  uint32_t hit_set_search_last_n;   ///<accumulate atmost N hit_sets for temperature

  uint32_t stripe_width;        ///< erasure coded stripe size in bytes

  uint64_t expected_num_objects; ///< expected number of objects on this pool, a value of 0 indicates
                                 ///< user does not specify any expected value
  bool fast_read;            ///< whether turn on fast read on the pool or not

  pool_opts_t opts; ///< options

  /// application -> key/value metadata
  map<string, std::map<string, string>> application_metadata;

private:
  vector<uint32_t> grade_table;

public:
  uint32_t get_grade(unsigned i) const {
    if (grade_table.size() <= i)
      return 0;
    return grade_table[i];
  }
  void calc_grade_table() {
    unsigned v = 1000000;
    grade_table.resize(hit_set_count);
    for (unsigned i = 0; i < hit_set_count; i++) {
      v = v * (1 - (hit_set_grade_decay_rate / 100.0));
      grade_table[i] = v;
    }
  }

  pg_pool_t()
    : flags(0), type(0), size(0), min_size(0),
      crush_rule(0), object_hash(0),
      pg_num(0), pgp_num(0),
      last_change(0),
      last_force_op_resend(0),
      last_force_op_resend_preluminous(0),
      snap_seq(0), snap_epoch(0),
      auid(0),
      crash_replay_interval(0),
      quota_max_bytes(0), quota_max_objects(0),
      pg_num_mask(0), pgp_num_mask(0),
      tier_of(-1), read_tier(-1), write_tier(-1),
      cache_mode(CACHEMODE_NONE),
      target_max_bytes(0), target_max_objects(0),
      cache_target_dirty_ratio_micro(0),
      cache_target_dirty_high_ratio_micro(0),
      cache_target_full_ratio_micro(0),
      cache_min_flush_age(0),
      cache_min_evict_age(0),
      hit_set_params(),
      hit_set_period(0),
      hit_set_count(0),
      use_gmt_hitset(true),
      min_read_recency_for_promote(0),
      min_write_recency_for_promote(0),
      hit_set_grade_decay_rate(0),
      hit_set_search_last_n(0),
      stripe_width(0),
      expected_num_objects(0),
      fast_read(false),
      opts()
  { }

  void dump(Formatter *f) const;

  uint64_t get_flags() const { return flags; }
  bool has_flag(uint64_t f) const { return flags & f; }
  void set_flag(uint64_t f) { flags |= f; }
  void unset_flag(uint64_t f) { flags &= ~f; }

  bool ec_pool() const {
    return type == TYPE_ERASURE;
  }
  bool require_rollback() const {
    return ec_pool();
  }

  /// true if incomplete clones may be present
  bool allow_incomplete_clones() const {
    return cache_mode != CACHEMODE_NONE || has_flag(FLAG_INCOMPLETE_CLONES);
  }

  unsigned get_type() const { return type; }
  unsigned get_size() const { return size; }
  unsigned get_min_size() const { return min_size; }
  int get_crush_rule() const { return crush_rule; }
  int get_object_hash() const { return object_hash; }
  const char *get_object_hash_name() const {
    return ceph_str_hash_name(get_object_hash());
  }
  epoch_t get_last_change() const { return last_change; }
  epoch_t get_last_force_op_resend() const { return last_force_op_resend; }
  epoch_t get_last_force_op_resend_preluminous() const {
    return last_force_op_resend_preluminous;
  }
  epoch_t get_snap_epoch() const { return snap_epoch; }
  snapid_t get_snap_seq() const { return snap_seq; }
  uint64_t get_auid() const { return auid; }
  unsigned get_crash_replay_interval() const { return crash_replay_interval; }

  void set_snap_seq(snapid_t s) { snap_seq = s; }
  void set_snap_epoch(epoch_t e) { snap_epoch = e; }

  void set_stripe_width(uint32_t s) { stripe_width = s; }
  uint32_t get_stripe_width() const { return stripe_width; }

  bool is_replicated()   const { return get_type() == TYPE_REPLICATED; }
  bool is_erasure() const { return get_type() == TYPE_ERASURE; }

  bool supports_omap() const {
    return !(get_type() == TYPE_ERASURE);
  }

  bool requires_aligned_append() const {
    return is_erasure() && !has_flag(FLAG_EC_OVERWRITES);
  }
  uint64_t required_alignment() const { return stripe_width; }

  bool allows_ecoverwrites() const {
    return has_flag(FLAG_EC_OVERWRITES);
  }

  bool can_shift_osds() const {
    switch (get_type()) {
    case TYPE_REPLICATED:
      return true;
    case TYPE_ERASURE:
      return false;
    default:
      assert(0 == "unhandled pool type");
    }
  }

  unsigned get_pg_num() const { return pg_num; }
  unsigned get_pgp_num() const { return pgp_num; }

  unsigned get_pg_num_mask() const { return pg_num_mask; }
  unsigned get_pgp_num_mask() const { return pgp_num_mask; }

  // if pg_num is not a multiple of two, pgs are not equally sized.
  // return, for a given pg, the fraction (denominator) of the total
  // pool size that it represents.
  unsigned get_pg_num_divisor(pg_t pgid) const;

  void set_pg_num(int p) {
    pg_num = p;
    calc_pg_masks();
  }
  void set_pgp_num(int p) {
    pgp_num = p;
    calc_pg_masks();
  }

  void set_quota_max_bytes(uint64_t m) {
    quota_max_bytes = m;
  }
  uint64_t get_quota_max_bytes() {
    return quota_max_bytes;
  }

  void set_quota_max_objects(uint64_t m) {
    quota_max_objects = m;
  }
  uint64_t get_quota_max_objects() {
    return quota_max_objects;
  }

  void set_last_force_op_resend(uint64_t t) {
    last_force_op_resend = t;
    last_force_op_resend_preluminous = t;
  }

  void calc_pg_masks();

  /*
   * we have two snap modes:
   *  - pool global snaps
   *    - snap existence/non-existence defined by snaps[] and snap_seq
   *  - user managed snaps
   *    - removal governed by removed_snaps
   *
   * we know which mode we're using based on whether removed_snaps is empty.
   * If nothing has been created, both functions report false.
   */
  bool is_pool_snaps_mode() const;
  bool is_unmanaged_snaps_mode() const;
  bool is_removed_snap(snapid_t s) const;

  /*
   * build set of known-removed sets from either pool snaps or
   * explicit removed_snaps set.
   */
  void build_removed_snaps(interval_set<snapid_t>& rs) const;
  snapid_t snap_exists(const char *s) const;
  void add_snap(const char *n, utime_t stamp);
  void add_unmanaged_snap(uint64_t& snapid);
  void remove_snap(snapid_t s);
  void remove_unmanaged_snap(snapid_t s);

  SnapContext get_snap_context() const;

  /// hash a object name+namespace key to a hash position
  uint32_t hash_key(const string& key, const string& ns) const;

  /// round a hash position down to a pg num
  uint32_t raw_hash_to_pg(uint32_t v) const;

  /*
   * map a raw pg (with full precision ps) into an actual pg, for storage
   */
  pg_t raw_pg_to_pg(pg_t pg) const;
  
  /*
   * map raw pg (full precision ps) into a placement seed.  include
   * pool id in that value so that different pools don't use the same
   * seeds.
   */
  ps_t raw_pg_to_pps(pg_t pg) const;

  /// choose a random hash position within a pg
  uint32_t get_random_pg_position(pg_t pgid, uint32_t seed) const;

  void encode(bufferlist& bl, uint64_t features) const;
  void decode(bufferlist::iterator& bl);

  static void generate_test_instances(list<pg_pool_t*>& o);
};
```

### pool related command
function | command | extra info
:---|:---:|:---
list cluster pool|**ceph osd lspools**|*
Create pool|ceph osd pool create {pool name} {pg-num} [{pgp-num}] replicated(pool type) crush-rule-name expected-num-objects|*
associate pool to application|ceph osd pool application enable {pool name} {application name}|*
set pool quotas|ceph osd pool set-quota {pool name} [max_objects {obj-count}] [max_bytes {bytes}]|To remove a quota, set its value to 0.
Delete pool|ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]|To remove a pool the mon_allow_pool_delete flag must be set to true in the Monitor’s configuration. Otherwise they will refuse to remove a pool.
get pool crush rule|ceph osd pool get {pool-name} crush_rule|*
rename pool|ceph osd pool rename {current-pool-name} {new-pool-name}|If you rename a pool and you have per-pool capabilities for an authenticated user, you must update the user’s capabilities (i.e., caps) with the new pool name.
pool usage|rados df|*
snapshot of pool|ceph osd pool mksnap {pool-name} {snap-name}|*
remove snapshot|ceph osd pool rmsnap {pool-name} {snap-name}|*
set pool values|ceph osd pool set {pool-name} {key} {value}|*
get pool values|ceph osd pool get {pool-name} {key}|*
set object replica| ceph osd pool set {poolname}  size {num-replicas}|*
set object min replica|ceph osd pool set data min_size {num}|This ensures that no object in the data pool will receive I/O with fewer than min_size replicas.
get object replicas|ceph osd dump | grep 'replicated size'|*

## Ceph pool PG(Placement Group)
As mentioned before, the PG name is {pool-num}.{pgid} which is a directory on local file system of one OSD.
Placing data manner is mapping objects (data) into PGS not directly into OSD. PG is virtual/indirect layer for placing data; PG is mapped onto OSDs, why pg do this? PG can group objects which reduces the amount of per-object metadata we need track of and processed we need to run(it would be prohibitively expensive to track eg the placement history on a per-object basis). Use to PG to group object and track the object, the failure scale/balance effect/data moving can be in a compare smaller level to reduce the effect of data lossing/data balance,etc.
ncreasing the number of PGs can reduce the variance in per-OSD load across your cluster, but each PG requires a bit more CPU and memory on the OSDs that are storing it. We try and ballpark it at 100 PGs/OSD, although it can vary widely without ill effects depending on your cluster. You hit a bug in how we calculate the initial PG number from a cluster description.

### mapping objects into its should be, real placing position
***One object can place only one pool.***
***One object can only be mapped into one PG***
***One PG can be mapped into multi-OSD.(this is decided by pool size: object replica size)***
***One OSD can handle multi-PGS***
***Different objects can mapp into one PG***
***many PGs can map to one OSD***

A PG represents nothing but a grouping of objects; you configure the number of PGs you want, number of OSDs * 100 is a good starting point , and all of your stored objects are pseudo-randomly evenly distributed to the PGs. So a PG explicitly does NOT represent a fixed amount of storage; it represents 1/pg_num’th of the storage you happen to have on your OSDs.

> locator = object_name
> obj_hash = hash(locator)
> pg = obj_hash % num_pg
> OSDs_for_pg = crush(pg)  # returns a list of OSDs
> primary = osds_for_pg[0]
> replicas = osds_for_pg[1:]

### Mapping PGS TO OSDS

Each pool has a number of placement groups. CRUSH maps PGs to OSDs dynamically. When a Ceph Client stores objects, CRUSH will map each object to a placement group.

Mapping objects to placement groups creates a layer of indirection between the Ceph OSD Daemon and the Ceph Client. The Ceph Storage Cluster must be able to grow (or shrink) and rebalance where it stores objects dynamically. If the Ceph Client “knew” which Ceph OSD Daemon had which object, that would create a tight coupling between the Ceph Client and the Ceph OSD Daemon. Instead, the CRUSH algorithm maps each object to a placement group and then maps each placement group to one or more Ceph OSD Daemons. This layer of indirection allows Ceph to rebalance dynamically when new Ceph OSD Daemons and the underlying OSD devices come online. The following diagram depicts how CRUSH maps objects to placement groups, and placement groups to OSDs.

![object placement flow](http://docs.ceph.com/docs/master/_images/ditaa-c7fd5a4042a21364a7bef1c09e6b019deb4e4feb.png)

With a copy of the cluster map and the CRUSH algorithm, the client can compute exactly which OSD to use when reading or writing a particular object.

Calculating PG IDs

When a Ceph Client binds to a Ceph Monitor, it retrieves the latest copy of the Cluster Map. With the cluster map, the client knows about all of the monitors, OSDs, and metadata servers in the cluster. However, it doesn’t know anything about object locations.

    Object locations get computed.

The only input required by the client is the object ID and the pool. It’s simple: Ceph stores data in named pools (e.g., “liverpool”). When a client wants to store a named object (e.g., “john,” “paul,” “george,” “ringo”, etc.) it calculates a placement group using the object name, a hash code, the number of PGs in the pool and the pool name. Ceph clients use the following steps to compute PG IDs.

    The client inputs the pool name and the object ID. (e.g., pool = “liverpool” and object-id = “john”)
    Ceph takes the object ID and hashes it.
    Ceph calculates the hash modulo the number of PGs. (e.g., 58) to get a PG ID.
    Ceph gets the pool ID given the pool name (e.g., “liverpool” = 4)
    Ceph prepends the pool ID to the PG ID (e.g., 4.58).

***Computing object locations is much faster than performing object location query over a chatty session. The CRUSH algorithm allows a client to compute where objects should be stored, and enables the client to contact the primary OSD to store or retrieve the objects.***

### peering and set
Noted that Ceph OSD Daemons check each others heartbeats and report back to the Ceph Monitor. Another thing Ceph OSD daemons do is called ‘peering’, which is the process of bringing all of the OSDs that store a Placement Group (PG) into agreement about the state of all of the objects (and their metadata) in that PG. In fact, Ceph OSD Daemons [Report Peering Failure](http://docs.ceph.com/docs/master/rados/configuration/mon-osd-interaction/#osds-report-peering-failure) to the Ceph Monitors. Peering issues usually resolve themselves; however, if the problem persists, you may need to refer to the Troubleshooting Peering Failure section.

Note

Agreeing on the state does not mean that the PGs have the latest contents.

The Ceph Storage Cluster was designed to store at least two copies of an object (i.e., size = 2), which is the minimum requirement for data safety. For high availability, a Ceph Storage Cluster should store more than two copies of an object (e.g., size = 3 and min size = 2) so that it can continue to run in a degraded state while maintaining data safety.

Referring back to the diagram in [Smart Daemons Enable Hyperscale](http://docs.ceph.com/docs/master/architecture/#smart-daemons-enable-hyperscale), we do not name the Ceph OSD Daemons specifically (e.g., osd.0, osd.1, etc.), but rather refer to them as Primary, Secondary, and so forth. By convention, the Primary is the first OSD in the Acting Set, and is responsible for coordinating the peering process for each placement group where it acts as the Primary, and is the ONLY OSD that that will accept client-initiated writes to objects for a given placement group where it acts as the Primary.

When a series of OSDs are responsible for a placement group, that series of OSDs, we refer to them as an Acting Set. An Acting Set may refer to the Ceph OSD Daemons that are currently responsible for the placement group, or the Ceph OSD Daemons that were responsible for a particular placement group as of some epoch.

The Ceph OSD daemons that are part of an Acting Set may not always be up. When an OSD in the Acting Set is up, it is part of the Up Set. The Up Set is an important distinction, because Ceph can remap PGs to other Ceph OSD Daemons when an OSD fails.

Note

In an Acting Set for a PG containing osd.25, osd.32 and osd.61, the first OSD, osd.25, is the Primary. If that OSD fails, the Secondary, osd.32, becomes the Primary, and osd.25 will be removed from the Up Set. 

### Rebalancing
When you add a Ceph OSD Daemon to a Ceph Storage Cluster, the cluster map gets updated with the new OSD. Referring back to [Calculating PG IDs](http://docs.ceph.com/docs/master/architecture/#calculating-pg-ids), this changes the cluster map. Consequently, it changes object placement, because it changes an input for the calculations. The following diagram depicts the rebalancing process (albeit rather crudely, since it is substantially less impactful with large clusters) where some, but not all of the PGs migrate from existing OSDs (OSD 1, and OSD 2) to the new OSD (OSD 3). Even when rebalancing, CRUSH is stable. Many of the placement groups remain in their original configuration, and each OSD gets some added capacity, so there are no load spikes on the new OSD after rebalancing is complete.
![rebalance](http://docs.ceph.com/docs/master/_images/ditaa-b31e1f646135b9706000fa0799d572563dffac81.png)

### Data Consistency
As part of maintaining data consistency and cleanliness, Ceph OSDs can also scrub objects within placement groups. That is, Ceph OSDs can compare object metadata in one placement group with its replicas in placement groups stored in other OSDs. Scrubbing (usually performed daily) catches OSD bugs or filesystem errors. OSDs can also perform deeper scrubbing by comparing data in objects bit-for-bit. Deep scrubbing (usually performed weekly) finds bad sectors on a disk that weren’t apparent in a light scrub.

### User-visible PG States

State|Meaning
:---|:---
creating|the PG is still being created
active|requests to the PG will be processed
clean|all objects in the PG are replicated the correct number of times
down|a replica with necessary data is down, so the pg is offline
replay|the PG is waiting for clients to replay operations after an OSD crashed
splitting|the PG is being split into multiple PGs (not functional as of 2012-02)
scrubbing|the PG is being checked for inconsistencies    
degraded|some objects in the PG are not replicated enough times yet
inconsistent|replicas of the PG are not consistent (e.g. objects are the wrong size, objects are missing from one replica after recovery finished, etc.)    
peering|the PG is undergoing the Peering process
repair|the PG is being checked and any inconsistencies found will be repaired (if possible)   
recovering|objects are being migrated/synchronized with replicas
recovery_wait|the PG is waiting for the local/remote recovery reservations
backfilling|a special case of recovery, in which the entire contents of the PG are scanned and synchronized, instead of inferring what needs to be transferred from the PG logs of recent operations
backfill_wait|the PG is waiting in line to start backfill
backfill_toofull|backfill reservation rejected, OSD too full  
incomplete|a pg is missing a necessary period of history from its log. If you see this state, report a bug, and try to start any failed OSDs that may contain the needed information.
stale|the PG is in an unknown state - the monitors have not received an update for it since the PG mapping changed.
remapped|the PG is temporarily mapped to a different set of OSDs from what CRUSH specified

[more state description](http://www.xuxiaopang.com/2016/11/11/doc-ceph-table/#more)

### Peering
the process of bringing all of the OSDs that store a Placement Group (PG) into agreement about the state of all of the objects (and their metadata) in that PG. Note that agreeing on the state does not mean that they all have the latest contents.

1. Acting Set:the ordered list of OSDs who are (or were as of some epoch) responsible for a particular PG. ***primary*** the (by convention first) member of the acting set, who is responsible for coordination peering, and is the only OSD that will accept client initiated writes to objects in a placement group. ***replica*** a non-primary OSD in the acting set for a placement group (and who has been recognized as such and activated by the primary). ***stray*** an OSD who is not a member of the current acting set, but has not yet been told that it can delete its copies of a particular placement group

2. Up Set:the ordered list of OSDs responsible for a particular PG for a particular epoch according to CRUSH. Normally this is the same as the acting set, except when the acting set has been explicitly overridden via PG temp in the OSDMap.
3. current interval or past interval:a sequence of OSD map epochs during which the acting set and up set for particular PG do not change
4. recovery: ensuring that copies of all of the objects in a PG are on all of the OSDs in the acting set. Once peering has been performed, the primary can start accepting write operations, and recovery can proceed in the background.
5. PG info basic metadata about the PG’s creation epoch, the version for the most recent write to the PG, last epoch started, last epoch clean, and the beginning of the current interval. Any inter-OSD communication about PGs includes the PG info, such that any OSD that knows a PG exists (or once existed) also has a lower bound on last epoch clean or last epoch started.
6. PG log
   ***
   a list of recent updates made to objects in a PG. Note that these logs can be truncated after all OSDs in the acting set have acknowledged up to a certain point.
   ***
7. Authoritative History
   ***
   a complete, and fully ordered set of operations that, if performed, would bring an OSD’s copy of a Placement Group up to date.
   ***
8. missing set: Each OSD notes update log entries and if they imply updates to the contents of an object, adds that object to a list of needed updates. This list is called the missing set for that <OSD,PG>.
9. epoch:a (monotonically increasing) OSD map version number
10. last epoch start:the last epoch at which all nodes in the acting set for a particular placement group agreed on an authoritative history. At this point, peering is deemed to have been successful.
11. ***last epoch clean***:the last epoch at which all nodes in the acting set for a particular placement group were completely up to date (both PG logs and object contents). At this point, recovery is deemed to have been completed.
12. up_thru: before a primary can successfully complete the peering process, it must inform a monitor that is alive through the current OSD map epoch by having the monitor set its up_thru in the osd map. This helps peering ignore previous acting sets for which peering never completed after certain sequences of failures, such as the second interval below:

        acting set = [A,B]
        acting set = [A]
        acting set = [] very shortly after (e.g., simultaneous failure, but staggered detection)
        acting set = [B] (B restarts, A does not)
#### peering process


***
***PG temp*** This can be used to rebalance the data
    a temporary placement group acting set used while backfilling the primary osd. Let say acting is [0,1,2] and we are active+clean. Something happens and acting is now [3,1,2]. osd 3 is empty and can’t serve reads although it is the primary. osd.3 will see that and request a PG temp of [1,2,3] to the monitors using a MOSDPGTemp message so that osd.1 temporarily becomes the primary. It will select osd.3 as a backfill peer and continue to serve reads and writes while osd.3 is backfilled. When backfilling is complete, PG temp is discarded and the acting set changes back to [3,1,2] and osd.3 becomes the primary.
***

### [Map and PG Message handling](http://docs.ceph.com/docs/master/dev/osd_internals/map_message_handling/)

#### overview

The OSD handles routing incoming messages to PGs, creating the PG if necessary in some cases.

PG messages generally come in two varieties:
1. Peering Messages
2. Ops/SubOps

There are several ways in which a message might be dropped or delayed. It is important that the message delaying does not result in a violation of certain message ordering requirements on the way to the relevant PG handling logic:
1. Ops referring to the same object must not be reordered.
2. Peering messages must not be reordered.
3. Subops must not be reordered.

#### MOSDMAP
MOSDMap messages may come from either monitors or other OSDs. Upon receipt, the OSD must perform several tasks:

        Persist the new maps to the filestore. Several PG operations rely on having access to maps dating back to the last time the PG was clean.
        Update and persist the superblock.
        Update OSD state related to the current map.
        Expose new maps to PG processes via OSDService.
        Remove PGs due to pool removal.
        Queue dummy events to trigger PG map catchup.

Each PG asynchronously catches up to the currently published map during process_peering_events before processing the event. As a result, different PGs may have different views as to the “current” map.
One consequence of this design is that messages containing submessages from multiple PGs (MOSDPGInfo, MOSDPGQuery, MOSDPGNotify) must tag each submessage with the PG’s epoch as well as tagging the message as a whole with the OSD’s current published epoch.

#### MOSDPGOp/MOSDPGSubOp
See OSD::dispatch_op, OSD::handle_op, OSD::handle_sub_op

MOSDPGOps are used by clients to initiate rados operations. MOSDSubOps are used between OSDs to coordinate most non peering activities including replicating MOSDPGOp operations.

OSD::require_same_or_newer map checks that the current OSDMap is at least as new as the map epoch indicated on the message. If not, the message is queued in OSD::waiting_for_osdmap via OSD::wait_for_new_map. Note, this cannot violate the above conditions since any two messages will be queued in order of receipt and if a message is received with epoch e0, a later message from the same source must be at epoch at least e0. Note that two PGs from the same OSD count for these purposes as different sources for single PG messages. That is, messages from different PGs may be reordered.

MOSDPGOps follow the following process:

        OSD::handle_op: validates permissions and crush mapping. discard the request if they are not connected and the client cannot get the reply ( See OSD::op_is_discardable ) See OSDService::handle_misdirected_op See PG::op_has_sufficient_caps See OSD::require_same_or_newer_map
        OSD::enqueue_op

MOSDSubOps follow the following process:

        OSD::handle_sub_op checks that sender is an OSD
        OSD::enqueue_op

OSD::enqueue_op calls PG::queue_op which checks waiting_for_map before calling OpWQ::queue which adds the op to the queue of the PG responsible for handling it.

OSD::dequeue_op is then eventually called, with a lock on the PG. At this time, the op is passed to PG::do_request, which checks that:

        the PG map is new enough (PG::must_delay_op)
        the client requesting the op has enough permissions (PG::op_has_sufficient_caps)
        the op is not to be discarded (PG::can_discard_{request,op,subop,scan,backfill})
        the PG is active (PG::flushed boolean)
        the op is a CEPH_MSG_OSD_OP and the PG is in PG_STATE_ACTIVE state and not in PG_STATE_REPLAY

If these conditions are not met, the op is either discarded or queued for later processing. If all conditions are met, the op is processed according to its type:

        CEPH_MSG_OSD_OP is handled by PG::do_op
        MSG_OSD_SUBOP is handled by PG::do_sub_op
        MSG_OSD_SUBOPREPLY is handled by PG::do_sub_op_reply
        MSG_OSD_PG_SCAN is handled by PG::do_scan
        MSG_OSD_PG_BACKFILL is handled by PG::do_backfill
#### CEPH_MSG_OSD_OP processing
PrimaryLogPG::do_op handles CEPH_MSG_OSD_OP op and will queue it

        in wait_for_all_missing if it is a CEPH_OSD_OP_PGLS for a designated snapid and some object updates are still missing
        in waiting_for_active if the op may write but the scrubber is working
        in waiting_for_missing_object if the op requires an object or a snapdir or a specific snap that is still missing
        in waiting_for_degraded_object if the op may write an object or a snapdir that is degraded, or if another object blocks it (“blocked_by”)
        in waiting_for_backfill_pos if the op requires an object that will be available after the backfill is complete
        in waiting_for_ack if an ack from another OSD is expected
        in waiting_for_ondisk if the op is waiting for a write to complete

#### Peering Messages
See OSD::handle_pg_(notify|info|log|query)

Peering messages are tagged with two epochs:

        epoch_sent: map epoch at which the message was sent
        query_epoch: map epoch at which the message triggering the message was sent

These are the same in cases where there was no triggering message. We discard a peering message if the message’s query_epoch if the PG in question has entered a new epoch (See PG::old_peering_evt, PG::queue_peering_event). Notifies, infos, notifies, and logs are all handled as PG::RecoveryMachine events and are wrapped by PG::queue_* by PG::CephPeeringEvts, which include the created state machine event along with epoch_sent and query_epoch in order to generically check PG::old_peering_message upon insertion and removal from the queue.

Note, notifies, logs, and infos can trigger the creation of a PG. See OSD::get_or_create_pg.


### [Log Based PG](http://docs.ceph.com/docs/master/dev/osd_internals/log_based_pg/)

Currently, consistency for all ceph pool types is ensured by primary log-based replication. This goes for both erasure-coded and replicated pools.