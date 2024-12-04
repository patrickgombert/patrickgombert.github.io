---
layout: post
title: Using Log-Structured Merge-Trees
date: 2020-02-26
---

I've been working with log-structred merge-trees (LSMTs) at my day job for years and my guess is that a lot of people might also be working with LSMTs without necessarily knowing it. Databases like Cassandra, HBase, LevelDB, RocksDB, and others all use log structured merge trees under the hood to store data on disk. While it's not necessary to be familiar with every subsytem of a particular database, there's a lot to be gained from understanding what's happening beneath the hood. Hopefully this can act as a nice introduction and provide a useful mental model from LSMTs.

### Tradeoffs ###

LSMTs store sorted key/value pairs, just like a [b-tree](https://en.wikipedia.org/wiki/B-tree). The question then becomes, why would someone reach for a log-structured merge-tree over a traditional b-tree? Well, let's look at some of the tradeoffs.

First, and probably most important, is that writes to LSMTs are in constant time as opposed to O(log n) time with a b-tree. This is a _very_ nice attribute to have and for systems where fast writes are important. Since deletes are a special case of writes (more on this later), deletes are also constant time with LSMTs.

However, reads in a LSMNT are O(n) as opposed to O(log n) in a b-tree. 
One difference worth calling out, which we'll get to in more detail, is that LSMTs have more moving parts and tend to have more knobs to turn. This isn't good or bad per se, but it a tradeoff worth considering. Furthermore, LSMTs are immutable on disk whereas this isn't necessarily, or practically, true for b-trees.

Keeping the tradeoffs in the back our minds, let's now walk through the layers contained within LSMTs.

### Memtables ###

The first layer is the memtable which you can probably guess by the name is an in memory store. All writes are written into the active memtable and all reads start by searching the available memtables before going to structures on disk.

...

### Sorted String Tables ###

...

#### Tombstones ####

Since our data on disk is immutable and exists at multiple levels deleting data becomes a bit of a challenge. Furthermore, O(log n) writes are a nice property, so wouldn't it be great to extend that to deletes as well? This is where thombstones come in. Thombstones are records which are written to mark the deletion of a key. Intead of finding all instances of a key in the database, LSMTs write a new piece of data indicating a deletion. Since reads find the first instance of a key and ignore data at lower levels, they can follow the same pattern for thombstones. Once a read finds a thombstone it knows to return a not found response.

#### Compaction ####

...

#### Range Indexes ####

...

#### Manifest ####

Manifests hold all of the metadata for a specific iteration of data on disk. By an iteration, I mean a stable state that is a snapshot between compactions and memtable flushes. Manifests themselves are immutable and essentially maintain the files in levels 0-n. One interesting property is that multiple iterations of the LSMT can exists on disk, with multiple manifests, and the running system maintains a pointer to the 'live' manifest. This means that compactions, memtable flushes, backups and so forth can happen while allowing data to be 'immutable'. Old manifests and old files can be garbage collected, so to speak, as time moves forward and they're not longer referenced.

### Common Additions ###

Since memtables aren't persistent it's common to include a Write Ahead Log (WAL) which will store writes in the case of an unclean shutdown.

### Tuning ###

...
