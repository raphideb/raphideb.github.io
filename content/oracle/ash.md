---
title: "Active Session History"
type: "docs"
weight: 1
description: "Oracle's active session history demystified"
---
<img class="floatimg" src="/oracle/ash.jpg" alt="active_session_history" width="25%" height="25%"/>
<p>The view gv$active_session_history (ASH) records in memory every second the wait events of all active sessions. It is the single most important source of information for deep performance analysis on a session or even query level, like what a session was doing in a particular moment in time, what queries it was executing or which other sessions were blocking it. For all the data it contains it is surprisingly easy to use. First I will explain how the ASH works and what fields it provides, then I show you a basic script and how to extend it with all sorts of queries to get the info that you need.

Since the memory for ASH is limited, the data is only available for a few days or even hours. It is also usually gone when the instance was restarted, therefore, every 10 seconds a snapshot of the ASH data is stored permanently on disk and can be accessed through the view dba_hist_active_sess_history (DASH). </p>
<div style="clear: both;"></div>

## ASH Basics
All ASH/DASH data is stored at the CDB level and can be accessed either from the CDB or a specific PDB. However, when you want to map the USER_ID, CURRENT_OBJ\#, CURRENT_FILE\# etc to names stored in dba_views, it's best to query ASH from within the corresponding PDB.

A word about gv$active_session_history vs. v$active_session_history: While they are defacto the same on a non-RAC database, on a RAC database the v$ variant only shows sessions running on one instance, whereas gv$ shows all sessions an all instances of the database. I therefore made it a habbit to always use gv$active_session_history and include the inst_id in my queries. The same is true for many other v$ views, in RAC there's usually a gv$ variant (like gv$session). 

Note that the dba_hist views are always cluster wide and store the data for all instances.

### Useful Tables and Views
To fully leverage all the information that the ASH provides, it helps to know some other useful views related to performance analysis.

| Name | Description | Notes |
| :-- | :-- | :-- |
| v\$active_session_history | Wait events for all sessions of a node in 1-second snapshots | In memory, limited space |
| gv\$active_session_history | Same as above, but for all nodes in a RAC cluster | Has additional column "INST_ID" |
| dba_hist_active_sess_history | Historical 10-second snapshots of ASH | Stored on disk; "INSTANCE_NUMBER" instead of "INST_ID" |
| v\$ash_info | Statistics about ASH itself | "OLDEST_SAMPLE_TIME" shows how far back data goes |
| v\$sql | Stats for currently loaded SQL statements | One row per child cursor |
| dba_hist_sqltext | Historical SQL text for SQL_IDs no longer in shared area | Use `set long 10000` to see full text |
| dba_objects | Info on all objects in the DB | Useful for mapping to current_obj\# in ASH |
| dba_data_files | Info on all data files | Useful for mapping to current_file\# in ASH |


### Description of the fields
ASH and DASH have over 100 fields, and more are added with each release. The description of all fields is available here:
[https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HIST_ACTIVE_SESS_HISTORY.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HIST_ACTIVE_SESS_HISTORY.html)

#### Useful Fields
All fields of ASH/DASH can be useful in specific cases, but the following are the most commonly used:


| Column | Datatype | Description |
| :-- | :-- | :-- |
| INST_ID | NUMBER | Only in gv\$active_session_history, RAC instance where the session is running |
| INSTANCE_NUMBER | NUMBER | Only in dba_hist_active_sess_history, RAC instance where the session is running |
| SAMPLE_ID | NUMBER | ID of the sample |
| SAMPLE_TIME | TIMESTAMP(3) | Time at which the sample was taken |
| SESSION_ID | NUMBER | Session identifier; maps to V\$SESSION.SID |
| SESSION_SERIAL\# | NUMBER | Session serial number (uniquely identifies a session's objects); maps to V\$SESSION.SERIAL\# |
| SESSION_TYPE | VARCHAR2(10) | Session type: FOREGROUND/BACKGROUND |
| USER_ID | NUMBER | Oracle user identifier; maps to V\$SESSION.USER\# |
| SQL_ID | VARCHAR2(13) | SQL identifier of the SQL statement executed at the time of sampling |
| SQL_OPNAME | VARCHAR2(64) | SQL command name (INSERT, DELETE, UPDATE, etc.) |
| TOP_LEVEL_SQL_ID | VARCHAR2(13) | SQL identifier of the top-level SQL statement (for PL/SQL) |
| SQL_PLAN_HASH_VALUE | NUMBER | Numeric representation of the SQL plan for the cursor |
| EVENT | VARCHAR2(64) | If SESSION_STATE = WAITING, the event waited for; if ON CPU, this column is NULL |
| P1TEXT | VARCHAR2(64) | Text of the first additional parameter |
| P1 | NUMBER | First additional parameter |
| P2TEXT | VARCHAR2(64) | Text of the second additional parameter |
| P2 | NUMBER | Second additional parameter |
| P3TEXT | VARCHAR2(64) | Text of the third additional parameter |
| P3 | NUMBER | Third additional parameter |
| WAIT_CLASS | VARCHAR2(64) | Wait class name of the event |
| SESSION_STATE | VARCHAR2(7) | Session state: WAITING / ON CPU |
| TIME_WAITED | NUMBER | If WAITING, time spent waiting for that event (in microseconds) |
| BLOCKING_SESSION | NUMBER | Session ID of the blocking session |
| BLOCKING_INST_ID | NUMBER | Instance number of the blocker |
| CURRENT_OBJ\# | NUMBER | Object ID referenced |
| CURRENT_FILE\# | NUMBER | File number referenced |
| CURRENT_BLOCK\# | NUMBER | Block ID referenced |
| REMOTE_INSTANCE\# | NUMBER | Remote instance serving the block |
| PGA_ALLOCATED | NUMBER | PGA memory used (in bytes) |
| TEMP_SPACE_ALLOCATED | NUMBER | TEMP memory used (in bytes) |
| CON_DBID | NUMBER | Database ID of the pluggable database (PDB) |
| PROGRAM | VARCHAR2(48) | Name of the operating system program |
| MACHINE | VARCHAR2(64) | Client's operating system machine name |

**Good to Know:**
- Wait events that only affect the CDB, such as backup activities, are not shown when querying ASH in the PDB.
- A session in the PDB can have a blocking session from the CDB, e.g., the logwriter session. The wait events of the blocking session itself must then be analyzed via the CDB.

### Checking available ASH data
Before analyzing a problem in the ASH, you should ensure that the data for the relevant period is still available:

```sql
select oldest_sample_time from gv$ash_info;

-- output

OLDEST_SAMPLE_TIME
-------------------------------
03-SEP-21 03.34.03.575000000 AM
```

Query information before "OLDEST_SAMPLE_TIME" can only be viewed via DASH.

### Saving ASH data
If you want to analyze the problem later, it is advisable to save the entire ASH to a table to not lose the data in it.

```sql
create table ash_snapshot as select * from gv$active_session_history;
```

In scripts, you should then select from this table instead:

```sql
-- from gv$active_session_history
from ash_snapshot
```

**Good to Know:**
Queries that are no longer in ASH and ran less than 10 seconds may not appear in DASH either. In such cases, the waits of a query can no longer be meaningfully analyzed.

## How to write ASH scripts
As you will shortly see, it's very easy to start with a basic script to get an overview and then drill deeper to find the root cause of a particular problem. Even when you use scripts like Tanel Poder's "ashtop" it's good to have a basic understanding of how to query the ASH and get the maximum out of those scripts. I'll also show the difference between queries from ASH or DASH.

### Basic Script
Let's start with a basic query which will be the blueprint for our more complex queries. It returns the most important columns of all wait events in the last hour for all nodes of a RAC database and works in both CDB and PDB.


#### Query from ASH
```sql
set lines 400
set pages 100
column sample_time format a22 tru
column event format a30 tru

select sample_time, inst_id, sql_id, session_id, user_id, event, time_waited, current_obj#, blocking_session
from gv$active_session_history
where sample_time between sysdate-1/24 and sysdate
order by sample_time;
```

#### Query from DASH
Difference from the ASH version: 
- instance_number instead of inst_id
- dba_hist_active_sess_history instead of gv$active_session_history

```sql
set lines 400
set pages 100
column sample_time format a22 tru
column event format a30 tru

select sample_time, instance_number, sql_id, session_id, user_id, event, time_waited, current_obj#, blocking_session
from dba_hist_active_sess_history
where sample_time between sysdate-1/24 and sysdate
order by sample_time;
```

**Sample Output:**
| SAMPLE_TIME | INST_ID | SQL_ID | SESSION_ID | USER_ID | EVENT | TIME_WAITED | CURRENT_OBJ\# | BLOCKING_SESSION |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| 08-JUL-25 03.40.44.241 | 1 | b913hx8hf360t | 5059 | 89 | direct path read | 3764 | 45305959 |  |
| 08-JUL-25 03.41.07.142 | 2 | di8yqa8tfsgqu | 2312 | 89 | cursor: pin S wait on X | 234920 | 4463384 | 4801 |
| 08-JUL-25 03.41.07.142 | 2 | cy6kjxcbcz8p3 | 893 | 89 | db file parallel read | 3275 | 48367311 |  |
| 08-JUL-25 03.41.07.142 | 2 | di8yqa8tfsgqu | 17769 | 89 | cursor: pin S wait on X | 274117 | 4463382 | 4801 |

#### Mapping to User and Object Name
The user and object names can be joined to have names instead of ids, but only in the PDB:

```sql
set lines 400
set pages 100
column sample_time format a22 tru
column event format a30 tru
column object_name format a30

select sample_time, inst_id, sql_id, session_id, b.username, event, time_waited, c.object_name, blocking_session
from gv$active_session_history a
join dba_users b on a.user_id = b.user_id
join dba_objects c on a.current_obj# = c.object_id
where sample_time between sysdate-1/24 and sysdate
order by 1;
```

**Sample Output:**
| SAMPLE_TIME | INST_ID | SQL_ID | SESSION_ID | USERNAME | EVENT | TIME_WAITED | OBJECT_NAME | BLOCKING_SESSION |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| 08-JUL-25 03.40.54.854 | 2 | 84p8c9j3ctpua | 14553 | RAPHI_USER | cursor: pin S wait on X | 1191291 | TRACKING_TABLE | 11326 |
| 08-JUL-25 03.40.54.854 | 2 | 84p8c9j3ctpua | 9272 | RAPHI_USER | cursor: pin S wait on X | 1206884 | TRK_PK_IDX | 11326 |
| 08-JUL-25 03.42.08.156 | 3 | cxfgr5c2j9zgg | 7578 | RAPHI_USER | row cache lock | 5642933 | SUB_PART_MGR | 4975 |
| 08-JUL-25 03.42.08.156 | 3 | bz77a36407aty | 4622 | RAPHI_USER | row cache lock | 5650066 | SUB_PART_MGR | 4975 |

Now we are getting somewhere, we see the username and which objects where affected by the wait clearly.

#### Limiting the time range
The "sample_time" column can be used to limit the query period, allowing you to focus on the relevant part. This can be done relative to sysdate:
```sql
where sample_time between sysdate-1/24 and sysdate
```

or as a specific time range:
```sql
where sample_time between to_date('25-Nov-2020 00:40:00', 'DD-Mon-YYYY HH24:MI:SS')
and to_date('25-Nov-2020 00:50:00', 'DD-Mon-YYYY HH24:MI:SS')
```

or both combined:
```sql
where sample_time between to_date('25-Nov-2020 00:40:00', 'DD-Mon-YYYY HH24:MI:SS')
and sysdate;
```

#### Limiting the number of rows
In one hour, ASH can record a huge number of rows, but often only the last 100 are of interest. To focus only on them, reverse the sort order and limit the output with fetch:
```sql
where sample_time between sysdate-1/24 and sysdate
order by sample_time desc
fetch next 100 rows only
```

## Drilldown
With the basic script, you can get an overview of what was happening in the database during the selected period. The script can then be extended as needed to further narrow down the problem.

### Finding blocking sessions
If, for example, a blocking_session is responsible for many waits, you can look at this session in more detail — the number in the "blocking_session" column is the session_id of the blocking session:

```sql
where sample_time between sysdate-1/24 and sysdate
and session_id = 123456
```

### Tracing a specific query
Similarly, if you want to look at a specific query, just filter by sql_id:

```sql
where sample_time between sysdate-1/24 and sysdate
and sql_id = '1ps650r41s7qr'
```

### PL/SQL queries
In Oracle Enterprise Manager, you often only see the SQL_ID of the PL/SQL procedure among the top sessions. If you want to view all associated queries, filter by top_level_sql_id and specify the PL/SQL sql_id:

```sql
where sample_time between sysdate-1/24 and sysdate
and top_level_sql_id = '1ps650r41s7qr'
```

### Viewing specific events
You can easily identify sessions affected by a specific wait using the Event column. A list of all possible wait events can be obtained with this query:
```sql
select name, wait_class FROM V$EVENT_NAME ORDER BY name;
```

Alternatively, you can list only the waits that actually occur in ASH:
```sql
select distinct event from gv$active_session_history order by 1;
```

To filter for specific event types, extend the basic query, for example:

```sql
where sample_time between sysdate-1/24 and sysdate
-- pick one of these:
-- index contention
and event = 'enq: TX - index contention'
-- all transaction contentions
and event like 'enq: TX%'
-- read/write operations
and (event like 'db file%' or event like 'direct path%')
-- all cluster waits (full RAC)
and event like 'gc%'
```

### Other useful filters

- Waits over 1 second:
```sql
and time_waited > 1000000
```

- Sessions accessing a specific object:
```sql
and c.object_name = 'NAME_OF_OBJECT'
```

- Sessions accessing a specific file:
```sql
and current_file# = 134
```

- Sessions accessing the same block in the same file (e.g., in case of contention):
```sql
and current_file# = 134
and current_block# = 123456
```

## More ASH queries
Some other ASH queries which are not directly developed from the basic script above but can also be useful.

### Track PGA usage
With this query, you can check how much PGA each session_id has used:
```sql
select inst_id, session_id, session_serial#, max(pga_allocated)/1024/1024/1024 GB 
from gv$active_session_history 
where sample_time between sysdate-1/24 and sysdate 
group by inst_id,session_id, session_serial# 
order by 4;
```

### Track TEMP usage
Similarly, you can find the sessions that have used the most TEMP:

```sql
select inst_id, session_id, session_serial#, max(temp_space_allocated)/1024/1024/1024 GB 
from gv$active_session_history 
where sample_time between sysdate-1/24 and sysdate 
group by inst_id, session_id, session_serial# 
order by 4;
```

## Making sense of the output
Now that we know how to query the ASH it's time to speak about what to do with the information.

### TIME_WAITED
As mentioned above, ASH data is stored every second for all sessions. The "TIME_WAITED" column shows 0 when a session was still waiting for a particular event to end when it was added to ASH. When ordered by sample_time, the first non-zero value for a unique combination of inst_id/session_id/sql_id shows the total amount of time (in ms) the session had to wait on that event. When the "TIME_WAITED" and "EVENT" column are NULL (not 0) on the other hand, the session was on the CPU and not waiting for an event. You can select the session_state in your query if you want to confirm this. 

In this example, session 14553 was waiting for "cursor: pin S wait on X" since 3:40:54 for 4120324 microseconds, when it finally finished at 3:40:58. Session 7578 was running on the CPU and thus did not have any wait event.

| SAMPLE_TIME | INST_ID | SQL_ID | SESSION_ID | EVENT | TIME_WAITED |
| :-- | :-- | :-- | :-- | :-- | :-- |
| 08-JUL-25 03.40.54.154 | 2 | 84p8c9j3ctpua | 14553 | cursor: pin S wait on X | 0 |
| 08-JUL-25 03.40.55.134 | 2 | 84p8c9j3ctpua | 14553 | cursor: pin S wait on X | 0 |
| 08-JUL-25 03.40.56.112 | 2 | 84p8c9j3ctpua | 14553 | cursor: pin S wait on X | 0 |
| 08-JUL-25 03.40.57.252 | 2 | 84p8c9j3ctpua | 14553 | cursor: pin S wait on X | 4120324 |
| 08-JUL-25 03.40.58.252 | 2 | cxfgr5c2j9zgg|  7578 |  |  |

### Wait events
This is the true bread and butter of the ASH. Understanding wait events, why they occur, what impact they have, how to correlate them and ultimately how to solve them is what makes performance tuning an art form rather than a scientific method. I can not go into the details of every important wait event because most would warrant a dedicated blog post of it's own. However, the official Oracle documentation has a brief description of all wait events:

[https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/descriptions-of-wait-events.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/descriptions-of-wait-events.html)

A good way to start when someone tells you that the database was slow, without any particulars like a sql_id or session_id, is to look for any events with a time_waited higher than a second. If one wait event stands out, check the documentation or google it. AI like perplexity.ai are also very good at explaining certain wait events, but it helps if you have at least some understanding because especially with everything Oracle, AI tends to blatantly lie to your face (apparently because the Oracle documentation is unclear, as perplexity explained to me :D).

#### Event Parameters
When searching for wait events, it's often useful to select the "P1" column (and possibly P2, P3). These provide additional information about the wait event, e.g. for Row Cache Locks, which DC Cache was involved. The columns "P1TEXT" (and P2TEXT, P3TEXT) give a descriptive name to the value. To see what the individual P-values mean, you can look at v\$event_name:

```sql
set lines 400
col parameter1 format a20
col parameter2 format a20
col parameter3 format a20
select name, parameter1, parameter2, parameter3 FROM V$EVENT_NAME where name like 'row%lock' order by name;
```

**Sample Output:**
| NAME | PARAMETER1 | PARAMETER2 | PARAMETER3 |
| :-- | :-- | :-- | :-- |
| row cache lock | cache id | mode | request |

You can then extend the basic query accordingly:

```sql
set lines 400
set pages 100
column sample_time format a22 tru
column event format a30 tru

select sample_time, inst_id, sql_id, session_id, user_id, event, p1text, p1, time_waited
from gv$active_session_history
where sample_time between sysdate-1/24 and sysdate
and event like 'row%lock'
order by 1;
```

### Locks and blocking sessions
One of the most common reason for a sudden performance loss is the application itself accessing the database. This can have a very ~~stupid~~simple cause, like someone manually executed a query that was updating a column of all rows in a table while the application was running, resulting in tons of row locks. 

Other less obvious causes are normal application sessions running slower than usual, causing locks and blocking other sessions. Possible causes from my own experience:
- execution plan of a query has suddenly changed for whatever reason, like missing statistics or different bind values
- a network problem between the application and DB server 
- a network problem in a dataguard environment with MaxProtection between primary and standby
- accessing the same table on different instances in a RAC database, causing a lot of block shipping
- not enough SGA to handle increased application volume, resulting in more blocks read from disk
- not enough PGA to handle complex queries, resulting in more temp usage
- Oracle changing the default for the parameter db_lost_write_protection in 19.26c (this caused us real headache) 
- and many more

Not all of these issues can be resolved with ASH alone but you most likely will find out where to look. For example, if you see a blocking session appearing in many other sessions, check what that session was doing:
- what queries was it running
- what objects was it accessing
- what wait events did it spend the most time on
- was it also blocked by yet another session

The examples in the drill down section above should help you closing in on the source of the problem. 

## Common waits and what to do about them
This is an attempt of a cookie-cutter guide for some of the more common wait events. 

<table><tr><th>Symptom</th><th>Reasons</th><th>Actions</th></tr>
<tr><td><ul><li>Many logfile sync events with high time_waited (> 1000000)</li></ul></td>
<td><ul><li>overload of the database, too many sessions</li>
<li>network issues in dataguard</li>
<li>SAN/storage problems</li>
<li>high CPU usage</li></ul></td>
<td><ul><li>Check average active sessions are lower than nr of cpu cores</li>
<li>Check alert.log for many logswitches or remote sync timeouts</li>
<li>Check /var/log/messages and cluster logs (if using grid) for any link down errors</li>
<li>Check if SAN and network load is below it's limits</li>
<li>Check host cpu usage with "top" or "sar"</li></ul>

<tr><td><ul><li>High amount of enq: xxx  or lock wait events</li></ul></td>
<td><ul><li>Contention of some sorts:</li>
<li>enq: TX - transaction related (locks on same object)</li>
<li>enq: SN/SQ - sequences related </li>
<li>enq: IV or L[A-P] - library cache related</li>
<li>row cache lock - DDL related</li></ul></td>
<td><ul><li>Check for blocking sessions and current_obj#</li>
<li>Increase the sequence cache</li>
<li>Check for too many parses or parallel sessions - or bugs</li>
<li>Check for DDL on current_obj#, can also occur when a segment is being extended or any other dictionary operation affecting an object</li></ul>
</td></tr>

<tr><td><ul><li>High amount of gc: xxx wait evens</li></ul></td>
<td><ul><li>RAC related, usually different instances accessing the same objects</li></ul></td>
<td><ul><li>Check for similar queries running on different "INST_ID"</li>
<li>Check for different "INST_ID" accessing the same current_obj#</li>
<li>Ensure that Jumbo Frames are enabled on the interconnect</li>
<li>Use instance bound services to separate the workload and table access</li></ul></td></tr>
</table>

## Make your life easier
Now that you hopefully have a better understanding of how to work with the active session history, it's time to let you in on a secret: Tanel Poder's script "ashtop.sql" does all the heavy lifting for us and in most cases it is no longer needed to write a custom script ourselfs. The basic script in this post can replaced with this ashtop.sql query:
```sql
@ashtop inst_id,session_id,sql_id,user,objt,event2,time_waited,blocking_session 1=1 sysdate-1/24 sysdate;
```
Or to just get an overview of the top queries:
```sql
@ashtop sql_id,event2 1=1 sysdate-1/24 sysdate;
```
Bear in mind though that it only shows the top 15 queries, which makes it even more important that you know what to query and be as specific as you can.

I am a big fan of ashtop.sql and use it on a daily basis. I explain it in detail with more drill down examples here: [/oracle/ashtop](/oracle/ashtop)

