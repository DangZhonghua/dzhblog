
[more](http://docs.ceph.com/docs/master/rados/operations/control/)
1. ceph tell osd.* version
2. ceph tell mon.* version
3. ceph tell mds.* version
4. ceph osd tree
5. ceph osd crush dump
6. ceph osd stat
7. ceph osd map pool object :osdmap e33 pool 'dedup' (1) object 'bigobj' -> pg 1.3cb12612 (1.12) -> up ([0,1,2], p0) acting ([0,1,2], p0)
8. rados --pool poolname put objectname localfilename
9. ceph pg dump
10. ceph osd crush rule ls
11. ceph osd crush rule dump
12. ceph osd crush remove {name}
13. ceph osd crush add-bucket {bucket-name} {bucket-type}
14. ceph osd crush move {bucket-name} {bucket-type}={bucket-name}, [...]
15. ceph osd crush weight-set reweight-compat {name} {weight}
16. ceph pg repair {placement-group-ID}
17. ceph health detail
18. ceph status
19. ceph -s
20. rados list-inconsistent-pg rbd
21. rados list-inconsistent-obj {0.6, pgid}--format=json-pretty
22. ceph daemon osd.0 config show | less
23. ceph osd pool set {pool-name} pg_num {pg_num}
24. ceph osd pool get {pool-name} pg_num
25. ceph pg dump [--format {format}]
26. ceph pg dump_stuck inactive|unclean|stale|undersized|degraded [--format <format>] [-t|--threshold <seconds>]
27. ceph pg map {pg-id}
28. ceph pg {pg-id} query
29. ceph pg scrub {pg-id}
30. ceph pg force-recovery {pg-id} [{pg-id #2}] [{pg-id #3} ...]
31. ceph pg force-backfill {pg-id} [{pg-id #2}] [{pg-id #3} ...]
32. ceph pg cancel-force-recovery {pg-id} [{pg-id #2}] [{pg-id #3} ...]
33. ceph pg cancel-force-backfill {pg-id} [{pg-id #2}] [{pg-id #3} ...]
34. ceph osd find 0(osd id) find the osd location
35. ceph --version
36. ceph mon dump
37. eph mon getmap -o /tmp/monmap //retrieve current map
38. ceph mon stat
39. ceph quorum_status:how the monitor quorum, including which monitors are participating and which one is the leader
40. ceph [-m monhost] mon_status: query the status of a single monitor, including whether or not it is in the quorum.
41. ceph auth ls
42. ceph osd lspools
43. ceph pg dump_stuck stale
44. ceph pg dump_stuck inactive
45. ceph pg dump_stuck unclean
