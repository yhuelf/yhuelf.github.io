---
layout: post
title:  "What are these slow COMMIT in my PostgreSQL logs?"
date:   2021-09-30 12:15:25 +0200
tags: PostgreSQL performance pg_stat_statements
---

I recently came across these surprising logs over the course of an audit at one
of [Dalibo](https://dalibo.com/)'s client:
(the logs are strimmed for better readability and anonymisation).

```
Jul 26 00:07:51 postgres10[42230]: client=127.0.0.1 LOG:  duration: 406.403 ms  statement: select 1
Jul 26 00:08:02 postgres10[42237]: client=127.0.0.1 LOG:  duration: 780.613 ms  statement: COMMIT
```

I could come up with an explanation for the slow COMMIT (heavy writes => many
WAL buffers to sync on disk, and maybe WAL compression added some latency) but
I wouldn't know what to say to the client about this very
long `select 1`. This query clearly came from PgBouncer: "a simple do-nothing
query to check if the server connection is alive", but that didn't help much.

{% include babar.html url="/img/babar.jpg" description="Me (embarrassed) and
my client." %}

Let's have a look at `pg_stat_statements`:

```
postgres=# SELECT rolname, datname, query, calls, min_time, max_time, mean_time
  FROM pg_stat_statements s JOIN pg_database d ON (s.dbid = d.oid)
  JOIN pg_roles r ON (r.oid = s.userid) WHERE octet_length(query) < 15
  AND rolname = 'anon' AND datname = 'anon';

 rolname | datname |    query    |    calls    | min_time |  max_time   |      mean_time      
---------+---------+-------------+-------------+----------+-------------+---------------------
 anon    | anon    | COMMIT      |  2078240339 | 0.000247 |   59.077196 | 0.00119367747128697
 anon    | anon    | ROLLBACK    |          81 | 0.000689 |    0.013588 | 0.00160246913580247
 anon    | anon    | BEGIN       |  2121191643 | 0.000251 |   57.558841 | 0.00114355394816151
 anon    | anon    | DISCARD ALL | 17515013300 | 0.004349 | 1442.615547 |  0.0350356836662835
(4 rows)
```

The `select 1` aren't there, and the longest `COMMIT` lasts 59ms. So what's
going on? The normalized query `select $1` has probably been evicted from
pg_stat_statements over the course of a deallocation, although its frequency
is rather high. But the parameter `pg_stat_statements.max` is set to `1000`, and
this a heavy loaded server with about 30,000 tx/s and 1200 backends during the
busiest periods. Now, we still don't have any explaination about the
inconsistency between pg_stat_statements and the logs, regarding the slow
COMMITs.

So I started to look at the code (pg_stat_statements.c). Initially, I aimed at
understanding better how the query time was computed. But I didn't go too far,
because I read this line of comment:

```
Note about locking issues: to create or delete an entry in the shared
hashtable, one must hold pgss->lock exclusively
```

... and then it clicked in my mind: it was probably a locking issue!

Or maybe not.

Anyway, it's not too difficult to test this hypothesis. Let's use pgbench with
16 custom scenarii like this one:

```
BEGIN;
SELECT 1 FROM t1;
SELECT 1 FROM t2;
SELECT 1 FROM t3;
SELECT 1 FROM t4;
SELECT 1 FROM t5;
SELECT 1 FROM t6;
SELECT 1 FROM t7;
SELECT 1 FROM t8;
COMMIT;
```

The queries are unique, event after normalization.

Here is a script to create the scenarii and the tables:

```sh
#!/bin/bash

psql -c "CREATE DATABASE pgbench;"

for x in `seq 1 128`; do
        psql -d pgbench -c "CREATE TABLE t$x (id int);"
done


foo() {
        x=$1
        start=$((1 + x*8))
        stop=$((8 + x*8))

        echo "BEGIN;"
        for x in `seq $start $stop`; do
                echo "SELECT 1 FROM t$x;"
        done
        echo "COMMIT;"
}

for y in `seq 0 15`; do
        f="scenario_$y.sql"
        foo $y > $f
done
```

These 16 scenarii amounts to 128 unique queries, which we need to reproduce the
locking problem, since `pg_stat_statements.max` lowest possible value is `100`.

And here is another one to launch 16 pgbench in parallel, one per scenario:

```sh
#!/bin/bash

psql -d pgbench -c "SELECT pg_stat_reset();"
psql -d pgbench -c "SELECT pg_stat_statements_reset();"

for x in `seq 0 15`; do
        pgbench -d pgbench -T 100 -f scenario_$x.sql 2> /dev/null &
        pids[${i}]=$!
done

for pid in ${pids[*]}; do
    wait $pid
done

psql -c "select xact_commit from pg_stat_database where datname = 'pgbench';"
psql -d pgbench -c "select * from pg_stat_statements_info;"
```

The last line of the above script uses a view that appeared with PostgreSQL 14.
Indeed it seems I'm not the only one to have noticed this problem
[(see this thread on pgsql-hackers)](https://www.postgresql.org/message-id/0d9f1107772cf5c3f954e985464c7298%40oss.nttdata.com),
and it's now possible to know how many deallocations occured since the last reset
of pg_stat_statements. A high number for a small period indicates that
`pg_stat_statements.max` is too low.

Let's launch the pgbench script with `pg_stat_statements.max = 400`:

```
 xact_commit 
-------------
      958181
(1 row)

 dealloc |          stats_reset          
---------+-------------------------------
       0 | 2021-10-01 16:35:51.180729+02
```

So we have an average of 9582 tx/s, and no deallocations.

Let's launch the pgbench script again, with `pg_stat_statements.max = 100`:

```
 xact_commit 
-------------
      714498
(1 row)

 dealloc |          stats_reset          
---------+-------------------------------
  144878 | 2021-10-01 16:33:19.156198+02
```

That's 25% less transactions!

Also, in the logs, we observe 4 queries that lasted more than 20ms:

```
2021-10-04 08:43:53.584 CEST [54262] LOG:  duration: 23.487 ms  statement: SELECT 1 FROM t63;
2021-10-04 08:43:56.335 CEST [54270] LOG:  duration: 21.671 ms  statement: SELECT 1 FROM t21;
2021-10-04 08:44:04.226 CEST [54265] LOG:  duration: 20.455 ms  statement: SELECT 1 FROM t109;
2021-10-04 08:44:28.480 CEST [54271] LOG:  duration: 48.653 ms  statement: COMMIT;
```

... compared to only one when `pg_stat_statements.max` is set to `400` (and
pgbench lauched with rate limiting, so that the transaction rate matches the
previous test).

What is the cause of this performance drop? It is clearly linked to these 144878
deallocations, but is it really the locking problem that I thought of? Let's use
`perf` together with [Flame Graph](https://www.brendangregg.com/perf.html#FlameGraphs)
for a On-CPU analysis of one of the 16 backends:

![perf100](/img/pg_stat_statements_v14_max_100.svg)
![perf400](/img/pg_stat_statements_v14_max_400.svg)

The two flame graphs look similar, except for the leftmost tower of the top one
(`pg_stat_statements.max = 100`). Notice the WaitEventSetWait frame that
accounts for 7.4% of all samples.

> **Note:** You can download the svg files (right-click on the image) and run them
> in your browser to get the full Flame Graph experience.

If it's really a locking problem, a
[Off-CPU analysis](https://www.brendangregg.com/offcpuanalysis.html) is more adequate:

![off100](/img/pg_stat_statements_v14_max_100_offcpu.svg)
![offf400](/img/pg_stat_statements_v14_max_400_offcpu.svg)

This time, the two graphs are very different!

The tower containing the `futex_wait` accounts for 28% percent of the off-cpu
time, and I think it explains completely the performance drop. I'm unsure how to
explain the other big tower (in the middle), but here's my guess: this is off-cpu
time which would otherwise have been caused by involontary context switches, in
the absence of locking within pg_stat_statements.

# Conclusion

The default value of `pg_stat_statements.max` is `5000`. For some unknown
reason, the client decreased this value to `1000`. He probably assumed that the
performance penalty of having pg_stat_statements loaded would be lesser, but on
the contrary it resulted in a much bigger one. The documentation doesn't say
anything about the danger of setting this value too low.
Only recently, in version 14,
we can rely on the view `pg_stat_statements_info`, containing the
counter `dealloc`and the timestamp `stats_reset`, to get an idea about whether
or not this parameter is set too low. On my 4-cpu machine, the performance drop
is noticeable with more than 700 deallocations per second, but to be on the safe
side, I would recommend to increase this parameter if there's more than 10
deallocations per second, also in order to avoid the extra CPU cost of
these deallocations.
For PostgreSQL versions prior to `14`, there's no simple way of knowing if this
parameter is set high enough, but the presence of slow queries that shouldn't be
slow in your logs is a good hint. Anyway, I would recommend sticking with the
default value of `pg_stat_statements.max`, or increasing it, but never
decreasing it.

_license: © 2021 -- [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)_
