---
layout: post
title: "Vibe Coding In Anger: Part 1"
date: 2025-03-31
---

> what is vibe coding?

> "Vibe coding" isn't a widely established term in programming, but it can refer to a few different things depending on the context. In a casual sense, it might refer to coding that is done with a relaxed or creative mindset, focusing on the flow of work rather than strict structure or rules.

> what does the idiom 'in anger' refer to?

> The idiom "in anger" refers to doing something while feeling angry or under the influence of anger. It suggests that an action or behavior is driven by or motivated by strong feelings of anger. For example, if someone says, "She slammed the door in anger," it means she slammed the door because she was angry at the time. The phrase emphasizes that the emotion of anger played a key role in the action or response.

I asked an LLM to define the title, but to be clear I'm using "vibe coding" to refer to the act of letting an AI write software without much oversight. I'm using the idiom "in anger" to refer to doing something hands on as its intended to be done, rather than just learning or hearing about it.

In this post I'm going to be vibe coding in anger. I'm going to start with some motivation and some background, but afterwards I'm going to narrate my thoughts as I step throught the process. I truly have no intuition for how this is going to go.

I won't be using an LLM to help draft these blogposts. It will be my thoughts, mostly stream of conscious, interspersed with AI generated artifacts.

### My AI Background

I chat with LLMs occasionally and more or less use it as an interactive search. At work I use the code assistants I have available to me. They augment me in the work I do, like autocomplete but much better. This is useful. Sometimes they're in the way and produce incorrect code, or invent interfaces that don't exist. This isn't useful, but it's fine because I can spot it right away. Overall, the work of typing out code is not the bottleneck most of the time. When I use AI beyond a fancy autocomplete tool I do find that the AI can useful, but that it can fall apart pretty quickly when I need it to deeply understand the domain or the broader ecosystem.

Given the sentiment of a lot of the AI content I see it seems that I'm not doing at least some of this right. I figure now is a good time to be deliberate about interacting with AI in earnest. I've never vibe coded, but I do believe that the exercise will be good for me.

### Projects I Work On In My Free Time

When I take on a project in my free time it's usually to build a toy version of something that I use often, but I want to understand deeper by getting hands on. That has been things like writing [a lisp interpreter](https://github.com/patrickgombert/Hisp), or [a log structured merge tree](https://github.com/patrickgombert/lsmt), or maybe even [a lisp interpreter](https://github.com/patrickgombert/gisp). The most rewarding thing I worked on in my free time was an implementation of the [clojure runtime in clojure](https://github.com/mylesmegyesi/clojure.core) with a former co-worker. We didn't finish the project, but I learned about the guts of the clojure runtime and it paid off in both direct and indirect ways. Directly, in that I was able to apply it to projects I was working on at the time. Indirectly, in that I got exposed to some very interesting ideas and a way of thinking that has stayed with me.

I work on these projects until I feel like I got what I was looking for. That often means that they're not totally finished or totally polished. However, when I set it down I feel satisfied  and carry the learning and the experience with me.

I'm going to approach this project in the same way.

# Vibes

### How I Will Approach Vibe Coding

I will be following Harper Reed's [My LLM codegen workflow atm](https://harper.blog/2025/02/16/my-llm-codegen-workflow-atm/). I think that this workflow makes as much sense to me as anything else. From my perspective it seems to be intentional in the way it approaches the subject. At a minimum, this is absolutely a better workflow than what I would come up with on my own.

I'm going to try and keep my hands off of the code, which is hard for me. I'll definitely read the code, and read it carefully when needed. I'm going to try and provide the high level thoughts, but not pay attention to the low level details.

Also, I'm going to use [rust](https://www.rust-lang.org/). I am somewhat [familiar](https://github.com/patrickgombert/tetris) with it. It's also a popular language so I assume the LLM will have lots of training data relevant to the types of things I want to do.

#### The Project

I'm going to vibe code a time series database named Vibe Coded Time Series Data Base (VCTSDB). The code will be up on my [github](https://github.com/patrickgombert/vctsdb/). It's going to be whatever Cursor writes, unless otherwise noted. I put some thought into the specification, but I wanted to make the case that this will be tagged data that can be durably stored on my laptop. I am interested in putting a little query language on top. You can read the [specification](https://github.com/patrickgombert/vctsdb/blob/main/spec.md) in full if you'd like. I made some decisions that sounded simpler to me, such as not downsampling the data and having a forever retention. I did make the case that it should be durable and recover writes after failure.

# Let's Begin

I've generated a [prompt plan](https://github.com/patrickgombert/vctsdb/blob/main/prompt_plan.md) and had that summarized into a [TODO checklist](https://github.com/patrickgombert/vctsdb/blob/main/todo.md) per the [workflow](https://harper.blog/2025/02/16/my-llm-codegen-workflow-atm/) I am following. It's a little bit wishy-washy and I'm not sure how that will play out! I wanted to not be overly strict in the specification because I wanted to see how the system evolved and then try to constrain or liberate the exercise as it progressed. This is more or less how I approach these types of projects, but maybe that's not quite what is required. We'll see!

### Project Setup

I had already run _cargo new vctsdb_ and thus had checked off the first checkbox. This was not vibes of me, please forgive me for I have Sinned. Looking at the second checkbox I don't really need CI/CD, but I do want a test runner. The remaining checkboxes feel a bit less like project setup, but that's not for me to decide!

```
I have a TODO list within `todo.md` for my overall project plan. Could you execute on the 'Project Setup' items? Please note that I have already initialized this as a rust repository via `cargo new`. Also note that I do not require CI/CD, but do want to have a test suite. If you want to learn more please consult the `spec.md` file.
```

It responds with:

```
First, let's create the required directory structure:

mkdir -p src/{storage,ingestion,query} tests
```

I run this. It's being very literal and following the perscribed boundaries for the sub components _storage_, _ingestion_, and _query_.

It goes on to add the _tracing_ and _tracing-subscriber_ crates. I quickly look at this library because I'm not familiar with it. Cursor is saying that this is a logging dependency, but it has lots of things like cross thread spans and events (I guess these can be logged?). Sounds good to me.

It sets up _tracing_ in a main method, I don't really look too hard because this just looks like boiler plate code. It then asks to run _cargo build_, sounds good. But uh-oh, compilation failed when invoking the _build_ method on _FmtSubscriber_. It then jumps around a bit which is confusing.

```
I see we need to create the module files and fix the subscriber builder. Let's create the necessary module files first:

touch src/storage/{wal,lsm}.rs src/ingestion/{parser,formats}.rs src/query/{parser,planner,executor}.rs
```

Sure, why not. Maybe the act of  meandering through your tasks unites all coders, AI and human alike. Now it fixes the usage of _FmtSubscriber_ and replaces the hallucinated _build_ with _init_. It builds, but with a warning. Cursor does not care about the warning, again maybe it's just like me. It runs _cargo test_ and it fails compilation.

```
We need to make a few fixes:
1. Make the crate a library to be able to use it in tests
2. Fix the unused variable warning in main.rs
3. Fix the unused imports in the module files
Let's make these changes:
First, let's create a lib.rs file to expose our modules:
```

I accept and it creates the modules. It also decided to fix the warning! The test suite runs (it still only has the _assert!(true)_ dummy test). I ask it check off the items in _todo.md_ and it does. It then notes that we have two items left.

```
The remaining items in this section are:
1. Set up CI/CD pipeline (which you mentioned is not needed)
2. Configure performance metrics collection
```

I tell it to remove the CI/CD item and to complete the other item. It wants to add dependencies on _metrics_ and _metrics-exporter-prometheus_. I really don't think that we need to bring prometheus into this, but I inhale in my preference and exhale out a vibe and click _Run Command_. It adds some metrics boilerplate setup and lets me know that prometheus will be available at _http://localhost:9090/metrics_ while the application is running. I'm glad I vibed, because that might be fun to watch. It now has all of _Project Setup_ completed, each TODO item is checked off.

I ask cursor to run the application (I want to see the metrics ddashboard!), but we are missing the _tokio_ dependency. Cursor adds it, and then sets up some more boilerplate in _main_. We're ready to run and... there's an error. It hallucinated the incorrect interface for _metrics_ and it corrects its mistake.

It runs _cargo run_ again... and there's another error. It hallucinated some of the _prometheus_ setup incorrectly and I am a little bit curious about how it eventually finds the real API given the hallucination in the first attempt. There is tooling for this, which I'm sure I could inform the AI about. But, for now, I'm not going to worry about it and just let it iterate until it builds. Finally it builds and runs! _http://localhost:9090/metrics_ is an empty page.

```
<html><head></head><body></body></html>
```

I ask about this.

```
Why is localhost:9090/metrics producing an empty html page?
```

It says it's because we aren't capturing any metrics (makes sense), so it invents some fake metrics.

```
tokio::spawn(async {
        let mut interval = tokio::time::interval(Duration::from_secs(1));
        loop {
            interval.tick().await;
            // Record some test metrics
            metrics::record_ingestion(42.0);
            metrics::update_memory_usage(1024 * 1024); // 1MB
            metrics::record_query(15.5);
            metrics::record_wal_write(512);
            metrics::record_sstable_operation("compaction", 1);
        }
    });
```

Which... yeah ok. These certainly look like metrics we'll eventually be capturing. Let's fake them for now, sounds good. I see the metrics listed on the page and I ask to stop the process. The _Project Setup_ portion is complete!

### Phase 1: Core Storage Foundation

Now, let's move on to the actual project. The TODO list has two sub-sections for Phase 1, _Data Structure Implementation_ and _Write-Ahead Log (WAL)_. The data structures are very simple and should be slim structs. The WAL is not so simple and has lots of nuance. It'll be interacting with the disk and encompasses checksum validation, rules about rotating data out, corruption detection, etc. This is a high risk area for losing data and requires lots of testing and knowledge about how to implement it correctly.

I'll just start with the data structures and check those off first.

```
Let's move on to Phase 1: Core Storage Foundation. Go ahead and complete the Data Structure Implementation section.
```

It does what I said it would do and created some slim-ish structs, but they're a little heftier than I anctipated because there is lots of error handling. It also adds the _thiserror_ dependency. It asks to run the tests like a good little AI. The tests are interesting, it's mandating ASCII for all strings and non-negative time stamps. I do notice one very intersting test which I'll just keep an eye on:

```
  // Try to add point with non-increasing timestamp
  let invalid_point = DataPoint::new(1500, 44.0, tags);
  assert!(matches!(
      series.add_point(invalid_point).await,
      Err(DataError::NonIncreasingTimestamp)
  ));
```

This is absolutely not an invariant a time-series database would have. I just noticed that these tests are very literal transcriptions of the TODO which I glossed over earlier, but I'm going to run with it for now. Using only ASCII strings is whatever, I don't care either way. "Strictly increasing timestamps" is definitely not great. There's also a lock around around the points in the timeseries. I don't fully understand the relationship between these in-memory storage structs and the rest of the submodule, which is going to be persisting to disk. It doesn't really matter if I fully understand or see the path forward, because I can simply trust the process. I'll just pretend that I don't know about clocks on computers and let this all slide, but later on I might poke at some of these decisions if it causes problems.

The test pass! Let's move on to the WAL.

```
Let's continue and implement the WAL. Let's start with the WAL serialization format and Segment management sub-sections for now. I want to see how those look before continuing to the Recovery Process sub-section.
```

Cool! It generated a WAL implementation that writes entries in segments. It's kind of interesting that it chose JSON for the format of each entry. It's essentially the series name along with the _DataPoint_ which was generated earlier. It's also including a _crc_ in the entry JSON. This will be handy for debugging. I actually like this decision because you can make the format pluggable later on. Then you might have a situation where JSON can be used in test or for debugging, and a more compact format used in production.

It has a few tests for reading, writing, segment rotation, and checksum validation. I ask to run the tests and it fails compilation. It iterates on the errors, even manually implementing the _Debug_ trait for one of the WAL structs. It finally compiles, but the tests fail. It has a theory though!

```
We have two test failures to fix:
1. test_wal_entry_validation is failing because we're trying to read the header as a UTF-8 string, but it's written as binary JSON.
2. test_wal_segment_rotation is failing because we're not properly tracking the segment size.
```

I accept and it informs me of the changes

```
The key changes I made are:
1 Fixed segment size tracking:
    - Added update_size() method to Segment
    - Initialize segment size from file metadata
    - Update size after each write
    - Reduced test data size and segment size threshold
2 Fixed WAL entry validation:
    - Added newline after header JSON
    - Properly handle empty lines
    - Trim whitespace from entry JSON
    - Skip newline after CRC
    - Added more assertions in tests
3 Improved segment rotation:
    - Check for rotation after creating new segment
    - Better error messages in tests
    - More realistic segment size limits
```

We try again and it fails to compile, I allow it to iterate some more but it really feels like the AI is flailing now. The tests fail again, but Cursor provides some more valid sounding reasons for the failures. I let it keep iterating, but I am starting to doubt that it can get to a green test suite. It's like if the little engine that could had derailed, but still thought that it could get up the hill. It is fixing lots of little silly logical errors, or atleast that's what the prose it spits out suggests. I'm not paying close attention yet.

I did call out earlier that the WAL being messed up probably means that the application will suffer data loss, so this isn't totally inspiring. Finally it sort of feels like it gives up and states:

```
Let's add some debug logging to understand why the segment isn't rotating
```

I'm going to stop vibe coding for a minute and see if I can debug. My fear is that the flailing caused a mess and it seems like I shouldn't let it go further. This is unforunate though as I won't be vibes coding but coding coding (or even debugging coding). I want to keep this exercise rolling so no use getting totally stuck here.

It was setting up a _tracing_ logger in test and had not used the API correctly. I just rip it out for now and I'll use regular old _println!_. Also, there are so so many warnings, but one in particular is... relevant.

```
warning: methods `is_full` and `is_expired` are never used
  --> src/storage/wal.rs:81:8
   |
57 | impl Segment {
   | ------------ methods in this implementation
...
81 |     fn is_full(&self, max_size: u64) -> bool {
   |        ^^^^^^^
...
85 |     fn is_expired(&self, max_age: u64) -> bool {
   |        ^^^^^^^^^^
```

That's interesting because the test that tests the rotating of the WAL segment is failing! I would expect _is_full_ and _is_expired_ to trigger the rotation. It has this code block in the write path:

```
let needs_rotation = {
    let segment = segment_guard.as_ref().unwrap();
    let metadata = fs::metadata(&segment.path)?;
    let current_size = metadata.len();
    current_size >= self.max_segment_size
};
```

That's not really the problem, but it's duplacing code in sort of dumb ways. I swap this out for:

```
let needs_rotation =
    segment.is_full(self.max_segment_size) || segment.is_expired(self.max_segment_age);
```

Then I find the actual root cause. This is how it was choosing to name each segment file:

```
let timestamp = SystemTime::now()
    .duration_since(UNIX_EPOCH)
    .unwrap()
    .as_secs();
let filename = format!("segment_{}.wal", timestamp);
```

So, on writes the code keep saying "please rotate" and then it would rotate into the same file. This is because the rotations all happened within the same second. I suspect that it wants to sort these segments by time, so I extend this to include a UUID which will still retain the time in the file name.

```
let timestamp = SystemTime::now()
    .duration_since(UNIX_EPOCH)
    .unwrap()
    .as_secs();
let filename = format!("segment_{}_{}.wal", timestamp, Uuid::new_v4());
let path = self.directory.join(filename);
```

The tests pass now. I inform the agent that it can check the TODO items off. I didn't inspect much else too closely and to be honest I don't feel convinced of this implementation. Hunting down this one test fix did not inspire confidence. Regardless, we move on.

```
I was able to fix the tests. Feel free to check off the  WAL serialization format and Segment management sub sections of the TODO if you feel like these items have been completed.
```

The agent dutifully obliges. It immediately asks to move on to the next TODO tasks. Look at me, I am the scrum master now. My programmer is moving tasks and I am keeping them motivated.

I'm going to have Cursor move on to _Recovery Process_ and complete it before we wrap up the WAL with the _Tests_ section. I feel as though remaining focused it useful, especially for touchy bits of code.

```
Please implement the Recovery Process TODO items next
```

Something interesting then happens, it writes code and checks off the TODO boxes. I'm not sure if it compiles and passes the tests (I didn't even look at the code) so I just ask to run the tests. It, of course, doesn't compile. I find it odd that it checked the boxes off this time so confidently, but had not done that in the previous sections. It iterates a bit and moves from a compilation error to a test failure for the checksum validation. It actully corrects the test failure and passes. Nice! What's interesting is that now it's really feeling itself and wants to check off the _Tests_ section in the _todo.md_ entirely. I accept, but I'm also going to peek at the code and the tests it wrote.

It's kind of neat! It let's you replay all of the segments in the WAL by passing a callback function that gets invoked for each enty. The replay functionality verifies the integrity, but then there is also a parallel verify functionality which verifies the integrity but doesn't do anything with each entry. These are written totally independently. It seems as though you could implement the _verify_ function by passing a no-op callback to the _replay_ function and letting it run.

There's also a very funny quirk here. The _WalEntry_ looks like this (note the _crc_):

```rust
#[derive(Debug, Serialize, Deserialize)]
struct WalEntry {
    series_name: String,
    timestamp: i64,
    value: f64,
    tags: std::collections::HashMap<String, String>,
    crc: u32,
}
```

Each entry gets initialized on write like this:

```rust
let entry = WalEntry {
    series_name: series_name.to_string(),
    timestamp: point.timestamp(),
    value: point.value(),
    tags: point.tags().clone(),
    crc: 0, // Will be calculated below
};
```
What's funny is that _crc_ is always _0_ in this struct. The 'Will be calculated below' comment is true, but when it's calculated below it gets written separately outside of the struct's serialized JSON object. On disk this looks like:

```
{"magic":1463897088,"version":1,"created_at":1743467384}
{"series_name":"test_series","timestamp":16,"value":16.0,"tags":{"host":"server1"},"crc":0}
<binary checksum>
```

And then the pattern contiues with a JSON entry, newline, checksum, newline, so forth. It also does not have an interface to clean up committed segments. The WAL will just grow forever unless it decides to add segment file deletion functionality later.

It does have a test where it writes something into the middle of the file and then tests that it gets caught in the corruption detection (this is great!). I suspected that this might be simply introducing a JSON parse error so it's not even using the checksum, and I was able to verify that it was in fact hitting the JSON parse error stanza. This was cracking me up to be honest. I was imagining a testsuite where a tag string value (for example) gets changed in the JSON blob and it gets caught by the verification code. Or the checksum is changed or whatever. Introducing invalid JSON and then not exercising the checksum and calling that good to go when testing the checksum validation is so funny. The vibes are so good.

I'll set this down for the time being. The code thus far is [here](https://github.com/patrickgombert/vctsdb/tree/95ca9da3c6113e4a96f84cde4f570b27e6c4680a). Next will be _Phase 2: Storage Engine_ which gets into writing a Log Structured Merge-Tree to store the time indices on disk.
