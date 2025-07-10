---
title: "pgbench"
type: "docs"
weight: 1
description: "Using pgbench for custom benchmarks"
---
## The PostgreSQL benchmark utility

<img class="floatimg" src="/postgres/pgbench.jpg" alt="elephant" width="25%" height="25%"/>
<br><br>
<p>Including a benchmark utility with a database product demonstrates a strong commitment to performance and transparency. For PostgreSQL, the built-in tool is pgbench, which allows users to test various aspects of the database system. Not only that, it can also run benchmarks on any set of tables with any script that you created yourself, making it the perfect tool to benchmark a specific application workload.

The official documentation is here: [https://www.postgresql.org/docs/current/pgbench.html](https://www.postgresql.org/docs/current/pgbench.html)
</p>
<div style="clear: both;"></div>

## Installation
In most cases, pgbench is installed in your system’s binary directory when you install PostgreSQL, typically /usr/bin or in the PostgreSQL bindir:

```bash
ls $(pg_config --bindir)/pgbench

# output
/usr/lib/postgresql/17/bin/pgbench
```
By default, pgbench simulates a workload similar to the TPC-B benchmark, generating many small transactions involving single-row updates, lookups, and inserts. However, it’s straightforward to design custom benchmarks, as described further below.


### Initial Setup

Before using pgbench, the necessary tables must be set up in the database. You need to specify a scale factor (-s) during this process, the higher the scale factor, the larger the tables will be. For example, with a scale factor of 100, the pgbench_accounts table will be created with 10 million rows.

### Tables

The following tables will be dropped and recreated if they already exist:

- pgbench_accounts
- pgbench_branches
- pgbench_history
- pgbench_tellers

If you do not specify a database, the tables will be created in the database that psql connects you to (typically your OS user, e.g. postgres) in the publich schema. To create the tables, run:
```bash
pgbench -i -s 100
```

### Installation in dedicated database
If you want the tables to be created in their own database and schema, you need to create a role, database and schema first and set the search_path correctly:
```sql
psql

-- create a role:
CREATE ROLE pgbench WITH LOGIN;

-- create the database owned by that role
CREATE DATABASE pgbench OWNER pgbench;

-- connect to the newly created schema
\c pgbench
CREATE SCHEMA pgbench;

-- set search_path to the new schema
ALTER ROLE pgbench SET search_path TO pgbench;

-- grant the necessary rights
GRANT ALL ON SCHEMA pgbench TO pgbench;
exit;
```

Now you are ready to install pgbench into the newly created database:
```bash
pgbench -i -s 100 -d pgbench -U pgbench
```

## Run the benchmark
The most important parameters are:

- -j [number of jobs]
- -c [number of clients]
- -T [time to run in seconds]

To test the database without being influenced by other factors like cpu scheduling, I prefer to have the same number of jobs and clients and both do not exceed the count of half of my machine's CPU cores. 

TPC-B like example to run the benchmark for 10 minutes:
```bash
pgbench -c 20 -j 20 -T 600

-- or in a dedicated database
pgbench -c 20 -j 20 -T 600 -d pgbench -U pgbench
```

From here on I assume that the tables are created in the pgbench database and schema.

### Sample output
The most meaningful numbers to compare between runs are the tps (transactions per second) and the number of transactions actually processed:
```bash
postgres@plexus:~$ pgbench -c 20 -j 20 -T 60 -d pgbench -U pgbench
pgbench (17.5 (Debian 17.5-1.pgdg120+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 20
number of threads: 20
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 318422
number of failed transactions: 0 (0.000%)
latency average = 3.768 ms
initial connection time = 9.239 ms
tps = 5307.477304 (without initial connection time)
```

By comparing the tps you'll know if the changes you made were positive or not, a higher tps number usually means that you are on the right track.

## Create your own benchmark
In this example, I am creating a benchmark that more closely resembles TPC-DS, which is designed for decision support systems. TPC-DS workloads typically involve large-scale updates, inserts, and complex queries that operate on many rows — characteristic of typical data warehouse environments.

In general, while the default TPC-B benchmark primarily tests CPU/memory performance and random access I/O through many small transactions, a TPC-DS-like benchmark tends to generate more sequential I/O due to larger table scans and aggregations, often hitting the shared buffer better.

### Write the script
While it is possible to create your own tables and fill them with data, I am going to use the tables which I already created previously with "pgbench -i". Due to the underlying data, this script is by no means a true TPC-DS benchmark, which usually consists of multiple tables in a star schema. But it should give you an idea how to write a benchmark your self.

Copy/paste the following lines into a file, let's call it "dwhbench.sql":
```sql
BEGIN;

-- 1. Total account balance per branch (roll-up by branch)
SELECT b.bid, SUM(a.abalance) AS total_balance
FROM pgbench_branches b
JOIN pgbench_accounts a ON b.bid = a.bid
GROUP BY b.bid;

-- 2. Average, min, max account balance per branch (analytics)
SELECT b.bid,
       AVG(a.abalance) AS avg_balance,
       MIN(a.abalance) AS min_balance,
       MAX(a.abalance) AS max_balance
FROM pgbench_branches b
JOIN pgbench_accounts a ON b.bid = a.bid
GROUP BY b.bid;

-- 3. Top 10 branches by total account balance (reporting)
SELECT b.bid, SUM(a.abalance) AS total_balance
FROM pgbench_branches b
JOIN pgbench_accounts a ON b.bid = a.bid
GROUP BY b.bid
ORDER BY total_balance DESC
LIMIT 10;

-- 4. Number of accounts per branch with balance above a threshold (filtering)
SELECT b.bid, COUNT(a.aid) AS num_high_value_accounts
FROM pgbench_branches b
JOIN pgbench_accounts a ON b.bid = a.bid
WHERE a.abalance > 100000
GROUP BY b.bid
ORDER BY num_high_value_accounts DESC
LIMIT 10;

-- 5. Teller performance: total transactions and sum of deltas per teller
SELECT t.tid, COUNT(h.aid) AS num_transactions, SUM(h.delta) AS total_delta
FROM pgbench_tellers t
LEFT JOIN pgbench_history h ON t.tid = h.tid
GROUP BY t.tid
ORDER BY total_delta DESC
LIMIT 10;

-- 6. Daily transaction summary for the last 7 days (time-based reporting)
SELECT DATE(h.mtime) AS day, COUNT(*) AS num_transactions, SUM(h.delta) AS total_delta
FROM pgbench_history h
WHERE h.mtime >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY day
ORDER BY day DESC;

-- 7. Branches with most active tellers (multi-level aggregation)
SELECT b.bid, COUNT(DISTINCT t.tid) AS num_tellers, COUNT(h.aid) AS num_transactions
FROM pgbench_branches b
JOIN pgbench_tellers t ON b.bid = t.bid
LEFT JOIN pgbench_history h ON t.tid = h.tid
GROUP BY b.bid
ORDER BY num_transactions DESC
LIMIT 10;

-- 8. Insert a new transaction (history) with valid keys
INSERT INTO pgbench_history (tid, bid, aid, delta, mtime)
VALUES (
    (SELECT tid FROM pgbench_tellers ORDER BY random() LIMIT 1),
    (SELECT bid FROM pgbench_branches ORDER BY random() LIMIT 1),
    (SELECT aid FROM pgbench_accounts ORDER BY random() LIMIT 1),
    (random() * 1000 - 500)::INTEGER,
    now()
);

COMMIT;
```
You can modify the script to your liking or install the pgbench tables with a higher scale factor than 100 (-s 1000) if you need more load. 

### Run your script
Now execute the script with pgbench:
```bash
pgbench -c 20 -j 20 -T 60 -d pgbench -U pgbench -f dwhbench.sql
```

Sample Output - note that the tps is significantelly lower than in the TPC-B like runs before:
```bash
postgres@plexus:~$ pgbench -c 20 -j 20 -T 60 -d pgbench -U pgbench -f dwhbench.sql
pgbench (17.5 (Debian 17.5-1.pgdg120+1))
starting vacuum...end.
transaction type: dwhbench.sql
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 20
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 158
number of failed transactions: 0 (0.000%)
latency average = 8223.323 ms
initial connection time = 7.292 ms
tps = 2.432107 (without initial connection time)
```

## Compare TPC-B vs TPC-DS like execution
With the pgstat_snap extension I can now easily compare the execution of both benchmarks, the default pgbench benchmark and the dwhbench I created before. If you don't have the extension installed, you can download it here: [https://github.com/raphideb/pgstat_snap](https://github.com/raphideb/pgstat_snap)

For the tests, I first reset all pg_stat* and psgstat_snap* statistics:
```sql
psql
select pgstat_snap_reset(2);
```
Then I start collecting snapshots in one terminal window for 70 seconds:
```sql
call pgstat_snap_collect(1,70);
```
and immediately after start the benchmark in another terminal:
```bash
# for default TPC-B like
pgbench -c 20 -j 20 -T 60 -d pgbench -U pgbench

# for our custom TPC-DS like
pgbench -c 20 -j 20 -T 60 -d pgbench -U pgbench -f dwhbench.sql
```

### TPC-B like
The pgbench default benchmark executes statements many times per second but usually only affects one row per execution, as shown by the number of rows affected each second divided by the number of times the statement was called (rows_d/called_d). The number of blocks dirtied in the shared buffer is also often quite high, indicative of a lot of reads from the filesystem.
```sql
select * from pgstat_snap_diff order by 1;

    snapshot_time    |       queryid        |        query        | datname | usename | wait_event_type |  wait_event   | rows_d | calls_d |  exec_ms_d   | sb_hit_d | sb_read_d | sb_dirt_d | sb_write_d
---------------------+----------------------+---------------------+---------+---------+-----------------+---------------+--------+---------+--------------+----------+-----------+-----------+------------
 2025-07-06 11:56:10 | -3717316371818285038 | UPDATE pgbench_acco | pgbench | pgbench | IO              | DataFileRead  |   2646 |    2646 |  5515.640933 |    13334 |      4806 |      2988 |          8
 2025-07-06 11:56:11 | -9070982371082842952 | UPDATE pgbench_bran | pgbench | pgbench | Lock            | transactionid |   2630 |    2630 |   834.971431 |    11784 |         0 |         0 |          0
 2025-07-06 11:56:11 | -3717316371818285038 | UPDATE pgbench_acco | pgbench | pgbench | IO              | DataFileRead  |   2626 |    2626 |  5365.836472 |    13598 |      4594 |      2936 |        239
 2025-07-06 11:56:12 | -9070982371082842952 | UPDATE pgbench_bran | pgbench | pgbench | Lock            | transactionid |   2614 |    2614 |   783.287074 |    11692 |         0 |         0 |          0
 2025-07-06 11:56:12 | -3717316371818285038 | UPDATE pgbench_acco | pgbench | pgbench | IO              | DataFileRead  |   2619 |    2619 |  5390.995660 |    13498 |      4521 |      2898 |        531
 2025-07-06 11:56:13 | -9070982371082842952 | UPDATE pgbench_bran | pgbench | pgbench | Lock            | transactionid |   2570 |    2570 |   907.785515 |    11650 |         0 |         0 |          0
 2025-07-06 11:56:13 | -3717316371818285038 | UPDATE pgbench_acco | pgbench | pgbench | IO              | DataFileRead  |   2568 |    2568 |  5206.180391 |    13230 |      4336 |      2796 |       1112
 2025-07-06 11:56:14 | -3717316371818285038 | UPDATE pgbench_acco | pgbench | pgbench | IO              | DataFileRead  |   3111 |    3111 |  5063.160774 |    15935 |      5181 |      3356 |       2626
 2025-07-06 11:56:14 |  6874825970108732267 | INSERT INTO pgbench | pgbench | pgbench |                 |               |  10929 |   10929 |    73.726408 |    11169 |         0 |        69 |         86
 2025-07-06 11:56:17 | -9070982371082842952 | UPDATE pgbench_bran | pgbench | pgbench | Lock            | transactionid |  10880 |   10880 |  2454.568335 |    48787 |         0 |         0 |          0
 2025-07-06 11:56:17 | -3717316371818285038 | UPDATE pgbench_acco | pgbench | pgbench | IO              | DataFileRead  |   7770 |    7770 |  9218.660031 |    39796 |     12937 |      8368 |       7342
 2025-07-06 11:56:17 |  6874825970108732267 | INSERT INTO pgbench | pgbench | pgbench |                 |               |   7766 |    7766 |    51.563130 |     8122 |         0 |        41 |         65
 2025-07-06 11:56:18 | -9070982371082842952 | UPDATE pgbench_bran | pgbench | pgbench |                 |               |   4067 |    4067 |   903.844398 |    18373 |         0 |         0 |          0
 2025-07-06 11:56:18 | -3717316371818285038 | UPDATE pgbench_acco | pgbench | pgbench | IO              | DataFileRead  |   4061 |    4061 |  4356.176667 |    20545 |      6727 |      4308 |       3985
 2025-07-06 11:56:18 |  7882337555304992192 | UPDATE pgbench_tell | pgbench | pgbench |                 |               |  22763 |   22763 |   947.264902 |   115032 |         0 |         0 |          0
 2025-07-06 11:56:18 |  6874825970108732267 | INSERT INTO pgbench | pgbench | pgbench | Client          | ClientRead    |   4067 |    4067 |    26.740198 |     4240 |         0 |        31 |         53
 2025-07-06 11:56:19 | -9070982371082842952 | UPDATE pgbench_bran | pgbench | pgbench |                 |               |   4087 |    4087 |   894.378344 |    18372 |         0 |         0 |          0
 2025-07-06 11:56:19 |  7882337555304992192 | UPDATE pgbench_tell | pgbench | pgbench |                 |               |   4085 |    4085 |   176.151355 |    20660 |         0 |         0 |          0
 2025-07-06 11:56:19 | -3717316371818285038 | UPDATE pgbench_acco | pgbench | pgbench | IO              | DataFileRead  |   4088 |    4088 |  4107.538425 |    20958 |      6794 |      4407 |       3877
 2025-07-06 11:56:20 | -3717316371818285038 | UPDATE pgbench_acco | pgbench | pgbench |                 |               |   4162 |    4162 |  4032.420140 |    21028 |      6905 |      4433 |       3993
 2025-07-06 11:56:20 | -9070982371082842952 | UPDATE pgbench_bran | pgbench | pgbench | Client          | ClientRead    |   4158 |    4158 |   943.804925 |    18834 |         0 |         0 |          0
 2025-07-06 11:56:20 |  6874825970108732267 | INSERT INTO pgbench | pgbench | pgbench |                 |               |   8245 |    8245 |    53.479212 |     8369 |         0 |        54 |         85
 ```

### TPC-DS like
This benchmark tells a different story, some queries return one row per call, others 10 or 100 per call (rows_d/calls_d). Also note that the number of blocks dirtied and evicted from the buffer is near 0 (sb_dirt_d, sb_write_d) whereas in the TPC-B, blocks were constantly unloaded and loaded into the shared buffer.

```sql
select * from pgstat_snap_diff order by 1;
    snapshot_time    |       queryid        |        query        | datname | usename | wait_event_type |  wait_event   | rows_d | calls_d |  exec_ms_d   | sb_hit_d | sb_read_d | sb_dirt_d | sb_write_d
---------------------+----------------------+---------------------+---------+---------+-----------------+---------------+--------+---------+--------------+----------+-----------+-----------+------------
 2025-07-06 12:29:00 | -6153627433925560805 | SELECT b.bid, SUM(a | pgbench | pgbench | LWLock          | BufferMapping |    300 |       3 |  2090.854209 |    42644 |    457201 |         0 |          0
 2025-07-06 12:29:01 |  4503905990032662730 | SELECT b.bid,       | pgbench | pgbench |                 |               |    500 |       5 |  5575.932052 |    58345 |    774700 |         0 |          0
 2025-07-06 12:29:01 |  6510738175259849580 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |     80 |       8 | 14012.869241 |   108454 |   1224336 |         0 |          0
 2025-07-06 12:29:02 | -6153627433925560805 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |    200 |       2 |  2712.562823 |    41057 |    292153 |         0 |          0
 2025-07-06 12:29:02 |  6510738175259849580 | SELECT b.bid, SUM(a | pgbench | pgbench | IO              | DataFileRead  |     20 |       2 |  1752.724632 |    31600 |    301620 |         0 |          0
 2025-07-06 12:29:02 |  3571751447725039568 | INSERT INTO pgbench | pgbench | pgbench |                 |               |      4 |       4 | 11356.669967 |  2788994 |    469954 |         3 |         18
 2025-07-06 12:29:03 |  4503905990032662730 | SELECT b.bid,       | pgbench | pgbench |                 |               |    200 |       2 |  1537.711768 |    27657 |    305573 |         0 |          0
 2025-07-06 12:29:03 | -6153627433925560805 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |    100 |       1 |   695.818981 |     9872 |    156743 |         0 |          0
 2025-07-06 12:29:03 |  3571751447725039568 | INSERT INTO pgbench | pgbench | pgbench |                 |               |      3 |       3 |  8762.063660 |  2049418 |    394793 |         2 |          5
 2025-07-06 12:29:04 | -6153627433925560805 | SELECT b.bid, SUM(a | pgbench | pgbench | LWLock          | BufferMapping |    300 |       3 |  3760.906149 |    54054 |    445761 |         0 |          0
 2025-07-06 12:29:04 |  4503905990032662730 | SELECT b.bid,       | pgbench | pgbench |                 |               |    100 |       1 |   751.276443 |    15988 |    150627 |         0 |          0
 2025-07-06 12:29:04 |  3571751447725039568 | INSERT INTO pgbench | pgbench | pgbench |                 |               |      3 |       3 |  8790.948263 |  1885512 |    558699 |         3 |         12
 2025-07-06 12:29:04 |  6510738175259849580 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |     40 |       4 |  4158.021389 |    60654 |    605786 |         0 |          0
 2025-07-06 12:29:05 | -6153627433925560805 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |    300 |       3 |  6126.329681 |    34905 |    464880 |         0 |          0
 2025-07-06 12:29:05 |  3571751447725039568 | INSERT INTO pgbench | pgbench | pgbench |                 |               |      7 |       7 | 20024.136128 |  4948620 |    754539 |         3 |         23
 2025-07-06 12:29:05 |  4503905990032662730 | SELECT b.bid,       | pgbench | pgbench |                 |               |    300 |       3 |  2650.404371 |    41028 |    458807 |         0 |          0
 2025-07-06 12:29:06 |  6510738175259849580 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |     30 |       3 |  2509.879189 |    50945 |    448890 |         0 |          0
 2025-07-06 12:29:06 |  3571751447725039568 | INSERT INTO pgbench | pgbench | pgbench | LWLock          | BufferMapping |      2 |       2 |  5933.317393 |  1257252 |    372222 |         2 |         14
 2025-07-06 12:29:06 | -6153627433925560805 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |    200 |       2 |  2818.259800 |    28190 |    305020 |         0 |          0
 2025-07-06 12:29:06 |  4503905990032662730 | SELECT b.bid,       | pgbench | pgbench |                 |               |    100 |       1 |   775.519352 |    14036 |    152579 |         0 |          0
 2025-07-06 12:29:07 | -6153627433925560805 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |    800 |       8 | 15163.238847 |   129899 |   1202881 |         0 |          0
 2025-07-06 12:29:07 |  4503905990032662730 | SELECT b.bid,       | pgbench | pgbench | IO              | DataFileRead  |    400 |       4 |  7343.899760 |    42480 |    623920 |         0 |          0
 2025-07-06 12:29:07 |  3571751447725039568 | INSERT INTO pgbench | pgbench | pgbench |                 |               |      2 |       2 |  5753.122885 |  1257700 |    371774 |         2 |         13
 2025-07-06 12:29:07 |  6510738175259849580 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |     10 |       1 |   697.975914 |    15494 |    151121 |         0 |          0
 2025-07-06 12:29:08 |  4503905990032662730 | SELECT b.bid,       | pgbench | pgbench | IO              | DataFileRead  |    200 |       2 |  1544.049097 |    25949 |    307281 |         0 |          0
 2025-07-06 12:29:08 | -6153627433925560805 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |    100 |       1 |  1979.149370 |    25598 |    140997 |         0 |          1
 2025-07-06 12:29:08 |  3571751447725039568 | INSERT INTO pgbench | pgbench | pgbench |                 |               |      1 |       1 |  3134.628218 |   629162 |    185575 |         1 |          6
 2025-07-06 12:29:08 |  6510738175259849580 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |     20 |       2 |  1773.277487 |    25037 |    308183 |         0 |          0
 2025-07-06 12:29:09 |  4503905990032662730 | SELECT b.bid,       | pgbench | pgbench |                 |               |    700 |       7 | 14010.989075 |    70769 |   1095416 |         0 |          0
 2025-07-06 12:29:09 |  6510738175259849580 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |     40 |       4 |  5428.666678 |    53151 |    613269 |         0 |          0
 2025-07-06 12:29:09 | -6153627433925560805 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |    200 |       2 |  3986.136102 |    36939 |    296251 |         0 |          0
 2025-07-06 12:29:09 |  3571751447725039568 | INSERT INTO pgbench | pgbench | pgbench |                 |               |      3 |       3 |  8991.218544 |  2000799 |    443412 |         2 |         12
 2025-07-06 12:29:10 |  6510738175259849580 | SELECT b.bid, SUM(a | pgbench | pgbench |                 |               |     30 |       3 |  2466.541453 |    41033 |    458802 |         0 |          0
 2025-07-06 12:29:10 |  4503905990032662730 | SELECT b.bid,       | pgbench | pgbench |                 |               |    100 |       1 |  2159.522420 |    12549 |    154046 |         0 |          0
 2025-07-06 12:29:10 |  3571751447725039568 | INSERT INTO pgbench | pgbench | pgbench |                 |               |      1 |       1 |  2846.755759 |   628987 |    185750 |         1 |         10
 ```

## Conclusion
 pgbench is a very powerful tool to benchmark a PostgreSQL database however you want. It's very easy to create your own application workload and compare it across different systems or before/after an upgrade, especially when combined with the pgstat_snap extension. 

 happy benchmarking ;)

