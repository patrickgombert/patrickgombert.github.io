---
layout: post
title: "Vibe Coding In Anger: Part 4"
date: 2025-04-26
---

[Last time](https://patrickgombert.com/2025/vibe-coding-in-anger-part-3/) on our quest to vibe code a toy time series database we finished up the _Storage Engine_. We now have a Log-Structured Merge-Tree implementation which can persist datapoints and query for ranges of timestamps against those datapoints. This time we'll be working on _Phase 3: Ingestion Pipeline_ which can be found in the TODO checklist. Once this phase is finished we should be able to start pushing data into vctsdb from other processes. My goal is to have Cursor write a shell script that we can push the results of _iostat_ or something in vctsdb, and hopefully we should see files get written to disk, metrics update, etc.

Looking at the TODO, first up is _Parser Trait_. Like the _Data Structure Implementation_ from _Phase 1_ I expect the trait and any structs to be slim. Maybe we'll see some minimal error handling.

> Let's move on to the first subsection of 'Phase 3: Ingestion Pipelines', specifically the 'Parser trait' subsection. Please go ahead and implement this and check off the TODO items.

It jumped ahead a bit and completed both _Parser Trait_, and _JSON parser_, but that's fine. JSON will be the first implementation, so we can exercise the trait. _Parser_ is a very simple trait, with a single _parse_ function and a _parse_batch_ convenience method. There is a _ParserError_ enum that will be common across all implementations. There's also a _supported_formats_ function which returns a collection of strings. For the json parser it returns _application/json_ and _json_. This feels weird, because I don't see how this is going to fit in to a larger component. Will the protocol have a handshake where the client specifies the type or something? It's unclear to me now. The parser trait is just parsing bytes so we'll definitely need the protocol to specify the format and structure of those bytes. We'll have to wait and see where it goes.

You know the deal though, it enters The Cycle. However, this time it exits pretty quickly with only one issue regarding parsing timestamps! By inspecting the tests we can see what the JSON format will look like:

```json
{
  "timestamp": 1000,
  "value": 42.5,
  "series": "test_series"
}
```

It's not totally clear where the tags go, because the tests don't have any tags and the implementation does this:

```rust
let mut tags = HashMap::new();
if let Some(series) = obj.get(self.field_mapping.get("series").unwrap()) {
    if let Some(series_str) = series.as_str() {
        tags.insert("series".to_string(), series_str.to_string());
    }
}

points.push(DataPoint::new(timestamp, value, tags));
```

It looks for a key named _series_ and pushes that as the only tag? I don't see a way for other tags to get in through the parser. That's a key (no pun intended) feature so this will surely come up once we start pushing data in. It did, however, add this feature which made me laugh:

```json
let mut field_mapping = HashMap::new();
field_mapping.insert("timestamp".to_string(), "ts".to_string());
field_mapping.insert("value".to_string(), "val".to_string());
field_mapping.insert("series".to_string(), "name".to_string());

let parser = JsonParser::with_field_mapping(field_mapping);
let input = r#"{
    "ts": 1000,
    "val": 42.5,
    "name": "test_series"
}"#.as_bytes();
```

Now this is configuration! We love to see it!

Next, it added some _Validation Middleware_ which validates an individual _DataPoint_. The _Parser_ conveniently returns a _Vec_ of _DataPoint_ so I assume that the _ValidationMiddleware_ will be hooked up to the output of the _Parser_. It has a config with some pretty outrageous defaults, although they'll work for what we're doing now:

```rust
max_series: 100_000,
max_tag_values: 10_000,
max_value: f64::MAX,
min_value: f64::MIN,
```

However, the implementation for this is not what I expected. It's planning on keeping the number of values for each tag and the number of unique series in-memory. These in-memory counts get updated each time a _DataPoint_ is passed to _validate_ which looks like something like this (note that _series_ is a special case tag here as well):

```rust
let series_name = point.tags().get("series")
    .ok_or_else(|| ValidationError::ValueSanityCheck("Missing series tag".to_string()))?;
let series_count = self.series_counts.entry(series_name.clone())
    .or_insert(0);

(...)

*series_count += 1;

for (key, value) in point.tags() {
  if key == "series" {
      continue; // Skip series tag as it's handled separately
  }

  let tag_values = self.tag_value_counts.entry(key.clone())
      .or_insert_with(HashMap::new);
  let value_count = tag_values.entry(value.clone())
      .or_insert(0);

  (...)

  *value_count += 1;
}
```

That's fine I guess. If we remember back to the _Storage Engine_ there were locks which made me think that it wanted to be able to handle multi-threaded inserts coming into the data storage layer. These counts, however, are not thread safe. I guess we have to assume that all _DataPoints_ move through _validate_ sequentially and either there is an enforced single writer or this is some kind of locking or queueing ahead of validation. There's also no mechanism currently to reload this data on start, it's completely ephemeral. There is no indication that the validation will work across process restarts.

The tests are failing after adding validation, so once again it enters The Cycle. I'm not paying close attention because it's doing pretty substantial code changes in response to repeated failures. After a few test runs Cursor somes to the conclusion that it needs to check unique series and unique tags in the implementation (or at least that's what it is being claimed). It's taking a long time. The repeatedly failing test is checking for cardinality of the tags and it can't seem to get it right. Finally it just gives up and the tests are still failing.

> Based on the code review, here's a comprehensive summary of the validation system: (...)

Ok. After half paying attention I think I know exactly what's wrong (and it's a really dumb error). Everything is in one big test, _test_validation_middleware_ and it starts by validating that the series cardinality is exceeded. Then it calls _validator.reset()_ to clear all of the in-memory tracking I mentioned before. Next, the test inserts a new tag and I guess it was assuming that the tag counts had not reset. But they had, because it's one big test and reset was explicitly called. I just throw in some tags to get us moving again, but I am trying to not fundamentally change any of the vibe code. I just want green tests before we move on. The irony is that the next TODO items are explicitly for adding more tests.

> Let's move on to the 'Tests' section so that we can finish 'Base Ingestion Interface'.

It generates some redundant-ish tests, but does add one interesting new test named _test_ingestion_throughput_.

```rust
// Measure parsing throughput
let start = Instant::now();
let points = parser.parse_batch(&inputs).unwrap();
let parse_duration = start.elapsed();
let parse_throughput = batch_size as f64 / parse_duration.as_secs_f64();

println!("Parsing throughput: {:.2} points/sec", parse_throughput);
println!("Validation throughput: {:.2} points/sec", validate_throughput);

// Assert minimum throughput requirements
assert!(parse_throughput > 1000.0, "Parsing throughput too low");
assert!(validate_throughput > 1000.0, "Validation throughput too low");
```

It wants to parse and validate 1,000 data points per second. That's a fun test for it to write, although those values seem somewhat low. Once more things are firmly in place it would be a fun exercise for me to ask the agent to validate some performance characters of the codebase in this type of fashion. Anyway, the tests are green on the first try! We check off the last section TODO items for _Base Ingestion Interface_ and we're ready to move on to _Pluggable Format Support_!

> Please move on to the 'Pluggable Format Support' section. Let's begin with only the 'Registry System' subsystem.

Even though it first fails to compile and then fails its tests, I'm still peeking at what it's doing here. The TODO has a checklist item for _Priority handling_ so it's not surprising that it's taking that very literally. I don't remember if I asked for some kind of priority handling in the original prompting setup or if that was an LLM creation. If I did ask for priority, I almost certainly was thinking of graceful degradation scenarios where we might want to save certain metrics when shedding load or something like that. Anyway, what the vibe coding LLM took it to mean was order the parsers by their priority. We'll see how the ordering is used in a second.

Anyway, it pairs a parser with a priority which creates a new _ParserEntry_ concept.

```rust
/// ParserEntry combines a parser with its priority
struct ParserEntry {
  parser: Arc<dyn Parser + Send + Sync>,
  priority: Priority,
}
```

These are stored in the _Parser Registry_.

```rust
/// ParserRegistry manages registered parsers and their priorities
pub struct ParserRegistry {
  /// Map from format name to parser entries
  parsers: RwLock<HashMap<String, Vec<ParserEntry>>>,
  /// Default parsers to try when format is unknown
  default_parsers: RwLock<Vec<ParserEntry>>,
}
```

You can either _get_parser_ by one of the _supported_formats_ strings or you can _parse_with_autodiscovery_ by throwing bytes at parsers until one succeeds. This is where the priority comes in, the order in which parsers are attempted are by their priority.

```rust
// Try each parser in priority order
let mut last_error = None;
for entry in default_parsers.iter() {
  match entry.parser.parse(input) {
    Ok(points) => return Ok(points),
    Err(err) => last_error = Some(err),
  }
}
```

If I'm being honest, that's sort of a goofy feature. But it is literally following the TODO and implementing _Format Autodiscovery_. I definitely didn't want this and am sort of upset that I let this slip by in the early stages. Regardless, we can work with it. Anyway, it's struggling with the tests and this one looks really obvious to me from the error message:

> thread 'ingestion::tests::test_registry_format_negotiation' panicked at 'called `Result::unwrap()` on an `Err` value: MissingField("time")'

It's messing up not one, but two of it's own configuration feature that it invented. This is kind of funny.

```rust
// Create default and custom JSON parsers
let default_parser = Arc::new(JsonParser::new());

let mut custom_mapping = HashMap::new();
custom_mapping.insert("timestamp".to_string(), "time".to_string());
custom_mapping.insert("value".to_string(), "measurement".to_string());
custom_mapping.insert("series".to_string(), "metric".to_string());
let custom_parser = Arc::new(JsonParser::with_field_mapping(custom_mapping));

// Register parsers with different priorities
registry.register(default_parser.clone(), Priority::Normal).unwrap();
registry.register(custom_parser.clone(), Priority::High).unwrap();

// Standard JSON data
let standard_data = r#"{
  "timestamp": 1000,
  "value": 42.5,
  "series": "test_series"
}"#.as_bytes();

// Test 1: With explicit format selection - should use default parser
// regardless of priority since we're explicitly requesting it
let points = registry.parse_with_format("json", standard_data).unwrap();
```

Not only are You Ain't Gonna Need It, but also You Ain't Gonna Navigate It! I get what this test is trying to do, it wants to have two "distinct" parsers, one with custom mappings and one without. It then wants to feed the two JSON formats through the auto discovery and see it work. Ok, that is at least showing the auto discovery feature working in the registry.

However, both parsers return the _"json"_ format from _supported_formats_. When the test calls _parse_with_format_ it's not really well defined which parser you'll use! They both are _"json"_! In this case it uses the _custom_parser_ since it happened to be registered second. This system is so weird and brittle and the agent is confusing itself trying to navigate it. Again, I step in to just stop the agent from spiraling and writing more code that doesn't address the problem. I change the test to only call _parse_with_autodiscovery_ since that's what we were interested in anyway.

```rust
let standard_points = registry.parse_with_autodiscovery(standard_data).unwrap();
assert_eq!(standard_points.len(), 1);
assert_eq!(standard_points[0].timestamp(), 1000);

let custom_points = registry.parse_with_autodiscovery(custom_data).unwrap();
assert_eq!(custom_points.len(), 1);
assert_eq!(custom_points[0].timestamp(), 2000);
```

It almost certainly doesn't make sense to register two parsers that claim the same format string, but for now we move on.

> Please check off the 'Registry system' TODO items

It does check off _Registry Sytem_, but then it skips _CSV Parser_ (correct) and tries to check off the _Tests_ section (incorrect).

> Please don't check off 'Tests' as you have not completed that section yet.

Great, the correct items are checked off and we're ready for _CSV parser_.

> Let's move on to the 'CSV Parser' subsection

It knocks this out of the park. Adding the CSV parser feels like a task perfectly set up for an AI agent. Here's an example of what the format looks like (this data was pulled from a test):

```rust
timestamp,value,series,region\n
1000,42.5,test_series,us-west\n
2000,43.5,test_series,us-east
```

You'll notice that CSV format accepts arbitrary tags, unlike the JSON format. Perfect!

The agent moves further along in the TODO checklist

> Now let's mark the tests as completed too, since we've added all the required tests

I agree, I think that this TODO item is satisfied. I mentioned early on that we would hopefully be able to push data in once the parsers are defined, but it's pretty clear that we're not that far along yet. The whole project looks like it gets tied together in _Phase 6: System Integration_. The code up to this point is [here](https://github.com/patrickgombert/vctsdb/tree/824297c14d8a8efb544d274d49c3fae80b74a5d8). Next time we'll start on _Phase 4: Query Foundation_ which I'm excited about!
