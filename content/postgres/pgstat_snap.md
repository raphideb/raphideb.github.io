---
title: "pgstat_snap"
linktitle: "pgstat_snap"
type: "docs"
weight: 10
description: "Extension for PostgreSQL to create timestamped snapshots of pg_stat_statements"
---
<img class="floatimg" src="/postgres/pgstat_snap.jpg" alt="active_session_history" width="25%" height="25%"/>
<p>The cumulative statistics system (CSS) in PostgreSQL and pg_stat_statements in particular lack any timing information, all values are cumulative and the only way to figure out the difference between query executions is to reset the stats every time or work with averages. 

With the pgstat_snap extension, you can create timestamped snapshots of pg_stat_statements and pg_stat_activity when needed. It also provides views that show the difference between every snapshot for every query and database. 

If you haven't already, download the extension from my github repo: [https://github.com/raphideb/pgstat_snap](https://github.com/raphideb/pgstat_snap)

The README.md is a shorter version of this post with just the essentials.</p>
<div style="clear: both;"></div>

## Installation
To install the extension, download these files from my github repo:
```
pgstat_snap--1.0.sql
pgstat_snap.control
```
And copy them to the extension directory of PostgreSQL
```bash
sudo cp pgstat_snap* $(pg_config --sharedir)/extension/
```
You can then install the extension in any database that has the pg_stat_statements extension enabled, superuser right are NOT needed:
```sql
create extension pgstat_snap;
```
It can also be installed into a different schema but be sure to have it included in the search_path:
```sql
create extension pgstat_snap schema my_schema;
```
This will create the following tables and views:
```
  pgstat_snap_stat_history   -> pg_stat_statements history (complete snapshot)
  pgstat_snap_act_history    -> pg_stat_activity history (complete snapshot)
  pgstat_snap_diff_all       -> view containing the sum and difference of each statement between snapshots
  pgstat_snap_diff           -> view containing only the difference of each statement between snapshots
```
## Requirements
pg_stat_statements must be loaded and tracking activated in the postgres config:  
```
shared_preload_libraries = 'pg_stat_statements'
```
Recommended settings:  
```
pg_stat_statements.track = all  
pg_stat_statements.track_utility = off
```
The extension has to be created in the database in which pgstat_snap will be installed:
```
create extension pg_stat_statements;
```
## Usage
### Collect snapshots
Start gathering snapshots with, e.g. every 1 second 60 times:
```sql
CALL pgstat_snap_collect(1, 60);
```
Or gather a snapshot every 5 seconds for 10 minutes:
```sql
CALL pgstat_snap_collect(5, 120);
```
**IMPORTANT:** on very busy clusters with many databases a lot of data can be collected, 500mb per minute or more. Don't let it run for a very long time with short intervals, unless you have the disk space for it.

After you collected the snapshots, check out the [Drilldown Section](/postgres/pgstat_snap/#drill-down) for how to query the views.

### Reset
Because everything is timestamped, a reset is usually not needed between CALLs to create_snapshot. But you can, for example to cleanup the tables and free some space. You can also reset pg_stats*.

Reset all pgstat_snap tables with:
```sql
SELECT pgstat_snap_reset();   -> reset only pgstat_snap.pgstat*history tables
SELECT pgstat_snap_reset(1);  -> also select pg_stat_statements_reset()
SELECT pgstat_snap_reset(2);  -> also select pg_stat_reset()
```
### Uninstall
To completely uninstall pgstat_snap, run:
```sql
DROP EXTENSION pgstat_snap;
```

### Help
The extension also has a help function, listing all views, tables and functions:
```sql
SELECT pgstat_snap_help();
```

## How it works 
The first argument to create_snapshot is the interval in seconds, the second argument is how many snapshots should be collected. Every *interval* seconds, *select * from pg_stat_statements* will be inserted into *pgstat_snap_stat_history* and *select * from pgstat_act_statements* into *pgstat_snap_act_history*. 

For every row, a timestamp will be added. Only rows where the "rows" column has changed will be inserted into *pgstat_snap_stat_history* and always only one row per timestamp, dbid and queryid. Every insert is immediately committed to be able to live follow the tables/views.

The views have a *_d* column which displays the difference between the current row and the last row where the query was recorded in the pgstat_snap_stat_history table. *NULL* values in *rows_d*, *calls_d* and so on mean, that no previous row for this query was found because it was executed the first time since create_snapshot was running. 

The views also contain the datname, wait events and the first 20 characters of the query, making it easier to identify queries of interest.

## Views description

### exec_ms_d
The views show the difference between each snapshot and not second by second. For example, when a query was called one time (calls_d=1) and has an execution time of 3 seconds (exec_ms_d=3000), it ran for 3 seconds since the last time it was executed. This is why you might see exec_ms_d higher than a second in one particular second. You can filter by queryid if you want to see exactly when a particular query was executed. 

### wait_event
This is a bit wonky as it only shows what one backend process executing a query was waiting for when the snapshot was collected. It does not mean that all processes executing that query had the same wait event but it might help point you in the right direction, for example if one query always has a lock.

### pgstat_snap_diff
This view only contains the difference between the previous and next execution of a queryid/dbid pair:
| Column Name | Description |
|-------------|-------------|
| **snapshot_time** | Timestamp |
| **queryid** | Query ID |
| **query** | Query Text (first 20 characters) |
| **datname** | Database Name |
| **usename** | Username |
| **wait_event_type** | Event Type - NULL if no wait occurred |
| **wait_event** | Wait Event - NULL if no wait occurred |
| **rows_d** | Difference in rows from the previous snapshot_time |
| **calls_d** | Difference in calls from the previous snapshot_time |
| **exec_ms_d** | Difference in total execution time from the previous snapshot_time |
| **sb_hit_d** | Difference in shared block hits from the previous snapshot_time |
| **sb_read_d** | Difference in shared block reads from the previous snapshot_time |
| **sb_dirt_d** | Difference in shared blocks dirtied from the previous snapshot_time |
| **sb_write_d** | Difference in shared blocks written from the previous snapshot_time |

#### Sample output
```sql

select * from pgstat_snap_diff order by 1;
    snapshot_time    |       queryid        |        query        |  datname  | usename  | wait_event_type |  wait_event   | rows_d | calls_d |  exec_ms_d   | sb_hit_d | sb_read_d | sb_dirt_d | sb_write_d
---------------------+----------------------+---------------------+-----------+----------+-----------------+---------------+--------+---------+--------------+----------+-----------+-----------+------------
 2025-07-10 20:33:25 |  8092674635626072858 | SELECT b.bid,       | dwhbench  | dwhuser  |                 |               |    200 |       2 |  3142.512490 |     2090 |    325798 |         0 |         14
 2025-07-10 20:33:25 | -1263534209317543048 | UPDATE pgbench_bran | oltpbench | oltpuser | Lock            | transactionid |   3249 |    3249 |  1768.453916 |    15380 |         3 |         1 |          1
 2025-07-10 20:33:25 |  7095169429556301570 | INSERT INTO pgbench | dwhbench  | dwhuser  | LWLock          | BufferMapping |      8 |       8 | 10730.907623 |   124847 |     93865 |         2 |       3375
 2025-07-10 20:33:26 |  8092674635626072858 | SELECT b.bid,       | dwhbench  | dwhuser  |                 |               |    300 |       3 |  2947.915426 |     8355 |    483493 |         0 |         37
 2025-07-10 20:33:26 | -2177783729771621131 | UPDATE pgbench_acco | oltpbench | oltpuser |                 |               |   6414 |    6414 |   541.561524 |    30523 |     12649 |      7321 |        904
 2025-07-10 20:33:26 |  5796608934964327314 | INSERT INTO pgbench | oltpbench | oltpuser | Client          | ClientRead    |  13879 |   13879 |   135.870508 |    14057 |        10 |        94 |         99
 2025-07-10 20:33:26 | -1263534209317543048 | UPDATE pgbench_bran | oltpbench | oltpuser | Lock            | transactionid |   3162 |    3162 |  1868.084329 |    14876 |         0 |         0 |          0
 2025-07-10 20:33:27 |  8431081492085281285 | UPDATE pgbench_tell | oltpbench | oltpuser |                 |               |   5889 |    5889 |   506.675114 |    29865 |         0 |         0 |          0
 2025-07-10 20:33:27 |  8537798620180856986 | SELECT b.bid, SUM(a | dwhbench  | dwhuser  |                 |               |    600 |       6 | 14137.163555 |     9804 |    973812 |         0 |         17
 2025-07-10 20:33:27 |  7095169429556301570 | INSERT INTO pgbench | dwhbench  | dwhuser  |                 |               |      4 |       4 |  5569.364955 |    43608 |     65748 |         2 |       4553
 2025-07-10 20:33:27 |  8092674635626072858 | SELECT b.bid,       | dwhbench  | dwhuser  |                 |               |    400 |       4 |  9745.461818 |     8764 |    646980 |         0 |         22
 2025-07-10 20:33:27 | -1263534209317543048 | UPDATE pgbench_bran | oltpbench | oltpuser | Lock            | transactionid |   2718 |    2718 |  1718.941857 |    12808 |         0 |         0 |          0
 2025-07-10 20:33:27 |  5981687818415711060 | SELECT b.bid, SUM(a | dwhbench  | dwhuser  |                 |               |     40 |       4 |  4591.836860 |    12552 |    643240 |         0 |         38
 ```

### pgstat_snap_diff_all
This view contains the difference between the previous and next execution of a queryid/dbid pair and the sum of the fields as recorded in pg_stat_statements at that time:

| Column Name | Description |
|-------------|-------------|
| **snapshot_time** | Timestamp |
| **queryid** | Query ID |
| **query** | Query Text (first 20 characters) |
| **datname** | Database Name |
| **usename** | Username |
| **wait_event_type** | Event Type - NULL if no wait occurred |
| **wait_event** | Wait Event - NULL if no wait occurred |
| **rows** | Value of rows at this time in pg_stat_statements |
| **rows_d** | Difference in rows from the previous snapshot_time |
| **calls** | Value of calls at this time in pg_stat_statements |
| **calls_d** | Difference in calls from the previous snapshot_time |
| **exec_ms** | Value of total execution time at this time in pg_stat_statements |
| **exec_ms_d** | Difference in total execution time from the previous snapshot_time |
| **sb_hit** | Value of shared block hits at this time in pg_stat_statements |
| **sb_hit_d** | Difference in shared block hits from the previous snapshot_time |
| **sb_read** | Value of shared block reads at this time in pg_stat_statements |
| **sb_read_d** | Difference in shared block reads from the previous snapshot_time |
| **sb_dirt** | Value of shared blocks dirtied at this time in pg_stat_statements |
| **sb_dirt_d** | Difference in shared blocks dirtied from the previous snapshot_time |
| **sb_write** | Value of shared blocks written at this time in pg_stat_statements |
| **sb_write_d** | Difference in shared blocks written from the previous snapshot_time |

#### Sample output
```sql
select * from pgstat_snap_diff_all order by 1;

    snapshot_time    |       queryid        |        query        |  datname  | usename  | wait_event_type |  wait_event   |  rows  | rows_d | calls  | calls_d |    exec_ms    |  exec_ms_d   | sb_hit  | sb_hit_d  | sb_read  | sb_read_d | sb_dirt | sb_dirt_d | sb_write | sb_write_d
---------------------+----------------------+---------------------+-----------+----------+-----------------+---------------+--------+--------+--------+---------+---------------+--------------+---------+-----------+----------+-----------+---------+-----------+----------+------------
 2025-07-10 20:33:25 |  5981687818415711060 | SELECT b.bid, SUM(a | dwhbench  | dwhuser  |                 |               |   3250 |     30 |    325 |       3 | 506209.382549 |  5098.012464 |  578683 | -51638449 | 52703041 |    485909 |       0 |         0 |     5078 |         29
 2025-07-10 20:33:25 |  8431081492085281285 | UPDATE pgbench_tell | oltpbench | oltpuser | Lock            | transactionid | 362116 |        | 362116 |         |  30769.931769 |              | 1840386 |           |       22 |           |     160 |           |       20 |
 2025-07-10 20:33:25 |  8092674635626072858 | SELECT b.bid,       | dwhbench  | dwhuser  |                 |               |  32600 |    200 |    326 |       2 | 595288.665764 |  3142.512490 |  884799 | -51349764 | 52560361 |    325798 |       0 |         0 |     9754 |         14
 2025-07-10 20:33:25 |  8537798620180856986 | SELECT b.bid, SUM(a | dwhbench  | dwhuser  |                 |               |  33300 |    400 |    333 |       4 | 688264.881853 |  4895.604348 | 2960269 | -48019037 | 51631691 |    652385 |       0 |         0 |    14832 |         20
 2025-07-10 20:33:25 | -1263534209317543048 | UPDATE pgbench_bran | oltpbench | oltpuser | Lock            | transactionid | 362121 |   3249 | 362121 |    3249 | 192172.014593 |  1768.453916 | 1688815 |   1688792 |       26 |         3 |      48 |         1 |       13 |          1
 2025-07-10 20:33:25 |  7095169429556301570 | INSERT INTO pgbench | dwhbench  | dwhuser  | LWLock          | BufferMapping |    321 |      8 |    321 |       8 | 445005.367489 | 10730.907623 | 4231400 |   -219327 |  4544592 |     93865 |     189 |         2 |   342185 |       3375 2025-07-10 20:33:26 |  8092674635626072858 | SELECT b.bid,       | dwhbench  | dwhuser  |                 |               |  32900 |    300 |    329 |       3 | 598236.581190 |  2947.915426 |  893154 | -51667207 | 53043854 |    483493 |       0 |         0 |     9791 |         37 2025-07-10 20:33:26 | -2177783729771621131 | UPDATE pgbench_acco | oltpbench | oltpuser |                 |               | 365288 |   6414 | 365288 |    6414 | 151503.887700 |   541.561524 | 2380434 |   1681465 |   711618 |     12649 |  511717 |      7321 |    75839 |        904 2025-07-10 20:33:26 |  5796608934964327314 | INSERT INTO pgbench | oltpbench | oltpuser | Client          | ClientRead    | 365282 |  13879 | 365282 |   13879 |   3669.242144 |   135.870508 |  370809 |    370766 |       53 |        10 |    2370 |        94 |     2562 |         99 2025-07-10 20:33:26 | -1263534209317543048 | UPDATE pgbench_bran | oltpbench | oltpuser | Lock            | transactionid | 365283 |   3162 | 365283 |    3162 | 194040.098922 |  1868.084329 | 1703691 |   1703665 |       26 |         0 |      48 |         0 |       13 |          0 2025-07-10 20:33:27 |  8431081492085281285 | UPDATE pgbench_tell | oltpbench | oltpuser |                 |               | 368005 |   5889 | 368005 |    5889 |  31276.606883 |   506.675114 | 1870251 |   1870229 |       22 |         0 |     160 |         0 |       20 |          0 2025-07-10 20:33:27 |  8537798620180856986 | SELECT b.bid, SUM(a | dwhbench  | dwhuser  |                 |               |  33900 |    600 |    339 |       6 | 702402.045408 | 14137.163555 | 2970073 | -48661618 | 52605503 |    973812 |       0 |         0 |    14849 |         17 2025-07-10 20:33:27 |  7095169429556301570 | INSERT INTO pgbench | dwhbench  | dwhuser  |                 |               |    325 |      4 |    325 |       4 | 450574.732444 |  5569.364955 | 4275008 |   -269584 |  4610340 |     65748 |     191 |         2 |   346738 |       4553 2025-07-10 20:33:27 |  8092674635626072858 | SELECT b.bid,       | dwhbench  | dwhuser  |                 |               |  33300 |    400 |    333 |       4 | 607982.043008 |  9745.461818 |  901918 | -52141936 | 53690834 |    646980 |       0 |         0 |     9813 |         22 2025-07-10 20:33:27 | -1263534209317543048 | UPDATE pgbench_bran | oltpbench | oltpuser | Lock            | transactionid | 368001 |   2718 | 368001 |    2718 | 195759.040779 |  1718.941857 | 1716499 |   1716473 |       26 |         0 |      48 |         0 |       13 |          0 2025-07-10 20:33:27 |  5981687818415711060 | SELECT b.bid, SUM(a | dwhbench  | dwhuser  |                 |               |   3290 |     40 |    329 |       4 | 510801.219409 |  4591.836860 |  591235 | -52111806 | 53346281 |    643240 |       0 |         0 |     5116 |         38
 ```

## Drill down

### Test setup
For the following examples I was running two instances of pgbench. One instance was connected to the database oltpbench and ran the default pgbench benchmark:
```bash
pgbench -c 20 -j 20 -T 600 -d oltpbench -U oltpuser
```
The second instance was running a datawarehouse like benchmark in the database dwhbench:
```bash
pgbench -c 20 -j 20 -T 600 -d dwhbench -U dwhuser -f dwhbench.sql
```
After 2 minutes I started collecting snapshots with pgstat_snap for a minute.

Tip: if you want to know how to create your own benchmark, check out this post: [pgbench](/postgres/pgbench/)

### Output format
Depending on screensize, you might want to set format to aligned, especially when querying *pstat_snap_diff_all*:
```sql
\pset format aligned
```
### What was happening, ordered by time
It is usually a good start to first get an overview of what was going on second-by-second. Let's dissect what we see when just focusing on seconds 20:33:25 and 26:
```sql
select * from pgstat_snap_diff order by 1;
    snapshot_time    |       queryid        |        query        |  datname  | usename  | wait_event_type |  wait_event   | rows_d | calls_d |  exec_ms_d   | sb_hit_d | sb_read_d | sb_dirt_d | sb_write_d
---------------------+----------------------+---------------------+-----------+----------+-----------------+---------------+--------+---------+--------------+----------+-----------+-----------+------------
 2025-07-10 20:33:25 |  8431081492085281285 | UPDATE pgbench_tell | oltpbench | oltpuser | Lock            | transactionid |        |         |              |          |           |           |
 2025-07-10 20:33:25 |  8092674635626072858 | SELECT b.bid,       | dwhbench  | dwhuser  |                 |               |    200 |       2 |  3142.512490 |     2090 |    325798 |         0 |         14
 2025-07-10 20:33:25 |  8537798620180856986 | SELECT b.bid, SUM(a | dwhbench  | dwhuser  |                 |               |    400 |       4 |  4895.604348 |     3399 |    652385 |         0 |         20
 2025-07-10 20:33:25 | -1263534209317543048 | UPDATE pgbench_bran | oltpbench | oltpuser | Lock            | transactionid |   3249 |    3249 |  1768.453916 |    15380 |         3 |         1 |          1
 2025-07-10 20:33:25 |  7095169429556301570 | INSERT INTO pgbench | dwhbench  | dwhuser  | LWLock          | BufferMapping |      8 |       8 | 10730.907623 |   124847 |     93865 |         2 |       3375
 2025-07-10 20:33:26 |  8092674635626072858 | SELECT b.bid,       | dwhbench  | dwhuser  |                 |               |    300 |       3 |  2947.915426 |     8355 |    483493 |         0 |         37
 2025-07-10 20:33:26 | -2177783729771621131 | UPDATE pgbench_acco | oltpbench | oltpuser |                 |               |   6414 |    6414 |   541.561524 |    30523 |     12649 |      7321 |        904
 2025-07-10 20:33:26 |  5796608934964327314 | INSERT INTO pgbench | oltpbench | oltpuser | Client          | ClientRead    |  13879 |   13879 |   135.870508 |    14057 |        10 |        94 |         99
 2025-07-10 20:33:26 | -1263534209317543048 | UPDATE pgbench_bran | oltpbench | oltpuser | Lock            | transactionid |   3162 |    3162 |  1868.084329 |    14876 |         0 |         0 |          0 
 ```
 OLTP:
- user "oltpuser" executed DMLs in the database "oltpbench"
- the first query with id "8431081492085281285" has a null value in rows_d and calls_d because it was executed for the first time since I started collecting snapshots. It might have been executed before but pgstat_snap can not go back in time
- the number of rows_d and calls_d is always the same for the user oltpuser. This means, every DML just updated or inserted a single row and the queries were called many times between snapshots
- the queries were also executed very fast, less than a millisecond (exec_ms_d_/calls_d), some processes had to wait for locks to be released

DWH:
- user "dwhuser" executed mainly SELECTS and few DMLs in the database "dwhbench"
- most queries affected more than one row per call, like the query "8092674635626072858", which returned 100 rows per call and was only called a few times between snapshots
- the queries usually ran for a second or more, "8092674635626072858" was called 2 times and the total exeuction time increased by 3.142 seconds (we'll verify this later)

### What was every query doing
Let's focus on the querid "8092674635626072858" a bit more, what was it doing? To get a better understanding, we can query pgstat_snap_diff_all which also has the summarized values as recorded in pg_stat_statemens when the snapshot was collected:
```sql
select snapshot_time,queryid,rows,rows_d,calls,calls_d,exec_ms,exec_ms_d from pgstat_snap_diff_all where queryid='8092674635626072858' order by 1;
    snapshot_time    |       queryid       | rows  | rows_d | calls | calls_d |    exec_ms    |  exec_ms_d
---------------------+---------------------+-------+--------+-------+---------+---------------+--------------
 2025-07-10 20:33:21 | 8092674635626072858 | 31500 |    200 |   315 |       2 | 574525.768201 |  3318.129896
 2025-07-10 20:33:22 | 8092674635626072858 | 32200 |    700 |   322 |       7 | 588935.828000 | 14410.059799
 2025-07-10 20:33:23 | 8092674635626072858 | 32400 |    200 |   324 |       2 | 592146.153274 |  3210.325274
 2025-07-10 20:33:25 | 8092674635626072858 | 32600 |    200 |   326 |       2 | 595288.665764 |  3142.512490
 2025-07-10 20:33:26 | 8092674635626072858 | 32900 |    300 |   329 |       3 | 598236.581190 |  2947.915426
 2025-07-10 20:33:27 | 8092674635626072858 | 33300 |    400 |   333 |       4 | 607982.043008 |  9745.461818
 2025-07-10 20:33:28 | 8092674635626072858 | 33600 |    300 |   336 |       3 | 610530.950563 |  2548.907555
```
- in the first example, we saw that at 20:33:25 it was called twice and the execution time increased by 3142ms. 
- there was no execution at 20:33:24, 20:33:23 is when it was executed the last time before 20:33:25. Nice, everything fits - or does it?
- at 20:33:27 it has been executed 4 times with an exec_ms_d of 9745ms. But it was also executed the second before, how can this be?

That's the limitation of pg_stat_statements, we can not trace sessions, everything is summarized over all executions and the views only show the difference between snapshots. We don't know when all the sessions were started that increased the calls count by 4, maybe one execution was slower than the others. Keep that in mind when trying to time how long one query was executed.

### Which database modified the most rows
It seems that more rows were modified in the oltpbench database, let's verify this:
```sql
select sum(rows_d),datname from pgstat_snap_diff where upper(query) not like 'SELECT%' group by datname;
  sum   |  datname
--------+-----------
 678031 | oltpbench
    165 | dwhbench
```
This is useful on busy clusters with many databases and you want to figure out which database might be doing the most I/O operations. Many DML in particular can slow a whole cluster down when standbys in SYNC mode are involved.

### Which query DML affected the most rows
With a slight modification we can also check which query did the most DML:
```sql
select sum(rows_d),queryid,query from pgstat_snap_diff where upper(query) not like 'SELECT%' group by queryid,query order by 1;
  sum   |       queryid        |        query
--------+----------------------+---------------------
    165 |  7095169429556301570 | INSERT INTO pgbench
 160834 |  8431081492085281285 | UPDATE pgbench_tell
 166739 | -2177783729771621131 | UPDATE pgbench_acco
 174205 |  5796608934964327314 | INSERT INTO pgbench
 176253 | -1263534209317543048 | UPDATE pgbench_bran
```
### What wait events happened the most
We can also check which wait events occured the most. Again, it's a bit wonky (see View description above) but if one event stands out you might want to investigate further:
```sql
select wait_event,count(*) from pgstat_snap_diff where wait_event_type is not null and wait_event_type <> 'Client' group by wait_event order by 2;
  wait_event   | count
---------------+-------
 tuple         |     1
 LockManager   |     1
 SpinDelay     |     1
 WALWrite      |     2
 DataFileWrite |     2
 BufferIo      |     2
 DataFileRead  |     9
 BufferMapping |    30
 transactionid |    54
```
### Access raw data directly
Lastly, if you don't trust the views you can always access the history tables directly, for example to see the statistics for the query "8092674635626072858" as they were in pg_stat_statements second by second:
```sql
select snapshot_time,queryid,rows,calls,total_exec_time from pgstat_snap_stat_history where queryid='8092674635626072858' order by 1;
    snapshot_time    |       queryid       | rows  | calls |  total_exec_time
---------------------+---------------------+-------+-------+-------------------
 2025-07-10 20:33:21 | 8092674635626072858 | 31500 |   315 | 574525.7682009998
 2025-07-10 20:33:22 | 8092674635626072858 | 32200 |   322 | 588935.8279999999
 2025-07-10 20:33:23 | 8092674635626072858 | 32400 |   324 | 592146.1532739999
 2025-07-10 20:33:24 | 8092674635626072858 | 32400 |   324 | 592146.1532739999
 2025-07-10 20:33:25 | 8092674635626072858 | 32600 |   326 | 595288.6657639999
 2025-07-10 20:33:26 | 8092674635626072858 | 32900 |   329 | 598236.5811899999
 2025-07-10 20:33:27 | 8092674635626072858 | 33300 |   333 | 607982.0430079999
 2025-07-10 20:33:28 | 8092674635626072858 | 33600 |   336 | 610530.9505629999
```

## Conclusion
Tracing query executions can be cumbersom with pg_stat_statements alone, which is why I wrote this extension. At my workplace we don't have that many performance problems with PostgreSQL to warrant setting up a centralized solution like pgwatch or pg_profile. But when I encounter a performance problem once in a blue moon on a cluster, especially with many databases, I can now quickly see which database or query might be responsible. All it takes is to load up the extension, collect some snapshots and query pgstat_snap_diff in whatever way I want. And when I'm done I drop the extension again with all the collected data to not have any dead waste in the database.  

I hope you'll find it useful too, please let me know if there's anything I could improve: <raphi@crashdump.ch>.

