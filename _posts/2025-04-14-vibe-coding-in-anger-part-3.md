---
layout: post
title: "Vibe Coding In Anger: Part 3"
date: 2025-04-14
---

[Previously](https://patrickgombert.com/2025/vibe-coding-in-anger-part-2/) in our quest to vibe code a little time series database we started building the _Storage Engine_. We built a memtable, a sorted-string table, and a process to flush the memtable to a sorted-string table. As always, the full codebase is available on [github](https://github.com/patrickgombert/vctsdb/).

This time I wanted to start with something I know that the agent can knockout in order to get the juices the flowing.

```
warning: `vctsdb` (bin "vctsdb") generated 38 warnings (12 duplicates)
```

I start off by just asking if it can fix the warnings.

> The project generates some warnings when it's built. Would you be able to fix the warnings?

It thinks for a long time and the model I'm using shares internal thoughts. For like almost every warning it thinks "I can just use #[allow(dead_code)]", which is sort of funny. It then comes to the same conclusion as me and adds.

> Refinement: Instead of liberally applying #[allow(dead_code)] to public API elements just because the library target doesn't use them currently (but tests or binaries might, or they are intended public API), I will focus on removing truly unused internal items (imports, variables, private helpers)

This is super interesting! It's mimicing the way a human might think about it, although I would like to be believe that most people would just go straight to actually removing dead code. Anyway, it starts by very sensibly running _cargo fix_ which gets pretty far. Then it starts to make code changes to address the remaining warnings and enters a spin cycle, mostly confusing itself around how to use _#[tokio::test]_ vs _#[test]_. Then the AI sort of abrubtly claims that it's done with a disclaimer.

> The remaining warnings are mostly dead_code warnings. As discussed before, these often flag code that's intended for future features or is only used within our test modules. Since the core logic seems sound and all tests pass, it's generally best to leave this code in place for now. We can revisit these warnings later if they persist after more features are implemented.

In the end we went from 38 warnings to 27. If I weren't vibe coding I would push harder on the idea of [YAGNI](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it), but for a vibe coder I don't really believe that dead code matters at all. I just wanted to see what it would do if I asked it to remove the warnings. As with everything else in this project, this was interesting! For some reason it seemed to think much more about why I'm asking and how it should approach the problem than with other prompts. I'm glad I asked and got to see this.

Now, let's move on to finishing _Phase 2: Storage Engine_ as we have the _Time-Based Indexing_ section left in the TODO. Again, I'm going to only prompt for a subsection at a time.

> Please start on the 'Index Structures' subsection of Time-Based Indexing

It does a fairly substantial rewrite to the _MemTable_. It has abandoned using _MemTableEntry_ to store points internally and now stores a series of points inside of a new _SeriesData_ struct. Every data point gets stored by the series name in a _SeriesData_. This makes sense, as now the _get_series_range_ consults the specific _SeriesData_. Conversely, the more general _get_range_ has become a little more awkward. Before, those were reversed and _get_series_range_ was the more awkward of the two. The _get_range_ function will now return each series' range sequentially. The datapoints won't be ordered totally by time in the resulting vec, but rather ordered for each time series and each time series is sequential in the output. If we have 'TS1' with datapoints at timestamps 1 and 4 and 'TS2' with datapoints at timestamps 2 and 3 the output will be: 'TS1 : 1', 'TS1 : 4', 'TS2 : 2', and 'TS2 : 3'.

It also included the minimum and maximum timestamps for each _SeriesData_ which allows for some useful bounds checking for these functions. There is no need to make a copy of a timeseries in the _get_series_ and _get_series_range_ functions if the range queried doesn't fall within the series range in the memtable. After these changes the tests even pass on the first try!

What's interesting is that the agent is now going very slowly and not very confidently. It only completed _MemTable time range tracking_ and didn't even go on to _SSTable metadata catalog_ even though it's in the subsection I asked it to work on. I let it know to check off the first item it completed and move on to the second item in the subsection. Then something really messy happens! It invented a new TODO item, checked it off, and then ran forward on the metadata catalog implementation.

```
### Time-Based Indexing
- [x] MemTable time range tracking
- [ ] Index structures:
  - [ ] MemTable time range tracking
  - [ ] SSTable metadata catalog
```

I ask it about this!

> I believe that you invented a new TODO item rather than checking off the existing one. 'MemTable time range tracking' already existed under 'Index structures' and I believe that you have completed that item. Could you remove the newly added redundant item and check off the original item?

It obliges and we're back to _SSTable metadata catalog_ and the compile -> fail -> test -> fail cycle. I am taking a peek at the implementation, but I'm not looking closely because I do expect it to change on the way to green. Eventually, once it's back to green it adds a little flourish in our chat. I think that it has PTSD from me asking to fix the warnings.

> There are some warnings about unused code, but those are expected since we haven't implemented all features yet. We can address those warnings as we implement more functionality.

It's ok little buddy, I trust you.

Looking at the _metadata catalog_ there is a lot of overlap with the data stored on the in-memory _SSTable_. It has its own _BlockInfo_ which looks suspiciously like the _BlockMetadata_ and in fact the AI generated some code that just copies it.

```rust
let metadata = table.metadata.read().await;

// Convert block metadata to BlockInfo
let blocks = metadata.blocks.iter().map(|block| BlockInfo {
    offset: block.offset,
    point_count: block.point_count,
    start_timestamp: block.start_timestamp,
    series_names: HashSet::new(), // Will be populated during block reads
}).collect();
```

The encapsulation of data is not clear at all between the various SSTable and SSTable metadata structs. Similar to the interface it created for managing blocks, each _SSTable_ is added to the _SSTableCatalog_ when it is passed to _add_table_. This function will mutate the struct and allow the table itself to be queried via _get_tables_in_range_ (the range here being time), _get_tables_for_series_, and _get_all_tables_.

I think that I can see where it's going and you can sort of squint and see the shape of a thing that wires this all together. When you start the database and read from disk, all of the blocks in the file get pulled into memory. Those in-memory blocks form the _SSTable_. Once all of the _SSTable_ files are in memory they are feed to the _SSTableCatalog_. Then you'll be able to route the query runner to the appropriate places across the _MemTable_ and various _SSTable_ via the _SSTableCatalog_. It's slowly circling around coming together.

Speaking of a query runner, that leads us to the next subsection _Query routing_. Let's jump in and see if it does, finally, start sticking the pieces into place.

> Let's move on to the 'Query Routing' subsection and begin with 'Time range overlap detection'. After that is complete then we can move on to 'Multi-level search'.

It generated a new _Query_ struct in _storage/lsm_ which will be the query interface for the raw data. This _Query_ is at a different level than the user facing query langauge that is coming later. Think of this as the lower level query that finds the right places to consult to find data. Anyway, at this point you know the drill, we're back in the cycle to green. It gets there rather quickly this time! It only added two tests. One tests if two _TimeRange_ overlap, and the other tests if a _TimeRange_ contains a specific point. This is pretty concerning as most of the functionality added is for the _QueryRouter_ which is looking across the _MemTable_ and the _SSTables_ (but not the _SSTableCatalog_ ???) to locate all of the touch points for a given _Query_. This is totally untested, although I suspect that it might be because it's only partially implemented.

```rust
// Check SSTables
let sstables = self.sstables.read().await;
for sstable in sstables.iter() {
    let metadata = sstable.metadata.read().await;
    // Check if SSTable's time range overlaps with query
    if query.time_range.overlaps(&TimeRange::new(
        metadata.min_timestamp,
        metadata.max_timestamp,
    )) {
        // TODO: Implement SSTable querying
        // For now, we'll just collect from MemTable
    }
}
```

_Multi-level search_ is the next _TODO.md_ item so maybe it's trying to save this implementation for that item. One more oddity before we move on that caught my eye is that the function for querying is called _route_query_, but it returns a _Vec_. It's just querying, although it does some routing under the hood. I tell it to check this item off and we move on.

> Perfect, let's move on to 'Multi-level search'.

It changed the _mod.rs_ to trim down on what's public from the _lsm_ module, which is intersting. Maybe it's starting to feel finalized? _Query_ now has a _with_series_ which is great! It's the same time ranged query, but with the filtering for a specific series. _route_query_ is now much larger as the TODO has been implemented!

There is something so interesting that is happening here and there is a theme developing. When it queries it's purposefully only allowing one _DataPoint_ per timestamp! This is not at all what I would expect. Even within the same series, we might have different tagged entries at the same timestamp. It does something like this:

```rust
let mut results = HashMap::new();

(..)

// Add points to results, newer points (from MemTable) take precedence
for point in filtered_points {
    results.entry(point.timestamp()).or_insert(point);
}

(..)

let mut sorted_results: Vec<DataPoint> = results.into_values().collect();
sorted_results.sort_by_key(|point| point.timestamp());
sorted_results
```

The _HashMap_ usage is just to ensure uniqueness of timestamps, because the output of the function is a _Vec_ of the _HashMap_ values. This is very intentional behavior. It even understood that in this type of scenario the _MemTable_ (aka newer writes) takes precedence. This would make sense with a system that has updates. However, the _DataPoints_ we're adding should be thought of as append only. We're never performing updates. That's why the AI didn't need to implement tombstones (or at least that is the reasoning I'm projecting onto it).

I need a way to poke at this as a vibe coding user, but the interface isn't really there yet for this. I think that this will be easy to show to the agent, but only once it's running and a human has a way to interact with it. For now, it's a series of tests and code that is slowly gelling together. There's no place for a vibe coder to have interacted with the system and have noticed anything yet.

Finally, like I predicted, it added a more substantial test now that the full implementation is in place. It finds data points across both a memtable and a sstable via the _QueryRouter_. Neat!

Finally, to wrap up the _Storage Engine_ I ask it to finish up the final TODO item.

> Please finish up the remaining 'Tests' TODO item

It adds a test for a point query and a range query across the entire LSMT (so across the memtable and a SST), which is great. But then I see the last test which really catches my eye, _test_index_update_consistency_. To my dismay it's not really testing consistency in any way. It's adding new data into the memtable and querying it and then adding a second SST and querying it. This is where I wish I wasn't vibe coding because I would step out and start quizzing the AI on what consistency guarantees it believes that the system offers. Regardless, the tests don't pass so we enter The Cycle. Interestingly, it's the range query that's not passing. It seems to be flailing on how to route a time range to the correct SST(s).

Then it does something that would make me rip my hair out if I had any. Remember earlier how I was complaing about how it captures points in a _HashMap_ and then sorts the values into a _Vec_? Well, it goes and removes the _HashMap_. However, that makes more tests fail because it's not using the newer memtable values for the time (which it shouldn't be using as a key anyway!). The fix is then to go back to the _HashMap_, but read the SSTs first and then overwrite it with the memtable's entries. This, hilariously, gets us back to only 1 failed test.

It then ditches the _HashMap_ again, and I'm getting the fear that I'll have to stop vibe coding. We're back to 4 tests failing and the next proposed solution does this:

```rust
pub async fn route_query(&self, query: &Query) -> Vec<DataPoint> {
  let mut results = Vec::new();
  let mut seen_timestamps = HashSet::new();

  (..)
}
```

Followed by the agent trying to pull a fast one by not running the tests.

> No further changes are needed to the implementation at this point, as both methods are working as intended and are being used correctly in the query router implementation.

I respond by asking a sheepish question:

> Are the tests passing?

Ah well, it doesn't compile. After some time it decides to put  the memtable query first again. This really feels like a death spiral. The agent is such a good developer though as it gives me a very positive status update.

> We're getting closer!

I'm like half paying attention but it's littering access to _seen_timestamps_ everywhere and things have broken down. For the second time during this project I have to stop vibe coding and begin coding coding.

It takes about 10 seconds to understand what happened. It's the timestamp overwriting, which is so painful to see. We have a test that looks like this:

```rust
let series = TimeSeries::new("test_series".to_string()).unwrap();
memtable_guard.insert(&series, &DataPoint::new(150, 1.0, HashMap::new())).await.unwrap();
memtable_guard.insert(&series, &DataPoint::new(200, 2.0, HashMap::new())).await.unwrap();

let sstable = SSTable::new(&sstable_path).unwrap();
let block = DataBlock {
    start_timestamp: 100,
    timestamp_deltas: vec![0, 50],
    values: vec![0.5, 1.5],
    series_names: vec!["test_series".to_string(), "test_series".to_string()],
    tags: vec![HashMap::new(), HashMap::new()],
};
sstable.write_block(block).await.unwrap();

let sstables = Arc::new(RwLock::new(vec![Arc::new(sstable)]));
let router = QueryRouter::new(memtable, sstables);
```

We then have 2 datapoints in the memtable, at timestamp 150 and 200. Then, there are two additional datapoints in the SST at timestamp 100 and 150. Notice that there are two datapoints at timestamp 150. As I've noted, there was a decision made by the AI to only allow one of those writes to be reflected. In this case it would have to be the write from the memtable since that's the newer write. The test then goes on to this piece which passes!

```rust
// Test complete range query
let query = Query::with_series(90, 210, "test_series".to_string());
let results = router.route_query(&query).await;

// Verify all points are present and in order
assert_eq!(results.len(), 3);
assert_eq!(results[0].timestamp(), 100);
assert_eq!(results[0].value(), 0.5);
assert_eq!(results[1].timestamp(), 150);
assert_eq!(results[1].value(), 1.0);
assert_eq!(results[2].timestamp(), 200);
assert_eq!(results[2].value(), 2.0);
```

There is only 1 datapoint in the _results_ at timestamp 150, as expected. It has the correct value of 1.0, which is from the memtable and not the 1.5 from the SST. Great! This is the rest of the test:

```rust
// Test partial range query
let query2 = Query::with_series(120, 170, "test_series".to_string());
let results2 = router.route_query(&query2).await;
assert_eq!(results2.len(), 2);
assert_eq!(results2[0].timestamp(), 150);
assert_eq!(results2[0].value(), 1.0);
```

Between the timestamps 120 and 170 there is only the datapoint at timestamp 150. There is only 1, but it's expecting 2 in this test! Probably because it wrote 2! But also it only asserts one of the values? There are no assertions on _results1[1]_. The LLM seems to have totally confused itself. I change the assertion to say that _results2_ has _len()_ of 1.The tests all pass! I ask it to check off the remainder of _Phase 2: Storage Engine_.

Next time we'll start on _Phase 3: Ingestion Pipeline_, which should provide the interface for pushing data into VCTSDB. The code up through the end of today's blog is [here](https://github.com/patrickgombert/vctsdb/tree/733c7d9ecd1517ada1af425125b5b81976388b02).
