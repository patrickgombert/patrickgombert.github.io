---
layout: post
title: "Vibe Coding In Anger: Part 6"
date: 2025-06-17
---

[Last time](https://patrickgombert.com/2025/vibe-coding-in-anger-part-5/) on our journey to build a vibe coded time series database we completed the _Query Parser_, including the lexer and the ability to generate an AST for the query language. This time we'll be working on the _Execution Framework_ which should wrap up _Phase 4: Query Foundation_ from the TODO. The AI will be generating the query planner and the framework for executing the queries. It should be fun! As always, the code is available on [github](https://github.com/patrickgombert/vctsdb).

Jumping right in I decide to give it a little warning about jumping ahead since it has done that on a few occasions.

> Please start on the 'Query planner' TODO items in the 'Execution Framework' section. Please don't move on to any other tasks until 'Query planner' has been completed.

It generated two new pieces of code, a _IndexInfo_ in the _storage_ module and a _QueryPlanner_ in the _query_ module. Let's start by looking at the _IndexInfo_ since it's going to be used by the _QueryPlanner_.

```rust
pub struct IndexInfo {
    pub name: String,
    pub time_range: TimeRange,
    pub tag_keys: Vec<String>,
    pub estimated_rows: usize,
}
```

There's an index over some _TimeRange_, in this case always an absolute time range, and in that time range are some known tag keys. You can ask some obvious questions of the _IndexInfo_ such as 'does my provided time range intersect with your timerange?' and 'does the index satisfy this tag filter?' which leads you to the ability to skip querying certain indices (this is a new concept?) when query planning (I assume). The various _IndexInfo_ are never populated or updated. They're disconnected from the flushing process and really it looks like a lower fidelity version of the _SSTableCatalog_ which is also disconnected from everything. Nothing is connected. Well, it's even worse, the _QueryPlanner_ is connected to the _IndexInfo_ which is connected to nothing.

It would make much more sense to populate and actually use the _SSTableCatalog_ and have the _QueryPlanner_ make use of that. The _SSTableCatalog_ doesn't really know about keys, but it does know about series names, so I guess we really want some kind of combination where the _SSTableCatalog_ can track tags. We also have the _QueryRouter_ in the _storage_ module which is also hooked up to nothing, but can route to the correct memtable/SST based on a time range and a series name. We have concentric data encapsulation which is starting with the LSMT core and losing fidelity outwards to the _IndexInfo_. None of it is actually wired together. As a vibe coder none of this apparent to me, everything I glance at looks right enough.

Looking at the _QueryPlanner_ we have a thing which can take the _Query_ from the AST and produce a _QueryPlan_ which looks like this:

```rust
#[derive(Debug, Clone)]
pub struct QueryPlan {
    pub index_selections: Vec<IndexSelection>,
    pub group_by: Vec<String>,
    pub order_by: Vec<(String, bool)>,
    pub limit: Option<usize>,
    pub offset: Option<usize>,
}
```

The _QueryPlan_ holds on to the indices that could contain data (I presume) and then holds what is essentially the same info that came in on the _Query_ (the AST Query, not the storage Query) but without the type information. This all feels sloppy to me. I know that the entire project fits into context. It's greenfield and not really referencing anything outside of the project, and yet nothing feels like it's supposed to fit together. It's many differnt jigsaw puzzles of nearly identical pictures mixed together. Which _Query_ am I supposed to use? Do they fit together? No matter how hard you squint you can't see the pieces fitting correctly.

Anyway, moving on to the remainder of _Execution Framework_ we have the _Execution Framework_ and _Tests_ sections left, so let's knock those out.

> Please complete the 'Execution Framework' section of the TODO

It then did something funny and marked the TODO items done and said, in the extremely verbose LLM way, that all of the items were completed. I don't think so! Since I'm vibe coding, maybe I'll just remark on the fact that I didn't see any code generated.

> Have the 'Parallel data fetching' and 'Early result pruning' sections been completed? I didn't see any new code written.

After some back and forth it acquieses and writes some code as well as checks the TODO items off again. We now have a _QueryExecutor_!

```rust
pub struct QueryExecutor {
  /// The active MemTable
  memtable: Arc<RwLock<MemTable>>,
  /// The SSTable catalog
  sstables: Arc<RwLock<Vec<Arc<SSTable>>>>,
  /// Execution configuration
  config: ExecutionConfig,
  /// Current memory usage
  memory_usage: Arc<Mutex<usize>>,
  /// Cancellation flag
  cancelled: Arc<Mutex<bool>>,
}
```

If you remember, the storage module has a _QueryRouter_ which looks like this:

```rust
pub struct QueryRouter {
  /// The active MemTable
  memtable: Arc<RwLock<MemTable>>,
  /// The SSTable catalog
  sstables: Arc<RwLock<Vec<Arc<SSTable>>>>,
}
```

You'll rememvber that the _QueryRouter_ returns a collection of _DataPoint_. Looking at the new _QueryExecutor_ it also returns a collection of _DataPoint_, but wrapped in a _ExecutionResult_. The _QueryRouter_ and _QueryExecutor_ do not interact at all. In fact, the two more or less have the same logic where they first consult the memtable and then walk the sstables. The logic to inspect each block in the sstables is almost exactly the same (it's worth looking at the code for [QueryRouter](https://github.com/patrickgombert/vctsdb/blob/9efa156851374f3c0a56cee2220282131a7f7d5f/src/storage/lsm/query.rs#L83) and [QueryExecutor](https://github.com/patrickgombert/vctsdb/blob/9efa156851374f3c0a56cee2220282131a7f7d5f/src/query/executor.rs#L108)). All in all the _QueryExecutor_ is a more featureful _QueryRouter_. So, what else is happening in this thing? There is a _ExecutionConfig_ which has the following knobs:

```rust
pub struct ExecutionConfig {
  /// Maximum number of concurrent tasks
  pub max_concurrent_tasks: usize,
  /// Memory limit in bytes
  pub memory_limit: usize,
  /// Timeout for query execution
  pub timeout: Duration,
}
```

The _QueryExecutor_ is meant to be instantiated per individual query, so this config can be set per query. That's neat and makes it easy to tweak these values in a live system. Now, looking at the knobs _max_concurrent_tasks_ is never referenced. There is parallel query execution against multiple SSTs in _execute_query_internal_ where it looks like this config would be used. You might wait for a task to finish when the max concurrent tasks have been reached before starting a new task. That would make sense to me, but it's not set up that way. Currently, it just executes an unbounded number of concurrent tasks regardless of the config. Similarly, the _QueryExecutor_ has a _memory_usage_ member which is initialized to 0. Inside of _execute_query_internal_ we check the value of _memory_usage_ vs the configured limit:

```rust
let mut usage = memory_usage.lock().await;
if *usage > self.config.memory_limit {
  return Err(ExecutionError::MemoryLimitExceeded);
}
```

This would be interesting if _memory_usage_ were ever updated. It's not even clear necessary what it's supposed to be counting. I suppose it's the total in memory size of the _DataPoint_ that are being collected. Finally, there's a _timeout_ which is handed off to tokio (which is an asynchronous library in rust). That part is neat because it is a timeout for the process that itself spawns n processes. If the outermost process times out it gives up and returns to the user. One out of three config values doing something ain't bad.

Looking at one of the tests we can get a feel for the interface:

```rust
// Create executor
let config = ExecutionConfig {
  max_concurrent_tasks: 2,
  memory_limit: 1024 * 1024, // 1MB
  timeout: Duration::from_secs(5),
};
let executor = QueryExecutor::new(memtable, sstables, config);

// Execute query
let mut query = Query::new();
query.from = "test_series".to_string();
query.time_range = Some(TimeRange::Absolute { start: 400, end: 1100 });
let results = executor.execute_query(&query).await.unwrap();
```

Remember, this is the AST's _Query_, not the storage's _Query_. I think that we have the components where, with some glue, we could actually run this thing to some extent. Like I could envision the glue code needed to accept metrics via the ingestion _Parser_, have that go into the _storage_, and then throw a string query at the lexing code which can then invoke the _QueryExecutor_. I'm not saying that the glue code is trivial, or that these pieces fit togther in a coherent way (theme of this post), but that you can see it from a distance.

Next time, we'll expand on the _Query Foundation_ by starting on the _Aggregation System_. The power of a time series database is in aggregating the data points! That should be exciting! For reference, the code up to this point can be found [here](https://github.com/patrickgombert/vctsdb/tree/9efa156851374f3c0a56cee2220282131a7f7d5f).
