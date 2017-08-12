---
title: "2017 08 12 Query Planner"
date: 2017-08-12T19:20:06+02:00
draft: false
tags: ["pg", "sql"]
---

Types of index scans
====================

Indexes
=======

Sequential Scan:
----------------
* Read every row in the table
* No reading of index. Reading from indexes is also expensive.
* Best way to access small tables, that fits in a single disk block. There is no point in this case to use the index. Block size in pg is 8K. Also the best way to read access all rows in a table.

Index Scan:
-----------
* Index Scan or Bitmap Index Scan: Read the index, hoping that we won't have to go to disk. Index vs. Bitmap is not about what kind of index we're using. It's actually two different ways of using the index. Index scan reads the index in alternation, bouncing between table and index, row at a time. The bitmap index will read all the relevant info from the index first, then read from the table based on what the index tells us.
* Performance gain when reading a few rows from a big table. It goes to the index, finds the rows
* Only method of accessing a table the can return rows in sorted order - useful in combination with LIMIT.
* Disadvantage: Causes random I/O against the base table. Read a row from the index, then a row from the table, and so on.

Bitmap Index Scan:
------------------
* Scans all index rows before examining base table. This populates a TID bitmap.
* Table I/O is sequential, with skips.; results in physical order.
* Can efficiently combine data from multiple indeces. the TID bitmap can handle boolean ANDs and ORs.
* Handles LIMIT poorly.
* Adjust page cost parameters to use Index scans vs Bitamp scans on queries where it makes sense.

SMALL INDEXES ARE BETTER. Less columns are better. Avoid multicolumn indexes.

Join planning
=============
* Fixing the join order and join strategy is the harder part of query planning.
* When search space is small, planner does a nearly exhaustive search. When search space is too large, planner uses GEQO (genetic planner) to limit planning time and mem usage.

Join Strategies
---------------
Nested loop
-----------

    for (each outer tuple)
      for (each inner tuple)
        if (join condition is met)
          emit result row;

For a 3 table join, a, b and c, reduce problem to a two table join: (a && b) && c, or a && (b && c), or ...

Merge join
----------
* Only handles equality joins
* Puts both input relations in sorted order (sort or index scan), and match up on equal values by scanning through the two in parallel.
* Only way to handle really big datasets, however merge joins are slower.

Hash join
---------
* Also only equality joins.
* Hash each row from the inner relation to ceate a hash table. Then hash the outer relation and probe the hash table for matches.
* Very fast - but requires enough memory to store inner tuples. Can get around this using multiple "batches". If work_mem is 1MB, and the hash requires 2MB. The planner does this.
* Not guaranteed to retain input ordering. This is because the planner is capable of doing the hash join in multiple batches. So when combined with a nested loop there may be a higher cost.

Join Removal
------------
* It can detect that a LEFT OUTER join is not used and remove it entirely. Ex: SELECT foo.a, foo.b from foo LEFT OUTER JOIN bar on foo.a = bar.a
* For example, big view where you request a subset of attributes.

Join Reordering
---------------
* It doesn't matter what the order in which you write the JOINs, postgres will decide the best plan. Available for ages...
* Can't reorder in the presence of LEFT JOINs.
* It will reorder joins as much as it can, but will be constrained by left joins.

These two queries are different and can't be reordered:

    SELECT * FROM
      (foo JOIN bar ON foo.x = bar.x)
      LEFT JOIN baz ON foo.y = baz.y

    SELECT * FROM
      (foo LEFT JOIN baz ON foo.y = baz.y)
      JOIN bar ON foo.x = bar.x

EXPLAIN estimates
-----------------
* number of rows estimates
* average width (byte) of each row (not that important)
* cost is in "abstract cost units". Ex: Relative costs of an index fetch vs a sequential page seek. These are not miliseconds.
* First cost is startup, second number is total
* Actual time is miliseconds, provided by EXPLAIN ANALYZE
* When looking at EXPLAIN ANALYZE output, look for:
  * where is the most time spent? In a bad query plan, find which branch was most costly? Identify which table is more problematic? Add an index?
  * Are the estimates and actuals very different? Have you analyzed the table recently. But sometimes these problems are caused by correlation between columns, and may require redesigning schema or the query. This is a rare/complex case though.
  * It's useful to remove JOINs or conditions that don't matter to the problem. Like remove a 5 table join which can be boiled down to a 2 table join. And if you are able to reproduce the problem with the 2 table join query, it'll be much simpler to troubleshoot.

Aggregates and DISTINCT
=======================

Plain aggregate
---------------
* Aggregating everything into one group, producing one row. SELECT count(*) from foo where x = 2;
* Gets more complex when there's a group by and there are many rows in the result. In that case, sorted and hashed aggregates.

Sorted aggregate
----------------
* Sort the data, reads through it, when you see a new value, aggregate the prior group.

Hashed aggregate
----------------
* Insert each input row into a hash table based on the gruoping columns. At the end, aggregate all the groups.
* Fast.

Statistics
==========
* Seq scan vs. index scan vs. bitmap index scan. Nested loop vs. merge join vs. hash join?
* ANALYZE gathers all stats, and decides. With bad statistics, the plans will suck.

Confusing the planner
---------------------
* Examples of queries where the planner is not able to estimate properly, no matter how good the stats, etc.
* Planner may underestimate row count, choosing index scan over a sequential scan, or nested loop instead of a hash or merge join.
* Planner overestimates row count, so it may choose a seq scan instead of an index scan, or a merge or hash join instead of a nested loop.
* Small values for LIMIT tilt the planner toward plans that have a fast start and magnify the effect of bad estimates. You can hack this out

    SELECT * FROM foo where a= 1 AND b = 1

* Impossible to know the number of rows that will match both of these criteria. The planner is affected by this.

    SELECT * FROM foo WHERE (a + 0) = a

* Planner can't know, will assume 0.5% of rows will match.

Query planner params
--------------------

* seq_page_cost (1.0), random_page_cost (4.0) - reduce these if DB is largely cashed.
* default_statistics_target (100) - Level of detail of stats gathering. Can be overridden on specific columns.
* enable_hashjoin, enable_sort, etc. Useful for testing queries and plans. Don't use these in production. If you must, use SET to disable them for the one session/query.
* work_mem - Amount of memory per sort or hash. Each query can use many sorts or hashes.
* from_collapse_limit, join_collapse_limit, geqo_threshold. Can be reaised, but some queries can blow up when this is increased. Careful.

Things that are slow
--------------------
* DISTINCT
* Pl/pgsql loops.
* Repeated calls to SQL or PL/pgsql functions. SELECT id, some_function(id) FROM table; <- bad.

[https://gist.github.com/hgmnz/883144]