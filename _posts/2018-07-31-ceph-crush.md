---
layout: post
title: "My First post"
date: 2018-07-31 23:29:01
categories:
- Foo
tags:
---

#Ceph data locator
Ceph rados is object data storage cluster:the physical storage is osd(like HDD) and
Placement Group(PG). One OSD can serve many PG. One PG can reside on multiple OSD, like
replicate pool (one object can be multiple copies for preventing data loss).
But how to decide which osd will store one certain object: Rados use two layer map to
locate the object storage service(OSD), the answer is: calcualtion.

Every thing which will into rados is object. Object has name, like "object1".

The first calculation is: calculte WHICH pg will store the object:
hash("object1") = ps, the ps is the pg's seed. PG = {poolid, ps, preferred}

The second calculation is: calculate which osds will store the object:
the PG's ps will be as seed be passed into crush algorithm to calculate the OSDS.