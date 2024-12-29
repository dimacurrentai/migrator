# `migrator`

## Mission

Build an end-to-end usable and valuable product under the patronage of SDM.

The product is the OLTP engine aimed at making cross-storage migrations easier.

## Vision

* "Never write SAGAs or compensating transactions again!"
* "Free up development resouces consumed by the time sinks such as manual cross-storage rollbacks!"
* "Eliminate the surface area of any and all errors introduced by imperfect humans implementing SAGAs!"

## Intended Audience

The company needs to reconcile two or more data storages, or even two or more shards / replicas of one storage.

The portrait of the ideal customer is a mid-to-large size business that is:

1. not latency-sensitive,
2. has a decent amount of data that has to be OLTP-consistent,
3. needs to orchestrate cross-storage-boundary logically-atomic transactions,
4. must execute the above with no downtime, and
5. has internalized the pain of wasting a lion share of dev team's resources on this problem, with unpredictable timelines and potentially costly mistakes.
 
## Non-Goals

The `migrator` adds some ~100ms of latency by design.

If your application is latency-sensitive, and if you are not interested in separating the "optimistic happy path" from the "consistent baseline path", the `migrator` is not for you.

Also, the `migrator` is currently being designed for short-term use case of migrating the databases. Upon introduction, eventually, it is expected to have done its job of reconciling several data sources, after which it can be switched off and phased out.

It remains to be seen whether the ideas behind the `migrator` have value for an average business use case in the longer term. In other words, the `migrator`, as it is, does not claim to become the source-of-truth of cross-storage data forever. Instead, the claims of `migrator` are to speed up zero-downtime data migrations, to minimize the amount of development resources required to run such migrations, and to reduce surface error for potentially expensive human errors.

Ultimately, we believe the DSL for durably-executed transactions and daemon tasks is the future. Doubly so when the transactions, as well as consistent reads, are logically executed in the strictly-sequential order. The middle ground solution may well be to keep our DSL in production, having it transpiled via some git hook into the programming language of choice, interfacing with the single-source-of-truth database of choice. The feasibility of this approach remains to be seen, and such work is out of scope of the first prototype.

## Approach

In a nutshell, `migrator` logically takes ownership of the data and becomes the single source-of-truth for what was scattered across disparate storages.

One could say `migrator` creates this single source-of-truth out of thin air, since if the data used to live across several loosely-connected storages, there was no single source-of-truth to begin with.

For the period of the migration, various data sources become eventually consistent replicas of the true view of the data. Mutations, and strongly consistent reads, must go through the migrator.

To make this possible, the `migrator` must:

* Know the schema of all the entities that need to be accessed and migrated,
* Have the logic of mutations, as well as of consistent reads, implemented in the simplified DSL of the `migrator`,
* Ensure each DSL operation executed in a strictly-bounded small number of CPU cycles (by potentially introducing explicit join tables as needed),
* Be aware of how to read and parse data entities from, as well as how to serialize and push them into various now-disparate data sources, and
* Have the low-throughput "background job" migration logic for infrequently- or not-accessed keys as the "durable daemon", executed by `migrator` itself.

In addition to explictly materializing whatever join tables are necessary (such as the counters of users in various groups), it is also essential that:

* Each read and then persisted object in each database has the integer data version field, which this migrator will keep up-to-data, and
* No external party, except the `migrator` itself, should modify the data in external storages during the migration process.

If the latter invariant is not held, since the `migrator` keeps checksums of most recent versions of data for each key, it will halt the migration process highlighting what exactly are the object that got modified externally while they should not have been. The delta between the expected and the actual value for the offending keys will be promptly presented to the user.

## Implementation

Underneath, the `migrator` has:

* The OLTP transaction engine, to safely run txns across storages.
* A total order broadcast engine, so that the txns are serizeable isoluation-wise.
* A DSL to run these txns that ensures every endpoint completes in a bounded, and small, number of CPU cycles.
* The durable (["maroon"](https://dimakorolev.substack.com/p/durable-execution) execution framework.

The durable execution framework is the heart of the system. It operates on `2N+1` nodes that maintain consensus.

## Implementation Details

To be expanded.

* Hashes for values.
* Execution hints/traces.
* Keep the few latest hashes, and the execution hints/traces of how to quickly go from version `(k-1)` to version `k`.
* Interleave mutations with (concurrent) consistent reads.
* Daemons to (opportunistically and slowly) run long-lasting tasks.
* LRU caching, instructions-conveyor-style prefetching.
* Extenrnally-facing "warmup" endpoints.
* Bindings/wrappers/connectors to fetch and to push data from and to external sources.
* Conflict resolution logic for data onboarding.
* Fail-by-design for DSL txns that failed to fetch some key data from some key storages.

## Areas to Dive Deeper

To be expanded.

* Indexes: Who and when builds them?
* Time, indexes (the "smart median" of `2N+1` self-organized nodes? only monotonically increading? NTP?)
* Failover procedures.
* Monitoring and alerting.

## Further Reading

* [SDM Educational Designs](https://github.com/SysDesignMeetup/sdm?tab=readme-ov-file#educational-designs), since we want to do it right.
* The [Look](https://github.com/SysDesignMeetup/look) vision (looks like ~40% .. ~60% of ideas are already there, hehe).
