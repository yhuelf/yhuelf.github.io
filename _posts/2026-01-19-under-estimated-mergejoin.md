---
title: "The strange case of the underestimated Merge Join node"
layout: post
date:   2026-01-19 09:15:25 +0200
tags: optimizer,planner
---

This post appeared first on the [Dalibo blog](https://blog.dalibo.com/2026/01/12/under-estimated-mergejoin.html).

*Brest, France, 19 January 2026*

We recently encountered a strange optimizer behaviour, reported by one of our customers:

> **Customer**: "Hi Dalibo, we have a query that is very slow on the first execution after a batch process,
> and then very fast. We initially suspected a caching effect, but then we noticed that the execution
> plan was different."
>
> **Dalibo**: "That's a common issue. Autoanalyze didn't have the opportunity to process the table after the batch
> job had finished, and before the first execution of the query. You should run the `VACUUM ANALYZE` command
> (or at least `ANALYZE`) immediately after your batch job."
>
> **Customer**: "Yes, it actually solves the problem, but… your hypothesis is wrong. We looked at `pg_stat_user_tables`,
> and are certain that the tables were not vacuumed or analyzed between the slow and
> fast executions. We don't have a production problem, but we would like to understand."
>
> **Dalibo**: "That's very surprising! we would also like to understand…"

So let's dive in!

<!--MORE-->

## Execution plans

The query is quite basic (table and column names have been anonymized):

```sql
SELECT    *
FROM      bar
LEFT JOIN foo ON (bar.a = foo.a)
WHERE     id = 10744501
ORDER BY  bar.x DESC, foo.x DESC;
```

Here's [the plan](https://explain.dalibo.com/plan/61cee01aa499bad0) of the first execution of the query after the batch job:

```
                                                                 QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------
 Sort (cost=17039.22..17042.11 rows=1156 width=786) (actual time=89056.476..89056.480 rows=6 loops=1)
   Sort Key: bar.x DESC, foo.x DESC
   Sort Method: quicksort Memory: 25kB
   Buffers: shared hit=2255368 read=717581 dirtied=71206 written=11997
   -> Merge Right Join (cost=2385.37..16980.41 rows=1156 width=786) (actual time=89056.428..89056.432 rows=6 loops=1)
         Inner Unique: true
         Merge Cond: (foo.a = bar.a)
         Buffers: shared hit=2255365 read=717581 dirtied=71206 written=11997
         -> Index Scan using foo_fk1 on foo (cost=0.57..145690068.16 rows=80462556 width=734) (actual time=89050.555..89050.557 rows=1 loops=1)
               Buffers: shared hit=2255360 read=717574 dirtied=71206 written=11997
         -> Sort (cost=2384.81..2387.70 rows=1156 width=52) (actual time=5.853..5.854 rows=6 loops=1)
               Sort Key: bar.a
               Sort Method: quicksort Memory: 25kB
               Buffers: shared hit=5 read=7
               -> Index Scan using bar_fk1 on bar (cost=0.58..2326.00 rows=1156 width=52) (actual time=1.514..5.808 rows=6 loops=1)
                     Index Cond: (bar.id = 10744501)
                     Buffers: shared hit=5 read=7
 Settings: effective_cache_size = '20GB', random_page_cost = '2', work_mem = '100MB'
 Planning:
   Buffers: shared hit=11073 read=10738 dirtied=2
 Planning Time: 209.686 ms
 Execution Time: 89056.610 ms
```

And here's [the plan](https://explain.dalibo.com/plan/7d75010b3gb693g7) of the 2nd execution of the query:

```
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------
 Sort (cost=544154.99..544157.54 rows=1020 width=789) (actual time=0.222..0.224 rows=6 loops=1)
   Sort Key: bar.x DESC, foo.x DESC
   Sort Method: quicksort Memory: 25kB
   Buffers: shared hit=39
   -> Nested Loop Left Join (cost=1.15..544104.02 rows=1020 width=789) (actual time=0.087..0.164 rows=6 loops=1)
         Buffers: shared hit=36
         -> Index Scan using bar_fk1 on bar (cost=0.58..2052.42 rows=1020 width=52) (actual time=0.073..0.127 rows=6 loops=1)
               Index Cond: (bar.id = 10744501)
               Buffers: shared hit=12
         -> Index Scan using foo_fk1 on foo (cost=0.57..528.77 rows=265 width=737) (actual time=0.004..0.004 rows=0 loops=6)
               Index Cond: (foo.a = bar.a)
               Buffers: shared hit=24
 Settings: effective_cache_size = '20GB', random_page_cost = '2', work_mem = '100MB'
 Planning:
   Buffers: shared hit=329
 Planning Time: 10.390 ms
 Execution Time: 0.373 ms
```

## Searching for the culprit

The main difference between the two plans is the join strategy. It's a
[Merge Join](https://postgrespro.com/blog/pgsql/5969770) in the first one, and a **Nested Loop Join** in the second one.

Since the two tables haven't been analyzed or vacuumed, we know the statistics[^stats] didn't change between the
two executions. We observe that the first execution modifies a lot of buffers (`dirtied=71206`), and that's our first clue.

[^stats]: Not only those contained in `pg_statistics`, but also those in `pg_class`.

    The latter can also be updated by an index creation or a reindexation, but this was not the case here.

I was surprised by the
[high total cost of the outer index scan](https://explain.dalibo.com/plan/61cee01aa499bad0#plan/node/3)
in the
`Merge Join` node (145 millions), especially since the cost of the `Merge Join` itself is only 16,980.
In fact, the planner knows that only a small fraction
of one of the two `Merge Join` child nodes will be executed.
This occurs, for example, when the histograms of the join clause columns (`foo.a` and `bar.a`, in our case) have only
a small overlap, or no overlap
(I would like to thank Robert Haas for teaching me this when I asked about it on the
[PostgreSQL Hacking Discord](https://discord.gg/bx2G9KWyrY).

In our case, the histograms are as follows:

* histogram of column **foo.a**: `{1532523860,1532923673, <97 more values>, 1573407772,1573803559}`
* histogram of column **bar.a**: `{16877,15720140, <97 more values>, 1485178901,1499389426}`

So they actually don't overlap at all.

When I saw this, I suddenly remembered a case from a few years ago when we had to deal
with a query whose planning time was absurdly high on the first execution. There were many index tuples
pointing to dead heap tuples.

When the optimizer computes the selectivity of one histogram,
it may probe the index to determine the actual extreme values.
(The function is `get_actual_variable_endpoint()` [^get_actual_variable_endpoint],
called by `get_actual_variable_range()`
when the first or last histogram entry is accessed
by the function `ineq_histogram_selectivity()`).

[^get_actual_variable_endpoint]: See [selfuncs.c](https://github.com/postgres/postgres/blob/6c99c715ddb338e169c2ffd2a4cf754fa510cccb/src/backend/utils/adt/selfuncs.c#L6753)


At that time, this function could end up
reading many heap pages, and also writing index pages in order to set the
[dead flag](https://www.cybertec-postgresql.com/en/killed-index-tuples/)
for the index tuples pointing to dead heap tuples. This explained the very high planning time.

However, in November 2022, a
[patch](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9c6ad5eaa957bdc2132b900a96e0d2ec9264d39c)
from Simon Riggs limited the number of heap pages `get_actual_variable_endpoint()` could fetch.
After reading 100 heap pages, it gives up, preventing the planning time explosion.

As a consequence, we have this strange case where the first execution of a query is planned
differently from the second one.
The first time, `get_actual_variable_endpoint()` gives up, so the planner uses
the extreme value recorded in the histogram.
The second time, if enough of the index tuples pointing to dead heap
tuples were _killed_, `get_actual_variable_endpoint()` successfully returns
the _actual_ extreme value.
Thus, the planner works with more accurate information.

## Verifying the hypothesis

### 1st execution

Assuming our hypothesis is correct, on the first execution, `get_actual_variable_endpoint()` gives up. Since our
histograms don't overlap at all, the planner could estimate a null total cost for the `Merge Join` node. However,
the planner is wary of extremely low or large selectivity estimates[^default_cutoff],
so it estimates (by simplifying) that it will run at least a small fraction
(0.01 %, or one-hundredth of the histogram resolution)
of the outer child node, and, at worst, almost all (99.99 %) of the inner child node to find the first matching tuple, and
all of it to find them all. So the startup cost of the `Merge Join`
node is close to the total cost of the inner child node. Its run cost (total cost minus startup cost)
should be close to `0.0001` of the total cost of the outer child node.

Let's do the math. In the above plan, the `Merge Join` startup cost is 2385, which is slightly higher than the
total cost of the inner index scan multiplied by 0.9999 (2326). Therefore, it's consistent.
The run cost equals 16,980 - 2,385 = 14,595, which is slightly higher than the total cost of the outer index
scan multiplied by 0.0001 (145,690,068 × 0.0001 = 14,569). Once again, this is consistent with our hypothesis.

[^default_cutoff]: See [selfuncs.c](https://github.com/postgres/postgres/blob/REL_18_STABLE/src/backend/utils/adt/selfuncs.c#L996)

### 2nd execution

Assuming our hypothesis is correct, on the second execution, `get_actual_variable_endpoint()` successfully returns the
actual extreme values, thanks to the already killed tuples in the index.
And we expect the total cost of the `Merge Join` to
be higher than the total cost of the `Nested Loop Join` node in the above "fast" plan, which is 544,104.

We can't say anything
very accurate here, because we didn't ask our customer for the real minimum value `foo.a`
and the maximum value `bar.a`.
Besides, it would be very tedious to unfold the computations
performed by the planner. Suffice it to say that if PostgreSQL estimates that it will have to run at least
0.5 % of the outer index scan, then the total cost of the `Merge Join` node exceeds
0.005 × 145,690,068 = 728,450, which is greater than the total cost of the `Nested Loop` node.
So the planner chooses the `Nested Loop`.


## A script to reproduce the case

The following script reproduces the case. First, we create two tables and disable autovacuum for them.

We insert one million rows into `foo`, with values for column `a` between 200,000 and 299,999.

We insert one million rows into `bar`, with values for column `a` between 100,000 and 199,999
(without any overlap with the `foo.a`).

We run `VACUUM ANALYZE` on both tables, and then insert 100,000 rows in `foo`,
with values for column `a` between 185,000 and 195,000 (inclusive),
so there is now some overlap. The autovacuum is inhibited here,
and we can verify that the histograms don't overlap in the statistics view. We create the indexes.

Then we delete all the rows that were inserted by the last `INSERT`,
except the last value (`a` = 195,000).
So the minimum value for `foo.a` is now 195,000, but the beginning of the index still contains many
entries pointing to dead heap tuples, and `get_actual_variable_endpoint()` will abort and return `false`
on the first execution.

When you run the script, the last command should output a plan
with an underestimated `Merge Join` node.
You'll see _Heap Fetches_, hinting that dead tuples entries are cleaned in the index.

Running this EXPLAIN command a second time
will result in a `Nested Loop Join`.
With `SET enable_nestloop TO off`, you can
verify that the cost of the `Merge Join`is much higher.

```sql
DROP TABLE IF EXISTS foo, bar;

CREATE UNLOGGED TABLE foo(id int, a int);
CREATE UNLOGGED TABLE bar(id int, a int);

ALTER TABLE foo SET (autovacuum_enabled = off);
ALTER table bar SET (autovacuum_enabled = off);

INSERT INTO foo SELECT i,  200000 + random() * 100000 FROM generate_series(1, 999999) i;
INSERT INTO bar SELECT i%100000, 100000 + i/10 FROM generate_series(1, 999999) i;

VACUUM ANALYZE foo, bar;

INSERT INTO foo SELECT 1000000 + i, 185000 + i/10 FROM generate_series(1, 100000) i;

SELECT * FROM pg_stats WHERE tablename IN ('foo', 'bar') AND ATTNAME = 'a';

CREATE INDEX ON foo(a);
CREATE INDEX ON bar(id, a);

DELETE FROM foo WHERE a >= 185000 and a < 195000;

EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT foo.a FROM foo JOIN bar USING (a) WHERE bar.id = 2047 ORDER BY a;
```

## Conclusion

We found a case in which the query plan can change between two executions, while the data and statistics
remain exactly the same. As far as I know, there are no other known cases[^1], but if you know of one, or
have any questions or feedback about this article, please contact Frédéric Yhuel at
<mailto:frederic.yhuel@dalibo.com>.

One more thing: typically, estimation errors in the planner result in an
output row count that differs greatly from the actual count, for a given node. This kind of
discrepancy is easy to spot, and graphical tools like [explain.dalibo.com](https://explain.dalibo.com) and [explain.depesz.com](https://explain.depesz.com) highlight it.
A common example is when the planner selects a `Nested Loop Join` instead of a `Hash Join` because the estimated
number of output rows of the outer child node is significantly underestimated. The reverse also happens, albeit
less frequently. However, we have encountered another type of estimation error that is much more difficult to spot,
because it comes from the direct use of indexes by the planner.

[^1]: I can think of one other case, involving GEQO and a different GEQO seed, but it's not very interesting to me.
