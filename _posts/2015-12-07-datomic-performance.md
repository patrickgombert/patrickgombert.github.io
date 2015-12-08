---
layout: post
title: Datomic Performance
date: 2015-12-07
---

I've recently been tasked with improving the performance of a long running data import which ultimately transacts into a datomic database. Through profiling, conversations with cognitect, and simple benchmarks we've had a couple of huge wins. During this time I've had a few takeaways that I'd like to share. While all of these are found within the [datomic docs](http://docs.datomic.com/) I thought it would be helpful to put the specific techniques I ended up using all in one place.

Of course, your milage may vary.

### Querying indexes

Datomic has four [indexes](http://docs.datomic.com/indexes.html) that it maintains. EAVT, AEVT, VAET, and AVET which are different orderings of (E)ntity, (A)ttribute, (V)alue, and (T)ransaction. Of the four indexes, AVET is _not_ enabled by default and can be transacted with your schema by supplying :db/index to be true or is implied by :db/unique. The docs claim that AVET is an expensive index and therefore not enabled by default. However, I've found the AVET index most useful for our particular use case.

We are maintaining relationships within datomic and we use entity ids as our foreign keys. Our schema sort of looks like this:

```clojure
[{:db/id          #db/id[:db.part/db]
  :db/ident       :t1/id
  :db/valueType   :db.type/string
  :db/cardinality :db.cardinality/one
  :db/index       true
  :db.install/_attribute :db.part/db}
 {:db/id          #db/id[:db.part/db]
  :db/ident       :t2/id
  :db/valueType   :db.type/string
  :db/cardinality :db.cardinality/one
  :db/index       true
  :db.install/_attribute :db.part/db}
 {:db/id             #db/id[:db.part/db]
  :db/ident          :link
  :db/isComponent    true
  :db/valueType      :db.type/ref
  :db/cardinality    :db.cardinality/many
  :db.install/_attribute :db.part/db}
 {:db/id             #db/id[:db.part/db]
  :db/ident          :link/target
  :db/valueType      :db.type/ref
  :db/cardinality    :db.cardinality/one
  :db.install/_attribute :db.part/db}])
```

The `:link/target` will always point to a entity id. When we're importing data we have the need to query datomic with an attribute and value in order to get an entity id quite often.

When we first attached [yourkit](https://www.yourkit.com/) in order to profile we noticed something interesting, up to 50% of our time was being spent in what looked like datomic's query parser. Furthermore, after running some benchmarks we were seeing that we could improve performance by an order of magnitude by querying the AVET index directly instead of writing the equivalent query. A simple benchmark might look like the following:

```clojure
(defn transact-t1s [c]
  (d/transact (d/connect datomic-url) (map #(hash-map :t1/id (str %) :db/id (d/tempid (* -1 %))) (range c))))

(defn run-test [c]
  (let [db (d/db (d/connect datomic-url))]
    ; warmup
    (doseq [i (range c)]
      (d/q '[:find ?id :in $ ?attr :where [?id :t1/id ?attr]] db (str i))
      (d/datoms db :avet :t1/id (str i)))
    (time
      (doseq [i (range c)]
        (d/q '[:find ?id :in $ ?attr :where [?id :t1/id ?attr]] db (str i))))
    (time
      (doseq [i (range c)]
        (d/datoms db :avet :t1/id (str i))))))

(run-test 1000)
; "Elapsed time: 83.204651 msecs"
; "Elapsed time: 2.487131 msecs"
```

According to the [metrics](http://docs.datomic.com/monitoring.html) that datomic was giving us we were hitting the peer's cache 100% of the time during our benchmarks and our production runs. Using the `datoms` function directly and querying indexes allowed us to save a ton of time. However, querying the index _only_ works with transacted datoms, so any database with non-transacted data (such as the result of calling `with`) will not work.

Of course, our benchmark might be wrong and our usecase might not be typical so again, YMMV.

### Avoiding Querying Non Transacted Data

This one might be obvious, but we had perfomance issues with querying non-transacted databases. We attempted to transact only once, at the end of our import process and therefore needed to write the `:link/target` of relationships against tempids. In order to ensure  that we had unique ids with the correct metadata we would query the existing but not transacted links. What we found was that as our inputs grew (in our case towards a few million) our query time would degrade about 2x-3x. Simply transacting at safe points during our import allowed us to instead query existing links against transacted data. Once we did this our queries were much faster and we saw that we no longer degraded over time.

### Put The Most Contraining Clause First

This one is stated directly in the [best practices](http://docs.datomic.com/best-practices.html#most-selective-clauses-first) which are a good read. We tended to generate queries which meant that it was often important for us to view the expanded form of our queries to ensure that we weren't doing operations out of order. With datomic it's simple enough to take any parts of the query and compare the counts returned to figure out the most sensible ordering.

Another issue with generating queries is the way the peer cache is computed. The [unevaluated query itself is used for caching](http://docs.datomic.com/best-practices.html#parameterize-queries) which means it's often better to parameterize queries than to generate the inputs directly into the query.
