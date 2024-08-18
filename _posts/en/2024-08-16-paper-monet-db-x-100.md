---
layout: default
title: "Paper: MonetDB/X100: Hyper-Pipelining Query Execution"
lang: en
---

# Paper: MonetDB/X100: Hyper-Pipelining Query Execution

This articles is notes taken from the paper: [MonetDB/X100: Hyper-Pipelining Query Execution](http://cidrdb.org/cidr2005/papers/P19.pdf). It was published in 2005 at [The Conference on Innovative Data Systems Research (CIDR)](https://www.cidrdb.org/) Conference.

> ### Series: Papers adopted by DuckDB
>
> DuckDB can handle analytical query workloads incredibly fast. This series is my notes from the publications adopted by DuckDB (listed [here](https://duckdb.org/why_duckdb.html#standing-on-the-shoulders-of-giants)).
>
> - Vectorized Query Engine : this article
> - Fast Serializable MVCC (coming soon)
> - Join Ordering Optimization (coming soon)
> - Unnesting Subqueries (coming soon)

## What is it about?

This paper proposes a vectorized query execution model in DBMS for better utilization of modern CPUs, comparing it against traditional volcano iterator model.

- Performance comparision is through TPC-H benchmark: 22 complex SQL queries including large scans, joins, aggregations and nested queries.

- The paper doesn't directly mention, but MonetDB is a columnar DB and the comparisons suggest that the mentioned traditional volcano model is assumed to be row-oriented DB.

## Interpretation of query

* Traditional: volcano iterator model = tuple by tuple 

    Each tuple must repeated go through various operators like filters, joins and aggregations -> high interpretation overhead & limited opportunities for CPU parallelism.

* Proposed: vectorized processing 

    Applies filter, join and aggregation operations to entire data vectors at once.

## CPU cache utilization

* Traditional: entire tuple needs to be loaded into memory, so unused data uses cacheline as well when data is accessed, leading to more frequent cache misses.

* Proposed: only relevant data is loaded into memory

    * Spacial locality: data of a single column is stored contiguously in memory -> better cache hits & contiguous memory accesses

    * Temporal locality: once data is loaded into cache, it is used repeatedly before the next chunk is loaded.

Example:
* Table: `sale_id INT, date DATE, product_id INT, quantity INT, price FLOAT`

* Query: `SELECT SUM (quantity * price) AS total_revenue FROM sales;`

* Cacheline:

    * Row-oriented: unused columns on cacheline

        `[ sale_id | date | product_id | quantity | price | sale_id... ]`

    * Columnar data: only used data is cached
            
        `[ quantity | quantity | quantity | quantity... ]`
    

## CPU Pipeline Execution

* Traditional: tuple-by-tuple & row-oriented

    * CPU stalls while data is loaded due to frequent cache misses as mentioned above.

    * Instructions need to be repeatedly fetched per tuple (No SIMD).

    * Conditions such as `IF` or `WHERE` per tuple lead to branch mispredictions.

    * Data / control hazards = operation dependencies. Operations need to wait for previous operations to finish before proceeding.

* Proposed: vectorized processing

    * Less cache misses = less CPU stalls.

    * Vectorized data align well with [SIMD](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data) can significantly optimize data processing.

    * Less or no dependency between operations, so CPU pipelines can plan and execute operations effectively. 