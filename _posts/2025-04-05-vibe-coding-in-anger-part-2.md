---
layout: post
title: "Vibe Coding In Anger: Part 2"
date: 2025-04-05
---

[Previously](https://patrickgombert.com/2025/vibe-coding-in-anger-part-1/) in our quest to vibe code a little time series database we defined a specification and distilled that into a checklist in _TODO.md_ for an AI agent to work through. We had the AI set up a rust project and then had it implement the Write-Ahead Log (WAL) as its first real meaningful task. This time we'll work on the _Storage Engine_ and implement a Log-Structured Merge-Tree (LSMT). We'll also set up the time based indices we'll use to effectively query the data. Again, I'll be vibe coding and not doing much hands on. I'll mostly let the agent drive and prompt the agent at a very high level on occasion. As a reminder, the code is available on [github](https://github.com/patrickgombert/vctsdb/).

### Phase 2: Storage Engine

I find that I like when I give Cursor one task to do rather than many cohesive tasks at once. In my experience (and I'm probably not doing this totally correctly) it has been the case that the agent can get lost when there's too much to do at once, especially when it relates to pieces of the system fitting together. For that reason I'm going to step through the TODO list slowly, starting with the _MemTable Implementation_.

> We're now moving on to 'LSM Tree Foundation'. Please start on the 'MemTable implementation' subsection. Once that's complete we can move on and hook it up to the SST and flush process.

I've written a [half-baked and half-finished LSMT](https://github.com/patrickgombert/lsmt) so I'm a bit of an expert here. I was feeling myself so my MemTable was an immutable persistent red-black tree and assumed a single writer. The AI decided to wrap a _BTreeMap_ in a _RwLock_. Interestingly, it decided to also wrap a _usize_ in a _RwLock_ independent of the _BTreeMap_ to track the number of entries that have been written. On write both of the data's and size's locks are acquired. On read, only the data's lock is acquired. It also stores all entries by the timestamp as the key, which means that iteration requires calling _get_range_ and then getting a full _Vec_ copy of the dataset. Not to toot my own horn, but the peristent datastructure implementation let callers receive an iterator that didn't require locking and could be held in tact through any modification to the underlying datastructure. This does some of that too, but requires a full copy.

The code written by the AI is failing compilation, but this failure is actually interesting because it's forcing the agent to reconcile the skeleton architecture with the implementation architecture.

> I see there's an error with the LSM module structure. We have both src/storage/lsm.rs and src/storage/lsm/mod.rs files, which is causing ambiguity. Let's fix this by removing the lsm.rs file and keeping the module structure.

I accept and after that it's back to the typicaly fail compilation -> fail test -> fail compilation cycle. It exits pretty quickly, however, and it's back to green! As a vibe coder, this is great, we love green! As a Software Engineer, I have some thoughts that I'll share and then we'll move on and keep them in the back of our head for later.

I wrote a little test, take a second and guess what the result is.

```rust
#[test]
async fn test_memtable_double_insert() {
    let memtable = MemTable::new(1000);
    let series = TimeSeries::new("test_series".to_string()).unwrap();
    memtable.insert(&series, &DataPoint::new(1000, 1.0, HashMap::new())).await.unwrap();
    memtable.insert(&series, &DataPoint::new(1000, 2.0, HashMap::new())).await.unwrap();

    assert_eq!(memtable.size().await, 2);
    let retrieved = memtable.get_range(999, 1001).await;
    assert_eq!(retrieved.len(), 2);
}
```

The size is two, but there is only one entry. Similar to the WAL segment filename issue earlier, it's clobbering metrics that are inserting with the same timestamp. However, this time it didn't get caught in any of the tests Cursor generated, so it's a logic error waiting for the right opportunity to steal your data. With the WAL, if you'll remember, the tests at least failed and I had to momentarily stop vibe coding to fix it. This one really bothers me because it's both losing data that has the same timestamp and is lying about the size because it had decided to track the size independently of the data. It's wrong and it's lying!

The return type of _insert_ is _Result<bool, MemTableError>_ where the _bool_ let's the caller know if the MemTable has exceeded the configured size and it's time to flush the data to disk. In the most degenerate case you could always be flushing exactly one datapoint no matter how big the configured size is. Not great, but also as a vibe coder I might not even ever notice the issue.

There is also no delete or tombstoning, but I think that's a good idea! You don't really delete metrics, unless maybe something goes wrong with the process emitting metrics and you want to clean up.

Before I prompt for the _SSTable Format_ I do notice that it asks for _Columnar storage layout_, _Timestamp delta encoding_, and _File format versioning_. I'm not totally sure where this is going. I don't fully know what _Columnar storage layout_ means exactly. I really don't know what _Timestamp delta encoding_ means, what is the delta of the timestamps and how is that encoded? Who knows! However, _File format versioning_ makes perfect sense and is a good idea. I'm very curious to see what the SST file ends up looking like on disk and how it will efficiently (or not efficiently) find specific metrics.

> Now that we have our MemTable, let's move on to the "SSTable format" sub-section.

It wrote a block based SST format! That's fun! I'll let it go through the compile/test cycle before taking a peek at the implementation. It's struggling hard with the _debug_ attribute for _SSTable_, since it holds an _Arc_ and locks and whatnot. Finally, it manually implements _debug_. It then can't format the SST file correctly and keeps getting EOF errors on read. It's spiraling again, but I haven't looked closely yet. It's littering _flush_ calls all over which is... not confidence inspiring.

But then it makes it to green! I was really afraid that I would have to do more coding coding to fix the problem, but it was able to exit the spiral on its own. Let's take a closer look, but on first glance this looks like it had the right idea.

The interface looks has a function to _create_ the _SSTable_ at a specific file path. You then hand it _DataBlocks_ via the _write_block_ function and it appends that block to the file. It updates the in-memory metadata on the _SSTable_ to capture the block information, but that metadata never is written into the file. When you want to open an existing _SSTable_ you invoke _open_, but this is where it becomes totally incoherent. The opened file initializes its metadata like this:

```rust
let metadata = SSTableMetadata {
    point_count: 0,
    min_timestamp: i64::MAX,
    max_timestamp: i64::MIN,
    series_names: Vec::new(),
    blocks: Vec::new(),
};
```

And then returns this metadata on the _SSTable_ out of _open_. There's a _read_block_ function, but that assumes that the _SSTable_ has the metadata to be able to read a specific block! However, the interface gives you no way to do that.

```rust
let block_metadata = metadata_guard
     .blocks
     .get(block_index)
     .ok_or(SSTableError::InvalidBlockIndex)?;

// Seek to block start
file_guard.seek(std::io::SeekFrom::Start(block_metadata.offset))?;
```

The way, with the current implementation, that it could populate this metadata from disk is to traverse the whole file, because each block is written at various points and you have to find all of them. Ideally a list of blocks with their timestamp ranges and offset in the file would be in the top of the file. Then you could rebuild the useful metadata without having to traverse the whole file. Way further down in the TODO in the _Time-Based Indexing_ section there is a checklist item for _SSTable metadata catalog_, so maybe this is coming later, although I doubt it. The tests are funny because they never actually like _drop_ the _SSTable_ and rebuild the struct from disk. They instead write specific blocks and read them back from disk, but with a _SSTable_ that has retained its metadata in memory.

```rust
// Write the block
sstable.write_block(block).await.unwrap();

// Read the block back
let read_block = sstable.read_block(0).await.unwrap();
```

A better interface might be to accept all of the _DataBlock_ entries on write and then write the file sequentially, starting with the header that contains the block metadata and then persisting each block in _write_block_ like it does currently (sans the metadata in memory stuff). Then, _open_ can read the header with the block metadata and will populate the in-memory struct that can seek to any block, as needed. It also decided to write the tags as JSON, but nothing else as JSON. I love that for it.

Regardless, the tests are green and that means that the vibes are satisfied. I ask it to check off the TODO items and move on to _Flush process_.

> Please check off the items in 'SSTTable format'. Let's move on to 'Flush Process'.

You know the drill, it enters the compile -> fail -> test -> fail cycle. This one is interesting because it is having to modify _MemTable_ in order to make the API work for the new flush process. Thus far, we haven't seen the agent attempt to edit existing code to accomodate new code because each piece has been more or less independent. It's fun to see this show up in the chat!

> Now let's update the FlushManager to work with the updated MemTable

I am not looking at the code yet because the cycle to get to green often involves big code rewrites, but I am genuinely excited to see what it's attempting to do here. It's also noticably slowing down which either means that this is a Tough Problem that requires Thought, or that the backend LLM service is struggling.

It gets back to green, so let's take a look! _FlushManager_ is basically a container for an optional async process that is performing a flush from the memtable to disk. If the async _JoinHandle_ present, then we can't flush again because one is in progress. If it's empty then we can kick one off.

Looking at the async work... there's some issues. Let's start with draining the memtable. The _start_flush_ function takes a _RwLock_ that contains a _MemTable_. This makes sense in the big picture. We have to atomically swap the flushed memtable with a fresh memtable so that any new writes that won't be flushed at this moment will go to the new memtable. The owner of this read-write locked memtable doesn't exist yet in the project, so this flushing construct isn't "wired up" at this time. That's fine, the project is being built bottom up.

Now, I mentioned the atomic swap for the memtable and it's important to note that this is a place where writes can be lost if done incorrectly. I'll cut some code out of this snippet, but I have faith that you all can spot the issue.

```rust
// Take a read lock on the MemTable
let memtable_guard = memtable.read().await;
let data = memtable_guard.get_data().await;

// Create a new empty MemTable for atomic swap
let new_memtable = MemTable::new(memtable_guard.capacity());

// Write all data points to the SSTable
for (series_name, points) in data {
    (..)
}

// Atomically swap the MemTables
drop(memtable_guard);
let mut memtable_guard = memtable.write().await;
*memtable_guard = new_memtable;
```

Does this lose writes. You bet! Cursor added _get_data_ to the _MemTable_ and that makes a copy of the internal _HashMap_. An order of operations with this setup that one could imagine would be:

- Make a copy of the original memtable's contents
- Accept a new write to the original memtable
- Write the copy of the contents to disk
- Swap a new memtable with the old memtable

Where did the write in step 2 go? It's gone! The AI did write "Take a read lock on the MemTable" as a comment, so maybe it was confused here? I tried to prove the write loss in test, but we can't actually read back SSTs from disk yet. So I got a little bit creative. I inserted a _println!_ into the flush function that shared the size of the data along with a 1 second sleep after the data read. Then I wrote this test:

```
#[tokio::test]
async fn test_write_loss() {
    let temp_dir = tempdir().unwrap();
    let mut flush_manager = FlushManager::new(temp_dir.path().to_path_buf());
    let memtable = Arc::new(RwLock::new(MemTable::new(1000)));
    let series = TimeSeries::new("test_series".to_string()).unwrap();

    memtable.write().await.insert(&series, &DataPoint::new(1000, 1.0, HashMap::new())).await.unwrap();

    flush_manager.start_flush(memtable.clone()).await.unwrap();

    memtable.write().await.insert(&series, &DataPoint::new(1001, 2.0, HashMap::new())).await.unwrap();

    flush_manager.wait_for_flush().await.unwrap();

    assert!(memtable.read().await.is_empty().await);
}
```

When I run I see:

```
data size: 1
test storage::lsm::flush::tests::test_write_loss ... ok
```

There are two writes in the test, but only 1 is in the dataset flushed to disk. At the end of the test there are no items in the memtable. The second write disappeared. As with the memtable using the timestamp as a key internally and losing same timestamp writes, this is another instance of something waiting for the right moment to drop data (this time it's a race condition). I do wonder if there was a way to read the flushed SST back if the AI would have written a test to do so. It _might_ have caught it in that case, although you have to craft a specific test that writes after flush. The solution here is to atomically swap the memtables first and then iterate over the data from the old memtable since it'll no longer be accepting writes.

The other issue that jumped out to me is that there are no levels to the SSTs, and no compaction. Way down in _Phase 6_ there is a checklist item for _Background Compaction_ so we might be coming back to this. Right now the flush mechanism writes a single SST file per flush. The way this typically works is that each layer has a series of files, and inside of those files are ordered keys. You could imagine SST file layouts that look like this:

```
| File 1   | File 2    | File 3    |
-----------------------------------| etc
| T1 - T10 | T11 - T20 | T21 - T30 |
```

You would sequentially flush these files by some kind of configured file size. The next time you flush you would put those files to a layer 2 and the new files would be layer 1. Eventually you would compact these layers and combine the entires and write new files. If we had layers that looked like this:

```
| File 1   |
|----------|
| T9 - T12 |

| File 1   | File 2   |
|----------|----------|
| T1 - T5  | T6 - T10 |
```

We could imagine these compacting into a new layer where the dataset is ordered and it's easy to find a specific timestamp.

```
| File 1  | File 2  | File 3    |
|---------|---------|-----------|
| T1 - T4 | T5 - T9 | T10 - T12 |
```

Every flush in the vibe coded implementation, as it exists, just makes a new file. We could imagine files looking like this after 3 flushes, as an example:

```
| File 1   | File 2   | File 3   |
|----------|----------|----------|
| T1 - T10 | T8 - T12 | T9 - T12 |
```

There's no coherent way to find things in these files. If you want to find a time series data point for T10 you would consult every file. But again, this issue is less concerning because expanded funcionality might just be coming later. I will try and embue the correct vibes to make this come true.

This seems like a good place to set this down, but we'll have to break _Phase 2_ into two parts. We're now able to accept writes to a memtable, and perform a flush to disk. The code up to this point is [here](https://github.com/patrickgombert/vctsdb/tree/cf4524d29643b46b3e9b18774db7c4e0fed7684b). Next time, we'll implement _Time-Based Indexing_ which will have to search for datapoints across the memtable and any flushed sstables.

