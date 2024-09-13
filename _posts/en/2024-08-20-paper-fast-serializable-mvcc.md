---
layout: default
title: "Paper: Fast Serializable Multi-Version Concurrency Control for Main-Memory Database Systems"
lang: en
image:
    path: /assets/images/screenshot.png
---

# Paper: Fast Serializable Multi-Version Concurrency Control for Main-Memory Database Systems

This articles is notes taken from the paper: [Fast Serializable Multi-Version Concurrency Control for Main-Memory Database Systems](https://db.in.tum.de/~muehlbau/papers/mvcc.pdf) published in 2015.

> ### Series: Papers adopted by DuckDB
>
> DuckDB can handle analytical query workloads incredibly fast. This series is my notes from the publications adopted by DuckDB (listed [here](https://duckdb.org/why_duckdb.html#standing-on-the-shoulders-of-giants)).
>
> - [Vectorized Query Engine](/2024/08/16/paper-monet-db-x-100.html)
> - Fast Serializable MVCC: this article
> - Join Ordering Optimization (coming soon)
> - Unnesting Subqueries (coming soon)

## What is it about?

This paper proposes an implementation of multiversion concurrency controll ([MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)) which has less overhead and less locking while providing serializability. Most of MVCC implementations out there provide snapshot isolation ([SI](https://en.wikipedia.org/wiki/Snapshot_isolation)).

Overall, the approach is to provide the serializable isolation level by checking possible conflicts by the commit times while using snapshots for a transaction to data reads.

> Database Isolation Levels: Snapshot vs Serializable
>
> I compared snapshot isolation level and serializable isolation level [here](/2023/09/07/snapshot-vs-serializable.html).
>
> Also it seems DuckDB has implemented its MVCC based on the concept of this paper, however without the serializable transaction controls. I haven't confirmed 100%, but it seems DuckDB provides snapshot isolation level currently, which makes sense to me as its focus is more on OLAP instead of OLTP.

## Storage Locations of Versions

* Conventional MVCC: scattered versions

    * Older versions are kept, and a background process cleans them when they are obsolete.

    * Storages in DBMS are dynamically allocated: different times of version entries lead to different locations and pages.

    * DBMS creates new versions on different pages to minimize contentition between concurrent transactions.

    * Versions are created after they are logged for recovery. New versions typically get created on separate storage locations so original versions can be preserved for recovery.

    * Data gets fragmented overtime in DBMS because available gaps are utilized by the system: versions are scattered across different locations.

* Proposed MVCC: centralized versioning management

    The newest data is updated in place and previous versions (before-images) are stored in undo buffers.

    * Reduces the complexity of managing multiple versions scattered across DB.

    * At every successful commit, obsolete versions before this commit time get cleaned up. 

    * Helps performance with cache-friendly format.

> The paper does not directly state it, however a good example of scattered versions is PostgreSQL in my opinion.
>
> PostgreSQL is designed to store versions in its permanent storage for multiple benefits such as simplicity and durability. However this decision brings [`VACUUM`](https://www.postgresql.org/docs/current/sql-vacuum.html) business and a very expensive cost to retrieve accurate counts of records in a large table. Separating the version storage from perpament storage seems to be a good design of providing snapshots of records without complexities.

## Reduced Locking

* Conventional MVCC: 

    * Rows might be locked to prevent other transactions from updating them.

    * Locks or similar mechanisms might be used to control visibility of versions.

    * Version list might be locked while being read for stable view of the data.

* Proposed MVCC:

    Changes happen in place within undo buffers.

    * Precision locking: only the localized areas being updated are locked.
    
        Changes are confined to small and specific region of memory. 

        * Leads to shorter lock hold times.

        * Other transactions can read and update another part of undo buffers.

## Selective Checks

* Conventional MVCC:

    * Not only committed, but also uncommitted transactions might be checked.

    * Not only modifications, but also reads would be checked in conservative approaches. It is typically to handle scenarios like [write skew anomaly](https://en.wikipedia.org/wiki/Snapshot_isolation#Definition).

    * All the conflict types such as read-write, write-write and even read-read might be checked.

    * Checks of conflicting transactions might happen at global level, which would result in global locking. 

* Proposed MVCC:

    * Only checks committed transactions that could affect the read predicates of ongoing transactions.

    * Defers much of conflict detection until commit time with a more focused scope.

## Efficiant Validation and Conflict Resolution

* Conventional MVCC:

    * Might require expensive global locks to ensure serializability 

    * Might use timestamps:

        * Atomic operations might be required to ensure uniqueness of timestamps, which leads to wait time.

        * Comparison process can be complex, and the system must validate read and write operations against the timestamps of potentially numerous other transaction.

        * If a transaction with an earlier timestamp has not yet committed, a transaction with a later timestamp might be forced to wait, leading to delay.

        * Rollback process can be expensive because it requires undoing all changes made by the aborted transaction, restoring the previous state, and possibly restarting the transaction.

* Proposed MVCC:

    * Validates serializability by checking conflicts only at commit time.

    * Versions in undo logs are version chains only with delta.

    * Versions in undo logs are indexed by IDs and/or timestamps.

    * Predicates of transactions are logged for fast validation of serializability.

# Benchmarks

* TPC-C: write-heavy benchmark of order entries (8% read and 92% write transactions)

    * Proposed MVCC: 

        * `100,000 TPS` (transactions per second) 
        
        * Performance cost of about 20% compared to single-version concurrency control

        * Scaled linearly up to 20 cores, however beyond that might require reducing global synchronization. [Silo](https://wzheng.github.io/silo.pdf) implements this.

    * 2PL (two-phase locking) in HyPer: 5 times slower

    * Conventional MVCC: `50,000 TPS`

* TATP: benchmark of point accesses & whole-record updates (80% read and 20% write transactions), simulating a telecomm app

    * Proposed MVCC: `407,564 TPS` shows minimal overhead comparing to single-version control and overperformed conventional MVCC.

    * Single-version control: `421,940 TPS`

    * Conventional MVCC: `340,715 TPS`

    


