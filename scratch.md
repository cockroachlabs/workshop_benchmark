## Demo Round 3 -- WORKLOAD binary
```
## Download workload binary if needed
wget https://edge-binaries.cockroachdb.com/cockroach/workload.LATEST
chmod 755 workload.LATEST
cp -i workload.LATEST /usr/local/bin/workload
chmod u+x /usr/local/bin/workload
```

Create simple schema to test logging function
```sql
create database mybench_noidx;
use mybench_noidx;

create table mylog (
    id uuid DEFAULT gen_random_uuid() primary key,
    ts timestamp DEFAULT now(),
    eventname string
);

create database mybench_idx;
use mybench_idx;

create table mylog (
    id uuid DEFAULT gen_random_uuid() primary key,
    ts timestamp DEFAULT now(),
    eventname string
);
create index idx_eventname_ts on mylog (eventname, ts);


create database mybench_hashidx;
use mybench_hashidx;

create table mylog (
    id uuid DEFAULT gen_random_uuid() primary key,
    ts timestamp DEFAULT now(),
    eventname string
);

set session experimental_enable_hash_sharded_indexes=on;
create index idxhash_eventname_ts on mylog (ts, eventname)  USING HASH WITH BUCKET_COUNT = 23;


-- Queries to run by each thread
insert into mylog (eventname) select 'startup' from generate_series(1,100);
insert into mylog (eventname) select 'crash' from generate_series(1,1);
insert into mylog (eventname) select 'not found' from generate_series(1,50);

-- Dash Board Query
select eventname, count(*) from mylog as of system time '-15s' where ts > now()::timestamp - INTERVAL '30s' group by 1 order by 1;

workload run querybench --query-file myqueries.sql --db mybench_noidx --duration 120s --concurrency 3 
workload run querybench --query-file myqueries.sql --db mybench_idx --duration 120s --concurrency 3 
workload run querybench --query-file myqueries.sql --db mybench_hashidx --duration 120s --concurrency 3 

```