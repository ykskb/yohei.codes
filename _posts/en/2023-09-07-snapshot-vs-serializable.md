---
layout: default
title: "DB Isolation Levels: Snapshot vs Serializable"
lang: en
image:
    path: /assets/images/screenshot.png
---

# Database Isolation Levels: Snapshot vs Serializable

This page is an article based on notes I took while researching database isolation levels. Here, I focus on the differences between the snapshot isolation level and the serializable isolation level.

## Write Skew Anomaly

A common scenario to compare snapshot isolation level and serializable isolation level is the write skew anomaly. In the snapshot isolation level model, this scenario is fundamentally not handled. This is because the conflict detection in snapshot isolation only targets changes to the same record and does not account for the order of the commit sequence, which is a design choice made for performance reasons. Below is the scenario:

* Transaction to update on-call doctors

    * The system needs to ensure that there is at least one doctor on call at all times.

    * T1 (Transaction 1) wants to cancel Doctor A’s on-call duty if there is another doctor on call.

    * T2 wants to cancel Doctor B’s on-call duty if there is another doctor on call.

    * T1 and T2 occur at overlapping times in the timeline.

    Transaction sequence:

    1. T1 reads from the DB snapshot that both Doctor A and Doctor B are on call.

    2. T2 reads from the DB snapshot that both Doctor A and Doctor B are on call.

    3. T1 performs the update to cancel Doctor A’s on-call duty (successful).

    4. T2 performs the update to cancel Doctor B’s on-call duty (successful).

    5. As a result, the system fails to ensure that at least one doctor is on call, violating the requirement.

In a serializable isolation level, to ensure the sequential integrity of transactions, only one of T1 or T2 (whichever started first) will succeed, and the other transaction will either roll back and retry, be blocked until the first transaction finishes and then retry, or fail. This example involves a type of conflict known as a read-write conflict.

Of course, implementing a lock is one way for developers to resolve this issue. For example, `SELECT FOR UPDATE` locks the data read by the ongoing transaction so that it cannot be read by other transactions until the transaction is complete. In the example above, once T1 reads the on-call data for Doctors A and B, T2 would not be able to read that data until T1 completes. This ensures that when T2 completes its read, T1 has already committed, allowing T2 to see that Doctor A’s on-call duty has been canceled and to make an informed decision about whether to cancel Doctor B’s on-call duty. In major databases, the default isolation level is typically `READ COMMITTED` or `REPEATABLE READ`, so unless transactions are implemented with proper locking there'll always be a possibility of write skew. Its practical impact follows the next section.

## Practical Impact

[The example from Wikipedia](https://en.wikipedia.org/wiki/Snapshot_isolation#Definition) illustrates a scenario where bank transactions could potentially allow a customer to withdraw more than their balance by performing simultaneous withdrawal transactions across multiple accounts. The scenario assumes that a customer has multiple accounts and is allowed to overdraw individual accounts as long as the total balance is non-negative. While this may seem like a theoretical example, it is plausible when considering the real-world user experience during withdrawals. For instance, if an account holder initiates simultaneous transactions to withdraw `$200` from two accounts, each with a balance of `$100`, the snapshot isolation level database may allow both transactions to succeed, resulting in a total balance of `-$200`.

Though these examples might appear theoretical, real-world cases of such attacks have been reported. According to a paper by members of Stanford InfoLab, "[ACIDRain: Concurrency-Related Attacks on Database-Backed Web Applications](http://www.bailis.org/papers/acidrain-sigmod2017.pdf)," there have been reports of data corruption severe enough to bankrupt a Bitcoin exchange, excessive usage of gift cards on e-commerce sites, and the collapse of product inventory data.