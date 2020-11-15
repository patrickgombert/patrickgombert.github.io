---
layout: post
title: Using Log-Structured Merge-Trees
date: 2020-04-25
---

I've been working with log-structred merge-trees (LSMTs) at my day job for years and my guess is that a lot of people might also be working with LSMTs without necessarily knowing it. Databases like Cassandra, HBase, LevelDB, RocksDB, and others all use log structured merge trees under the hood to store data on disk. While it's not necessary to be familiar with every subsytem of a particular database, there's a lot to be gained from understanding what's happening beneath the hood. Hopefully this can act as a nice introduction and provide a useful mental model for LSMTs.

### Tradeoffs ###

LSMTs store sorted key/value pairs, just like a [b-tree](https://en.wikipedia.org/wiki/B-tree). The question then becomes, why would someone reach for a log-structured merge-tree over a traditional b-tree? Well, let's look at some of the tradeoffs.

First, and probably most important, is that writes to LSMTs are in constant time as opposed to O(log n) time with a b-tree. This is a _very_ nice attribute to have and for systems where fast writes are important. Since deletes are a special case of writes (more on this later), deletes are also constant time with LSMTs.

However, reads in a LSMNT are O(n) as opposed to O(log n) in a b-tree.
One difference worth calling out, which we'll get to in more detail, is that LSMTs have more moving parts and tend to have more knobs to turn. This isn't good or bad per se, but it a tradeoff worth considering. Furthermore, LSMTs are immutable on disk whereas this isn't necessarily, or practically, true for b-trees.

Keeping the tradeoffs in the back our minds, let's now walk through the layers contained within LSMTs.

### Memtables ###

The first layer is the memtable which you can probably guess by the name is an in memory store. All writes are written into the active memtable and all reads start by searching the available memtables before going to structures on disk.

Memtables are mutable, ephemeral structures which are dumped to disk periodically. When this occurs, a new, empty memtable is created and begins accepting writes in place of the previous memtable. Typically, databases will expose some kind of setting for the maximum size of a memtable and once that maximum size is reached the flushing process will trigger.

### Static Sorted Table ###

The format used when a memtable is flushed to disk is a series of files known as Static Sorted Table (SSTs). Contained within each SST are key value pairs sorted lexicographically. Each file is immutable so once a memtable is flushed the files will not change until they reach their end of life.

If a user wishes to query for a given key the file that contains the key's range can be binary searched until the corresponding key is either found within the file or not found within the file. Since each file contains a run of keys and keys do not repeat between files, we know that if a key is not in a file responsible for its range then the key does not exist in the set of SST files.

#### Levels ####

As more and more memtables are flushed to disk we end up with more and more data that needs to be searched when executing queries. In order to keep "fresh" data more readily available SSTs can be configured to be stored on levels. Each level comprises of a set of SSTs with newer files in the higher levels and older files in the lower levels. Since keys can be duplicated between levels, higher levels have precidence when searching for a given key. The number and size of each level are typically configurable for a given database.

#### Tombstones ####

Since our data on disk is immutable and exists at multiple levels deleting data becomes a bit of a challenge. Furthermore, O(log n) writes are a nice property, so wouldn't it be great to extend that to deletes as well? This is where tombstones come in. Tombstones are records which are written to mark the deletion of a key. Intead of finding all instances of a key in the database, LSMTs write a new piece of data indicating a deletion. Since reads find the first instance of a key and ignore data at lower levels, they can follow the same pattern for tombstones. Once a read finds a tombstone it knows to return a not found response.

#### Compaction ####

LSMTs can store information on disk that is no longer or relevant and no longer accessible to the user. Keys can be repeated between levels as new values are assigned to keys. Key/value pairs that have been tombstoned can appear and we can even have the cases where tombstones are not longer relevant when the key is rewritten after deletion. Needless to say all of this data takes up space when it's not necessary to do so. However, since our data is immutable that presents a bit of a conundrum.

This is where compaction comes in. Compaction is a background process where each level is "merged" with the lower level to product a new level. Each key will have the freshest value, or be removed if applicable. Once the merging process is done and new SSTs are produced the database can swap to using the news set of SSTs and remove the old SSTs in order to reclaim space.

#### Range Indexes ####

While point reads are possible and a part of LSMT APIs, LSMTs are at their best when performing range scans. Range scans are what they sound like, a user specifies a begin and end range and are given a cursor which allows data to be pulled at the caller's leisure and the LSMT will track the progress through the range.

Under the hood, the cursor tracks the position in each level. When new data is requested it peeks forward in each level and finds the next key in order and moves the pointer forward for the elvel with the next key. If multiple values are found for keys on two or more levels it takes the highest level's value since it is the freshest value.

#### Manifest ####

Manifests hold all of the metadata for a specific iteration of data on disk. By an iteration, I mean a stable state that is a snapshot between compactions and memtable flushes. Manifests themselves are immutable and essentially maintain the files in levels 0-n. One interesting property is that multiple iterations of the LSMT can exists on disk, with multiple manifests, and the running system maintains a pointer to the 'live' manifest. This means that compactions, memtable flushes, backups and so forth can happen while allowing data to be 'immutable'. Old manifests and old files can be garbage collected, so to speak, as time moves forward and they're not longer referenced.

### Common Additions ###

Since memtables aren't persistent it's common to include a Write Ahead Log (WAL) which will store writes in the case of an unclean shutdown.

Another common addition is the use of bloom filters for point reads. Since we only know the range of an SST file, but not it's exact contents, we can give each file a bloom filter which will help us determine is a given file definitely does not have a key. That way we can skip searching the file and speed up our execution time.

One more interesting feature that appears commonly is write stalling. Write stalling is a form of backpressure that can be applied when writes are happening to fast for the system to handle. When conditions arise such that writes need to be slowed down or stopped the memtable can simple wait to processes requests and thus apply backpressure up to the application layer.

### Using Log Structured Merge Trees ###

Hopefully this provides a nice high level mental model for understanding and using log structures merge trees. It's worth taking the time to familiarize yourself with the various knobs and features available in databases that make use of LSMTs.
